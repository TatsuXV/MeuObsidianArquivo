#### 1. Teoria

**O que é uma thread**

Um _processo_ (seu programa Java rodando) tem memória isolada do resto do sistema operacional. Uma **thread** é uma linha de execução _dentro_ desse processo. Um processo pode ter várias threads, e todas elas compartilham a mesma memória heap (objetos, atributos estáticos) — mas cada thread tem sua própria _stack_ (variáveis locais, chamadas de método).

Isso é a diferença chave: threads do mesmo processo **compartilham estado**. É exatamente isso que cria os problemas que você vai treinar hoje (race condition) e é por isso que concorrência é considerada difícil — não é a criação da thread que é complicada, é coordenar acesso a memória compartilhada sem corromper dado.

**Duas formas de criar uma thread em Java**

1. Estender `Thread` e sobrescrever `run()`.
2. Implementar `Runnable` e passar pro construtor de `Thread`.

Na prática, **prefira `Runnable`**. Java não tem herança múltipla — se sua classe já estende `Thread`, ela não pode estender mais nada. Além disso, `Runnable` separa a _tarefa_ (o que precisa ser feito) do _mecanismo de execução_ (a thread em si), o que é mais alinhado com o resto do ecossistema Java moderno (inclusive `ExecutorService`, que você vai ver mais pra frente — não vou entrar em detalhe agora porque é tópico futuro).

**start() vs run() — a armadilha nº1**

- `thread.start()` → cria uma nova thread de sistema operacional e o `run()` roda _nela_, concorrentemente com quem chamou.
- `thread.run()` → executa o método `run()` como uma chamada de método comum, na thread atual. **Nenhuma concorrência acontece.**

Isso é confundido o tempo todo por quem está começando.

**Ciclo de vida de uma thread**

`Thread.State` tem 6 valores:

- `NEW` — criada, `start()` ainda não chamado
- `RUNNABLE` — rodando ou pronta pra rodar (o SO decide o escalonamento)
- `BLOCKED` — esperando entrar numa seção `synchronized` que outra thread ocupa
- `WAITING` — esperando indefinidamente (ex: `Object.wait()` sem timeout, `Thread.join()` sem timeout)
- `TIMED_WAITING` — como `WAITING`, mas com prazo (`Thread.sleep(ms)`, `wait(ms)`)
- `TERMINATED` — `run()` terminou

**synchronized — exclusão mútua**

Toda instância de objeto em Java tem um "monitor" (lock) implícito. `synchronized` (em método ou bloco) garante que só uma thread por vez execute aquele trecho _para o mesmo objeto de lock_. Isso resolve race condition, mas tem custo de performance e pode causar deadlock se mal usado.

**Onde isso aparece no backend real**

Em Spring Boot você raramente cria `Thread` manualmente — o Tomcat/servlet container já usa um pool de threads pra atender requisições HTTP concorrentemente. Mas entender threads importa porque:

- Beans `@Component`/`@Service` são **singletons por padrão** — se você guardar estado mutável num atributo de instância desses beans, múltiplas requisições concorrentes vão pisar uma na outra (bug clássico de produção).
- `@Async` do Spring roda em outra thread — se você não entender join/exceção em thread, vai debugar isso no escuro.
- Cache em memória compartilhado entre requisições precisa de sincronização.

---

#### 2. Exemplo de código comentado

java

```java
public class ThreadBasicoExemplo {

    public static void main(String[] args) throws InterruptedException {
        // Runnable = a tarefa, sem amarrar à mecânica de Thread
        Runnable tarefa = () -> {
            for (int i = 1; i <= 3; i++) {
                System.out.println(Thread.currentThread().getName() + " -> " + i);
                try {
                    Thread.sleep(200); // pausa a thread atual por 200ms, SEM travar as outras
                } catch (InterruptedException e) {
                    // Boa prática: se você captura InterruptedException e não vai
                    // propagar, restaure o "interrupt flag" da thread.
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        };

        Thread t1 = new Thread(tarefa, "worker-1");
        Thread t2 = new Thread(tarefa, "worker-2");

        t1.start(); // cria a thread de SO e chama run() nela
        t2.start(); // roda "ao mesmo tempo" que t1

        // join() bloqueia a main até a thread terminar.
        // Sem isso, o programa pode terminar antes das threads acabarem.
        t1.join();
        t2.join();

        System.out.println("Ambas terminaram.");
    }
}
```

Saída não é determinística — a ordem de `worker-1` e `worker-2` pode intercalar de formas diferentes a cada execução. Isso é esperado: o escalonador do SO decide.

**Exemplo de race condition + correção com synchronized:**

java

```java
class Contador {
    private int valor = 0;

    // SEM synchronized aqui, valor++ (que é ler + somar + escrever)
    // pode ser interrompido no meio por outra thread, perdendo incrementos.
    public synchronized void incrementar() {
        valor++;
    }

    public synchronized int getValor() {
        return valor;
    }
}
```

---

#### 3. Armadilhas comuns

1. **Chamar `run()` em vez de `start()`** — parece funcionar (o código roda), mas não há concorrência nenhuma; é só uma chamada de método sequencial.
2. **Não sincronizar acesso a estado compartilhado** — `valor++` parece uma operação atômica mas não é (é ler, somar, escrever — três passos). Com duas threads incrementando sem `synchronized`, você perde incrementos silenciosamente. Não dá exceção, só dá resultado errado — o tipo de bug mais difícil de rastrear.
3. **Engolir `InterruptedException` sem restaurar o flag** — fazer só `catch (InterruptedException e) {}` esconde o pedido de interrupção e pode deixar a thread num estado que ninguém mais consegue parar de forma limpa.
4. **Usar `Thread.stop()` ou `Thread.suspend()`** — são métodos deprecated e perigosos (podem deixar objetos em estado inconsistente, liberando locks no meio de uma operação). Nunca use.

---

#### 4. Exercícios práticos

**1. Fácil**  
Crie duas threads. Cada uma deve imprimir seu próprio nome e os números de 1 a 5, com uma pausa de 100ms entre cada número. Use `join()` para garantir que a `main` só imprima "Fim" depois que as duas threads terminarem.

**2. Médio**  
Crie uma classe `ContadorInseguro` com um método `incrementar()` que faz `valor++`, **sem** `synchronized`. No `main`, crie 1000 threads (ou 100, se sua máquina travar) que cada uma chama `incrementar()` uma vez, dê `join()` em todas, e imprima o valor final. Rode algumas vezes e observe que o valor final quase nunca é 1000. Depois, corrija criando uma versão `synchronized` e mostre que agora sempre dá o valor certo.

**3. Difícil**  
Implemente um produtor-consumidor simples: uma thread "produtora" adiciona números (0, 1, 2, ...) numa fila (uma `LinkedList` comum, não uma fila thread-safe pronta), uma thread "consumidora" remove e imprime. Use `wait()`/`notify()` no próprio objeto de lock para a consumidora esperar quando a fila estiver vazia, e a produtora avisar quando adicionar algo. Critério de pronto: rodar sem `NoSuchElementException` e sem a consumidora fazer polling em loop apertado (busy-wait).

**4. Desafio**  
Implemente, sem usar `ExecutorService` nem `Semaphore` prontos do `java.util.concurrent`, um limitador manual que garante que no máximo **3 threads** de um total de 10 executem sua tarefa simultaneamente (as outras devem esperar até uma "vaga" abrir). Dica: você vai precisar de um contador compartilhado protegido por `synchronized`/`wait`/`notify`. Critério de pronto: logar o momento em que cada thread começa e termina, e confirmar visualmente no log que nunca há mais de 3 rodando ao mesmo tempo.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Ex1 {
    public static void main(String[] args) throws InterruptedException {
        Runnable tarefa = () -> {
            for (int i = 1; i <= 5; i++) {
                System.out.println(Thread.currentThread().getName() + ": " + i);
                try { Thread.sleep(100); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        };
        Thread a = new Thread(tarefa, "A");
        Thread b = new Thread(tarefa, "B");
        a.start();
        b.start();
        a.join(); // main espera A terminar
        b.join(); // main espera B terminar
        System.out.println("Fim");
    }
}
```

Raciocínio: sem os dois `join()`, "Fim" poderia imprimir antes das threads acabarem, porque `start()` não bloqueia — ele só dispara a thread e devolve o controle imediatamente pra quem chamou.

**Exercício 2**

java

```java
class ContadorInseguro {
    private int valor = 0;
    public void incrementar() { valor++; }
    public int getValor() { return valor; }
}

class ContadorSeguro {
    private int valor = 0;
    public synchronized void incrementar() { valor++; }
    public synchronized int getValor() { return valor; }
}

public class Ex2 {
    public static void main(String[] args) throws InterruptedException {
        ContadorInseguro c = new ContadorInseguro();
        Thread[] threads = new Thread[1000];
        for (int i = 0; i < 1000; i++) {
            threads[i] = new Thread(c::incrementar);
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println("Inseguro: " + c.getValor()); // provavelmente < 1000
    }
}
```

Raciocínio: `valor++` compila para "ler valor, somar 1, escrever valor". Se a thread A lê `valor=42`, é pausada pelo escalonador, a thread B também lê `valor=42`, soma e escreve `43`, depois A retoma, soma seu `42+1` e escreve `43` de novo — um incremento foi perdido. `synchronized` serializa o acesso: só uma thread por vez entra no método, então não existe mais essa janela de sobreposição.

_Alternativa com trade-off:_ em vez de `synchronized`, dá pra usar `java.util.concurrent.atomic.AtomicInteger` (`incrementAndGet()`), que usa instruções atômicas de CPU (CAS — compare-and-swap) em vez de lock. Pra um contador simples, `AtomicInteger` costuma ser mais rápido sob alta concorrência porque evita o custo de aquisição de lock. Mas isso é assunto de uma sessão futura (Concorrência mais avançada) — aqui o objetivo era entender `synchronized`.

**Exercício 3**

java

```java
import java.util.LinkedList;

public class Ex3 {
    public static void main(String[] args) {
        LinkedList<Integer> fila = new LinkedList<>();
        Object lock = fila; // usamos a própria fila como monitor

        Thread produtor = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                synchronized (lock) {
                    fila.add(i);
                    System.out.println("Produziu: " + i);
                    lock.notify(); // acorda o consumidor se ele estiver esperando
                }
                try { Thread.sleep(50); } catch (InterruptedException e) {}
            }
        });

        Thread consumidor = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                synchronized (lock) {
                    while (fila.isEmpty()) {
                        try {
                            lock.wait(); // libera o lock e dorme até notify()
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                    }
                    int valor = fila.removeFirst();
                    System.out.println("Consumiu: " + valor);
                }
            }
        });

        produtor.start();
        consumidor.start();
    }
}
```

Raciocínio: `wait()` só pode ser chamado dentro de um bloco `synchronized` no mesmo objeto — ele libera o lock enquanto espera (senão o produtor nunca conseguiria entrar pra adicionar algo) e o readquire automaticamente quando é acordado por `notify()`. O `while (fila.isEmpty())` em vez de `if` é importante: entre o `notify()` e a consumidora acordar de fato, outra coisa pode ter mudado o estado (isso evita o problema clássico de "spurious wakeup" e de múltiplos consumidores brigando pelo mesmo item).

**Exercício 4**

java

```java
public class Ex4 {
    private static final int LIMITE = 3;
    private static int emExecucao = 0;
    private static final Object lock = new Object();

    public static void main(String[] args) {
        for (int i = 1; i <= 10; i++) {
            int id = i;
            new Thread(() -> tarefa(id)).start();
        }
    }

    private static void tarefa(int id) {
        synchronized (lock) {
            while (emExecucao >= LIMITE) {
                try { lock.wait(); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
            emExecucao++;
            System.out.println("Thread " + id + " começou. Em execução: " + emExecucao);
        }

        try { Thread.sleep(500); } catch (InterruptedException e) {} // simula trabalho

        synchronized (lock) {
            emExecucao--;
            System.out.println("Thread " + id + " terminou. Em execução: " + emExecucao);
            lock.notifyAll(); // acorda TODAS as threads esperando, pra reavaliar a vaga
        }
    }
}
```

Raciocínio: uso `notifyAll()` em vez de `notify()` aqui porque várias threads podem estar esperando vaga ao mesmo tempo — `notify()` acordaria só uma, arbitrariamente, e as outras ficariam esperando sem necessidade até a próxima liberação. Note que o "trabalho simulado" (`Thread.sleep(500)`) fica **fora** do bloco `synchronized` de propósito: se ficasse dentro, nenhuma outra thread conseguiria nem checar a condição de vaga enquanto essa estivesse "trabalhando", travando o paralelismo que o exercício pede pra demonstrar.