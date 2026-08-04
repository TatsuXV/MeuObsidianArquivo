#### 1. Teoria

**Por que precisa de um "modelo de memória" pra começo de conversa**

No tópico anterior você já viu que threads compartilham a mesma heap. O que não ficou explícito é: **isso não significa que uma thread "vê" imediatamente o que outra thread escreveu.** O motivo é hardware e otimização de compilador:

- CPUs modernas têm caches (L1/L2/L3) por núcleo. Uma escrita feita por uma thread rodando no núcleo A pode ficar "presa" no cache daquele núcleo por um tempo antes de ser propagada pra memória principal (RAM) — e só a partir daí outras threads (rodando em outros núcleos) enxergam a mudança.
- O compilador JIT e o próprio processador podem **reordenar** instruções pra otimizar performance, desde que isso não mude o resultado observado _dentro daquela mesma thread_ (chamado de "as-if-serial" — do ponto de vista de quem escreveu o código sequencialmente, tudo parece na ordem certa). O problema é que outra thread, observando de fora, pode enxergar essas mudanças de estado numa ordem diferente da que está escrita no código-fonte.

Sem um contrato formal, cada JVM/CPU poderia se comportar diferente, e código concorrente "correto" numa máquina quebraria silenciosamente em outra. É exatamente esse contrato que o **Java Memory Model (JMM)** define.

**Contexto histórico (confirmado agora, porque é justamente detalhe de versão que não vale chutar):**  
O modelo de memória original do Java (versões 1 a 4) era amplamente considerado quebrado pela comunidade. Ele foi substituído pelo JSR-133, liderado por Jeremy Manson, Brian Goetz e William Pugh, a partir do Java 5 em 2004 — e, com pequenos refinamentos, é esse modelo do JSR-133 que continua valendo hoje. Não é um recurso de linguagem que você "usa" diretamente feito uma classe; é a especificação que garante que `volatile`, `synchronized` e `final` funcionam de forma previsível. [JSR 133 (Java Memory Model) FAQ +2](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html)

**O conceito central: happens-before**

O objetivo de design declarado no próprio JSR-133 é garantir que programas corretamente sincronizados tenham semântica sequencialmente consistente — ou seja, se comportem como se todas as operações executassem numa ordem total, de forma coerente entre as threads. A ferramenta que garante isso é a relação **happens-before**: é importante notar que happens-before é uma ordem parcial, não total — ou seja, ela não diz "tudo acontece nessa ordem única", ela diz "_se_ A happens-before B, então todo mundo vai enxergar A antes de B"; mas ações não relacionadas por happens-before não têm garantia nenhuma de ordem entre si. [University of Maryland Department of Computer Science](http://www.cs.umd.edu/~pugh/java/memoryModel/CommunityReview.pdf)[University of Maryland Department of Computer Science](http://www.cs.umd.edu/~pugh/java/memoryModel/CommunityReview.pdf)

As regras principais de happens-before que você usa no dia a dia:

1. **Program order dentro da mesma thread**: cada ação numa thread happens-before a próxima ação na mesma thread (isso é o "as-if-serial" que mencionei).
2. **Monitor lock**: um `unlock` de um monitor (fim de bloco `synchronized`) happens-before qualquer `lock` subsequente do _mesmo_ monitor (por qualquer thread). É isso que garante que tudo que a thread A escreveu dentro do `synchronized` fica visível pra thread B quando ela entra no mesmo bloco depois.
3. **Escrita em `volatile`**: uma escrita numa variável `volatile` happens-before qualquer leitura subsequente _dessa mesma variável_.
4. **`Thread.start()`**: happens-before qualquer ação dentro da thread iniciada.
5. **Última ação de uma thread**: happens-before o retorno de `Thread.join()` bem-sucedido nela.
6. **Campos `final`**: o modelo antigo não tratava campos `final` de forma diferente de qualquer outro campo — o que significava que, em teoria, outra thread poderia ver o valor padrão/zero de um campo `final` em vez do valor atribuído no construtor. O JSR-133 corrigiu isso: hoje, se um objeto é construído corretamente (sem "escapar" a referência `this` durante a construção), qualquer thread que enxergue a referência do objeto depois do construtor terminar vê o valor correto do campo `final`, sem precisar de sincronização extra. [slideshare](https://www.slideshare.net/slideshow/java-memory-model-23207253/23207253)

**O que isso significa na prática pro seu código**

Sem nenhum desses mecanismos, uma escrita feita por uma thread pode **nunca** ficar visível pra outra — não é só "demora", pode travar num cache de núcleo indefinidamente do ponto de vista prático, ou o compilador pode até eliminar uma leitura repetida assumindo (corretamente, dentro das regras do JMM, mas de forma surpreendente pra quem não conhece isso) que o valor "não muda".

**Onde isso aparece no backend real**

- Um `boolean flag` compartilhado entre threads sem `volatile` (ex: um flag de "pare de processar" lido por uma thread de worker, escrito por outra) pode nunca ser enxergado pela thread que está no loop — ela pode ficar rodando pra sempre, mesmo que a outra thread já tenha "escrito" `flag = true`.
- **Double-checked locking** mal feito (padrão clássico de inicialização lazy de singleton) é o exemplo didático mais citado de bug de JMM — sem `volatile` no campo, outra thread pode enxergar uma referência de objeto "parcialmente construída".
- Beans `@Singleton`/`@Service` do Spring com estado mutável compartilhado entre requisições (mencionei isso no tópico de Threads) são exatamente o tipo de cenário onde falta de happens-before vira bug intermitente em produção — funciona 99% do tempo, falha esporadicamente sob carga, e é infernal de reproduzir em ambiente de dev com pouca concorrência.

---

#### 2. Exemplo de código comentado

**Problema de visibilidade sem `volatile` (pode não terminar nunca, ou terminar de forma imprevisível):**

java

```java
public class VisibilidadeExemplo {

    // SEM volatile: nada garante que a thread leitora
    // vai enxergar a mudança feita pela thread escritora.
    private static boolean pare = false;

    public static void main(String[] args) throws InterruptedException {
        Thread leitora = new Thread(() -> {
            int contador = 0;
            while (!pare) {
                contador++; // loop apertado — o JIT pode inclusive otimizar
                            // isso assumindo que "pare" nunca muda dentro do loop,
                            // já que não há nenhuma barreira de memória aqui
            }
            System.out.println("Parou depois de " + contador + " iterações");
        });

        leitora.start();
        Thread.sleep(1000); // dá tempo da leitora "esquentar"
        pare = true; // essa escrita pode nunca "chegar" na thread leitora
        System.out.println("Sinal de parada enviado");
    }
}
```

**Corrigido com `volatile` (garante happens-before entre escrita e leitura):**

java

```java
public class VisibilidadeCorrigida {

    // volatile: toda escrita é imediatamente visível a qualquer
    // thread que leia essa variável depois (happens-before garantido
    // pela regra 3 acima), e o compilador não pode reordenar/otimizar
    // a leitura assumindo que o valor não muda.
    private static volatile boolean pare = false;

    public static void main(String[] args) throws InterruptedException {
        Thread leitora = new Thread(() -> {
            int contador = 0;
            while (!pare) {
                contador++;
            }
            System.out.println("Parou depois de " + contador + " iterações");
        });

        leitora.start();
        Thread.sleep(1000);
        pare = true;
        System.out.println("Sinal de parada enviado");
    }
}
```

**O clássico double-checked locking — versão quebrada e corrigida:**

java

```java
class SingletonQuebrado {
    private static SingletonQuebrado instancia; // SEM volatile

    static SingletonQuebrado getInstance() {
        if (instancia == null) {                 // 1ª checagem (sem lock, por performance)
            synchronized (SingletonQuebrado.class) {
                if (instancia == null) {          // 2ª checagem (com lock, evita corrida)
                    instancia = new SingletonQuebrado();
                    // PROBLEMA: "new SingletonQuebrado()" não é uma operação
                    // atômica indivisível do ponto de vista do JMM. Ela envolve:
                    // (a) alocar memória, (b) rodar o construtor, (c) atribuir
                    // a referência a "instancia". O compilador/CPU pode
                    // REORDENAR (c) antes de (b) terminar completamente —
                    // outra thread pode ver "instancia" != null mas apontando
                    // pra um objeto ainda não totalmente construído.
                }
            }
        }
        return instancia;
    }
}

class SingletonCorrigido {
    // volatile aqui impede especificamente essa reordenação problemática
    private static volatile SingletonCorrigido instancia;

    static SingletonCorrigido getInstance() {
        if (instancia == null) {
            synchronized (SingletonCorrigido.class) {
                if (instancia == null) {
                    instancia = new SingletonCorrigido();
                }
            }
        }
        return instancia;
    }
}
```

---

#### 3. Armadilhas comuns

1. **Achar que `synchronized` só serve pra evitar race condition (exclusão mútua).** Ele também garante _visibilidade_ (happens-before entre unlock e lock subsequentes) — as duas coisas andam juntas, mas são conceitos diferentes, e é comum estudante decorar só a parte de "evita dois incrementos ao mesmo tempo" e esquecer da parte de visibilidade de memória.
2. **Achar que `volatile` substitui `synchronized` pra qualquer caso.** `volatile` garante visibilidade e impede certas reordenações, mas **não garante atomicidade de operações compostas**. `volatile int contador; contador++;` ainda tem race condition — `++` continua sendo ler+somar+escrever, e `volatile` não torna isso uma operação atômica.
3. **Double-checked locking sem `volatile`** — é o exemplo #1 citado em toda entrevista sobre JMM porque parece funcionar na maioria dos testes locais (poucos núcleos, JIT não tão agressivo) e falha esporadicamente em produção sob carga real.
4. **Confundir "a variável está desatualizada" com "é só coincidência/sorte que funcionou".** Se seu código depende de happens-before e você não tem nenhum dos mecanismos (synchronized/volatile/final/join corretamente usados), qualquer comportamento "correto" que você observou rodando localmente **não é garantia de nada** — o JMM permite explicitamente que isso quebre em outra JVM, outro hardware, ou até na mesma máquina sob outra carga.

---

#### 4. Exercícios práticos

**1. Fácil**  
Rode o exemplo `VisibilidadeExemplo` (sem `volatile`) da seção 2 na sua máquina algumas vezes. Depois rode `VisibilidadeCorrigida`. Anote o que observou (é possível que na sua máquina/JVM específica o primeiro exemplo "funcione" mesmo sem `volatile` — isso não invalida o problema, só mostra que o JMM _permite_ esse comportamento incorreto, não que ele _sempre_ aconteça). Explique com suas palavras por que "funcionou na minha máquina" não é prova de que o código está correto do ponto de vista do JMM.

**2. Médio**  
Implemente uma classe `ContadorComVisibilidade` com um campo `volatile int valor` e dois métodos: `incrementar()` (faz `valor++`, sem `synchronized`) e `getValor()`. Crie 100 threads que chamam `incrementar()` uma vez cada, dê `join()` em todas, e imprima o valor final. Rode várias vezes. Depois responda: o valor final bate sempre com 100? Por quê (ou por que não), considerando o que `volatile` garante e o que ele NÃO garante?

**3. Difícil**  
Implemente uma classe `ConfiguracaoImutavel` com dois campos `final` (`String nome` e `int versao`), preenchidos no construtor. Crie um cenário com duas threads: a thread A cria a instância (`new ConfiguracaoImutavel(...)`) e publica a referência num campo estático **sem nenhuma sincronização** (nem `volatile`, nem `synchronized`); a thread B fica em loop checando se aquele campo estático não é mais `null` e, quando não for, lê e imprime `nome` e `versao`. Critério de pronto: mesmo sem `volatile` na publicação da referência, os campos `final` lidos pela thread B devem sempre mostrar os valores corretos do construtor (nunca valor "zerado" ou parcial) — isso demonstra na prática a garantia especial que `final` ganhou a partir do JSR-133. Documente no código, em um comentário, por que isso funciona mesmo sem sincronização explícita na publicação.

**4. Desafio**  
Reimplemente `SingletonQuebrado` da seção 2, mas em vez de usar `volatile`, resolva o problema de outra forma válida: usando o padrão **initialization-on-demand holder** (uma classe estática aninhada que só é carregada quando `getInstance()` é chamado pela primeira vez, aproveitando as garantias de inicialização de classe da JVM — pesquise "class initialization happens-before" ou "JLS class loading" se precisar, e cite a fonte que confirmar isso). Explique no comentário do código por que essa alternativa dispensa tanto `synchronized` quanto `volatile` no caminho comum (depois da primeira chamada), e que garantia do JMM ela usa por baixo dos panos.

---

#### 5. Gabarito comentado

**Exercício 1**

Não há "código de solução" fixo aqui — o objetivo era observação. Raciocínio esperado na resposta: mesmo que `VisibilidadeExemplo` termine "corretamente" na sua máquina em todos os testes, isso é uma coincidência de implementação (JIT específico, JVM específica, número de núcleos, carga da máquina no momento) e **não** uma garantia da linguagem. O JMM define o que é _permitido_ acontecer, não o que _vai_ acontecer numa execução específica — código que depende de comportamento não garantido pelo JMM é, por definição, um bug latente, mesmo que "passe no teste" hoje.

**Exercício 2**

java

```java
class ContadorComVisibilidade {
    private volatile int valor = 0;
    public void incrementar() { valor++; }
    public int getValor() { return valor; }
}

public class Ex2JMM {
    public static void main(String[] args) throws InterruptedException {
        ContadorComVisibilidade c = new ContadorComVisibilidade();
        Thread[] threads = new Thread[100];
        for (int i = 0; i < 100; i++) {
            threads[i] = new Thread(c::incrementar);
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println("Valor final: " + c.getValor());
    }
}
```

Raciocínio: o valor final **não bate sempre com 100** (mesma causa raiz do Exercício 2 da sessão de Threads). `volatile` garante que toda leitura de `valor` enxerga a escrita mais recente feita por qualquer thread — ou seja, resolve o problema de _visibilidade_. Mas `valor++` continua sendo três passos (ler, somar, escrever) executados sem exclusão mútua — duas threads podem ler o mesmo valor "atual" antes de qualquer uma escrever o resultado somado, perdendo um incremento. Isso é exatamente a diferença entre visibilidade (o que `volatile` resolve) e atomicidade (o que precisaria de `synchronized` ou `AtomicInteger` pra resolver — o mesmo ponto do gabarito anterior).

**Exercício 3**

java

```java
class ConfiguracaoImutavel {
    final String nome;
    final int versao;

    ConfiguracaoImutavel(String nome, int versao) {
        this.nome = nome;
        this.versao = versao;
    }
}

public class Ex3JMM {
    // Propositalmente SEM volatile, para demonstrar a garantia de "final"
    private static ConfiguracaoImutavel config;

    public static void main(String[] args) throws InterruptedException {
        Thread publicadora = new Thread(() -> {
            try { Thread.sleep(50); } catch (InterruptedException e) {}
            config = new ConfiguracaoImutavel("prod", 3);
        });

        Thread leitora = new Thread(() -> {
            while (config == null) {
                // busy-wait só para fins didáticos deste exercício;
                // em código real isso pediria sincronização adequada
                // de qualquer forma, por outros motivos de design
            }
            // Graças à garantia de "final" do JSR-133 (safe publication),
            // se a referência "config" já não é mais null aqui,
            // os campos final dela JÁ estão com os valores corretos do
            // construtor — não existe janela onde vemos a referência
            // não-nula mas com "nome" ainda em seu valor padrão (null).
            System.out.println("nome=" + config.nome + " versao=" + config.versao);
        });

        leitora.start();
        publicadora.start();
        leitora.join();
        publicadora.join();
    }
}
```

Raciocínio: essa é justamente a lacuna que o JSR-133 fechou. No modelo antigo, nada tratava campos `final` de forma diferente de qualquer outro campo, então em teoria uma thread poderia ver o valor padrão/zero de um campo `final` em vez do valor atribuído no construtor. A partir do JSR-133, a JVM garante que a atribuição a um campo `final` no construtor happens-before qualquer uso da referência ao objeto por outra thread, **desde que o objeto não "escape"** durante a construção (ex: não passar `this` pra fora do construtor antes dele terminar). É por isso que `final` é frequentemente recomendado como a forma _mais simples_ de publicar objetos imutáveis com segurança entre threads, sem precisar de `volatile`/`synchronized` na publicação. [slideshare](https://www.slideshare.net/slideshow/java-memory-model-23207253/23207253)

**Exercício 4**

java

```java
public class SingletonHolder {
    private SingletonHolder() {}

    // Classe estática aninhada: só é CARREGADA (e portanto só o campo
    // estático INSTANCE só é inicializado) na primeira vez que
    // Holder.INSTANCE é referenciado — ou seja, na primeira chamada
    // de getInstance(). Isso é "lazy" sem precisar de nenhuma checagem
    // manual de null.
    private static class Holder {
        static final SingletonHolder INSTANCE = new SingletonHolder();
    }

    public static SingletonHolder getInstance() {
        return Holder.INSTANCE;
        // Nenhum synchronized, nenhum volatile aqui. A garantia vem de
        // outro lugar: a especificação da JVM garante que a
        // inicialização de uma classe é feita de forma segura entre
        // threads (a JVM usa lock interno só durante o carregamento
        // da classe, uma única vez) — e a conclusão dessa inicialização
        // happens-before qualquer uso subsequente da classe por
        // qualquer thread. Depois que a classe "Holder" já foi
        // carregada uma vez, chamadas seguintes de getInstance() são
        // apenas leitura de um campo estático já publicado com
        // segurança — sem custo de lock a cada chamada, diferente do
        // double-checked locking, que paga o custo de pelo menos uma
        // checagem de null a cada chamada mesmo depois de inicializado
        // (embora seja um custo pequeno).
    }
}
```

Raciocínio: essa alternativa terceiriza a garantia de "inicialize uma vez, com segurança entre threads" pro próprio mecanismo de carregamento de classes da JVM, em vez de reimplementar isso manualmente com `synchronized`/`volatile`. É considerado por muita gente mais simples e menos propenso a erro do que double-checked locking, justamente porque você não precisa raciocinar sobre reordenação de instruções — a JVM já garante isso pra inicialização de classe por design.