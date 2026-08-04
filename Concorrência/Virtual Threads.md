#### 1. Teoria

**O problema que motivou Virtual Threads**

No modelo tradicional (que você viu no tópico anterior), cada `Thread` Java é mapeada **1:1** para uma thread do sistema operacional. Threads de SO são "caras": cada uma reserva alguns MB de stack, e o SO tem um limite prático de quantas consegue escalonar bem — na prática, algumas milhares por processo antes do sistema degradar. Isso é um problema real em backend: se cada requisição HTTP ocupa uma thread do pool do Tomcat até terminar (inclusive enquanto espera banco de dados, chamada HTTP externa, etc.), seu throughput fica limitado pelo tamanho do pool, não pela capacidade real da CPU — a maior parte do tempo a thread está _parada esperando I/O_, não computando.

**O que é uma Virtual Thread**

Virtual threads são uma implementação leve de threads fornecida pela JDK, e não pelo sistema operacional — uma forma de "user-mode threads", conceito que já existia em outras linguagens (goroutines em Go, processos em Erlang). A diferença de escalonamento: para platform threads (as tradicionais), a JDK depende do escalonador do próprio SO; para virtual threads, a JDK tem seu próprio escalonador, que atribui as virtual threads a platform threads (chamadas de "carrier threads"), que aí sim são escalonadas normalmente pelo SO. Isso é escalonamento **M:N**: milhões de virtual threads podem, ao longo do tempo, se revezar sobre um número pequeno de carrier threads (por padrão, aproximadamente o número de núcleos de CPU disponíveis). [Belief Driven Design](https://belief-driven-design.com/looking-at-java-21-virtual-threads-bd181/)[OpenJDK](https://openjdk.org/jeps/425)

O ponto central: quando uma virtual thread bloqueia numa operação de I/O (leitura de socket, query de banco, chamada HTTP), a JDK **desmonta** ela da carrier thread, liberando a carrier thread para rodar outra virtual thread enquanto a primeira espera. Quando o I/O termina, a virtual thread é remontada numa carrier thread (não necessariamente a mesma) e continua de onde parou. Tudo isso é transparente — seu código continua escrito no estilo síncrono/bloqueante de sempre (`repository.findById()`, sem `CompletableFuture`, sem callback), mas na prática o processo inteiro fica muito mais eficiente sob carga de I/O.

**Virtual Thread ainda é `Thread`**

Isso é importante: uma virtual thread ainda é um `java.lang.Thread`, exatamente como uma platform thread tradicional — mesma API (`start()`, `join()`, `interrupt()`, `Thread.currentThread()`). O que muda é a implementação por baixo, não a interface que você usa. [JRebel](https://www.jrebel.com/blog/what-are-virtual-threads)

**Linha do tempo (confirmado agora via busca, porque isso é justamente o tipo de detalhe que não dá pra chutar):**  
JEP 444, Virtual Threads, finaliza esse recurso com base no feedback das duas rodadas anteriores de preview: JEP 436 (Second Preview), entregue na JDK 20; e JEP 425 (Preview), entregue na JDK 19. Ou seja: **preview na 19 e 20, definitivo (produção) a partir da JDK 21** (LTS, setembro/2023). [infoq](https://www.infoq.com/news/2023/04/virtual-threads-arrives-jdk21/)

**Como criar (a forma moderna, sem precisar de `Runnable`/`Thread` manual toda hora)**

java

```java
Thread.ofVirtual().start(() -> System.out.println("rodando em virtual thread"));

// ou, o jeito mais comum em backend real:
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> processarRequisicao());
}
```

**Quando usar — e quando NÃO usar**

- **Bom encaixe:** tarefas I/O-bound com alto volume de concorrência (ex: um servidor atendendo dezenas de milhares de requisições que passam a maior parte do tempo esperando banco/API externa). É exatamente o cenário de um backend Spring Boot típico.
- **Não ajuda:** tarefas CPU-bound (cálculo pesado, processamento de imagem, etc.). Se a thread nunca bloqueia esperando I/O, não tem "vaga" pra outra virtual thread aproveitar — você ainda está limitado pelo número de núcleos de CPU disponíveis nas carrier threads. Trocar `ExecutorService` de pool fixo por virtual threads nesse caso não traz ganho e pode até confundir.
- **Anti-padrão:** **fazer pool de virtual threads.** Elas são baratas de criar (diferente de platform threads) — a prática recomendada é criar uma nova por tarefa e descartar, não reaproveitar num pool. `Executors.newVirtualThreadPerTaskExecutor()` já reflete isso: uma virtual thread nova por task submetida.

**Armadilha histórica que merece nota — o "pinning"**

Isso é importante você saber porque tutoriais mais antigos (baseados em JDK 21–23) vão insistir muito nisso como regra fixa, e a informação mudou: na JDK 21, uma virtual thread que entrava num bloco `synchronized` não conseguia se desmontar da carrier thread mesmo que depois bloqueasse em I/O dentro daquele bloco — isso é chamado de "pinning" (a virtual thread fica "pregada" na carrier thread, anulando o ganho de eficiência, e em casos extremos podendo até causar deadlock sob alta carga). A recomendação da época era trocar `synchronized` por `ReentrantLock` em código quente. [OpenJDK](https://openjdk.org/jeps/491)

Isso **mudou**: a partir da JDK 24, via JEP 491, virtual threads passaram a conseguir se desmontar mesmo dentro de métodos/blocos `synchronized`, encerrando o problema de pinning que antes forçava autores de bibliotecas a migrar para `ReentrantLock`. Ou seja: se seu projeto vai rodar em JDK 24+ (o que é razoável esperar hoje), essa armadilha específica deixou de existir na prática — mas é bom você saber que ela existiu, porque vai aparecer em muito conteúdo/pergunta de entrevista desatualizada, e o entrevistador pode não estar atualizado sobre a mudança. [OpenJDK](https://openjdk.org/jeps/444)

**Onde isso aparece no backend real**

Spring Boot 3.2+ tem suporte nativo pra rodar o processamento de requisições em virtual threads (`spring.threads.virtual.enabled=true`), trocando o pool de platform threads do Tomcat por uma virtual thread por requisição. O ganho prático: uma aplicação que antes precisava de tuning cuidadoso de tamanho de pool de threads (e sofria com "thread starvation" sob pico) passa a escalar de forma muito mais natural sob carga I/O-bound — que é o perfil da maioria dos serviços CRUD/API que consultam banco.

---

#### 2. Exemplo de código comentado

java

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.List;
import java.util.ArrayList;

public class VirtualThreadExemplo {

    public static void main(String[] args) throws Exception {
        // newVirtualThreadPerTaskExecutor(): cria UMA virtual thread nova
        // para cada tarefa submetida. Não é um "pool" no sentido tradicional —
        // não há reaproveitamento de thread, e isso é intencional (são baratas).
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {

            List<Future<String>> resultados = new ArrayList<>();

            // Simula 10 mil "requisições" concorrentes, cada uma esperando
            // um pouco (simulando I/O, tipo uma chamada de banco/API).
            // Com platform threads, 10 mil threads de SO simultâneas seria
            // impraticável na maioria das máquinas.
            for (int i = 0; i < 10_000; i++) {
                int id = i;
                Future<String> resultado = executor.submit(() -> {
                    Thread.sleep(100); // aqui a virtual thread se DESMONTA da
                                        // carrier thread; a carrier fica livre
                                        // pra rodar outra virtual thread nesse meio-tempo
                    return "Tarefa " + id + " concluída em "
                            + Thread.currentThread(); // note: imprime algo como
                                                       // VirtualThread[#123]/runnable@ForkJoinPool-...
                });
                resultados.add(resultado);
            }

            // Coleta os resultados (bloqueia até cada Future terminar)
            long concluidas = resultados.stream()
                    .map(f -> {
                        try { return f.get(); } catch (Exception e) { return null; }
                    })
                    .filter(r -> r != null)
                    .count();

            System.out.println("Total concluído: " + concluidas);
        } // try-with-resources: o executor é fechado automaticamente aqui,
          // aguardando as tarefas em andamento antes de encerrar
    }
}
```

Compare mentalmente com o exercício 2 da sessão anterior (1000 threads tradicionais) — lá, `new Thread()` mil vezes já pesa. Aqui, 10 mil virtual threads rodam tranquilamente porque cada uma não consome uma thread de SO dedicada o tempo todo — só enquanto está ativamente executando, não enquanto está bloqueada em `sleep`/I/O.

---

#### 3. Armadilhas comuns

1. **Usar virtual threads para tarefa CPU-bound esperando ganho de performance.** Se a tarefa não bloqueia em I/O, virtual threads não trazem vantagem sobre um pool de platform threads bem dimensionado — você continua limitado pelo número de núcleos.
2. **Fazer pool de virtual threads** (ex: guardar um `ExecutorService` de virtual threads como se fosse um pool fixo reutilizável tipo `newFixedThreadPool`). O modelo mental correto é "uma virtual thread descartável por tarefa", não "reaproveitar threads caras".
3. **Confiar cegamente em conteúdo desatualizado sobre `synchronized` + pinning.** Como você viu acima, isso era um problema real até JDK 23 e foi resolvido na JDK 24 (JEP 491) — mas ainda é o tipo de "pegadinha" que aparece em entrevista formulada com base em material antigo. Vale saber explicar os dois lados: "era um problema, foi corrigido a partir de tal versão".
4. **Uso pesado de `ThreadLocal` em código pensado pra virtual threads.** `ThreadLocal` funciona com virtual threads, mas como você pode ter _milhões_ delas ativas, `ThreadLocal`s "pesados" (guardando objetos grandes) por thread podem consumir memória de forma desproporcional. Isso não é uma proibição, é um cuidado de dimensionamento.

---

#### 4. Exercícios práticos

**1. Fácil**  
Reescreva o Exercício 1 da sessão de Threads (as duas threads "A" e "B" imprimindo 1 a 5 com pausa de 100ms) usando `Thread.ofVirtual()` em vez de `new Thread(...)`. Confirme, imprimindo `Thread.currentThread()`, que o nome/tipo da thread mudou (deve aparecer algo como `VirtualThread[...]` em vez de `Thread[A,...]`).

**2. Médio**  
Escreva um programa que compara tempo total de execução entre: (a) criar 50.000 **platform threads**, cada uma só fazendo `Thread.sleep(50)` e terminando; (b) fazer a mesma coisa com 50.000 **virtual threads** via `Executors.newVirtualThreadPerTaskExecutor()`. Meça o tempo total com `System.nanoTime()` antes/depois, e dê `join()`/espere todas terminarem nos dois casos. Critério de pronto: rodar os dois e comparar o tempo total (é esperado — e o objetivo do exercício é você observar isso na prática, não só ler que é assim — que a versão (a) demore muito mais ou até falhe/trave dependendo da sua máquina, enquanto (b) termine rápido).

**3. Difícil**  
Crie um método que simula uma "chamada de API externa" fazendo `Thread.sleep(200)` (representando espera de rede) e depois retorna uma `String`. Usando `Executors.newVirtualThreadPerTaskExecutor()`, dispare 1000 chamadas concorrentes desse método e colete todos os resultados usando `Future`. Depois, imprima quanto tempo o processamento total levou. Critério de pronto: o tempo total deve ficar próximo de ~200ms (ou pouco mais), e não de `1000 × 200ms` — isso prova, na prática, que as 1000 "chamadas" rodaram concorrentemente, não em sequência.

**4. Desafio**  
Escreva um pequeno "servidor" simulado: um método `atenderRequisicao(int id)` que faz `Thread.sleep(100)` (I/O) e, **dentro** de um bloco `synchronized` num objeto de lock compartilhado, faz mais um `Thread.sleep(50)` (simulando, por exemplo, escrita num recurso compartilhado protegido). Dispare 500 dessas "requisições" concorrentes via `newVirtualThreadPerTaskExecutor()` e meça o tempo total. Depois, pesquise (usando busca na web, se disponível) qual é a versão mínima de JDK que você precisaria rodar para que esse `synchronized` não cause pinning, e explique no comentário do código o que aconteceria de diferente rodando esse mesmo código numa JDK 21 pura vs. numa JDK 24+.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Ex1VT {
    public static void main(String[] args) throws InterruptedException {
        Runnable tarefa = () -> {
            for (int i = 1; i <= 5; i++) {
                System.out.println(Thread.currentThread() + ": " + i);
                try { Thread.sleep(100); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        };

        Thread a = Thread.ofVirtual().unstarted(tarefa);
        Thread b = Thread.ofVirtual().unstarted(tarefa);

        a.start();
        b.start();
        a.join();
        b.join();
        System.out.println("Fim");
    }
}
```

Raciocínio: `Thread.ofVirtual()` retorna um _builder_ (`Thread.Builder.OfVirtual`). `.unstarted(tarefa)` cria a thread sem iniciar (equivalente a `new Thread(tarefa)` na API antiga); dá pra usar `.start(tarefa)` direto se não precisar guardar a referência antes de iniciar. A API de `join()`/`interrupt()` é idêntica à de platform threads — essa continuidade de API é justamente um dos objetivos de design declarados do JEP 444.

**Exercício 2**

java

```java
import java.util.concurrent.*;

public class Ex2VT {
    public static void main(String[] args) throws InterruptedException {
        int total = 50_000;

        long inicioPlatform = System.nanoTime();
        Thread[] platformThreads = new Thread[total];
        for (int i = 0; i < total; i++) {
            platformThreads[i] = new Thread(() -> {
                try { Thread.sleep(50); } catch (InterruptedException e) {}
            });
            platformThreads[i].start();
        }
        for (Thread t : platformThreads) t.join();
        long fimPlatform = System.nanoTime();
        System.out.println("Platform threads: " +
                (fimPlatform - inicioPlatform) / 1_000_000 + "ms");

        long inicioVirtual = System.nanoTime();
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < total; i++) {
                executor.submit(() -> {
                    try { Thread.sleep(50); } catch (InterruptedException e) {}
                });
            }
        } // fechamento do try-with-resources espera todas as tarefas terminarem
        long fimVirtual = System.nanoTime();
        System.out.println("Virtual threads: " +
                (fimVirtual - inicioVirtual) / 1_000_000 + "ms");
    }
}
```

Raciocínio: no bloco de platform threads, 50 mil threads de SO simultâneas sobrecarregam o escalonador do SO (memória de stack reservada por thread, custo de context-switch) — dependendo da sua máquina isso pode até lançar `OutOfMemoryError: unable to create new native thread`. No bloco de virtual threads, como cada `Thread.sleep()` desmonta a virtual thread da carrier thread, um número pequeno de carrier threads (perto do número de núcleos) dá conta de "hospedar" as 50 mil virtual threads ao longo do tempo, então o tempo total fica perto de apenas um pouco mais que 50ms (o tempo de um único `sleep`), não crescendo proporcionalmente à quantidade de tarefas.

**Exercício 3**

java

```java
import java.util.concurrent.*;
import java.util.*;

public class Ex3VT {
    public static void main(String[] args) throws Exception {
        long inicio = System.nanoTime();

        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<String>> futures = new ArrayList<>();
            for (int i = 0; i < 1000; i++) {
                int id = i;
                futures.add(executor.submit(() -> chamarApiExterna(id)));
            }
            for (Future<String> f : futures) {
                f.get(); // espera cada resultado
            }
        }

        long fim = System.nanoTime();
        System.out.println("Tempo total: " + (fim - inicio) / 1_000_000 + "ms");
    }

    static String chamarApiExterna(int id) throws InterruptedException {
        Thread.sleep(200); // simula espera de rede
        return "Resposta " + id;
    }
}
```

Raciocínio: o tempo total ficar perto de ~200ms (e não 200.000ms, que seria `1000 × 200ms` sequencial) é a prova empírica de que as 1000 chamadas rodaram concorrentemente. Isso simula exatamente o cenário real que motiva virtual threads: um serviço que faz muitas chamadas de I/O (bancos, APIs) simultâneas sem precisar de código assíncrono explícito (`CompletableFuture`, callbacks) — o código continua no estilo bloqueante/síncrono de sempre.

**Exercício 4**

java

```java
import java.util.concurrent.*;

public class Ex4VT {
    private static final Object lock = new Object();

    public static void main(String[] args) throws Exception {
        long inicio = System.nanoTime();

        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 500; i++) {
                int id = i;
                executor.submit(() -> atenderRequisicao(id));
            }
        }

        long fim = System.nanoTime();
        System.out.println("Tempo total: " + (fim - inicio) / 1_000_000 + "ms");
    }

    static void atenderRequisicao(int id) throws InterruptedException {
        Thread.sleep(100); // I/O fora do lock — desmonta normalmente

        synchronized (lock) {
            Thread.sleep(50); // I/O DENTRO do synchronized
        }
    }
}
```

Raciocínio (comentário pedido no exercício): rodando essa mesma classe numa **JDK 21, 22 ou 23**, o `Thread.sleep(50)` dentro do bloco `synchronized` causaria pinning — a virtual thread ficaria presa à carrier thread durante esses 50ms, incapaz de liberar a carrier pra outra virtual thread rodar nesse meio-tempo. Com 500 requisições competindo por um número pequeno de carrier threads (perto do número de núcleos da sua máquina), isso serializaria boa parte do trabalho que deveria ser concorrente, e o tempo total cresceria de forma proporcional ao número de requisições em vez de ficar perto do tempo de uma única execução. A partir da **JDK 24** (JEP 491), esse mesmo código não causa mais pinning: a virtual thread consegue se desmontar da carrier mesmo dentro do bloco `synchronized`, então o comportamento fica igual ao do `sleep` de fora do lock — tempo total próximo de pouco mais de 150ms (100 + 50), independente de rodar em 500 ou 5000 requisições, contanto que a máquina tenha carrier threads suficientes pra não virar gargalo de CPU pura em algum outro ponto.