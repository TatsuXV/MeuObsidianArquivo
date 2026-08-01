#### 1. Teoria

`final` é um modificador que **trava** algo contra modificação futura. O que exatamente ele trava depende de onde é aplicado — em Java, `final` tem três usos completamente diferentes: em variável, em método e em classe. É importante não confundir os três, porque o efeito de cada um é distinto.

**`final` em variável**  
Uma variável marcada `final` só pode ser atribuída **uma vez**. Depois disso, tentar reatribuir é erro de compilação.

Importante entender a diferença entre **variável final** e **objeto imutável** — isso confunde muita gente:

java

```java
final List<String> lista = new ArrayList<>();
lista.add("item"); // OK! A lista pode mudar de conteúdo.
lista = new ArrayList<>(); // ERRO de compilação — a REFERÊNCIA não pode mudar.
```

`final` trava a **referência** (o "endereço" que a variável aponta), não o conteúdo do objeto. Se você quer um objeto verdadeiramente imutável, isso é responsabilidade da própria classe do objeto (ex: `String` é imutável por design; `List.of(...)` retorna uma lista imutável), não do `final`.

**`final` em método**  
Um método `final` numa classe **não pode ser sobrescrito** (`@Override`) por nenhuma subclasse. Usado quando você quer garantir que o comportamento daquele método seja idêntico em toda a hierarquia — por exemplo, pra proteger uma regra de negócio crítica ou uma etapa de um algoritmo (padrão _Template Method_) de ser alterada por engano numa subclasse.

**`final` em classe**  
Uma classe `final` **não pode ser estendida** (`extends`) por nenhuma outra classe. É o oposto de uma classe pensada pra herança. Exemplos do próprio JDK: `String`, `Integer` e todas as classes wrapper são `final` — isso é proposital, para garantir que o comportamento delas nunca seja alterado por uma subclasse mal-intencionada ou descuidada, o que quebraria garantias que todo o resto da linguagem assume sobre elas.

**Diferença de `final` para `static`:**  
São conceitos ortogonais e frequentemente confundidos por quem está começando. `static` diz respeito a **pertencer à classe em vez de à instância**. `final` diz respeito a **não poder ser reatribuído/sobrescrito/estendido**. Eles costumam aparecer juntos em constantes (`public static final int MAX = 100;`), mas resolvem problemas diferentes: `static` faz o campo existir uma única vez, compartilhado por todas as instâncias; `final` impede que esse valor único seja alterado depois de definido.

**Onde aparece no dia a dia de backend Java/Spring:**

- Constantes de configuração: `public static final int MAX_RETRIES = 3;`
- Campos de classes imutáveis (DTOs, Value Objects): construtor preenche, `final` garante que nunca muda depois — fundamental em ambiente multi-thread (um servidor Spring atende requisições em paralelo).
- Parâmetros de método marcados `final` (opcional, mas alguns times exigem por convenção) — evita reatribuir o parâmetro por engano dentro do método.
- Variáveis usadas dentro de lambdas/classes anônimas: só podem capturar variáveis `final` ou _effectively final_ (que nunca são reatribuídas, mesmo sem a palavra-chave explícita) — isso vai aparecer muito quando você chegar em Streams e Lambda Expressions.
- Records (que você acabou de ver) usam `final` internamente por baixo dos panos — todo campo de um `record` é implicitamente `final`.

---

#### 2. Exemplo de código comentado

java

```java
public class ContaBancaria {

    // final em campo de instância + inicializado no construtor:
    // padrão comum pra representar um dado que nunca muda depois de criado o objeto
    private final String numeroConta;
    private double saldo; // este pode mudar, então NÃO é final

    // Constante de classe: static (uma cópia só, da classe) + final (nunca muda)
    public static final double TAXA_SAQUE = 2.50;

    public ContaBancaria(String numeroConta, double saldoInicial) {
        this.numeroConta = numeroConta; // única atribuição permitida
        this.saldo = saldoInicial;
        // this.numeroConta = "outro"; // ERRO se descomentado: já foi atribuído
    }

    public void sacar(double valor) {
        this.saldo -= (valor + TAXA_SAQUE);
    }

    // método final: nenhuma subclasse de ContaBancaria pode sobrescrever a
    // regra de cálculo de taxa — protege uma regra de negócio crítica
    public final double calcularTaxaSaque() {
        return TAXA_SAQUE;
    }
}

// Classe final: ninguém pode herdar de ContaPoupanca e alterar seu comportamento
public final class ContaPoupanca extends ContaBancaria {
    public ContaPoupanca(String numeroConta, double saldoInicial) {
        super(numeroConta, saldoInicial);
    }
    // Se eu tentasse: @Override public double calcularTaxaSaque() {...}
    // isso NÃO compilaria, porque o método na superclasse é final.
}
```

java

```java
// Exemplo de "effectively final" em uso com lambda
public void exemploLambda() {
    int limite = 10; // nunca reatribuída depois -> effectively final

    Runnable tarefa = () -> {
        System.out.println("Limite é: " + limite); // pode capturar, porque é effectively final
    };

    // limite = 20; // se eu descomentar isso, a linha do lambda acima
                     // deixa de compilar, pois limite deixou de ser effectively final
    tarefa.run();
}
```

---

#### 3. Armadilhas comuns

1. **Achar que `final` em objeto/coleção torna o conteúdo imutável.** Como mostrado na Teoria, `final List<String> lista` permite `lista.add(...)` normalmente. Quem confunde isso acaba com bugs de estado mutável escondido, achando que "protegeu" o dado.
2. **Marcar tudo como `final` por hábito sem necessidade, em parâmetro de método.** Não é errado, mas em times que não têm essa convenção, isso só adiciona ruído visual sem ganho real — é mais estilo de código do que regra técnica. Vale saber que existe, mas não é unanimidade em todo projeto.
3. **Tentar sobrescrever método `final` herdado e não entender o erro de compilação.** Iniciante às vezes gasta tempo tentando descobrir "por que meu `@Override` não compila" sem perceber que o método na superclasse foi marcado `final` de propósito.
4. **Esquecer de inicializar um campo `final` em TODOS os caminhos do construtor.** Se uma classe tem mais de um construtor, ou um construtor com `if/else`, o compilador exige que o campo `final` seja garantidamente atribuído em qualquer caminho de execução — senão dá erro de compilação ("variable might not have been initialized").

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie uma classe `Pessoa` com um campo `final String nome`, atribuído via construtor. Tente escrever um método `renomear(String novoNome)` que reatribui `this.nome`. Comente no código o que acontece e por quê.  
_Critério de pronto:_ o projeto demonstra (via comentário ou captura do erro do compilador) que a reatribuição não compila, explicando a razão em uma frase.

**Exercício 2 (Médio)**  
Crie uma classe `Configuracao` com uma constante `public static final int TIMEOUT_PADRAO = 30;` e um campo de instância `private final List<String> permissoes`, inicializado no construtor a partir de uma lista recebida como parâmetro. Depois, escreva um método `adicionarPermissao(String p)` que adiciona um item à lista `permissoes`.  
_Critério de pronto:_ o código compila (já que `final` não impede mutação da lista), e você escreve um comentário explicando por que isso não contradiz o `final`. Bônus: refatore para usar `List.copyOf()` no construtor e mostre o que muda (o `add` passa a lançar exceção em runtime).

**Exercício 3 (Difícil)**  
Crie uma classe abstrata `FormaGeometrica` com um método `final double calcularAreaComDesconto(double percentualDesconto)`, que internamente chama um método abstrato `double calcularArea()` (que cada subclasse implementa) e aplica o desconto. Crie duas subclasses (`Quadrado`, `Circulo`) que só implementam `calcularArea()`. Demonstre por que faz sentido `calcularAreaComDesconto` ser `final` aqui (o que aconteceria de errado se não fosse).  
_Critério de pronto:_ código compila, as duas subclasses calculam a área com desconto corretamente, e existe uma explicação escrita (comentário ou texto) do risco de negócio evitado ao tornar o método `final`.

**Exercício 4 (Desafio)**  
Escreva um método que recebe uma `List<Runnable>` vazia e, dentro de um loop `for` de 0 a 4, cria e adiciona 5 lambdas `Runnable` nessa lista — cada lambda, ao ser executada, deve imprimir o número da iteração em que foi criada (0, 1, 2, 3, 4). Depois, execute todas as `Runnable`s da lista. Este exercício testa sua compreensão de _effectively final_ em capturas de lambda dentro de loop — pesquise/raciocine sobre por que usar a variável de controle do `for` diretamente na lambda não funciona, e qual ajuste resolve isso.  
_Critério de pronto:_ ao rodar, o programa imprime corretamente 0, 1, 2, 3, 4 (cada lambda "lembra" do número certo, não todas imprimindo 4 ou 5).

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Pessoa {
    private final String nome;

    public Pessoa(String nome) {
        this.nome = nome; // única atribuição válida
    }

    public void renomear(String novoNome) {
        // this.nome = novoNome; // ERRO DE COMPILAÇÃO:
        // "cannot assign a value to final variable nome"
        // Porque 'nome' já recebeu seu único valor permitido no construtor.
    }

    public String getNome() {
        return nome;
    }
}
```

_Raciocínio:_ isso ilustra o caso de uso mais comum de `final` em campo — modelar um dado que é parte da **identidade** do objeto e não deveria mudar depois de criado. Se `Pessoa` realmente precisasse de um nome mutável, o campo simplesmente não seria `final`, e `renomear` funcionaria normalmente com um setter.

**Exercício 2**

java

```java
import java.util.ArrayList;
import java.util.List;

public class Configuracao {
    public static final int TIMEOUT_PADRAO = 30;

    private final List<String> permissoes;

    public Configuracao(List<String> permissoesIniciais) {
        this.permissoes = new ArrayList<>(permissoesIniciais);
        // Referência final atribuída uma vez; o objeto ArrayList em si continua mutável.
    }

    public void adicionarPermissao(String p) {
        this.permissoes.add(p); // OK — não é reatribuição de 'permissoes', é mutação do conteúdo
    }
}
```

_Raciocínio:_ isso compila normalmente porque `final` protege apenas a referência `permissoes` contra apontar para uma lista diferente — nunca protegeu o conteúdo.

**Versão bônus com `List.copyOf()`:**

java

```java
public Configuracao(List<String> permissoesIniciais) {
    this.permissoes = List.copyOf(permissoesIniciais); // lista verdadeiramente imutável
}

public void adicionarPermissao(String p) {
    this.permissoes.add(p); // agora lança UnsupportedOperationException em RUNTIME
}
```

_Trade-off:_ a versão com `ArrayList` é mutável e flexível, mas exige disciplina do time pra não abusar. A versão com `List.copyOf()` é genuinamente imutável, mas qualquer tentativa de `adicionarPermissao` quebra em runtime — nesse caso, o método `adicionarPermissao` deixaria de fazer sentido como está escrito; a classe precisaria ser redesenhada (ex: retornar uma nova instância de `Configuracao` em vez de mutar).

**Exercício 3**

java

```java
public abstract class FormaGeometrica {

    public final double calcularAreaComDesconto(double percentualDesconto) {
        double area = calcularArea();
        return area - (area * percentualDesconto / 100);
    }

    public abstract double calcularArea();
}

public class Quadrado extends FormaGeometrica {
    private final double lado;

    public Quadrado(double lado) {
        this.lado = lado;
    }

    @Override
    public double calcularArea() {
        return lado * lado;
    }
}

public class Circulo extends FormaGeometrica {
    private final double raio;

    public Circulo(double raio) {
        this.raio = raio;
    }

    @Override
    public double calcularArea() {
        return Math.PI * raio * raio;
    }
}
```

_Raciocínio:_ isso é o padrão **Template Method** — a "receita" de como calcular o desconto (pegar a área, aplicar o percentual) é uma regra de negócio única que deve ser idêntica pra toda forma geométrica que existir no sistema, hoje ou no futuro. Se `calcularAreaComDesconto` **não** fosse `final`, uma subclasse poderia sobrescrevê-lo e aplicar a fórmula de desconto errada (ex: um desenvolvedor futuro, sem entender a regra, poderia criar `PoligonoEspecial` que calcula desconto de forma diferente por engano ou por atalho, gerando inconsistência de negócio entre formas). O `final` aqui é uma trava contra esse tipo de erro — só o cálculo da _área_ varia por subclasse (`calcularArea`, que é `abstract`, não `final`); o cálculo do _desconto_ é regra fixa da superclasse.

**Exercício 4**

java

```java
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String[] args) {
        List<Runnable> tarefas = new ArrayList<>();

        for (int i = 0; i < 5; i++) {
            final int numeroCapturado = i;
            // Criamos uma NOVA variável 'final' a cada iteração, copiando o valor de 'i'.
            // Essa variável é local ao corpo do loop, então cada lambda captura
            // sua PRÓPRIA cópia, congelada no valor daquele momento.
            tarefas.add(() -> System.out.println("Iteração: " + numeroCapturado));
        }

        for (Runnable tarefa : tarefas) {
            tarefa.run();
        }
        // Saída: Iteração: 0, Iteração: 1, Iteração: 2, Iteração: 3, Iteração: 4
    }
}
```

_Raciocínio:_ a variável de controle `i` de um `for` tradicional **é reatribuída a cada iteração** (`i++`), então ela nunca é _effectively final_ — o compilador simplesmente não deixaria você escrever `() -> System.out.println(i)` diretamente ali dentro, pois isso não compila. A solução clássica é declarar uma variável nova dentro do corpo do loop (`numeroCapturado`), que é recriada a cada passagem e nunca reatribuída depois de criada — ela sim é _effectively final_, e cada lambda "trava" seu próprio valor no momento em que foi criada. É exatamente o mesmo princípio por trás do porquê `for (String item : lista)` (for-each) permite lambdas dentro do corpo sem esse problema: a variável do for-each também é recriada a cada iteração, nunca reatribuída.