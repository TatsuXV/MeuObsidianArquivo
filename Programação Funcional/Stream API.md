### 1. Teoria

**O que é a Stream API?**

É a API funcional para processar **coleções de dados** de forma declarativa: em vez de escrever `for` explícito dizendo _como_ iterar, você descreve _o que_ quer que aconteça com cada elemento, encadeando operações. Ela é, no fundo, a aplicação prática de tudo que vimos no bloco até agora — `map`, `filter`, `reduce` são high order functions prontas que recebem as functional interfaces (`Function`, `Predicate`, `BinaryOperator`) que você já sabe implementar com lambda.

**Ponto mais importante pra entender antes de qualquer coisa: Stream não é uma estrutura de dados.**

Um `Stream` não guarda elementos — ele é um **pipeline de computação** sobre uma fonte de dados (uma `List`, um array, etc.), que só executa quando necessário. Isso tem duas consequências diretas:

1. **Streams são de uso único.** Depois que você chama uma operação terminal, o stream é consumido — tentar usá-lo de novo lança `IllegalStateException: stream has already been operated upon or closed`.
2. **Operações intermediárias são lazy (preguiçosas).** Elas não fazem nada sozinhas — só descrevem uma etapa do pipeline. Nada é de fato executado até você chamar uma operação **terminal**.

java

```java
Stream<String> s = lista.stream().filter(nome -> nome.startsWith("A"));
// até aqui, NADA foi executado — filter só registrou a intenção

s.forEach(System.out::println); // SÓ AGORA o pipeline roda de fato, elemento por elemento
```

**Duas categorias de operação — essa distinção é a espinha dorsal do tópico:**

|Categoria|O que faz|Retorna|Exemplos|
|---|---|---|---|
|**Intermediária**|Transforma/filtra o pipeline|Outro `Stream` (permite encadear)|`filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `skip`, `peek`|
|**Terminal**|Consome o stream e produz um resultado final|Um valor concreto (não-Stream), ou `void`|`forEach`, `collect`, `reduce`, `count`, `anyMatch`, `findFirst`, `toArray`|

**As operações principais:**

- **`filter(Predicate<T>)`** — mantém só os elementos que satisfazem a condição.
- **`map(Function<T,R>)`** — transforma cada elemento de tipo `T` em tipo `R`.
- **`flatMap(Function<T, Stream<R>>)`** — igual `map`, mas "achata" streams aninhados. Use quando cada elemento produz uma **coleção**, e você quer um único stream "plano" no final, não um stream de coleções.
- **`sorted()` / `sorted(Comparator<T>)`** — ordena.
- **`distinct()`** — remove duplicados (usa `equals()`).
- **`limit(n)` / `skip(n)`** — pega os primeiros `n` / pula os primeiros `n`.
- **`reduce(...)`** — combina todos os elementos num único valor (soma, concatenação, etc). Tem duas formas: com valor inicial (`reduce(identity, BinaryOperator)`, retorna `T` direto) e sem valor inicial (`reduce(BinaryOperator)`, retorna `Optional<T>`, porque um stream vazio não tem o que reduzir).
- **`collect(Collector)`** — a operação terminal mais usada: converte o stream de volta pra uma estrutura concreta. `Collectors.toList()`, `Collectors.toSet()`, `Collectors.toMap(keyFn, valueFn)`, `Collectors.joining(", ")`, `Collectors.groupingBy(classificadorFn)`, `Collectors.partitioningBy(predicado)`, `Collectors.counting()`.

**`map` vs `flatMap` — a confusão mais comum do tópico:**

java

```java
List<List<Integer>> listaDeListas = List.of(List.of(1,2), List.of(3,4));

listaDeListas.stream().map(lista -> lista.size());
// Stream<Integer> -- um tamanho por sublista: [2, 2]

listaDeListas.stream().flatMap(List::stream);
// Stream<Integer> -- todos os elementos juntos, achatados: [1, 2, 3, 4]
```

**`Optional` aparece em vários terminais** (`findFirst`, `findAny`, `min`, `max`, `reduce` sem identity) exatamente porque um stream pode estar vazio, e a API força você a lidar com essa possibilidade explicitamente em vez de devolver `null`.

**Streams primitivos (`IntStream`, `LongStream`, `DoubleStream`):** evitam o overhead de autoboxing (`Integer` em vez de `int`) quando você trabalha com muitos números. `IntStream.range(0, 10)` gera `0..9`; `IntStream.rangeClosed(1, 10)` gera `1..10` (inclusive no fim). Um `Stream<Integer>` normal pode ser convertido pra `IntStream` com `.mapToInt(...)`, e vice-versa com `.boxed()`.

**Onde isso aparece em backend real:** é praticamente onipresente — transformar `List<Entity>` em `List<DTO>` (`.map(EntityMapper::toDto)`), filtrar resultados de repository antes de retornar numa resposta, agrupar pedidos por status (`Collectors.groupingBy(Pedido::status)`) pra montar um dashboard, ou simplesmente somar valores de uma lista sem loop manual.

### 2. Exemplo de código comentado

java

```java
import java.util.*;
import java.util.stream.*;

public class ExemploStreamAPI {

    record Funcionario(String nome, String departamento, double salario) {}

    public static void main(String[] args) {

        List<Funcionario> funcionarios = List.of(
            new Funcionario("Ana", "TI", 6000),
            new Funcionario("Bruno", "TI", 8500),
            new Funcionario("Carla", "RH", 4500),
            new Funcionario("Diego", "RH", 5200),
            new Funcionario("Elen", "TI", 4000)
        );

        // filter + map + collect: pipeline clássico
        List<String> nomesTIAcimaDeCinco = funcionarios.stream()
                .filter(f -> f.departamento().equals("TI"))   // mantém só TI
                .filter(f -> f.salario() > 5000)               // e salário > 5000
                .map(Funcionario::nome)                        // extrai só o nome
                .toList();                                     // atalho moderno pra Collectors.toList()
        System.out.println(nomesTIAcimaDeCinco); // [Ana, Bruno]

        // reduce: soma total da folha salarial
        double totalSalarios = funcionarios.stream()
                .mapToDouble(Funcionario::salario) // Stream<Funcionario> -> DoubleStream (evita boxing)
                .sum();                              // método terminal específico de streams primitivos
        System.out.println(totalSalarios); // 28200.0

        // groupingBy: agrupar funcionários por departamento
        Map<String, List<Funcionario>> porDepartamento = funcionarios.stream()
                .collect(Collectors.groupingBy(Funcionario::departamento));
        System.out.println(porDepartamento.get("RH").size()); // 2

        // groupingBy + downstream collector: contar quantos por departamento (sem trazer os objetos inteiros)
        Map<String, Long> contagemPorDepartamento = funcionarios.stream()
                .collect(Collectors.groupingBy(Funcionario::departamento, Collectors.counting()));
        System.out.println(contagemPorDepartamento); // {RH=2, TI=3}

        // findFirst retorna Optional -- obriga a tratar o caso "não encontrado"
        Optional<Funcionario> maiorSalarioTI = funcionarios.stream()
                .filter(f -> f.departamento().equals("TI"))
                .max(Comparator.comparingDouble(Funcionario::salario));
        maiorSalarioTI.ifPresentOrElse(
            f -> System.out.println("Maior salário TI: " + f.nome()),
            () -> System.out.println("Nenhum funcionário de TI encontrado")
        );

        // flatMap: exemplo com listas aninhadas
        List<List<String>> times = List.of(List.of("Ana", "Bruno"), List.of("Carla"));
        List<String> todosOsNomes = times.stream()
                .flatMap(List::stream) // achata List<List<String>> em Stream<String>
                .toList();
        System.out.println(todosOsNomes); // [Ana, Bruno, Carla]
    }
}
```

### 3. Armadilhas comuns

- **Reutilizar um stream depois de uma operação terminal.** Uma vez consumido, ele acabou. Se precisar reprocessar a mesma fonte, chame `.stream()` de novo a partir da coleção original.

java

```java
  Stream<Integer> s = List.of(1,2,3).stream();
  s.forEach(System.out::println);
  s.count(); // IllegalStateException: stream has already been operated upon or closed
```

- **Achar que operações intermediárias já executam algo.** Um `.map(x -> { System.out.println("processando"); return x; })` sem operação terminal depois **não imprime nada** — o pipeline nunca roda. Isso confunde quem espera comportamento "imediato", como em loops tradicionais.
- **Usar `map`/`forEach` para causar efeito colateral e mutar estado externo**, em vez de usar `collect` pra construir o resultado. Isso já apareceu na sessão de Lambda (Exercício 3), mas na Stream API a tentação é ainda maior:

java

```java
  // Ruim: usa stream como se fosse for-each imperativo com mutação externa
  List<String> nomes = new ArrayList<>();
  funcionarios.stream().forEach(f -> nomes.add(f.nome()));

  // Correto: deixa o collect construir o resultado
  List<String> nomes2 = funcionarios.stream().map(Funcionario::nome).toList();
```

- **Confundir `map` com `flatMap`** quando a função de transformação já retorna uma coleção/stream — resultando em `Stream<Stream<X>>` ou `Stream<List<X>>` em vez do stream achatado esperado.
- **Usar `parallelStream()` por padrão, achando que é sempre mais rápido.** Para coleções pequenas, o overhead de gerenciar paralelismo geralmente é maior que o ganho. Além disso, operações com efeito colateral ou dependentes de ordem (`forEach` mutando estado externo, por exemplo) podem produzir resultado incorreto ou não-determinístico em paralelo. Regra prática: só considere `parallelStream()` com volume de dados grande o suficiente pra justificar, e meça antes de assumir que ajudou.
- **Chamar `.get()` num `Optional` retornado por `findFirst()`/`max()`/etc sem checar `.isPresent()` antes**, arriscando `NoSuchElementException` quando o stream estiver vazio. Prefira `.orElse(...)`, `.orElseGet(...)` ou `.ifPresentOrElse(...)`, como no exemplo acima.

### 4. Exercícios práticos

**1. Fácil**  
Dada `List<Integer> numeros = List.of(3, 7, 2, 9, 4, 1, 8)`, use Stream API para: (a) filtrar só os números pares, (b) ordená-los, (c) coletar em uma nova `List<Integer>`. Deve ser uma única cadeia de operações encadeadas, sem loop manual.

**2. Médio**  
Dada a lista de `Funcionario` do exemplo acima (`nome`, `departamento`, `salario`), use Stream API para calcular a **média salarial por departamento**, retornando um `Map<String, Double>`. Dica: existe um `Collector` pronto especificamente pra média — pesquise `Collectors.averagingDouble` (ou confirme o nome exato antes de usar, seguindo a regra de precisão desta trilha).

**3. Difícil**  
Dada `List<String> frases = List.of("java e poderoso", "stream simplifica codigo", "pratica leva a fluencia")`, use `flatMap` para produzir uma única `List<String>` com **todas as palavras únicas** de todas as frases (sem repetição), ordenadas alfabeticamente. Critério de pronto: o resultado não deve ter palavras duplicadas mesmo que apareçam em frases diferentes.

**4. Desafio**  
Dada a lista de `Funcionario`, use `Collectors.partitioningBy` para dividir os funcionários em dois grupos: os que ganham acima da média geral da empresa e os que ganham na média ou abaixo. O resultado deve ser um `Map<Boolean, List<Funcionario>>`. Você vai precisar calcular a média geral **antes** de particionar (não dá pra fazer isso numa única passada de stream sem reprocessar, já que a condição de cada elemento depende de um agregado de todos — pense em por que isso exige duas operações terminais separadas, não uma só).

### 5. Gabarito comentado

**1. Fácil**

java

```java
List<Integer> numeros = List.of(3, 7, 2, 9, 4, 1, 8);

List<Integer> paresOrdenados = numeros.stream()
        .filter(n -> n % 2 == 0)
        .sorted()
        .toList();

System.out.println(paresOrdenados); // [2, 4, 8]
```

Raciocínio: `filter` é intermediária e lazy, `sorted()` também é intermediária (só ordena quando o pipeline for de fato executado), e `toList()` é a operação terminal que dispara tudo e materializa o resultado. A ordem das operações importa pra legibilidade (filtrar antes de ordenar processa menos elementos no `sorted`), mas não muda o resultado final neste caso.

**2. Médio**

java

```java
Map<String, Double> mediaPorDepartamento = funcionarios.stream()
        .collect(Collectors.groupingBy(
                Funcionario::departamento,
                Collectors.averagingDouble(Funcionario::salario)
        ));

System.out.println(mediaPorDepartamento);
// {RH=4850.0, TI=6166.666666666667}
```

Raciocínio: esse é o padrão "collector composto" — `groupingBy` recebe um segundo argumento, chamado _downstream collector_, que diz o que fazer com cada grupo depois de formado. Sem o segundo argumento, `groupingBy` te devolveria `Map<String, List<Funcionario>>` (como no exemplo da Teoria); com `Collectors.averagingDouble` como downstream, cada grupo já sai processado direto como a média, sem você precisar de um segundo `.stream()` manual em cima do resultado do primeiro `groupingBy`.

**3. Difícil**

java

```java
List<String> frases = List.of(
    "java e poderoso",
    "stream simplifica codigo",
    "pratica leva a fluencia"
);

List<String> palavrasUnicas = frases.stream()
        .flatMap(frase -> Arrays.stream(frase.split(" "))) // cada frase vira Stream<String> de palavras, achatado
        .distinct()
        .sorted()
        .toList();

System.out.println(palavrasUnicas);
// [a, codigo, e, fluencia, java, leva, poderoso, pratica, simplifica, stream]
```

Raciocínio: `frase.split(" ")` devolve um array de `String`; `Arrays.stream(...)` converte esse array num `Stream<String>`. Como `flatMap` espera que a função de transformação retorne um `Stream`, cada frase vira um mini-stream de palavras, e o `flatMap` os achata todos num único `Stream<String>` — se fosse `map` em vez de `flatMap`, o resultado seria um `Stream<Stream<String>>` (ou você precisaria de outro passo pra achatar manualmente). `distinct()` remove repetições (usa `equals()` de `String`), e `sorted()` ordena alfabeticamente por último.

**4. Desafio**

java

```java
double mediaGeral = funcionarios.stream()
        .mapToDouble(Funcionario::salario)
        .average()          // retorna OptionalDouble, não Optional<Double>
        .orElse(0.0);        // trata o caso de lista vazia

Map<Boolean, List<Funcionario>> particionados = funcionarios.stream()
        .collect(Collectors.partitioningBy(f -> f.salario() > mediaGeral));

System.out.println(particionados.get(true));  // acima da média
System.out.println(particionados.get(false)); // na média ou abaixo
```

Raciocínio: isso **exige duas operações terminais separadas** porque `mediaGeral` é um agregado que depende de **todos** os elementos, mas o `Predicate` do `partitioningBy` avalia **um elemento por vez**. Não existe como calcular "a média de tudo" e "comparar cada item com essa média" numa única passada de stream, porque quando o pipeline chega no primeiro elemento pra decidir a partição, ainda não viu os outros — a média só existe depois que o stream inteiro já foi consumido no primeiro `.average()`. É por isso que o stream original é percorrido duas vezes aqui (uma para `average()`, outra para `partitioningBy`), cada vez chamando `.stream()` de novo a partir de `funcionarios` (a lista original, não um stream reaproveitado — reforçando a regra de "stream de uso único" vista na Teoria). Note também `OptionalDouble` em vez de `Optional<Double>`: é a versão especializada de `Optional` usada pelos streams primitivos (`IntStream`, `DoubleStream`, etc.), evitando boxing.