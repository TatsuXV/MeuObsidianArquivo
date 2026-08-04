#### 1. Teoria

**O que `volatile` garante — as duas coisas, com precisão**

1. **Visibilidade**: toda leitura de uma variável `volatile` enxerga a escrita mais recente feita por qualquer thread (regra "escrita em volatile happens-before leitura subsequente da mesma variável", que você já viu no tópico de JMM).
2. **Ordenação (impede reordenação)**: o compilador/CPU não pode reordenar instruções ao redor de um acesso `volatile` de forma que quebre a semântica esperada. Tecnicamente isso é implementado via barreiras de memória (`LoadLoad`, `StoreStore`, `LoadStore`, `StoreLoad`) inseridas pela JVM ao redor do acesso `volatile` — você não precisa decorar os nomes das barreiras, só saber que existe um custo real de performance nisso (por isso `volatile` não é "grátis", e não deve ser usado em todo campo por precaução).

**O que `volatile` NÃO garante — o ponto que mais cai em entrevista**

- **Não garante atomicidade de operações compostas.** Você já viu isso duas vezes (Threads e JMM): `volatile int x; x++;` continua tendo race condition, porque `++` é ler+somar+escrever, três passos separados, e `volatile` só garante que cada passo individual (a leitura, a escrita) é visível — não que os três juntos aconteçam sem interferência de outra thread no meio.
- **Não garante visibilidade dos _campos internos_ de um objeto referenciado por uma variável `volatile`.** Essa é a pegadinha menos óbvia: se você tem `private volatile MinhaClasse obj;`, o `volatile` garante que a _referência_ `obj` (qual objeto ela aponta) é vista corretamente por outras threads. Mas se depois de publicar `obj`, outra thread mudar um campo _não-volatile_ dentro daquele objeto, essa mudança específica não tem garantia de visibilidade só por causa do `volatile` na referência.
- **Não substitui `synchronized` quando você precisa de exclusão mútua** (impedir que duas threads executem um trecho ao mesmo tempo). `volatile` não tem conceito de "lock" — não bloqueia thread nenhuma, só controla visibilidade/ordenação.

**Um detalhe correto e específico sobre `long`/`double` que costuma surpreender**

Para os fins do modelo de memória do Java, uma única escrita a um valor `long` ou `double` não-volatile é tratada como duas escritas separadas: uma pra cada metade de 32 bits. Isso significa que, em teoria (a especificação permite isso, mesmo que a maioria das JVMs modernas em 64 bits não tenha esse problema na prática), uma thread poderia ler um `long` "misturado" — 32 bits de uma escrita antiga e 32 bits de uma escrita mais nova, um valor que nunca foi escrito por inteiro por ninguém. Isso é chamado de "word tearing". Escritas e leituras de `long`/`double` declarados `volatile`, por outro lado, são sempre atômicas. Na prática: se você tem um `long`/`double` compartilhado entre threads sem sincronização nenhuma, declarar `volatile` já resolve esse problema específico de tearing (mesmo sem resolver o problema de operações compostas como `+=` que já vimos). [cmu](https://wiki.sei.cmu.edu/confluence/x/tFZMBQ)[cmu](https://wiki.sei.cmu.edu/confluence/x/tFZMBQ)

**Quando usar `volatile` de fato**

- **Flags de controle** (o clássico `pare`/`shutdown` que você viu no exemplo de Threads) — um valor escrito por uma thread, lido por outra(s), sem operação composta em cima.
- **Publicação segura de referência de objeto imutável** — quando o próprio objeto referenciado é imutável (todos os campos `final`), `volatile` na referência é suficiente e mais barato que `synchronized`.
- **Double-checked locking** (você já viu isso no JMM) — embora o padrão _holder_ que você implementou no exercício 4 anterior seja geralmente preferido justamente por dispensar `volatile`/`synchronized` no caminho comum.

**Quando NÃO usar — prefira `java.util.concurrent.atomic`**

Para contadores e operações do tipo "ler-modificar-escrever" que precisam ser atômicas, a ferramenta certa é `AtomicInteger`, `AtomicLong`, `AtomicReference`, etc. (do pacote `java.util.concurrent.atomic`, introduzido junto com o JSR-133 no Java 5) — eles usam instruções de CPU de compare-and-swap (CAS) para fazer a operação composta inteira de forma atômica, sem precisar de `synchronized`. Isso é mais avançado e será aprofundado em um tópico futuro (Concorrência mais avançada), mas vale já saber que essa é a alternativa correta quando `volatile` sozinho não é suficiente.

**Onde isso aparece no backend real**

Flags de "aplicação pronta para receber tráfego" em health checks customizados, campos de configuração recarregada em runtime (ex: um feature flag lido de um arquivo/serviço externo, atualizado periodicamente por uma thread de background e lido por todas as threads de requisição), e a implementação interna de vários mecanismos do próprio Spring (ex: beans lazy) usam `volatile` exatamente por esses motivos.

---

#### 2. Exemplo de código comentado

java

```java
public class VolatileExemplo {

    // Caso 1: flag simples — uso ideal de volatile
    private static volatile boolean aplicacaoPronta = false;

    // Caso 2: referência a objeto imutável — publicação segura
    private static volatile Configuracao configAtual =
            new Configuracao("v1", 100);

    // Caso 3: long compartilhado — volatile evita "word tearing"
    private static volatile long ultimoTimestampProcessado = 0L;

    static void recarregarConfiguracao(Configuracao nova) {
        // A troca da REFERÊNCIA inteira é atômica e visível
        // imediatamente para quem ler configAtual depois disso.
        configAtual = nova;
    }

    static Configuracao lerConfiguracaoAtual() {
        return configAtual; // sempre vê a versão mais recente publicada
    }

    // Classe imutável: todos os campos final, sem setters.
    // É isso que torna seguro publicar via volatile sem synchronized extra.
    static final class Configuracao {
        final String versao;
        final int limiteRequisicoes;

        Configuracao(String versao, int limiteRequisicoes) {
            this.versao = versao;
            this.limiteRequisicoes = limiteRequisicoes;
        }
    }
}
```

**Exemplo do erro sutil — `volatile` na referência não protege os campos internos mutáveis:**

java

```java
class Contador {
    int valor; // NÃO é volatile — campo mutável dentro do objeto
}

public class ArmadilhaReferencia {
    private static volatile Contador contador = new Contador();

    static void incrementar() {
        contador.valor++; // essa escrita NÃO tem garantia de visibilidade,
                           // mesmo que "contador" (a referência) seja volatile!
                           // volatile protegeu a TROCA de objeto, não os
                           // campos mutáveis DENTRO do objeto atual.
    }
}
```

---

#### 3. Armadilhas comuns

1. **Usar `volatile` esperando atomicidade em operação composta** (`x++`, `x += 1`, `if (x == null) x = novo`) — já reforçado múltiplas vezes porque é o erro nº1 do tópico.
2. **Achar que `volatile Objeto obj` protege os campos internos de `obj`.** Só protege a referência em si (qual objeto `obj` aponta), não mudanças feitas nos campos mutáveis daquele objeto depois.
3. **Usar `volatile` em vez de `synchronized` quando o que você precisa é exclusão mútua**, não só visibilidade — sintoma comum: código "parece" resolver a race condition nos testes locais (poucas threads, timing favorável) mas falha sob carga real.
4. **Espalhar `volatile` "por precaução" em todo campo compartilhado.** Cada acesso `volatile` tem custo real (impede certas otimizações do JIT, insere barreiras de memória). Use onde o padrão de acesso realmente pede (flag simples, publicação de imutável) — para contadores e acumuladores, prefira `AtomicInteger`/`AtomicLong`; para lógica mais complexa envolvendo múltiplos campos relacionados, prefira `synchronized`.

---

#### 4. Exercícios práticos

**1. Fácil**  
Escreva uma classe `StatusServico` com um campo `private volatile boolean disponivel = false;`, um método `marcarDisponivel()` e um método `estaDisponivel()`. Crie uma thread que fica em loop checando `estaDisponivel()` e, assim que for `true`, imprime "Serviço disponível!" e encerra o loop. Na `main`, espere 500ms e chame `marcarDisponivel()`. Confirme que a thread reage à mudança.

**2. Médio**  
Demonstre a armadilha da seção 2 (`volatile` na referência não protegendo campos internos mutáveis) na prática: crie a classe `Contador` do exemplo (campo `int valor` não-volatile) e uma referência estática `volatile Contador contador`. Dispare 1000 threads que cada uma faz `contador.valor++`. Depois, crie uma segunda versão onde `Contador` usa `AtomicInteger` internamente (`AtomicInteger valor = new AtomicInteger(0)`, incrementando com `valor.incrementAndGet()`). Compare o resultado final das duas versões e explique a diferença observada.

**3. Difícil**  
Escreva uma classe `RelogioCompartilhado` com um campo `private long tempoAtual` (**sem** `volatile`, de propósito) atualizado repetidamente por uma thread escritora (em loop rápido, escrevendo `System.nanoTime()`) e lido repetidamente por uma thread leitora, que imprime o valor lido a cada leitura. Rode por alguns segundos. Depois, adicione `volatile` ao campo e rode de novo. Documente no comentário: por que, na prática, é difícil observar diretamente o "word tearing" mencionado na teoria mesmo sem `volatile` (dica: relacione com o fato de que a maioria das JVMs modernas em hardware de 64 bits já implementa escrita de `long` como uma operação atômica de CPU, mesmo quando a especificação não exige isso) — e por que, mesmo assim, declarar `volatile` continua sendo a prática recomendada, já que o comportamento sem ele não é _garantido_ por especificação em nenhuma plataforma.

**4. Desafio**  
Implemente um `CacheSimples<K, V>` que guarda um único par imutável de "última chave consultada / último valor retornado" como otimização (uma espécie de cache de tamanho 1), usando uma classe interna imutável `Entrada` (campos `final`) e uma referência `private volatile Entrada ultimaEntrada`. O método `Optional<V> buscarUltima(K chave)` deve retornar o valor cacheado se a chave bater com a última consultada, ou `Optional.empty()` caso contrário — sem usar `synchronized` em lugar nenhum. O método `atualizar(K chave, V valor)` deve trocar `ultimaEntrada` por uma nova instância de `Entrada`. Escreva um teste com múltiplas threads chamando `atualizar()` e `buscarUltima()` concorrentemente, e explique no comentário por que essa implementação é segura entre threads (thread-safe) mesmo sem nenhum bloco `synchronized`.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
class StatusServico {
    private volatile boolean disponivel = false;
    void marcarDisponivel() { disponivel = true; }
    boolean estaDisponivel() { return disponivel; }
}

public class Ex1Volatile {
    public static void main(String[] args) throws InterruptedException {
        StatusServico status = new StatusServico();

        Thread verificadora = new Thread(() -> {
            while (!status.estaDisponivel()) {
                // aguardando
            }
            System.out.println("Serviço disponível!");
        });

        verificadora.start();
        Thread.sleep(500);
        status.marcarDisponivel();
        verificadora.join();
    }
}
```

Raciocínio: sem `volatile`, o JIT poderia otimizar o loop `while (!status.estaDisponivel())` assumindo que o valor nunca muda dentro daquela thread (já que nada dentro do loop escreve nele), transformando na prática num loop infinito que nunca reavalia a condição de verdade contra a memória principal. `volatile` impede exatamente essa otimização.

**Exercício 2**

java

```java
class Contador {
    int valor;
}

class ContadorAtomico {
    private final AtomicInteger valor = new AtomicInteger(0);
    void incrementar() { valor.incrementAndGet(); }
    int getValor() { return valor.get(); }
}

public class Ex2Volatile {
    public static void main(String[] args) throws InterruptedException {
        // Versão 1: volatile na referência, campo interno não-volatile
        Contador contador = new Contador();
        Thread[] t1 = new Thread[1000];
        for (int i = 0; i < 1000; i++) {
            t1[i] = new Thread(() -> contador.valor++);
            t1[i].start();
        }
        for (Thread t : t1) t.join();
        System.out.println("Com campo simples: " + contador.valor);

        // Versão 2: AtomicInteger
        ContadorAtomico atomico = new ContadorAtomico();
        Thread[] t2 = new Thread[1000];
        for (int i = 0; i < 1000; i++) {
            t2[i] = new Thread(atomico::incrementar);
            t2[i].start();
        }
        for (Thread t : t2) t.join();
        System.out.println("Com AtomicInteger: " + atomico.getValor());
    }
}
```

Raciocínio: a primeira versão tipicamente termina com um valor menor que 1000 (mesma race condition de sempre — `valor++` não é atômico, e aqui nem tem `volatile` protegendo o campo interno). A segunda versão sempre termina exatamente em 1000, porque `incrementAndGet()` executa a operação composta inteira (ler, somar, escrever) como uma única operação atômica via CAS, sem depender de lock nem de `volatile`.

**Exercício 3**

java

```java
public class Ex3Volatile {
    private static long tempoAtual; // depois testar com volatile

    public static void main(String[] args) throws InterruptedException {
        Thread escritora = new Thread(() -> {
            for (int i = 0; i < 1_000_000; i++) {
                tempoAtual = System.nanoTime();
            }
        });
        Thread leitora = new Thread(() -> {
            for (int i = 0; i < 1_000_000; i++) {
                long v = tempoAtual;
                // em teoria, sem volatile, "v" poderia conter metade de
                // uma escrita antiga + metade de uma escrita nova
                // (word tearing) — na prática, JVMs modernas em hardware
                // de 64 bits tendem a implementar essa escrita como uma
                // única instrução atômica de CPU, então isso raramente
                // (ou nunca) é observável empiricamente numa máquina
                // comum, mesmo sem volatile.
            }
        });
        escritora.start();
        leitora.start();
        escritora.join();
        leitora.join();
        System.out.println("Concluído");
    }
}
```

Raciocínio (o que vai no comentário, resumido aqui): mesmo sem observar o bug na prática, ele continua sendo _permitido_ pela especificação em qualquer plataforma que não dê garantia explícita de atomicidade pra `long`/`double` de 64 bits — e código que "passa" só porque a JVM/hardware atual dá uma garantia a mais do que a linguagem exige não é código correto, é código que depende de um detalhe de implementação não documentado. Declarar `volatile` é a única forma de ter essa garantia _pela especificação da linguagem_, portável entre JVMs/plataformas diferentes.

**Exercício 4**

java

```java
import java.util.Optional;

class CacheSimples<K, V> {
    private static final class Entrada<K, V> {
        final K chave;
        final V valor;
        Entrada(K chave, V valor) {
            this.chave = chave;
            this.valor = valor;
        }
    }

    private volatile Entrada<K, V> ultimaEntrada;

    Optional<V> buscarUltima(K chave) {
        Entrada<K, V> snapshot = ultimaEntrada; // lê a referência UMA vez
        if (snapshot != null && snapshot.chave.equals(chave)) {
            return Optional.of(snapshot.valor);
        }
        return Optional.empty();
    }

    void atualizar(K chave, V valor) {
        ultimaEntrada = new Entrada<>(chave, valor); // troca a referência inteira
    }
}
```

Raciocínio: essa implementação é thread-safe sem `synchronized` por dois motivos combinados: (1) `Entrada` é imutável (campos `final`), então, pela garantia de publicação segura de `final` que você viu no tópico de JMM, qualquer thread que enxergue uma instância de `Entrada` através da referência já vê seus campos corretamente preenchidos, sem necessidade de sincronização adicional; (2) `volatile` na referência `ultimaEntrada` garante que a troca feita por `atualizar()` fica visível para `buscarUltima()` chamado por outra thread, e a leitura da referência é feita uma única vez (`snapshot`), evitando qualquer inconsistência de ler `chave` de uma entrada e `valor` de outra caso a referência mude no meio da checagem. Essa combinação — objeto imutável + referência `volatile` — é um padrão comum pra caches/configurações pequenas que mudam com pouca frequência, exatamente porque evita o custo de `synchronized` no caminho de leitura (que, num cache, tende a ser muito mais frequente que o de escrita).