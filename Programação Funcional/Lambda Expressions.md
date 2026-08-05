### 1. Teoria

Você já viu lambda de relance no Bloco 3 (OOP), como sintaxe. Essa sessão é a **aplicação de verdade**: como lambdas funcionam por baixo, o que elas podem e não podem capturar, e onde isso quebra na prática.

**O que é uma lambda expression, tecnicamente?**

É uma implementação anônima e compacta de uma **functional interface** (o que vimos na sessão anterior). O compilador olha o **tipo alvo** (target type) esperado — por exemplo, um parâmetro declarado como `Predicate<String>` — e usa isso pra saber qual método a lambda está implementando. A lambda em si não tem tipo próprio "solto"; ela só existe em contexto de um tipo alvo compatível.

java

```java
Predicate<String> p = s -> s.isEmpty();
//                    ^ isso só compila porque o compilador sabe, pelo tipo
//                      declarado à esquerda, que precisa implementar test(String)
```

**Sintaxe — as três formas equivalentes:**

java

```java
(String s) -> { return s.isEmpty(); }   // forma completa
(s) -> s.isEmpty();                     // tipo inferido, sem chaves (expressão única)
s -> s.isEmpty();                       // parênteses opcionais com 1 parâmetro só
```

Regra prática: se o corpo é uma única expressão que já é o valor de retorno, sem chaves e sem `return`. Se precisar de mais de uma instrução, chaves e `return` explícito (quando aplicável) voltam a ser obrigatórios.

**Closures — o ponto mais importante da sessão.**

Uma lambda pode **capturar** variáveis do escopo onde foi criada. Mas só pode capturar variáveis locais que sejam **effectively final** — ou seja, que nunca são reatribuídas depois de inicializadas, mesmo que você não tenha escrito a palavra-chave `final`.

java

```java
int base = 10;
Function<Integer, Integer> soma = x -> x + base; // ok, "base" nunca muda depois disso
```

java

```java
int contador = 0;
Runnable r = () -> contador++; // ERRO DE COMPILAÇÃO
// "contador" é reatribuído (contador++), então deixa de ser effectively final
```

**Por que essa regra existe?** Porque, por baixo, a lambda não guarda uma _referência_ à variável local — ela guarda uma **cópia do valor** no momento da captura (captura por valor, não por referência). Se a linguagem permitisse mutar essa variável depois, você teria duas cópias divergindo silenciosamente, o que é uma fonte clássica de bug em outras linguagens. Java evita esse problema inteiro proibindo a mutação.

**Isso é diferente de capturar atributos de instância ou variáveis estáticas** — esses _podem_ ser mutados livremente dentro de uma lambda, porque não são variáveis locais, são referenciados através do objeto/classe, não copiados:

java

```java
class Contador {
    private int valor = 0;
    Runnable incrementar = () -> valor++; // ok! "valor" é atributo, não variável local
}
```

**Method references — açúcar sintático pra lambda que só chama um método existente:**

|Tipo|Sintaxe|Equivalente em lambda|
|---|---|---|
|Método estático|`Classe::metodoEstatico`|`x -> Classe.metodoEstatico(x)`|
|Método de instância de objeto específico|`objeto::metodo`|`x -> objeto.metodo(x)`|
|Método de instância de um parâmetro arbitrário|`Classe::metodo`|`x -> x.metodo()`|
|Construtor|`Classe::new`|`x -> new Classe(x)`|

Use method reference quando a lambda **só encaminha** a chamada sem lógica extra — é mais legível e sinaliza intenção. Se tem qualquer lógica além do encaminhamento puro, fica lambda mesmo.

**Onde isso aparece em backend real:** toda vez que você escreve `.stream().map(Produto::getNome)` em vez de `.map(p -> p.getNome())`, é method reference. Em Spring, é comum ver em `.orElseThrow(EntidadeNaoEncontradaException::new)` — um construtor sendo referenciado como `Supplier`.

### 2. Exemplo de código comentado

java

```java
import java.util.function.Function;
import java.util.function.Supplier;
import java.util.List;

public class ExemploLambda {

    record Produto(String nome, double preco) {}

    public static void main(String[] args) {

        List<Produto> produtos = List.of(
            new Produto("Teclado", 250.0),
            new Produto("Mouse", 80.0)
        );

        // Lambda "cheia", com corpo em bloco — usada quando há mais de uma instrução
        Function<Produto, String> descricaoCompleta = produto -> {
            String nomeFormatado = produto.nome().toUpperCase();
            return nomeFormatado + " - R$ " + produto.preco();
        };
        produtos.forEach(p -> System.out.println(descricaoCompleta.apply(p)));

        // Method reference de método de instância de um parâmetro arbitrário:
        // equivalente a "p -> p.nome()"
        Function<Produto, String> pegarNome = Produto::nome;
        System.out.println(pegarNome.apply(produtos.get(0))); // "Teclado"

        // Closure capturando uma variável effectively final do escopo externo
        double taxaDesconto = 0.1; // nunca reatribuída depois -> effectively final
        Function<Produto, Double> aplicarDesconto =
            produto -> produto.preco() * (1 - taxaDesconto);
        System.out.println(aplicarDesconto.apply(produtos.get(0))); // 225.0

        // Method reference de construtor
        Supplier<StringBuilder> fabricaDeStringBuilder = StringBuilder::new;
        StringBuilder sb = fabricaDeStringBuilder.get();
        sb.append("gerado via method reference");
        System.out.println(sb);
    }
}
```

### 3. Armadilhas comuns

- **Tentar mutar uma variável local capturada** (o erro de `contador++` visto na teoria). A mensagem de erro do compilador é `variable ... is accessed from within a lambda expression, needs to be final or effectively final` — quando você ver isso, o problema quase sempre é esse.
- **"Contornar" a regra do effectively final com um array de 1 posição ou um objeto wrapper mutável** (`int[] contador = {0}; () -> contador[0]++;`). Isso _compila_, porque o array em si nunca é reatribuído (só seu conteúdo interno muda), mas é um workaround feio que geralmente indica que você deveria estar usando uma classe com estado próprio (ou `AtomicInteger`, se for concorrência) em vez de forçar isso numa lambda.
- **Usar lambda com efeito colateral em vez de retorno de valor**, especialmente dentro de streams — por exemplo, uma lambda dentro de `.map()` que modifica uma lista externa em vez de transformar e retornar o valor. Isso quebra o estilo funcional que a Stream API pressupõe e pode gerar comportamento inconsistente (streams paralelas, por exemplo, não garantem ordem de execução desses efeitos colaterais).
- **Confundir o "this" dentro de uma lambda com o "this" de uma classe anônima.** Dentro de uma lambda, `this` se refere à instância da classe **externa** onde a lambda foi escrita (porque lambda não cria um novo escopo de `this`). Dentro de uma classe anônima, `this` se refere à própria classe anônima. Isso pega gente que troca lambda por classe anônima (ou vice-versa) sem perceber essa diferença de comportamento.

### 4. Exercícios práticos

**1. Fácil**  
Dada uma `List<String> nomes = List.of("ana", "bruno", "carla")`, use `.forEach()` com method reference (não lambda) pra imprimir cada nome em maiúsculas. Dica: você vai precisar combinar `System.out::println` com alguma transformação — pense se dá pra fazer isso só com method reference ou se vai precisar de uma lambda em algum ponto, e explique por quê.

**2. Médio**  
Escreva um método `static Function<Integer, Integer> criarMultiplicador(int fator)` que retorna uma `Function` capaz de multiplicar qualquer número inteiro pelo `fator` recebido. Teste criando dois multiplicadores diferentes (`vezes3` e `vezes10`) e aplicando ambos no mesmo valor de entrada. Isso testa se você entendeu closure de parâmetro de método (que é sempre effectively final por padrão, a menos que você o reatribua no corpo do método).

**3. Difícil**  
Explique (em comentário no código, sem precisar rodar) por que o código abaixo **não compila**, e reescreva-o de uma forma que funcione, preservando a intenção original (contar quantos produtos de uma lista têm preço acima de um limite):

java

```java
List<Produto> produtos = ...;
int total = 0;
produtos.forEach(p -> {
    if (p.preco() > 100) {
        total++; // por que isso não compila?
    }
});
```

**4. Desafio**  
Crie uma `Function<Function<Integer,Integer>, Function<Integer,Integer>> aplicarDuasVezes` que recebe uma função `f` e retorna uma nova função equivalente a aplicar `f` duas vezes seguidas (`f(f(x))`). Teste com `f = x -> x + 3` e confirme que `aplicarDuasVezes.apply(f).apply(5)` retorna `11`. Esse exercício força você a lidar com lambda que retorna lambda — um padrão real em pipelines de configuração no Spring (ex: `Function` compostas em `RouterFunction` do Spring WebFlux).

### 5. Gabarito comentado

**1. Fácil**

java

```java
List<String> nomes = List.of("ana", "bruno", "carla");
nomes.forEach(n -> System.out.println(n.toUpperCase()));
```

Raciocínio: **não dá** pra fazer isso só com method reference puro, porque você precisa encadear duas operações (`toUpperCase()` e depois `println`), e method reference só encaminha _uma_ chamada direta. `String::toUpperCase` sozinho seria `Function<String,String>`, não `Consumer<String>` — não bate com o que `forEach` espera. Você teria que quebrar em duas etapas (`.map(String::toUpperCase).forEach(System.out::println)`, usando Stream), mas com `List` direto (sem stream) a lambda é o caminho mais direto. Isso ilustra bem o limite de method reference: ótimo pra encaminhamento simples, mas não substitui composição de lógica.

**2. Médio**

java

```java
static Function<Integer, Integer> criarMultiplicador(int fator) {
    return numero -> numero * fator;
}

Function<Integer, Integer> vezes3 = criarMultiplicador(3);
Function<Integer, Integer> vezes10 = criarMultiplicador(10);

System.out.println(vezes3.apply(5));  // 15
System.out.println(vezes10.apply(5)); // 50
```

Raciocínio: `fator` é parâmetro do método `criarMultiplicador` — cada chamada do método cria uma "cópia" independente desse parâmetro, e como ele nunca é reatribuído dentro do método, é effectively final por padrão. Cada `Function` retornada carrega consigo o valor de `fator` daquela chamada específica — é exatamente esse mecanismo (closure) que permite `vezes3` e `vezes10` terem comportamentos independentes mesmo vindo do mesmo método.

**3. Difícil**

java

```java
List<Produto> produtos = ...;

// PROBLEMA ORIGINAL: "total" é variável local reatribuída (total++) dentro da lambda,
// então deixa de ser effectively final -> erro de compilação.

// SOLUÇÃO: trocar o forEach manual por Stream API com filter + count,
// que resolve isso sem precisar de estado mutável compartilhado.
long total = produtos.stream()
        .filter(p -> p.preco() > 100)
        .count();
```

Raciocínio: a causa raiz é a mesma da armadilha comum #1. A solução correta **não é** contornar com array de 1 posição — é reconhecer que "contar quantos elementos satisfazem uma condição" é exatamente o problema que `filter().count()` resolve de forma funcional, sem precisar de nenhuma variável mutável externa. Isso antecipa o que veremos em Stream API: o estilo funcional evita esse tipo de problema estruturalmente, em vez de você ter que trabalhar em volta dele.

**4. Desafio**

java

```java
Function<Function<Integer,Integer>, Function<Integer,Integer>> aplicarDuasVezes =
    f -> (x -> f.apply(f.apply(x)));

Function<Integer,Integer> somaTres = x -> x + 3;
System.out.println(aplicarDuasVezes.apply(somaTres).apply(5)); // 11
```

Raciocínio: `aplicarDuasVezes` recebe uma função `f` e retorna **outra lambda** que captura `f` (que é effectively final — é o parâmetro `f` do lambda externo, nunca reatribuído). Quando você chama `.apply(somaTres)`, recebe de volta essa lambda interna já "fechada" sobre `somaTres`; ao chamar `.apply(5)` nela, ela executa `f.apply(f.apply(5))` = `f.apply(8)` = `11`. Trade-off de legibilidade: dá pra escrever sem os parênteses extras em volta de `x -> ...` (Java aceita `f -> x -> f.apply(f.apply(x))`), mas os parênteses deixam explícito, pra quem lê, que uma lambda está retornando outra — vale a verbosidade extra aqui.