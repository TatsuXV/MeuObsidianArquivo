#### 1. Teoria

**Atributos** (também chamados de _campos_ ou _variáveis de instância_) são os dados que uma classe declara para descrever o estado de um objeto. **Métodos** são os blocos de código que descrevem o comportamento desse objeto — o que ele sabe fazer.

java

```java
class Produto {
    String nome;      // atributo
    double preco;     // atributo

    void aplicarDesconto(double percentual) { // método
        preco = preco - (preco * percentual / 100);
    }
}
```

Diferença importante com coisas que se parecem:

- **Atributo vs. variável local**: atributo é declarado no corpo da classe, existe enquanto o objeto existir, e recebe um **valor padrão automático** se você não inicializar (`0` para números, `false` para boolean, `null` para objetos). Variável local é declarada dentro de um método, só existe durante a execução dele, e o Java **não** dá valor padrão — se você tentar usar sem inicializar, é erro de compilação.
- **Atributo vs. parâmetro**: parâmetro é a variável que recebe o valor passado na chamada do método; some quando o método termina, igual variável local.
- **Atributo de instância vs. atributo estático**: o que estamos vendo aqui é atributo de instância — cada objeto tem sua própria cópia. Existe também o atributo estático (compartilhado entre todos os objetos da classe), mas isso é o próximo item do checklist (Static Keyword), então vamos aprofundar isso na próxima sessão.

**Onde isso aparece no dia a dia de backend**: toda entidade JPA (`@Entity`) que você vai mapear pro banco no Spring Data JPA é basicamente isso — uma classe com atributos que viram colunas, e métodos (getters/setters, ou lógica de negócio) que operam sobre esses atributos. Entender bem esse conceito aqui é a base de tudo que vem depois.

#### 2. Exemplo de código comentado

java

```java
public class ContaBancaria {

    // Atributos: estado da conta. Cada objeto ContaBancaria terá
    // sua própria cópia independente desses valores.
    private String titular;
    private double saldo;

    // Construtor: inicializa os atributos quando o objeto é criado.
    public ContaBancaria(String titular, double saldoInicial) {
        this.titular = titular;      // "this.titular" é o atributo
        this.saldo = saldoInicial;   // "saldoInicial" é o parâmetro
    }

    // Método: comportamento que opera sobre o estado (atributos).
    public void depositar(double valor) {
        if (valor <= 0) {
            throw new IllegalArgumentException("Valor de depósito deve ser positivo");
        }
        this.saldo += valor; // acessa e modifica o atributo da instância
    }

    public double getSaldo() {
        return this.saldo; // método que expõe o valor do atributo
    }
}
```

Ponto não óbvio: dentro do construtor, `titular` (parâmetro) e `this.titular` (atributo) têm o mesmo nome de propósito. O `this` é o que desambiguiza — sem ele, `titular = titular` seria o parâmetro atribuindo a ele mesmo, e o atributo continuaria `null`.

#### 3. Armadilhas comuns

1. **Esquecer o `this` quando parâmetro e atributo têm o mesmo nome** — o compilador não avisa, o código roda, mas o atributo nunca é de fato atualizado (bug silencioso clássico).
2. **Achar que atributo primitivo não inicializado dá erro igual variável local** — não dá; ele recebe valor padrão (`0`, `false`, etc.), o que pode mascarar um bug de "esqueci de inicializar isso".
3. **Deixar atributos `public` direto** — funciona, mas quebra encapsulamento (tópico futuro, mas já vale o alerta: qualquer código externo pode alterar o estado sem validação nenhuma).
4. **Confundir atributo de instância com estático antes de aprender `static`** — achar que todos os objetos "compartilham" o valor de um atributo comum é erro típico; cada objeto tem sua cópia até você aprender e usar `static` de propósito.

#### 4. Exercícios práticos

**1. Fácil** — Crie uma classe `Pessoa` com atributos `nome` (String) e `idade` (int). Adicione um método `apresentar()` que imprime algo como `"Olá, meu nome é X e tenho Y anos."`. Critério de pronto: instanciar dois objetos `Pessoa` diferentes e chamar `apresentar()` em cada um, mostrando dados independentes.

**2. Médio** — Refaça a classe `ContaBancaria` do exemplo acima (pode copiar), mas adicione um método `sacar(double valor)` que: lança `IllegalArgumentException` se o valor for negativo ou zero, e lança uma exceção (pode ser `IllegalStateException`) se o saque deixar o saldo negativo. Critério de pronto: testar saque válido, saque inválido (valor negativo) e saque que estouraria o saldo.

**3. Difícil** — Crie uma classe `Retangulo` com atributos `base` e `altura` (double). Implemente `calcularArea()`, `calcularPerimetro()`, e um método `ehMaiorQue(Retangulo outro)` que retorna `boolean` comparando a área deste retângulo com a do outro passado como parâmetro. Critério de pronto: criar dois retângulos diferentes e comprovar que `ehMaiorQue` responde corretamente nos dois sentidos (A maior que B, e B maior que A).

**4. Desafio** — Crie uma classe `CarrinhoSimples` que representa um carrinho com **exatamente 3 itens fixos** (sem usar coleções ainda — isso é tópico futuro do Bloco 4). Use atributos como `nomeItem1`, `precoItem1`, `nomeItem2`, `precoItem2`, `nomeItem3`, `precoItem3`. Implemente um método `calcularTotal()` e um método `aplicarDescontoTotal(double percentual)` que valida se `percentual` está entre 0 e 100 (lançando exceção se não estiver) e retorna o total já com desconto aplicado, **sem alterar os preços originais dos itens**. Critério de pronto: o método de desconto não pode mutar o estado interno dos preços — só retornar o valor calculado.

#### 5. Gabarito comentado

**1. Pessoa**

java

```java
public class Pessoa {
    private String nome;
    private int idade;

    public Pessoa(String nome, int idade) {
        this.nome = nome;
        this.idade = idade;
    }

    public void apresentar() {
        System.out.println("Olá, meu nome é " + nome + " e tenho " + idade + " anos.");
    }
}

// Uso:
Pessoa p1 = new Pessoa("Ana", 28);
Pessoa p2 = new Pessoa("Bruno", 34);
p1.apresentar(); // Olá, meu nome é Ana e tenho 28 anos.
p2.apresentar(); // Olá, meu nome é Bruno e tenho 34 anos.
```

Raciocínio: cada `Pessoa` tem sua própria cópia de `nome` e `idade` — é exatamente o conceito de atributo de instância na prática, dois objetos, dois estados independentes.

**2. ContaBancaria com saque**

java

```java
public void sacar(double valor) {
    if (valor <= 0) {
        throw new IllegalArgumentException("Valor de saque deve ser positivo");
    }
    if (valor > this.saldo) {
        throw new IllegalStateException("Saldo insuficiente");
    }
    this.saldo -= valor;
}
```

Raciocínio: a ordem das validações importa — primeiro valida a entrada (`valor` em si é inválido?), depois valida a regra de negócio (o estado atual permite essa operação?). Isso é um padrão que você vai repetir a vida toda em backend.

**3. Retangulo**

java

```java
public class Retangulo {
    private double base;
    private double altura;

    public Retangulo(double base, double altura) {
        this.base = base;
        this.altura = altura;
    }

    public double calcularArea() {
        return base * altura;
    }

    public double calcularPerimetro() {
        return 2 * (base + altura);
    }

    public boolean ehMaiorQue(Retangulo outro) {
        return this.calcularArea() > outro.calcularArea();
    }
}
```

Raciocínio: `ehMaiorQue` reutiliza `calcularArea()` em vez de recalcular `base * altura` na mão duas vezes — evita duplicação e, se a fórmula de área mudar um dia, você só corrige em um lugar. Repare que `outro.calcularArea()` acessa o método de **outro objeto** da mesma classe — isso é permitido porque o método está _dentro_ da própria classe `Retangulo`.

**4. CarrinhoSimples**

java

```java
public class CarrinhoSimples {
    private String nomeItem1;
    private double precoItem1;
    private String nomeItem2;
    private double precoItem2;
    private String nomeItem3;
    private double precoItem3;

    public CarrinhoSimples(String nomeItem1, double precoItem1,
                            String nomeItem2, double precoItem2,
                            String nomeItem3, double precoItem3) {
        this.nomeItem1 = nomeItem1;
        this.precoItem1 = precoItem1;
        this.nomeItem2 = nomeItem2;
        this.precoItem2 = precoItem2;
        this.nomeItem3 = nomeItem3;
        this.precoItem3 = precoItem3;
    }

    public double calcularTotal() {
        return precoItem1 + precoItem2 + precoItem3;
    }

    public double aplicarDescontoTotal(double percentual) {
        if (percentual < 0 || percentual > 100) {
            throw new IllegalArgumentException("Percentual deve estar entre 0 e 100");
        }
        double total = calcularTotal();
        return total - (total * percentual / 100);
        // note: não alteramos precoItem1/2/3 — só retornamos um valor calculado
    }
}
```

Raciocínio: o ponto central do desafio é perceber que "aplicar desconto no total" não deveria mutar o preço de cada item individualmente — são conceitos diferentes (preço unitário vs. valor final da compra). Esse é um exercício mental que se conecta direto com o próximo Bloco (Coleções): você já deve estar sentindo que ter 3 atributos separados pra "itens" é falta de jeito — é exatamente por isso que `List`/`ArrayList` existe, mas isso é assunto do Bloco 4, não vamos adiantar.