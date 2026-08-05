### 1. Teoria

**O que é Functional Composition?**

É o processo de **combinar funções menores em uma função maior**, onde a saída de uma vira a entrada da próxima. Você já _implementou_ isso na mão nos exercícios anteriores (`compor`, `encadear`) — esta sessão é sobre os métodos **default prontos** que `Function`, `Predicate` e `Consumer` já oferecem pra fazer exatamente isso, sem você reescrever o mecanismo.

**Os métodos de composição por interface:**

#### `Function<T, R>`

|Método|Assinatura|Ordem de execução|
|---|---|---|
|`andThen`|`f.andThen(g)`|executa **`f` primeiro**, depois passa o resultado pra `g` → `g(f(x))`|
|`compose`|`f.compose(g)`|executa **`g` primeiro**, depois passa o resultado pra `f` → `f(g(x))`|

Essa diferença de ordem é a fonte nº1 de confusão nesse tópico — veja a seção de armadilhas.

java

```java
Function<Integer, Integer> somaUm = x -> x + 1;
Function<Integer, Integer> vezesDois = x -> x * 2;

somaUm.andThen(vezesDois).apply(3);  // (3+1)*2 = 8   -> soma primeiro, depois multiplica
somaUm.compose(vezesDois).apply(3);  // (3*2)+1 = 7   -> multiplica primeiro, depois soma
```

#### `Predicate<T>`

|Método|Comportamento|
|---|---|
|`and`|`p1.and(p2)` — true só se **ambos** forem true (com curto-circuito: se `p1` for false, `p2` nem é avaliado)|
|`or`|`p1.or(p2)` — true se **pelo menos um** for true (com curto-circuito: se `p1` for true, `p2` nem é avaliado)|
|`negate`|`p1.negate()` — inverte o resultado (o que você já implementou na mão no exercício anterior)|

java

```java
Predicate<Integer> ehPositivo = n -> n > 0;
Predicate<Integer> ehPar = n -> n % 2 == 0;

Predicate<Integer> positivoEPar = ehPositivo.and(ehPar);
positivoEPar.test(4);  // true
positivoEPar.test(-4); // false (curto-circuito: ehPositivo já falha, ehPar nem roda)
```

#### `Consumer<T>`

|Método|Comportamento|
|---|---|
|`andThen`|`c1.andThen(c2)` — executa `c1`, depois `c2`, **ambos no mesmo valor de entrada** (não passa resultado adiante, porque `Consumer` não retorna nada)|

java

```java
Consumer<String> imprimir = System.out::println;
Consumer<String> imprimirTamanho = s -> System.out.println("Tamanho: " + s.length());

imprimir.andThen(imprimirTamanho).accept("Java");
// Java
// Tamanho: 4
```

**Diferença conceitual importante com `Function.andThen` vs `Consumer.andThen`:** no `Function`, `andThen` **encadeia** — a saída de um vira entrada do próximo. No `Consumer`, `andThen` **sequencia** — os dois recebem o _mesmo_ valor original, um depois do outro, porque `Consumer` não produz saída nenhuma pra passar adiante. É o mesmo nome de método, semânticas diferentes, porque a forma da interface é diferente.

**Onde isso aparece em backend real:** validação em cadeia é o exemplo clássico — `Predicate<Pedido> pedidoValido = temItens.and(clienteAtivo).and(enderecoValido)`, cada regra isolada e testável, combinadas de forma declarativa. Em pipelines de processamento (ex: transformar um DTO em entidade passando por normalização, validação e enriquecimento), `Function.andThen` encadeado evita um método gigante com tudo misturado.

### 2. Exemplo de código comentado

java

```java
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.function.Consumer;

public class ExemploComposicao {

    record Usuario(String nome, String email, int idade) {}

    public static void main(String[] args) {

        // --- Function: andThen vs compose ---
        Function<String, String> paraMinuscula = String::toLowerCase;
        Function<String, String> removerEspacos = s -> s.replace(" ", "");

        // andThen: primeiro paraMinuscula, DEPOIS removerEspacos
        Function<String, String> normalizarEmail = paraMinuscula.andThen(removerEspacos);
        System.out.println(normalizarEmail.apply("Ana ROCHA@Email.com"));
        // "anarocha@email.com"

        // --- Predicate: and / or / negate combinados ---
        Predicate<Usuario> temNomeValido = u -> u.nome() != null && !u.nome().isBlank();
        Predicate<Usuario> temEmailValido = u -> u.email() != null && u.email().contains("@");
        Predicate<Usuario> ehMaiorDeIdade = u -> u.idade() >= 18;

        Predicate<Usuario> usuarioValido = temNomeValido
                .and(temEmailValido)
                .and(ehMaiorDeIdade);

        Usuario u1 = new Usuario("Bruno", "bruno@email.com", 25);
        Usuario u2 = new Usuario("", "sememail", 15);

        System.out.println(usuarioValido.test(u1)); // true
        System.out.println(usuarioValido.test(u2)); // false

        // Predicate negado reaproveitando a composição já pronta
        Predicate<Usuario> usuarioInvalido = usuarioValido.negate();
        System.out.println(usuarioInvalido.test(u2)); // true

        // --- Consumer: andThen encadeando ações sobre o MESMO valor ---
        Consumer<Usuario> logar = u -> System.out.println("Processando: " + u.nome());
        Consumer<Usuario> validarEAvisar = u -> {
            if (!usuarioValido.test(u)) {
                System.out.println("ATENÇÃO: usuário inválido -> " + u.nome());
            }
        };

        Consumer<Usuario> pipelineDeProcessamento = logar.andThen(validarEAvisar);
        pipelineDeProcessamento.accept(u2);
        // Processando: 
        // ATENÇÃO: usuário inválido -> 
    }
}
```

### 3. Armadilhas comuns

- **Trocar `andThen` por `compose` sem perceber a inversão de ordem.** É o erro mais comum do tópico: `f.andThen(g)` executa `f` primeiro; `f.compose(g)` executa `g` primeiro. Mnemônico que ajuda: em `f.compose(g)`, leia como "`f` de `g`" — matematicamente é `f(g(x))`, igual à notação de composição de funções `f∘g` que você viu (ou vai ver) em matemática.
- **Achar que `Consumer.andThen` encadeia valor, igual `Function.andThen`.** Não encadeia — os dois `Consumer`s recebem o **mesmo** valor de entrada original. Se você precisa que a saída de uma ação alimente a próxima, `Consumer` é a interface errada; você precisa de `Function`.
- **Não aproveitar o curto-circuito de `and`/`or` e colocar a condição mais cara primeiro.** Se uma das condições envolve, por exemplo, uma consulta ao banco e a outra é uma checagem de campo em memória, coloque a checagem barata primeiro no `.and(...)` — se ela falhar, a condição cara nem é avaliada.
- **Empilhar `andThen`/`compose` em excesso até o código ficar ilegível numa linha só.** Composição funcional é poderosa, mas 5+ `.andThen()` encadeados numa expressão só vira tão difícil de ler quanto o código imperativo que ela deveria substituir. Nomeie passos intermediários com variáveis (como fizemos com `temNomeValido`, `temEmailValido` acima) em vez de compor tudo inline.

### 4. Exercícios práticos

**1. Fácil**  
Dadas `Function<Integer, Integer> somaCinco = x -> x + 5;` e `Function<Integer, Integer> elevarAoQuadrado = x -> x * x;`, escreva duas composições diferentes usando `andThen` e `compose` a partir das mesmas duas funções, aplique ambas em `x = 2`, e explique (por escrito) por que os resultados são diferentes.

**2. Médio**  
Modele validação de senha usando `Predicate<String>` compostos: crie `temTamanhoMinimo` (mínimo 8 caracteres), `temNumero` (contém pelo menos um dígito) e `temLetraMaiuscula` (contém pelo menos uma letra maiúscula) como Predicates separados, depois combine os três num único `Predicate<String> senhaForte` usando `.and()`. Teste com pelo menos 3 senhas diferentes (uma que falha em cada critério isoladamente, e uma que passa em todos).

**3. Difícil**  
Usando `Consumer<T>.andThen`, monte um "pipeline de auditoria" que, para um `record Pedido(String id, double valor)`, execute em sequência: (1) imprimir `"Auditando pedido " + id`, (2) se `valor > 1000`, imprimir um aviso de "pedido de alto valor", (3) imprimir uma linha de separação `"---"`. Monte isso combinando 3 `Consumer`s distintos com `andThen` (não escreva tudo numa lambda só) e explique, por escrito, por que `Consumer.andThen` é adequado aqui e `Function.andThen` não seria.

**4. Desafio**  
Combine tudo que vimos nos últimos 3 itens (Functional Interfaces, High Order Functions e Composition): escreva uma high order function `static <T> Function<T, T> aplicarSeValido(Predicate<T> condicao, Function<T, T> transformacao)` que retorna uma `Function<T, T>` — aplicando `transformacao` **somente se** `condicao.test(valor)` for verdadeira; caso contrário, retorna o valor original sem alteração. Teste com `Predicate<Integer> ehPar = n -> n % 2 == 0;` e `Function<Integer, Integer> dobrar = n -> n * 2;`, confirmando que números pares dobram e ímpares permanecem iguais.

### 5. Gabarito comentado

**1. Fácil**

java

```java
Function<Integer, Integer> somaCinco = x -> x + 5;
Function<Integer, Integer> elevarAoQuadrado = x -> x * x;

int resultadoAndThen = somaCinco.andThen(elevarAoQuadrado).apply(2);
// andThen: soma primeiro -> (2+5)=7, depois eleva ao quadrado -> 7*7=49
System.out.println(resultadoAndThen); // 49

int resultadoCompose = somaCinco.compose(elevarAoQuadrado).apply(2);
// compose: eleva ao quadrado primeiro -> (2*2)=4, depois soma -> 4+5=9
System.out.println(resultadoCompose); // 9
```

Raciocínio: os resultados diferem porque `andThen` executa a função "de fora" (`somaCinco`) primeiro, enquanto `compose` executa ela por último. `f.andThen(g)` é `g(f(x))`; `f.compose(g)` é `f(g(x))` — são literalmente ordens opostas de aplicação, então salvo casos onde as funções comutam matematicamente (o que não é o caso aqui, soma e quadrado não comutam), o resultado numérico será diferente.

**2. Médio**

java

```java
Predicate<String> temTamanhoMinimo = s -> s.length() >= 8;
Predicate<String> temNumero = s -> s.chars().anyMatch(Character::isDigit);
Predicate<String> temLetraMaiuscula = s -> s.chars().anyMatch(Character::isUpperCase);

Predicate<String> senhaForte = temTamanhoMinimo.and(temNumero).and(temLetraMaiuscula);

System.out.println(senhaForte.test("abc123"));       // false (curto: falha no tamanho)
System.out.println(senhaForte.test("abcdefgh"));      // false (tamanho ok, mas sem número/maiúscula)
System.out.println(senhaForte.test("Abcdefg1"));      // true
```

Raciocínio: cada `Predicate` isolado testa uma única regra de negócio, o que os torna reutilizáveis e testáveis individualmente (você poderia escrever um teste unitário só pra `temNumero`, por exemplo). `.and()` encadeado três vezes forma uma cadeia legível de regras, e o curto-circuito evita processar `.chars()` desnecessariamente quando o tamanho já falhou. `s.chars().anyMatch(Character::isDigit)` usa Stream API de forma antecipada aqui — é um bom preview do próximo item do bloco, mas o foco desse exercício é a composição dos Predicates, não a Stream em si.

**3. Difícil**

java

```java
record Pedido(String id, double valor) {}

Consumer<Pedido> logInicio = p -> System.out.println("Auditando pedido " + p.id());
Consumer<Pedido> avisoAltoValor = p -> {
    if (p.valor() > 1000) {
        System.out.println("Pedido de alto valor!");
    }
};
Consumer<Pedido> separador = p -> System.out.println("---");

Consumer<Pedido> pipelineAuditoria = logInicio.andThen(avisoAltoValor).andThen(separador);

pipelineAuditoria.accept(new Pedido("P001", 1500.0));
// Auditando pedido P001
// Pedido de alto valor!
// ---
```

Raciocínio: `Consumer.andThen` é adequado aqui porque cada etapa **não precisa do resultado da anterior** — todas operam sobre o mesmo `Pedido` original, só produzindo efeitos colaterais (impressão) em sequência. `Function.andThen` não serviria porque `Function` **transformaria** o `Pedido` a cada etapa (a saída de uma viraria entrada da próxima), o que não é a intenção aqui — você quer _observar_ o mesmo pedido três vezes, não transformá-lo progressivamente.

**4. Desafio**

java

```java
static <T> Function<T, T> aplicarSeValido(Predicate<T> condicao, Function<T, T> transformacao) {
    return valor -> condicao.test(valor) ? transformacao.apply(valor) : valor;
}

Predicate<Integer> ehPar = n -> n % 2 == 0;
Function<Integer, Integer> dobrar = n -> n * 2;

Function<Integer, Integer> dobrarSeForPar = aplicarSeValido(ehPar, dobrar);

System.out.println(dobrarSeForPar.apply(4)); // 8  (par -> dobra)
System.out.println(dobrarSeForPar.apply(5)); // 5  (ímpar -> mantém)
```

Raciocínio: essa função combina os três conceitos do bloco até aqui — `Predicate` e `Function` são os _contratos_ (Functional Interfaces), `aplicarSeValido` é a _high order function_ que orquestra os dois, e o operador ternário dentro da lambda retornada decide, em tempo de execução, se aplica a composição ou não. Esse padrão — transformação condicional encapsulada numa única `Function` — é útil quando você quer passar essa lógica adiante (ex: como argumento de outro método) sem expor um `if` solto no meio do código de quem chama.