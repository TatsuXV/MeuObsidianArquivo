### 1. Teoria

**`Collections`** (não confundir com a interface `Collection`, no singular, que é a raiz de `List`/`Set`/`Queue`) é uma **classe utilitária** do pacote `java.util`, cheia de métodos **estáticos** que operam sobre coleções já existentes. É conceitualmente parecida com a classe `Arrays` que você já usou (`Arrays.copyOf`, `Arrays.toString`) — só que voltada pro Collections Framework em vez de arrays puros.

Ela existe porque, historicamente, o Java optou por não colocar certas operações genéricas (ordenar, embaralhar, encontrar máximo/mínimo, sincronizar, tornar imutável) como métodos de instância dentro de cada implementação de `List`/`Set`/`Map` — em vez disso, centralizou tudo numa classe só, que funciona sobre a interface (`List`, `Collection`), não sobre a implementação específica.

Os métodos mais usados no dia a dia:

|Método|O que faz|
|---|---|
|`Collections.sort(list)`|Ordena a lista **no lugar** (in-place), usando ordem natural|
|`Collections.sort(list, comparator)`|Ordena usando um `Comparator` customizado|
|`Collections.reverse(list)`|Inverte a ordem dos elementos, no lugar|
|`Collections.shuffle(list)`|Embaralha aleatoriamente, no lugar|
|`Collections.max(collection)` / `Collections.min(collection)`|Retorna o maior/menor elemento|
|`Collections.frequency(collection, elemento)`|Conta quantas vezes um elemento aparece|
|`Collections.unmodifiableList(list)`|Retorna uma **view** imutável da lista (não uma cópia — mudanças na lista original ainda refletem na view)|
|`Collections.emptyList()` / `Collections.singletonList(e)`|Retornam listas imutáveis vazias / de um único elemento|
|`Collections.synchronizedList(list)`|Retorna uma versão thread-safe (sincronizada) da lista|

**Uma distinção importante que costuma confundir:** `Collections.unmodifiableList(lista)` **não copia** a lista — ela cria um "invólucro" (wrapper) que bloqueia modificações feitas _através dele_, mas se você ainda tiver uma referência à lista original mutável, alterá-la por ali **reflete** na view "imutável". Isso é bem diferente de `List.of(...)` (que vimos lá no início) ou de criar uma cópia de verdade com `new ArrayList<>(lista)`.

**Também vale diferenciar de `List.of()`/`Set.of()`/`Map.of()`:** esses são métodos de fábrica (factory methods) das próprias interfaces, introduzidos no Java 9, que criam coleções **genuinamente imutáveis desde a criação** (nem view de outra coisa) — e, diferente de `Collections.unmodifiableList`, não aceitam `null` como elemento. `Collections.unmodifiableXxx` é mais antigo e serve especificamente pra "proteger" uma coleção mutável já existente, envolvendo-a.

**Não confunda com Stream API:** muita coisa que hoje se resolveria com Stream (filtrar, mapear, coletar) tinha, historicamente, um jeito mais manual via `Collections` + loop. A Stream API (que vamos ver num bloco futuro) tornou vários usos de `Collections` menos necessários no dia a dia, mas os métodos de ordenação, min/max, imutabilidade e sincronização continuam bem usados.

**Onde aparece no dia a dia de backend:** `Collections.sort()` com `Comparator` customizado é comum quando você precisa ordenar uma lista de objetos por um critério de negócio (ex: ordenar pedidos por data). `Collections.unmodifiableList()` (ou `List.of()`, dependendo do caso) aparece quando você quer expor uma coleção interna de uma classe sem permitir que quem chama o método modifique o estado interno por acidente — um princípio de encapsulamento importante.

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class CollectionsExample {
    public static void main(String[] args) {
        List<Integer> numeros = new ArrayList<>(List.of(5, 2, 8, 1, 9, 3));

        // sort: ordena NO LUGAR (modifica a lista original), ordem natural
        Collections.sort(numeros);
        System.out.println("Ordenado: " + numeros); // [1, 2, 3, 5, 8, 9]

        // reverse: inverte no lugar
        Collections.reverse(numeros);
        System.out.println("Invertido: " + numeros); // [9, 8, 5, 3, 2, 1]

        // max e min: precisam de Comparable ou de um Comparator
        System.out.println("Máximo: " + Collections.max(numeros)); // 9
        System.out.println("Mínimo: " + Collections.min(numeros)); // 1

        // frequency: conta ocorrências
        List<String> letras = List.of("a", "b", "a", "c", "a", "b");
        System.out.println("Frequência de 'a': " + Collections.frequency(letras, "a")); // 3

        // sort com Comparator customizado - ordenando por critério de negócio
        List<String> nomes = new ArrayList<>(List.of("Bruno", "Ana", "Eduardo", "Carla"));
        Collections.sort(nomes, Comparator.comparing(String::length)); // ordena por tamanho da string
        System.out.println("Por tamanho: " + nomes); // [Ana, Bruno, Carla, Eduardo]

        // unmodifiableList: view imutável - CUIDADO, ainda é apoiada na lista original
        List<Integer> mutavel = new ArrayList<>(List.of(1, 2, 3));
        List<Integer> viewImutavel = Collections.unmodifiableList(mutavel);

        try {
            viewImutavel.add(4); // lança exceção - não pode modificar POR AQUI
        } catch (UnsupportedOperationException e) {
            System.out.println("Erro capturado: " + e);
        }

        // MAS a lista original ainda pode ser alterada, e isso reflete na "view"
        mutavel.add(4);
        System.out.println("View depois de alterar o original: " + viewImutavel); // [1, 2, 3, 4]

        // Comparando com List.of(): imutável de verdade, não é view de nada
        List<Integer> imutavelDeVerdade = List.of(1, 2, 3);
        try {
            imutavelDeVerdade.add(4);
        } catch (UnsupportedOperationException e) {
            System.out.println("List.of também bloqueia modificação: " + e);
        }
    }
}
```

---

### 3. Armadilhas comuns

1. **Achar que `Collections.unmodifiableList()` cria uma cópia protegida** — não cria. É uma _view_ sobre a lista original. Se você guarda a referência da lista mutável em algum lugar e continua alterando-a, essa alteração aparece na "view imutável" também. Pra proteção de verdade, combine com uma cópia: `Collections.unmodifiableList(new ArrayList<>(original))`.
2. **Chamar `Collections.sort()` numa lista imutável** (criada com `List.of()`, por exemplo) — lança `UnsupportedOperationException`, porque `sort` tenta modificar a lista no lugar, e listas de `List.of()` não permitem nenhuma modificação estrutural nem de conteúdo.
3. **Usar `Collections.max()`/`min()` numa coleção de objetos que não implementam `Comparable`, sem passar um `Comparator`** — lança `ClassCastException` em tempo de execução (não em compilação), porque o método tenta fazer cast pra `Comparable` internamente. Se sua classe não implementa `Comparable`, você precisa passar explicitamente um `Comparator` como segundo argumento.
4. **Esquecer que `sort`/`reverse`/`shuffle` modificam a lista original (in-place)**, quando às vezes você queria preservar a lista original intacta e trabalhar com uma cópia ordenada separada. Se precisar preservar o original, ordene uma cópia: `List<Integer> copiaOrdenada = new ArrayList<>(original); Collections.sort(copiaOrdenada);`.

---

### 4. Exercícios práticos

**1. Fácil**  
Dada `List<Integer> numeros = new ArrayList<>(List.of(34, 12, 89, 5, 67, 23));`, use métodos de `Collections` para: (a) encontrar e imprimir o maior e o menor valor sem ordenar a lista; (b) depois, ordenar a lista de forma decrescente (dica: existe um `Comparator` pronto pra "ordem natural invertida", ou você pode combinar `sort` com `reverse`). Critério de pronto: máximo `89`, mínimo `5`, lista final `[89, 67, 34, 23, 12, 5]`.

**2. Fácil/Médio**  
Dada `List<String> palavras = List.of("gato", "cão", "elefante", "rato", "boi", "hipopótamo");`, use `Collections.sort` com um `Comparator` customizado para ordenar as palavras da **mais curta para a mais longa**, e em caso de empate no tamanho, ordenar **alfabeticamente** (dica: `Comparator` tem um método pra encadear um critério de desempate: `.thenComparing(...)`). Critério de pronto: a saída deve ser `[boi, cão, gato, rato, elefante, hipopótamo]`.

**3. Médio**  
Crie uma classe `Produto` com campos `nome` (String) e `preco` (double). Crie uma `List<Produto>` com pelo menos 5 produtos com preços variados. Escreva um método `Produto produtoMaisCaro(List<Produto> produtos)` usando `Collections.max()` com um `Comparator` apropriado (a classe `Produto` **não** precisa implementar `Comparable` para isso funcionar — é justamente o ponto do exercício). Critério de pronto: o método deve retornar corretamente o produto de maior preço da lista, sem lançar `ClassCastException`.

**4. Difícil/Desafio**  
Implemente uma classe `Configuracoes` que guarda uma `List<String>` interna de "flags ativas" do sistema. Ela deve expor um método `List<String> getFlagsAtivas()` que retorna as flags para quem chamar, **mas sem permitir que quem recebeu o retorno consiga modificar a lista interna da classe** (nem adicionando, nem removendo, nem através de alterações posteriores feitas pela própria classe — ou seja, proteja contra os dois problemas: modificação direta pelo chamador E vazamento de referência mutável). Escreva um `main` que tenta quebrar essa proteção de duas formas diferentes (chamando `.add()` direto no retorno, e tentando guardar uma referência e mutá-la depois) e mostre que ambas falham ou não afetam o estado interno real. Critério de pronto: nenhuma tentativa externa de modificar a lista retornada deve conseguir alterar o estado interno de `Configuracoes`.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        List<Integer> numeros = new ArrayList<>(List.of(34, 12, 89, 5, 67, 23));

        System.out.println("Máximo: " + Collections.max(numeros)); // 89
        System.out.println("Mínimo: " + Collections.min(numeros)); // 5

        // Ordem natural crescente, depois invertida = decrescente
        Collections.sort(numeros);
        Collections.reverse(numeros);

        System.out.println("Decrescente: " + numeros); // [89, 67, 34, 23, 12, 5]
    }
}
```

_Raciocínio:_ `Collections.max`/`min` não exigem que a lista esteja ordenada — eles percorrem a coleção internamente comparando elemento a elemento, então funcionam igual antes ou depois do `sort`. Para a ordem decrescente, a combinação `sort` (ordem natural crescente) seguida de `reverse` é a forma mais direta e legível; alternativamente, `Collections.sort(numeros, Collections.reverseOrder())` faz o mesmo em uma linha só.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        List<String> palavras = new ArrayList<>(List.of("gato", "cão", "elefante", "rato", "boi", "hipopótamo"));

        Collections.sort(palavras,
            Comparator.comparing(String::length)   // critério principal: tamanho
                      .thenComparing(Comparator.naturalOrder())); // desempate: alfabético

        System.out.println(palavras); // [boi, cão, gato, rato, elefante, hipopótamo]
    }
}
```

_Raciocínio:_ `Comparator.comparing(String::length)` cria um comparador baseado no tamanho da string. `.thenComparing(...)` encadeia um **segundo critério**, aplicado só quando o primeiro empata — nesse caso, comparação alfabética natural. É assim que se resolve "ordenar por X, e em caso de empate, por Y" de forma legível, sem precisar escrever a lógica de comparação manualmente com `if/else` dentro de um `Comparator` customizado do zero.

**Exercício 3**

java

```java
public class Produto {
    String nome;
    double preco;

    Produto(String nome, double preco) {
        this.nome = nome;
        this.preco = preco;
    }

    @Override
    public String toString() {
        return nome + " (R$" + preco + ")";
    }
}

public class Exercicio3 {
    static Produto produtoMaisCaro(List<Produto> produtos) {
        return Collections.max(produtos, Comparator.comparingDouble(p -> p.preco));
    }

    public static void main(String[] args) {
        List<Produto> produtos = List.of(
            new Produto("Teclado", 150.0),
            new Produto("Monitor", 899.0),
            new Produto("Mouse", 79.0),
            new Produto("Cadeira", 650.0),
            new Produto("Notebook", 3200.0)
        );

        System.out.println(produtoMaisCaro(produtos)); // Notebook (R$3200.0)
    }
}
```

_Raciocínio:_ `Collections.max(collection, comparator)` é a versão que aceita um `Comparator` explícito como segundo argumento — exatamente pra cobrir o caso onde os objetos não têm uma "ordem natural" própria (não implementam `Comparable`). Aqui dizemos explicitamente "compare os produtos pelo campo `preco`" via `Comparator.comparingDouble`, sem precisar tocar na definição da classe `Produto`. Isso ilustra por que essa é a forma flexível de comparar: o mesmo `Produto` poderia ser comparado por preço em um lugar do código e por nome em outro, sem precisar de duas classes diferentes ou de escolher uma única "ordem natural" fixa na própria classe.

**Exercício 4**

java

```java
public class Configuracoes {
    private List<String> flagsAtivas = new ArrayList<>(List.of("modoDebug", "cacheAtivo", "logsDetalhados"));

    List<String> getFlagsAtivas() {
        // Cópia protegida + imutável: mesmo se alguém guardar essa referência
        // e tentar mutar depois, não afeta o estado interno de Configuracoes,
        // e nem sequer consegue mutar diretamente (UnsupportedOperationException)
        return Collections.unmodifiableList(new ArrayList<>(flagsAtivas));
    }

    public static void main(String[] args) {
        Configuracoes config = new Configuracoes();

        List<String> flags = config.getFlagsAtivas();

        // Tentativa 1: modificar direto o retorno
        try {
            flags.add("flagMaliciosa");
        } catch (UnsupportedOperationException e) {
            System.out.println("Tentativa 1 bloqueada: " + e);
        }

        // Tentativa 2: se o retorno FOSSE mutável, dava pra guardar e mutar depois -
        // mas como já é unmodifiable, nem essa segunda linha de defesa é necessária aqui.
        // Provamos que o estado interno real nunca mudou:
        System.out.println("Flags internas continuam: " + config.getFlagsAtivas());
    }
}
```

_Raciocínio:_ a proteção completa exige **duas camadas**: (1) `new ArrayList<>(flagsAtivas)` cria uma **cópia**, desconectando a lista retornada da lista interna real — isso resolve o problema de "vazamento de referência", onde só um `Collections.unmodifiableList(flagsAtivas)` sem cópia ainda deixaria a view refletir mudanças futuras feitas pela própria classe na lista original; (2) `Collections.unmodifiableList(...)` em volta dessa cópia bloqueia tentativas de modificação direta pelo chamador. Sem a camada (1), se `Configuracoes` chamasse `flagsAtivas.add(...)` internamente depois, isso apareceria na "view" que já tinha sido entregue pra fora — quebrando a expectativa de imutabilidade do ponto de vista de quem recebeu o retorno.