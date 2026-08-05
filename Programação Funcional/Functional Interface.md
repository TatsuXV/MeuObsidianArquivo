### 1. Teoria

**O que é uma Functional Interface?**

É uma interface Java que declara **exatamente um método abstrato**. Ela pode ter quantos métodos `default` e `static` você quiser — isso não conta pro limite. É por causa dessa restrição de "um método abstrato só" que o compilador consegue inferir, a partir de uma lambda `(a, b) -> a + b`, qual método essa lambda está implementando: só existe um candidato possível.

A anotação `@FunctionalInterface` é **opcional**, mas recomendada: ela não muda o comportamento em runtime, só faz o compilador **verificar em tempo de compilação** que a interface realmente tem só um método abstrato, e falha a build se alguém adicionar um segundo método abstrato por engano. É uma trava de segurança, não um requisito da linguagem.

**Detalhe que confunde iniciante:** métodos que a interface "herda" implicitamente de `Object` (como `equals`, `hashCode`, `toString`) **não contam** como métodos abstratos pro propósito dessa contagem. Então isto ainda é uma functional interface válida:

java

```java
@FunctionalInterface
interface Comparador<T> {
    int comparar(T a, T b);
    boolean equals(Object o); // não conta, Object já garante implementação
}
```

**Para que serve na prática (backend real):** você quase nunca vai _criar_ uma functional interface do zero no dia a dia — o pacote `java.util.function` já cobre praticamente todos os casos:

|Interface|Método abstrato|Uso típico|
|---|---|---|
|`Function<T, R>`|`R apply(T t)`|transformar um valor em outro|
|`BiFunction<T, U, R>`|`R apply(T t, U u)`|igual acima, mas com 2 entradas|
|`Predicate<T>`|`boolean test(T t)`|condição/filtro (ex: `list.stream().filter(...)`)|
|`Consumer<T>`|`void accept(T t)`|fazer algo com um valor sem retornar nada (ex: `forEach`)|
|`Supplier<T>`|`T get()`|fornecer um valor sob demanda (ex: valor padrão "lazy", `Optional.orElseGet`)|
|`UnaryOperator<T>`|`T apply(T t)`|`Function<T,T>` — entrada e saída do mesmo tipo|
|`BinaryOperator<T>`|`T apply(T t, T t2)`|`BiFunction<T,T,T>` — ex: usado em `reduce`|
|`Comparator<T>`|`int compare(T a, T b)`|ordenação (existe desde antes do Java 8, foi "retrofitada")|
|`Runnable`|`void run()`|ação sem entrada nem saída (também usada com Threads)|

Em Spring Boot isso aparece o tempo todo: `Predicate` em specifications do Spring Data JPA, `Function`/`Consumer` em beans de configuração, `Supplier` em `Optional.orElseGet(() -> buscarValorPadrao())`, `Comparator` pra ordenar listas de DTO antes de retornar numa resposta.

**Diferença com classe abstrata (pra não confundir):** uma classe abstrata pode ter estado (atributos), construtor, e múltiplos métodos abstratos. Uma functional interface não tem estado próprio e é restrita a um único método abstrato — ela existe _especificamente_ pra ser o "contrato" que uma lambda ou method reference implementa.

### 2. Exemplo de código comentado

java

```java
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.function.Supplier;

public class ExemploFunctionalInterface {

    // Functional interface customizada — às vezes você cria a sua
    // quando as prontas do java.util.function não expressam bem a intenção do domínio.
    @FunctionalInterface
    interface CalculadoraDeDesconto {
        double calcular(double precoOriginal);
    }

    public static void main(String[] args) {

        // Function<T, R>: recebe um Integer, devolve um Integer (dobro do valor)
        Function<Integer, Integer> dobro = x -> x * 2;
        System.out.println(dobro.apply(5)); // 10

        // Predicate<T>: recebe um valor, devolve boolean
        Predicate<Integer> ehPar = n -> n % 2 == 0;
        System.out.println(ehPar.test(7)); // false

        // Supplier<T>: não recebe nada, fornece um valor (útil pra "lazy evaluation")
        Supplier<String> mensagemPadrao = () -> "Usuário não encontrado";
        System.out.println(mensagemPadrao.get());

        // Interface funcional customizada, implementada com lambda
        CalculadoraDeDesconto descontoDez = preco -> preco * 0.9;
        System.out.println(descontoDez.calcular(200.0)); // 180.0

        // A mesma interface, agora com method reference (equivalente à lambda acima
        // quando o comportamento já existe como método estático em outra classe)
        CalculadoraDeDesconto descontoViaMetodo = ExemploFunctionalInterface::aplicarDescontoFixo;
        System.out.println(descontoViaMetodo.calcular(200.0));
    }

    static double aplicarDescontoFixo(double preco) {
        return preco - 15.0;
    }
}
```

### 3. Armadilhas comuns

- **Achar que `@FunctionalInterface` é obrigatória para a interface funcionar com lambda.** Não é — qualquer interface com um único método abstrato já é "funcional" por definição. A anotação só adiciona a verificação do compilador. Sem ela, se alguém adicionar um segundo método abstrato por acidente, o erro só aparece quando você tentar usar lambda ali, com uma mensagem menos clara.
- **Criar uma interface funcional customizada quando já existe uma pronta em `java.util.function` que resolve.** Isso é retrabalho e deixa o código menos familiar pra quem vai ler depois. Só crie a sua quando o nome do método genérico (`apply`, `test`, `accept`) prejudicar a legibilidade do domínio (como no exemplo `CalculadoraDeDesconto` acima, onde `calcular` é mais claro que `apply`).
- **Confundir `Function<T,R>` com `UnaryOperator<T>`.** `UnaryOperator<T>` é só um `Function<T,T>` (entrada e saída do mesmo tipo) — é comum ver gente declarando `Function<String,String>` quando `UnaryOperator<String>` já expressaria a intenção com mais clareza.
- **Esquecer que default/static methods não contam no limite de "um método abstrato".** É comum achar, por engano, que uma interface com um método abstrato + dois `default` é "inválida" como functional interface — não é, ela é perfeitamente válida.

### 4. Exercícios práticos

**1. Fácil**  
Crie uma `Predicate<String>` chamada `ehVazia` que retorna `true` se a string for nula ou vazia (`""`). Teste com pelo menos 3 valores diferentes usando `System.out.println`.

**2. Médio**  
Declare sua própria functional interface `Validador<T>` com um método `boolean validar(T valor)`. Use-a para validar se um `int` representa uma idade válida para cadastro (entre 18 e 120, inclusive). Implemente com lambda e teste com 3 valores (um abaixo, um dentro, um acima do intervalo).

**3. Difícil**  
Crie um método `Function<Integer, Integer> compor(Function<Integer, Integer> f, Function<Integer, Integer> g)` que retorna uma nova `Function` equivalente a aplicar `g` e depois `f` no resultado (ou seja, `f(g(x))`) — **sem usar** os métodos default `andThen`/`compose` que a própria interface `Function` já oferece (você vai implementar a composição manualmente, na "unha", pra entender o que esses métodos fazem por baixo). Critério de pronto: `compor(x -> x + 1, x -> x * 2).apply(3)` deve retornar `7`.

**4. Desafio**  
Implemente um `Supplier<Integer>` que funciona como um contador: cada chamada de `.get()` retorna um valor 1 maior que a chamada anterior, começando em 0 (primeira chamada retorna 0, segunda retorna 1, etc). Dica: pense em como uma lambda pode "lembrar" de estado entre chamadas sem usar uma variável de instância numa classe nomeada — e por que uma variável `local` comum não funcionaria aqui (isso te obriga a entender a regra de "effectively final" nas capturas de lambda, que será aprofundada com mais detalhe quando chegarmos em Lambda Expressions, mas o exercício já esbarra nela).

### 5. Gabarito comentado

**1. Fácil**

java

```java
Predicate<String> ehVazia = s -> s == null || s.isEmpty();

System.out.println(ehVazia.test(null));   // true
System.out.println(ehVazia.test(""));     // true
System.out.println(ehVazia.test("oi"));   // false
```

Raciocínio: `Predicate<T>` existe exatamente pra isso — expressar uma condição booleana reutilizável. Checar `null` antes de `isEmpty()` é essencial: se a ordem fosse invertida (`s.isEmpty() || s == null`), daria `NullPointerException` quando `s` fosse `null`, porque `isEmpty()` seria chamado antes do curto-circuito do `||` proteger.

**2. Médio**

java

```java
@FunctionalInterface
interface Validador<T> {
    boolean validar(T valor);
}

Validador<Integer> idadeValida = idade -> idade >= 18 && idade <= 120;

System.out.println(idadeValida.validar(15));  // false
System.out.println(idadeValida.validar(30));  // true
System.out.println(idadeValida.validar(150)); // false
```

Raciocínio: criar a interface própria aqui é um exercício didático — na prática, esse caso específico poderia perfeitamente usar `Predicate<Integer>` pronto do `java.util.function`, já que a semântica ("recebe um valor, devolve boolean") é idêntica. O ponto do exercício é você sentir na mão como o compilador liga a lambda ao método abstrato da sua própria interface.

**3. Difícil**

java

```java
static Function<Integer, Integer> compor(Function<Integer, Integer> f, Function<Integer, Integer> g) {
    return x -> f.apply(g.apply(x));
}

// Teste:
Function<Integer, Integer> somaUm = x -> x + 1;
Function<Integer, Integer> vezesDois = x -> x * 2;

System.out.println(compor(somaUm, vezesDois).apply(3)); // g primeiro: 3*2=6, depois f: 6+1=7
```

Raciocínio: a lambda retornada por `compor` **captura** `f` e `g` do escopo em que foi criada (isso é uma _closure_) e, quando finalmente é chamada com `.apply(x)`, executa `g` primeiro e passa o resultado pra `f`. Isso é literalmente o que o método default `andThen` faz por dentro: `f.andThen(g)` seria equivalente a escrever `x -> g.apply(f.apply(x))`, e `f.compose(g)` seria equivalente a `x -> f.apply(g.apply(x))` — que é exatamente o que você implementou aqui na mão. Trade-off: na prática você sempre vai preferir `f.compose(g)` pronto em vez de reescrever isso, é mais legível e menos propenso a erro de ordem (compose vs andThen é uma fonte comum de confusão, e ter implementado na mão ajuda a nunca mais errar qual é qual).

**4. Desafio**

java

```java
Supplier<Integer> contador = new Supplier<Integer>() {
    private int valor = -1;

    @Override
    public Integer get() {
        valor++;
        return valor;
    }
};

System.out.println(contador.get()); // 0
System.out.println(contador.get()); // 1
System.out.println(contador.get()); // 2
```

Raciocínio: aqui é onde a lambda "esbarra" numa limitação e o exercício força você a perceber isso. Uma lambda só pode capturar variáveis locais que sejam **effectively final** (nunca reatribuídas depois de inicializadas) — então não dá pra fazer `int valor = 0; Supplier<Integer> c = () -> valor++;`, porque isso reatribuiria `valor`, o que quebra a regra. Por isso a solução usa uma **classe anônima** implementando `Supplier<Integer>` em vez de lambda: classes anônimas têm seu próprio campo de instância (`valor`), que pode ser mutado livremente entre chamadas, diferente de uma variável capturada de fora. Isso é uma pista de coisa que vamos aprofundar em Lambda Expressions: a diferença entre "capturar por valor" (lambda) e "ter estado próprio" (classe anônima ou, alternativamente, um array de 1 posição ou um objeto mutável como _workaround_ pra lambda — mas isso é gambiarra, o jeito limpo pra estado mutável é classe anônima ou, melhor ainda, nem usar esse padrão e sim algo como `AtomicInteger`).