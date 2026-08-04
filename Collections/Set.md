### 1. Teoria

**`Set`** é uma interface do Collections Framework que representa uma coleção **sem elementos duplicados**. Ao contrário de `List`, ela não garante posição indexada (você não acessa "o elemento na posição 2" de um `Set`) e, dependendo da implementação, também não garante ordem de inserção.

As três implementações principais que você vai usar de verdade:

|Implementação|Ordem|Performance (add/contains/remove)|Permite `null`?|
|---|---|---|---|
|`HashSet`|Nenhuma garantida|O(1) médio|Sim (1 elemento)|
|`LinkedHashSet`|Ordem de inserção|O(1) médio (um pouco mais lento que HashSet)|Sim (1 elemento)|
|`TreeSet`|Ordenada (natural ou por `Comparator`)|O(log n)|Não (lança `NullPointerException`)|

**Como o `Set` garante "sem duplicados"?** Depende da implementação:

- `HashSet` usa **`hashCode()` + `equals()`** internamente (na prática, é um `HashMap` por baixo dos panos, onde cada elemento vira uma chave). Dois objetos são considerados "iguais" (e portanto duplicados) se `equals()` retorna `true` — mas o `hashCode()` também precisa bater, senão o objeto nem é comparado. É por isso que classes customizadas usadas em `HashSet` precisam sobrescrever `equals()` e `hashCode()` juntos, de forma consistente.
- `TreeSet` usa **`compareTo()`** (via `Comparable`) ou um `Comparator` passado no construtor — se `compareTo()` retorna `0` para dois elementos, o segundo é tratado como duplicado e descartado, **mesmo que `equals()` diga que são diferentes**. Isso é uma pegadinha real (ver Armadilhas).

**Não confunda com `List`:** `List` permite duplicados e é indexada; `Set` não permite duplicados e não é indexada (exceto `TreeSet`, que tem ordem, mas ainda não tem acesso por índice como `get(i)`).

**Onde aparece no dia a dia de backend:** `Set` é comum quando você precisa garantir unicidade — por exemplo, coletar IDs únicos de um resultado de query, representar roles/permissions de um usuário (`Set<Role>`), ou fazer operações de conjunto (interseção, união) entre dois grupos de dados. Em entidades JPA, é comum ver `Set<T>` em relacionamentos `@OneToMany`/`@ManyToMany` justamente para evitar duplicação de registros relacionados.

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class SetExample {
    public static void main(String[] args) {
        // HashSet: sem ordem garantida, mais rápido
        Set<String> hashSet = new HashSet<>();
        hashSet.add("banana");
        hashSet.add("maçã");
        hashSet.add("uva");
        hashSet.add("banana"); // duplicado - será ignorado silenciosamente

        System.out.println("HashSet: " + hashSet);
        System.out.println("Tamanho: " + hashSet.size()); // 3, não 4

        // LinkedHashSet: mantém ordem de inserção
        Set<String> linkedHashSet = new LinkedHashSet<>();
        linkedHashSet.add("banana");
        linkedHashSet.add("maçã");
        linkedHashSet.add("uva");
        System.out.println("LinkedHashSet: " + linkedHashSet); // sempre [banana, maçã, uva]

        // TreeSet: ordenado (ordem natural de String = alfabética)
        Set<String> treeSet = new TreeSet<>();
        treeSet.add("banana");
        treeSet.add("maçã");
        treeSet.add("uva");
        System.out.println("TreeSet: " + treeSet); // [banana, maçã, uva] - alfabético

        // Verificação de existência - operação clássica de Set, O(1) no HashSet
        boolean contemMaca = hashSet.contains("maçã");
        System.out.println("Contém maçã? " + contemMaca);

        // Operações de conjunto: interseção
        Set<String> grupoA = new HashSet<>(Set.of("java", "python", "go"));
        Set<String> grupoB = new HashSet<>(Set.of("python", "go", "rust"));

        Set<String> intersecao = new HashSet<>(grupoA);
        intersecao.retainAll(grupoB); // mantém só o que existe nos dois
        System.out.println("Interseção: " + intersecao); // [python, go] (ordem pode variar)

        // Operações de conjunto: união
        Set<String> uniao = new HashSet<>(grupoA);
        uniao.addAll(grupoB);
        System.out.println("União: " + uniao); // [java, python, go, rust] (ordem pode variar)
    }
}
```

---

### 3. Armadilhas comuns

1. **Usar objeto customizado em `HashSet` sem sobrescrever `equals()` e `hashCode()`** — sem isso, o `HashSet` usa a implementação padrão de `Object` (compara referência de memória), então dois objetos com os "mesmos dados" são tratados como diferentes, e duplicados passam despercebidos.
2. **`TreeSet` descarta elementos onde `compareTo()` retorna 0, mesmo que `equals()` diga que são diferentes** — se você define um `Comparator` que compara só por um campo (ex: idade), dois objetos com idades iguais mas nomes diferentes serão tratados como "duplicados" pelo `TreeSet` e um deles será silenciosamente descartado. Isso é uma fonte real de bugs difíceis de rastrear.
3. **Esperar ordem de inserção de um `HashSet`** — a ordem de iteração de um `HashSet` depende do hash interno, não da ordem em que você inseriu. Se ordem importa, use `LinkedHashSet`.
4. **Tentar usar `.get(index)` em `Set`** — essa operação não existe na interface `Set` (nem em `TreeSet`), porque `Set` não é indexado. Para percorrer, use `for-each` ou `Iterator`.

---

### 4. Exercícios práticos

**1. Fácil**  
Crie um `HashSet<String>` e adicione os nomes `"Ana"`, `"Bruno"`, `"Ana"`, `"Carla"` (nessa ordem). Imprima o `Set` e o `.size()`. Critério de pronto: o tamanho impresso deve ser `3`, comprovando que o duplicado foi ignorado.

**2. Fácil/Médio**  
Dada a lista `List<Integer> numeros = List.of(4, 2, 7, 2, 9, 4, 1, 7);`, converta-a para um `Set` de forma que o resultado final saia **ordenado de forma crescente** e sem duplicados. Imprima o resultado. Critério de pronto: a saída deve ser exatamente `[1, 2, 4, 7, 9]`.

**3. Médio**  
Crie uma classe `Aluno` com os campos `nome` (String) e `matricula` (int). Sobrescreva `equals()` e `hashCode()` considerando **apenas o campo `matricula`** como critério de igualdade (dois alunos são "iguais" se têm a mesma matrícula, mesmo com nomes diferentes). Adicione a um `HashSet<Aluno>` os seguintes alunos: `("João", 100)`, `("Maria", 200)`, `("João Pedro", 100)`. Imprima o tamanho do Set e explique o resultado. Critério de pronto: o tamanho deve ser `2`.

**4. Difícil/Desafio**  
Você tem dois grupos de emails cadastrados em dois sistemas diferentes: `Set<String> sistemaA` e `Set<String> sistemaB` (invente pelo menos 5 emails em cada, com alguma sobreposição proposital). Sem usar `retainAll`, `addAll` ou `removeAll` prontos, implemente manualmente (com loop e `Set` auxiliar) os métodos: `Set<String> interseccaoManual(Set<String> a, Set<String> b)` e `Set<String> diferencaManual(Set<String> a, Set<String> b)` (elementos que estão em `a` mas não em `b`). Depois, confira o resultado comparando com a versão usando `retainAll`/`removeAll` prontos — os dois devem bater. Critério de pronto: os dois métodos manuais devem produzir exatamente o mesmo resultado que as versões nativas do `Set`.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        Set<String> nomes = new HashSet<>();
        nomes.add("Ana");
        nomes.add("Bruno");
        nomes.add("Ana"); // duplicado, ignorado
        nomes.add("Carla");

        System.out.println(nomes);
        System.out.println("Tamanho: " + nomes.size()); // 3
    }
}
```

_Raciocínio:_ direto ao ponto — mostrar que `add()` de um elemento já presente simplesmente não faz nada (retorna `false` internamente, mas aqui a gente nem verifica o retorno), sem lançar exceção nem duplicar.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        List<Integer> numeros = List.of(4, 2, 7, 2, 9, 4, 1, 7);

        Set<Integer> ordenadoSemDuplicados = new TreeSet<>(numeros);

        System.out.println(ordenadoSemDuplicados); // [1, 2, 4, 7, 9]
    }
}
```

_Raciocínio:_ o construtor de `TreeSet` aceita uma `Collection` e já resolve duas coisas de uma vez: remove duplicados (comportamento de `Set`) e ordena (comportamento de `TreeSet`, usando a ordem natural de `Integer`, que é numérica crescente). Não precisa de passo intermediário.

**Exercício 3**

java

```java
public class Aluno {
    private String nome;
    private int matricula;

    public Aluno(String nome, int matricula) {
        this.nome = nome;
        this.matricula = matricula;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Aluno aluno = (Aluno) o;
        return matricula == aluno.matricula; // só matrícula importa
    }

    @Override
    public int hashCode() {
        return Objects.hash(matricula); // consistente com equals - só matrícula
    }
}

public class Exercicio3 {
    public static void main(String[] args) {
        Set<Aluno> alunos = new HashSet<>();
        alunos.add(new Aluno("João", 100));
        alunos.add(new Aluno("Maria", 200));
        alunos.add(new Aluno("João Pedro", 100)); // mesma matrícula do primeiro

        System.out.println("Tamanho: " + alunos.size()); // 2
    }
}
```

_Raciocínio:_ o `HashSet` usa `hashCode()` primeiro pra decidir em qual "balde" (bucket) interno colocar o objeto, e só depois usa `equals()` pra confirmar se já existe algo igual naquele balde. Como `hashCode()` e `equals()` foram implementados considerando só `matricula`, o terceiro aluno (`"João Pedro", 100`) tem o mesmo hash e é considerado `equals()` ao primeiro (`"João", 100`) — mesmo tendo nome diferente. O `Set` descarta o segundo `add()` dessa matrícula. É crítico que `equals()` e `hashCode()` usem sempre os **mesmos campos**, senão o contrato quebra e o comportamento do `HashSet` fica imprevisível.

**Exercício 4**

java

```java
public class Exercicio4 {
    static Set<String> interseccaoManual(Set<String> a, Set<String> b) {
        Set<String> resultado = new HashSet<>();
        for (String email : a) {
            if (b.contains(email)) {
                resultado.add(email);
            }
        }
        return resultado;
    }

    static Set<String> diferencaManual(Set<String> a, Set<String> b) {
        Set<String> resultado = new HashSet<>();
        for (String email : a) {
            if (!b.contains(email)) {
                resultado.add(email);
            }
        }
        return resultado;
    }

    public static void main(String[] args) {
        Set<String> sistemaA = new HashSet<>(Set.of(
            "ana@mail.com", "bruno@mail.com", "carla@mail.com", "diego@mail.com", "eva@mail.com"
        ));
        Set<String> sistemaB = new HashSet<>(Set.of(
            "carla@mail.com", "diego@mail.com", "fabio@mail.com", "gustavo@mail.com", "helena@mail.com"
        ));

        // Versões manuais
        Set<String> intersecManual = interseccaoManual(sistemaA, sistemaB);
        Set<String> difManual = diferencaManual(sistemaA, sistemaB);

        // Versões nativas para comparação
        Set<String> intersecNativa = new HashSet<>(sistemaA);
        intersecNativa.retainAll(sistemaB);

        Set<String> difNativa = new HashSet<>(sistemaA);
        difNativa.removeAll(sistemaB);

        System.out.println("Interseção manual == nativa? " + intersecManual.equals(intersecNativa)); // true
        System.out.println("Diferença manual == nativa? " + difManual.equals(difNativa)); // true
    }
}
```

_Raciocínio:_ a ideia aqui é entender o que `retainAll`/`removeAll` fazem por baixo dos panos: `retainAll` percorre e mantém só o que existe em ambos (interseção); `removeAll` percorre e mantém só o que existe em `a` mas não em `b` (diferença). Vale notar: `Set.equals(Set)` compara **conteúdo**, não ordem nem referência — dois `HashSet`s são considerados iguais se têm exatamente os mesmos elementos, independente da ordem interna de cada um. É por isso que dá pra comparar diretamente com `.equals()` mesmo sem garantia de ordem.