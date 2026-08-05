### 1. Teoria

**O que é uma High Order Function (função de alta ordem)?**

É uma função que faz pelo menos uma das duas coisas:

1. **Recebe outra função como parâmetro**, ou
2. **Retorna uma função como resultado**.

Você já usou isso nas duas sessões anteriores sem o rótulo formal — `criarMultiplicador` (Exercício 2 de Lambda) e `aplicarDuasVezes` (Exercício 4) são, por definição, high order functions: o primeiro _retorna_ uma função, o segundo _recebe e retorna_ uma função.

**Importante desfazer uma confusão comum:** "high order function" não é um tipo de dado nem uma interface — é uma **categoria/classificação** de função, baseada no que ela faz com outras funções. Em Java, isso se manifesta usando os tipos de `java.util.function` (que vimos no item 1) como tipo de parâmetro ou tipo de retorno. Não existe sintaxe especial pra "declarar uma high order function" — você só declara um método normal cuja assinatura usa `Function`, `Predicate`, `Consumer`, etc.

java

```java
// Isso É uma high order function: recebe uma Function como parâmetro
static int aplicarERetornar(int valor, Function<Integer, Integer> operacao) {
    return operacao.apply(valor);
}
```

java

```java
// Isso NÃO é: apenas usa lógica interna, não recebe nem retorna função
static int dobrar(int valor) {
    return valor * 2;
}
```

**Diferença com Functional Interface e Lambda (pra situar no que já vimos):**

- **Functional Interface** = o _contrato_ (o tipo) que define o formato da função.
- **Lambda / method reference** = a _implementação concreta_ desse contrato.
- **High Order Function** = um _método que trata funções como valores_ — recebendo-as como argumento ou devolvendo-as como resultado.

São três conceitos que se encaixam: você usa lambdas (implementação) de functional interfaces (contrato) _dentro_ de high order functions (o método que orquestra tudo isso).

**Por que isso importa em backend real:** é o mecanismo por trás de praticamente toda a Stream API (`map`, `filter`, `reduce` são todos high order functions — recebem uma função e a aplicam internamente) e de padrões de configuração no Spring, como builders funcionais e `RouterFunction` no WebFlux. Também é a base de **injeção de comportamento**: em vez de uma classe precisar de uma subclasse pra mudar de comportamento (herança), você passa a lógica variável como parâmetro função — geralmente resultando em código mais enxuto que o padrão _Strategy_ clássico da GoF implementado com interfaces + classes.

**Um padrão importante: "template" com buraco de comportamento.**

java

```java
// A "estrutura" do método é fixa (abre conexão, faz algo, fecha conexão),
// mas o "algo" no meio é injetado de fora via high order function.
static <T> T executarComConexao(Function<Conexao, T> operacao) {
    Conexao conexao = abrirConexao();
    try {
        return operacao.apply(conexao);
    } finally {
        conexao.fechar();
    }
}
```

Isso é exatamente o que o `JdbcTemplate` do Spring faz por dentro — você passa um `RowMapper` (uma functional interface) pra dizer _o que fazer_ com cada linha, e o template cuida de abrir/fechar conexão, tratar exceção, etc. Vamos ver isso com detalhe quando chegarmos em JDBC (Bloco 13), mas o mecanismo de fundo é exatamente High Order Function.

### 2. Exemplo de código comentado

java

```java
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.List;
import java.util.ArrayList;

public class ExemploHighOrderFunction {

    record Funcionario(String nome, double salario, String cargo) {}

    public static void main(String[] args) {

        List<Funcionario> funcionarios = List.of(
            new Funcionario("Ana", 5000, "Dev"),
            new Funcionario("Bruno", 8000, "Tech Lead"),
            new Funcionario("Carla", 4500, "Dev")
        );

        // High order function que RECEBE um Predicate como parâmetro:
        // filtra a lista de acordo com qualquer critério que você passar.
        List<Funcionario> devsAcimaDeCincoMil = filtrarPor(
            funcionarios,
            f -> f.cargo().equals("Dev") && f.salario() > 4800
        );
        System.out.println(devsAcimaDeCincoMil); // [] -- nenhum Dev passa dos dois critérios

        // High order function que RETORNA uma Function:
        // gera um "aumentador de salário" configurado com um percentual específico.
        Function<Funcionario, Double> aumentoDeDez = criarCalculadoraDeAumento(0.10);
        System.out.println(aumentoDeDez.apply(funcionarios.get(0))); // 5500.0

        // Combinando: passa a função gerada acima para outra high order function
        List<Double> novosSalarios = transformarTodos(funcionarios, aumentoDeDez);
        System.out.println(novosSalarios); // [5500.0, 8800.0, 4950.0]
    }

    // Recebe uma lista e UM PREDICATE (a condição é decidida por quem chama, não por este método)
    static <T> List<T> filtrarPor(List<T> lista, Predicate<T> condicao) {
        List<T> resultado = new ArrayList<>();
        for (T item : lista) {
            if (condicao.test(item)) {
                resultado.add(item);
            }
        }
        return resultado;
    }

    // RETORNA uma Function configurada com o percentual recebido (closure sobre "percentual")
    static Function<Funcionario, Double> criarCalculadoraDeAumento(double percentual) {
        return funcionario -> funcionario.salario() * (1 + percentual);
    }

    // Recebe lista + uma Function que descreve COMO transformar cada item
    static <T, R> List<R> transformarTodos(List<T> lista, Function<T, R> transformacao) {
        List<R> resultado = new ArrayList<>();
        for (T item : lista) {
            resultado.add(transformacao.apply(item));
        }
        return resultado;
    }
}
```

### 3. Armadilhas comuns

- **Reimplementar `filter`/`map`/`reduce` na mão (como fiz no exemplo acima, de propósito didático) em vez de usar Stream API.** O exemplo acima existe só pra você ver o mecanismo cru; na prática profissional, `filtrarPor` e `transformarTodos` seriam simplesmente `funcionarios.stream().filter(...)` e `.map(...)`. Vamos chegar em Stream API no próximo item do bloco — ela é, no fundo, uma coleção enorme de high order functions já prontas e otimizadas.
- **Passar lógica excessivamente complexa como lambda inline**, deixando a chamada do método ilegível (`lista.forEach(x -> { /* 15 linhas de lógica */ })`). Quando a lambda passa de poucas linhas, extraia pra um método nomeado e use method reference — o nome do método vira documentação.
- **Confundir "retornar uma função" com "executar a função e retornar o resultado dela".** Isso é um erro sutil de tipo, não de lógica:

java

```java
  static Function<Integer,Integer> criar(int fator) {
      return numero -> numero * fator; // retorna a FUNÇÃO (correto, se a intenção é essa)
  }
  static int criarErrado(int fator, int numero) {
      return numero * fator; // já executa e retorna o resultado, não é mais high order
  }
```

Isso parece óbvio isolado, mas em assinaturas mais longas (com generics, tipos aninhados) é fácil perder de vista qual dos dois você está de fato escrevendo.

- **Ignorar que high order functions genéricas (`<T, R>`) exigem que o compilador consiga inferir os tipos.** Se a inferência falhar (geralmente em cadeias muito compostas), o erro do compilador costuma ser confuso — nesses casos, declarar o tipo explicitamente na lambda (`(Funcionario f) -> ...` em vez de `f -> ...`) ajuda a debugar onde a inferência quebrou.

### 4. Exercícios práticos

**1. Fácil**  
Escreva uma high order function `static void executarNVezes(int n, Runnable acao)` que executa a `acao` recebida `n` vezes seguidas. Teste passando uma lambda que imprime `"Executando..."`.

**2. Médio**  
Escreva uma high order function `static <T> T obterOuPadrao(T valor, Predicate<T> condicaoValida, Supplier<T> valorPadrao)` que retorna `valor` se `condicaoValida.test(valor)` for verdadeiro, ou o resultado de `valorPadrao.get()` caso contrário. Teste com um `String` (ex: validar se não é vazia) e um `Integer` (ex: validar se é positivo), mostrando que o mesmo método genérico funciona pros dois tipos.

**3. Difícil**  
Escreva uma high order function `static <T> Predicate<T> negar(Predicate<T> predicado)` que recebe um `Predicate<T>` e retorna um novo `Predicate<T>` com a lógica invertida — **sem usar** o método default `.negate()` que `Predicate` já tem (é pra você implementar o mecanismo na mão, igual fizemos com composição de `Function` na sessão anterior). Teste criando `Predicate<Integer> ehPar = n -> n % 2 == 0;` e usando sua função `negar` pra obter o equivalente a "é ímpar".

**4. Desafio**  
Escreva uma high order function `static <T> Function<T, T> encadear(List<Function<T, T>> transformacoes)` que recebe uma **lista** de transformações e retorna uma única `Function` que aplica todas elas em sequência, na ordem da lista. Teste com uma lista de 3 `Function<Integer, Integer>` (ex: somar 1, depois multiplicar por 2, depois subtrair 3) e confirme o resultado aplicando em um valor de entrada calculado manualmente antes de rodar o código, pra comparar com a saída.

### 5. Gabarito comentado

**1. Fácil**

java

```java
static void executarNVezes(int n, Runnable acao) {
    for (int i = 0; i < n; i++) {
        acao.run();
    }
}

executarNVezes(3, () -> System.out.println("Executando..."));
// Executando...
// Executando...
// Executando...
```

Raciocínio: `Runnable` é a functional interface certa aqui porque a ação não recebe entrada nem produz saída — é só "faça essa coisa". `executarNVezes` é high order function porque recebe `acao` como parâmetro e a invoca internamente, sem saber (nem precisar saber) o que ela faz.

**2. Médio**

java

```java
static <T> T obterOuPadrao(T valor, Predicate<T> condicaoValida, Supplier<T> valorPadrao) {
    return condicaoValida.test(valor) ? valor : valorPadrao.get();
}

String nome = obterOuPadrao("", s -> !s.isEmpty(), () -> "Anônimo");
System.out.println(nome); // "Anônimo"

Integer idade = obterOuPadrao(25, n -> n > 0, () -> 0);
System.out.println(idade); // 25
```

Raciocínio: esse método combina **dois** parâmetros função de tipos diferentes (`Predicate<T>` e `Supplier<T>`), o que é comum em métodos utilitários reais. Repare que `valorPadrao` é um `Supplier`, não o valor padrão direto (`T`) — isso é proposital: se calcular o valor padrão fosse caro (ex: uma query no banco), você não ia querer pagar esse custo quando `valor` já for válido. É o mesmo raciocínio por trás de `Optional.orElseGet(Supplier)` versus `Optional.orElse(valor)`: `orElseGet` só executa o Supplier se realmente precisar, `orElse` sempre calcula o argumento, mesmo quando não vai usar.

**3. Difícil**

java

```java
static <T> Predicate<T> negar(Predicate<T> predicado) {
    return valor -> !predicado.test(valor);
}

Predicate<Integer> ehPar = n -> n % 2 == 0;
Predicate<Integer> ehImpar = negar(ehPar);

System.out.println(ehImpar.test(7)); // true
System.out.println(ehImpar.test(4)); // false
```

Raciocínio: igual fizemos com `compor` para `Function` na sessão de Lambda, aqui a lambda retornada captura `predicado` (effectively final, é o parâmetro do método) e, quando chamada, inverte o resultado do `.test()` original com `!`. Isso é literalmente o que `.negate()` já faz internamente — `predicado.negate()` é equivalente a `negar(predicado)` que você acabou de escrever. Trade-off: use sempre `.negate()` pronto na prática (é padrão, testado, e sinaliza intenção imediatamente pra quem lê); o valor deste exercício é só entender que não tem mágica nenhuma por trás do método default.

**4. Desafio**

java

```java
static <T> Function<T, T> encadear(List<Function<T, T>> transformacoes) {
    return valor -> {
        T resultado = valor;
        for (Function<T, T> transformacao : transformacoes) {
            resultado = transformacao.apply(resultado);
        }
        return resultado;
    };
}

List<Function<Integer, Integer>> passos = List.of(
    x -> x + 1,
    x -> x * 2,
    x -> x - 3
);

Function<Integer, Integer> pipeline = encadear(passos);
System.out.println(pipeline.apply(5));
// Cálculo manual: 5 +1=6, 6*2=12, 12-3=9 -> esperado 9
```

Raciocínio: a lambda retornada por `encadear` captura a **lista inteira** `transformacoes` (a referência à lista é effectively final — não é reatribuída — mesmo que a lista em si seja imutável aqui via `List.of`). Dentro do corpo, um loop comum (não-funcional) vai aplicando cada `Function` sequencialmente sobre um acumulador local `resultado`. Isso é conceitualmente o "embrião" de `Function.andThen()` encadeado múltiplas vezes (`f1.andThen(f2).andThen(f3)`), só que generalizado pra uma lista de tamanho arbitrário definida em runtime, em vez de um número fixo de `.andThen()` escritos no código-fonte — útil quando o número de transformações não é conhecido em tempo de compilação (ex: veio de configuração).