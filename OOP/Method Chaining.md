#### 1. Teoria

**Method Chaining** (encadeamento de métodos) é um estilo de escrita onde você chama vários métodos em sequência, numa única expressão, porque cada método **retorna uma referência** que permite continuar chamando o próximo método diretamente em cima do resultado. Não é uma feature da linguagem — é um **padrão de design**, viabilizado por uma escolha simples: fazer o método retornar `this` (ou um novo objeto) em vez de `void`.

**Como funciona por baixo:**

java

```java
objeto.metodoA().metodoB().metodoC();
```

Isso só compila se `metodoA()` retornar algo do tipo `objeto` (ou compatível), pois `.metodoB()` é chamado **em cima do valor retornado** por `metodoA()`, não em cima de `objeto` de novo.

**Duas variações de method chaining, com propósitos diferentes:**

1. **Fluent Interface mutável (retorna `this`)** — cada método na cadeia modifica o **mesmo objeto** e retorna a própria instância (`return this;`). Muito comum em _builders_ de configuração.
2. **Fluent Interface imutável (retorna um novo objeto)** — cada método na cadeia **não** modifica o objeto original; em vez disso, cria e retorna um **novo objeto** com a alteração aplicada. `String` funciona assim: `"abc".toUpperCase().trim()` — nenhum desses métodos altera a `String` original (que é imutável por design, como você viu em tópicos anteriores).

**Diferença de Method Chaining vs. Builder Pattern:**  
Muita gente confunde os dois porque geralmente andam juntos, mas são coisas diferentes: _Method Chaining_ é a **técnica sintática** (retornar algo encadeável). _Builder Pattern_ é um **padrão de projeto completo**, focado especificamente em construir objetos complexos passo a passo, geralmente terminando com um método `.build()` que retorna o objeto final pronto e imutável. Ou seja: todo Builder normalmente usa Method Chaining, mas nem todo Method Chaining é um Builder (ex: `StringBuilder.append("a").append("b")` usa chaining, mas não é o padrão Builder — é só uma classe mutável com chaining).

**Onde aparece no dia a dia de backend Java/Spring:**

- `StringBuilder`: `new StringBuilder().append("Olá").append(", ").append("mundo").toString();`
- Stream API (você vai ver a fundo em breve): `lista.stream().filter(x -> x > 0).map(x -> x * 2).collect(...);` — é encadeamento puro.
- `Optional`: `Optional.ofNullable(usuario).map(Usuario::getEmail).orElse("sem email");`
- Builders de configuração no Spring, como `ResponseEntity.status(200).header("X-Custom", "valor").body(dados);`
- Bibliotecas de teste como Mockito/AssertJ: `assertThat(resultado).isNotNull().hasSize(3).contains("x");`
- Query builders (ex: `Specification` no Spring Data JPA, ou builders de query de bibliotecas como jOOQ).

---

#### 2. Exemplo de código comentado

java

```java
// Exemplo 1: Fluent Interface MUTÁVEL — cada método retorna 'this' (o mesmo objeto)
public class ConfiguracaoServidor {
    private String host;
    private int porta;
    private boolean https;

    public ConfiguracaoServidor host(String host) {
        this.host = host;
        return this; // retorna a PRÓPRIA instância, já modificada
    }

    public ConfiguracaoServidor porta(int porta) {
        this.porta = porta;
        return this;
    }

    public ConfiguracaoServidor https(boolean https) {
        this.https = https;
        return this;
    }

    @Override
    public String toString() {
        return (https ? "https://" : "http://") + host + ":" + porta;
    }
}

// Uso: uma única expressão constrói e configura o objeto
ConfiguracaoServidor config = new ConfiguracaoServidor()
        .host("api.exemplo.com")
        .porta(443)
        .https(true);

System.out.println(config); // https://api.exemplo.com:443
```

java

```java
// Exemplo 2: Fluent Interface IMUTÁVEL — cada método retorna um NOVO objeto
public final class Ponto {
    private final int x;
    private final int y;

    public Ponto(int x, int y) {
        this.x = x;
        this.y = y;
    }

    // Cada método cria e retorna um Ponto NOVO — o objeto original nunca muda
    public Ponto mover(int deltaX, int deltaY) {
        return new Ponto(this.x + deltaX, this.y + deltaY);
    }

    public Ponto espelharX() {
        return new Ponto(-this.x, this.y);
    }

    @Override
    public String toString() {
        return "(" + x + ", " + y + ")";
    }
}

// Uso:
Ponto original = new Ponto(1, 1);
Ponto resultado = original.mover(2, 3).espelharX(); // encadeamento, cada passo gera novo objeto

System.out.println(original);  // (1, 1)  <- não mudou!
System.out.println(resultado); // (-3, 4)
```

java

```java
// Exemplo 3: Builder Pattern clássico — chaining + método .build() terminal,
// produzindo um objeto final IMUTÁVEL
public final class Pedido {
    private final String cliente;
    private final String produto;
    private final int quantidade;

    // Construtor privado: só o Builder pode criar um Pedido
    private Pedido(Builder builder) {
        this.cliente = builder.cliente;
        this.produto = builder.produto;
        this.quantidade = builder.quantidade;
    }

    public static class Builder {
        private String cliente;
        private String produto;
        private int quantidade = 1; // valor padrão

        public Builder cliente(String cliente) {
            this.cliente = cliente;
            return this;
        }

        public Builder produto(String produto) {
            this.produto = produto;
            return this;
        }

        public Builder quantidade(int quantidade) {
            this.quantidade = quantidade;
            return this;
        }

        public Pedido build() {
            // Aqui é o lugar certo para validar antes de criar o objeto final
            if (cliente == null || produto == null) {
                throw new IllegalStateException("cliente e produto são obrigatórios");
            }
            return new Pedido(this);
        }
    }

    @Override
    public String toString() {
        return quantidade + "x " + produto + " para " + cliente;
    }
}

// Uso:
Pedido pedido = new Pedido.Builder()
        .cliente("Maria")
        .produto("Teclado mecânico")
        .quantidade(2)
        .build(); // .build() encerra a cadeia e entrega o objeto pronto

System.out.println(pedido); // 2x Teclado mecânico para Maria
```

---

#### 3. Armadilhas comuns

1. **Esquecer de retornar `this` (ou o novo objeto) em algum método da cadeia.** Se um dos métodos, por engano, for declarado `void` ou esquecer o `return`, o encadeamento quebra ali — erro de compilação "cannot find symbol" no próximo `.metodo()`, porque você estaria chamando algo em cima de `void`.
2. **Confundir chaining mutável com imutável e causar bug de estado compartilhado.** Se você pensa que `.mover()` do Exemplo 2 modifica o objeto original (como no Exemplo 1), vai reusar `original` esperando que ele tenha mudado — e vai se surpreender que continua igual. Sempre confira, na documentação/Javadoc, se o método retorna `this` ou um novo objeto.
3. **Cadeias longas demais dificultando debug.** Uma cadeia de 6-7 chamadas numa linha só é difícil de debugar — se der `NullPointerException` no meio, o stack trace aponta pra linha inteira, não pro método específico que falhou. Prática comum: quebrar a cadeia em múltiplas linhas (um método por linha), o que também facilita ler o "fluxo" da operação.
4. **Fluent Interface mutável não é thread-safe por padrão.** Como cada chamada modifica o mesmo objeto (`this`), se duas threads chamarem métodos da mesma cadeia no mesmo objeto ao mesmo tempo, o resultado é imprevisível. Isso raramente é problema em builders locais (criados e descartados dentro de um método), mas é uma armadilha real se a instância for compartilhada entre threads — mais um motivo para builders geralmente produzirem um objeto imutável ao final (`.build()`).

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie uma classe `CalculadoraFluente` com um campo `double valor` (iniciado em 0). Implemente os métodos `somar(double n)`, `subtrair(double n)` e `multiplicar(double n)`, cada um retornando `this`, e um método `resultado()` que retorna o `double` final. Use encadeamento para calcular `(0 + 10 - 3) * 2` numa única expressão.  
_Critério de pronto:_ a expressão encadeada retorna `14.0`, sem variáveis intermediárias.

**Exercício 2 (Médio)**  
Crie uma classe imutável `Retangulo` (com `largura` e `altura` `final`) com os métodos `redimensionar(double fatorLargura, double fatorAltura)` e `rotacionar90Graus()` (que troca largura por altura), onde **cada método retorna um novo `Retangulo`**, nunca modificando `this`. Escreva um teste no `main` que cria um retângulo original, aplica uma cadeia de 2-3 transformações, e imprime tanto o original quanto o resultado, provando que o original não mudou.  
_Critério de pronto:_ o objeto original impresso ao final mantém os valores iniciais intactos, mesmo depois de "usado" numa cadeia de chamadas.

**Exercício 3 (Difícil)**  
Implemente o padrão Builder completo (como no Exemplo 3) para uma classe `Pizza`, com campos obrigatórios (`tamanho`) e opcionais (`List<String> ingredientes`, começando vazia). O método `.build()` deve lançar `IllegalStateException` se `tamanho` não foi definido. Adicione um método `adicionarIngrediente(String ingrediente)` no Builder que também retorna `this`, permitindo chamá-lo múltiplas vezes na mesma cadeia para ir empilhando ingredientes.  
_Critério de pronto:_ é possível construir uma pizza chamando `.adicionarIngrediente(...)` 3 vezes seguidas na mesma cadeia antes do `.build()`; tentar `.build()` sem definir `tamanho` lança a exceção esperada.

**Exercício 4 (Desafio)**  
Crie uma classe `ValidadorDeSenha` com método estático `validar(String senha)` que retorna uma nova instância de `ValidadorDeSenha`. Nessa instância, implemente uma cadeia de métodos de validação (`tamanhoMinimo(int n)`, `temNumero()`, `temMaiuscula()`, `temCaractereEspecial()`), onde cada método **acumula** um resultado interno (ex: uma `List<String>` de erros encontrados) e sempre retorna `this`, independente de a validação passar ou falhar (ou seja, a cadeia nunca "quebra" no meio, mesmo se uma validação anterior já tiver falhado). Ao final, um método `.getErros()` retorna a lista de mensagens de erro acumuladas (vazia se a senha for válida).  
_Critério de pronto:_ encadear todos os métodos de validação numa senha fraca (ex: `"abc"`) retorna uma lista com múltiplos erros (tamanho insuficiente, sem número, sem maiúscula, sem caractere especial); uma senha forte retorna lista vazia.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class CalculadoraFluente {
    private double valor = 0;

    public CalculadoraFluente somar(double n) {
        this.valor += n;
        return this;
    }

    public CalculadoraFluente subtrair(double n) {
        this.valor -= n;
        return this;
    }

    public CalculadoraFluente multiplicar(double n) {
        this.valor *= n;
        return this;
    }

    public double resultado() {
        return this.valor;
    }
}

// Uso:
double res = new CalculadoraFluente()
        .somar(10)
        .subtrair(3)
        .multiplicar(2)
        .resultado(); // 14.0
```

_Raciocínio:_ `resultado()` é intencionalmente o único método que **não** retorna `this` — ele é o método "terminal" da cadeia, que converte do mundo fluente (`CalculadoraFluente`) de volta pro tipo de dado real (`double`) que o chamador precisa.

**Exercício 2**

java

```java
public final class Retangulo {
    private final double largura;
    private final double altura;

    public Retangulo(double largura, double altura) {
        this.largura = largura;
        this.altura = altura;
    }

    public Retangulo redimensionar(double fatorLargura, double fatorAltura) {
        return new Retangulo(largura * fatorLargura, altura * fatorAltura);
    }

    public Retangulo rotacionar90Graus() {
        return new Retangulo(altura, largura); // troca os dois
    }

    @Override
    public String toString() {
        return largura + "x" + altura;
    }
}

public class Main {
    public static void main(String[] args) {
        Retangulo original = new Retangulo(10, 5);

        Retangulo resultado = original
                .redimensionar(2, 1)
                .rotacionar90Graus();

        System.out.println("Original: " + original);   // Original: 10.0x5.0
        System.out.println("Resultado: " + resultado);  // Resultado: 5.0x20.0
    }
}
```

_Raciocínio:_ `redimensionar` calcula `10*2=20` e `5*1=5`, gerando um novo `Retangulo(20, 5)`; `rotacionar90Graus` então troca para `(5, 20)`. Em nenhum momento `this.largura` ou `this.altura` do objeto `original` são reatribuídos — são `final`, então fisicamente não poderiam ser, o que reforça (e garante em tempo de compilação) que essa classe só pode funcionar no estilo imutável.

**Exercício 3**

java

```java
import java.util.ArrayList;
import java.util.List;

public final class Pizza {
    private final String tamanho;
    private final List<String> ingredientes;

    private Pizza(Builder builder) {
        this.tamanho = builder.tamanho;
        this.ingredientes = builder.ingredientes;
    }

    public static class Builder {
        private String tamanho;
        private final List<String> ingredientes = new ArrayList<>();

        public Builder tamanho(String tamanho) {
            this.tamanho = tamanho;
            return this;
        }

        public Builder adicionarIngrediente(String ingrediente) {
            this.ingredientes.add(ingrediente); // acumula, não substitui
            return this;
        }

        public Pizza build() {
            if (tamanho == null) {
                throw new IllegalStateException("tamanho é obrigatório");
            }
            return new Pizza(this);
        }
    }

    @Override
    public String toString() {
        return "Pizza " + tamanho + " com " + ingredientes;
    }
}

// Uso válido:
Pizza pizza = new Pizza.Builder()
        .tamanho("Grande")
        .adicionarIngrediente("Queijo")
        .adicionarIngrediente("Tomate")
        .adicionarIngrediente("Manjericão")
        .build();
// Pizza Grande com [Queijo, Tomate, Manjericão]

// Uso que lança exceção:
Pizza semTamanho = new Pizza.Builder()
        .adicionarIngrediente("Queijo")
        .build(); // IllegalStateException: tamanho é obrigatório
```

_Raciocínio:_ `adicionarIngrediente` funciona porque ele **acumula** num `ArrayList` interno do próprio `Builder` (mutável, faz sentido ser mutável — é uma etapa transitória de construção) e sempre retorna `this`, permitindo chamar quantas vezes precisar antes do `.build()`. A validação fica dentro de `.build()` porque é o único lugar que sabemos, com certeza, que o usuário terminou de configurar e quer o objeto final — validar antes disso seria prematuro (a pessoa ainda pode estar no meio da cadeia).

**Exercício 4**

java

```java
import java.util.ArrayList;
import java.util.List;

public class ValidadorDeSenha {
    private final String senha;
    private final List<String> erros = new ArrayList<>();

    private ValidadorDeSenha(String senha) {
        this.senha = senha;
    }

    public static ValidadorDeSenha validar(String senha) {
        return new ValidadorDeSenha(senha);
    }

    public ValidadorDeSenha tamanhoMinimo(int n) {
        if (senha.length() < n) {
            erros.add("Senha deve ter no mínimo " + n + " caracteres");
        }
        return this; // sempre retorna this, independente do resultado da checagem
    }

    public ValidadorDeSenha temNumero() {
        if (senha.chars().noneMatch(Character::isDigit)) {
            erros.add("Senha deve conter ao menos um número");
        }
        return this;
    }

    public ValidadorDeSenha temMaiuscula() {
        if (senha.chars().noneMatch(Character::isUpperCase)) {
            erros.add("Senha deve conter ao menos uma letra maiúscula");
        }
        return this;
    }

    public ValidadorDeSenha temCaractereEspecial() {
        if (senha.chars().allMatch(Character::isLetterOrDigit)) {
            erros.add("Senha deve conter ao menos um caractere especial");
        }
        return this;
    }

    public List<String> getErros() {
        return erros;
    }
}

// Uso:
List<String> errosFraca = ValidadorDeSenha.validar("abc")
        .tamanhoMinimo(8)
        .temNumero()
        .temMaiuscula()
        .temCaractereEspecial()
        .getErros();
// ["Senha deve ter no mínimo 8 caracteres", "Senha deve conter ao menos um número",
//  "Senha deve conter ao menos uma letra maiúscula", "Senha deve conter ao menos um caractere especial"]

List<String> errosForte = ValidadorDeSenha.validar("Abc123!@")
        .tamanhoMinimo(8)
        .temNumero()
        .temMaiuscula()
        .temCaractereEspecial()
        .getErros();
// [] (lista vazia)
```

_Raciocínio:_ essa é uma variação interessante do padrão: diferente do Exercício 1 (que interrompe implicitamente se algo der errado, tipo uma exceção), aqui a decisão de design é que a cadeia **nunca interrompe** — cada validação roda independente das anteriores terem passado ou não, e só **acumula** o problema na lista `erros`. Isso é comum em validação de formulário, onde você quer mostrar **todos** os problemas de uma vez pro usuário, não só o primeiro que apareceu. O `factory method` estático `validar(String senha)` (em vez de um construtor público) também é um detalhe de design proposital: deixa a leitura mais natural (`ValidadorDeSenha.validar("abc123")` já expressa a intenção), e mantém o construtor `private`.