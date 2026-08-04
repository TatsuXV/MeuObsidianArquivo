### 1. Teoria

**Array** é uma estrutura de dados nativa da linguagem (não é uma classe do Collections Framework). Tem **tamanho fixo definido na criação** e pode guardar tanto tipos primitivos (`int`, `double`, `boolean`...) quanto objetos.

**ArrayList** é uma classe do pacote `java.util`, parte do Collections Framework, que implementa a interface `List`. É **dinâmica** — cresce e encolhe conforme você adiciona/remove elementos — mas só guarda **objetos** (para primitivos, o Java faz autoboxing/unboxing automático por trás dos panos).

#### Diferenças principais

|Aspecto|Array|ArrayList|
|---|---|---|
|Tamanho|Fixo na criação|Dinâmico|
|Tipos|Primitivos e objetos|Só objetos (com autoboxing)|
|Tamanho atual|`array.length` (campo)|`list.size()` (método)|
|API|Mínima (poucos métodos, mais via classe utilitária `Arrays`)|Rica (`add`, `remove`, `contains`, `indexOf`...)|
|Performance|Levemente mais rápido, sem overhead de boxing|Overhead pequeno por causa do boxing/unboxing|
|Multidimensional|Natural (`int[][]`)|Mais verboso (`List<List<Integer>>`)|
|Type-safety|Covariante — permite erro em runtime (`ArrayStoreException`)|Genérico e invariante — erro pego em tempo de compilação|

**Não confunda:**

- `array.length` é um **campo público**, não um método — não tem parênteses.
- `list.size()` é um **método**, precisa dos parênteses.
- Um erro clássico de quem vem de outra linguagem: achar que dá pra "adicionar" um elemento num array. Não dá — array não cresce. Se você precisa de um array "maior", cria um array novo e copia o conteúdo (`Arrays.copyOf`).

**Onde isso aparece no dia a dia de backend:** você vai usar `ArrayList` (geralmente por trás da interface `List`) o tempo inteiro — retorno de métodos de service, resultado de queries no Spring Data JPA, DTOs, etc. Array puro aparece principalmente em cenários específicos: `String[] args` do `main`, matrizes, ou código sensível a performance onde o overhead de boxing importa.

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class ArrayVsArrayListExample {
    public static void main(String[] args) {
        // Array: tamanho fixo, definido na criação
        int[] numerosArray = new int[3]; // tamanho 3, não pode mudar depois
        numerosArray[0] = 10;
        numerosArray[1] = 20;
        numerosArray[2] = 30;
        // numerosArray[3] = 40; // ArrayIndexOutOfBoundsException - posição 3 não existe

        System.out.println("Tamanho do array: " + numerosArray.length); // campo, sem parênteses

        // ArrayList: tamanho dinâmico
        List<Integer> numerosList = new ArrayList<>();
        numerosList.add(10);
        numerosList.add(20);
        numerosList.add(30);
        numerosList.add(40); // sem problema, cresce sozinha

        System.out.println("Tamanho da lista: " + numerosList.size()); // método, com parênteses

        // Array guarda primitivo puro, sem boxing
        int[] primitivos = {1, 2, 3};

        // ArrayList guarda Integer - o "1" abaixo vira Integer.valueOf(1) automaticamente (autoboxing)
        List<Integer> comBoxing = new ArrayList<>();
        comBoxing.add(1);

        // Remover elemento: trivial em List, inexistente diretamente em array
        numerosList.remove(Integer.valueOf(20)); // remove o VALOR 20 (não o índice 20 - ver armadilha #2)
        System.out.println(numerosList); // [10, 30, 40]
    }
}
```

---

### 3. Armadilhas comuns

1. **Confundir `.length` com `.size()`** — array usa campo (`array.length`), List usa método (`list.size()`). Trocar um pelo outro é erro de compilação garantido.
2. **`list.remove(20)` vs `list.remove(Integer.valueOf(20))`** — em `List<Integer>`, existe `remove(int index)` (remove por posição) e `remove(Object o)` (remove por valor). `list.remove(20)` chama a versão por índice, não por valor. Isso pega muita gente de surpresa.
3. **`Arrays.asList()` não é totalmente mutável** — retorna uma lista de tamanho fixo, apoiada (backed) no array original. Dá pra usar `set()`, mas `add()` ou `remove()` lançam `UnsupportedOperationException`.
4. **Covariância de array pode quebrar em runtime** — Java permite `Object[] arr = new String[3];` compilar, mas `arr[0] = 1;` lança `ArrayStoreException` em tempo de execução, porque o array "lembra" que é um `String[]` por baixo. Com genéricos (`List<Object>`), esse tipo de erro é pego em **tempo de compilação**, não em runtime.

---

### 4. Exercícios práticos

**1. Fácil**  
Crie um array de 5 números inteiros (preenchido manualmente, sem loop) e some todos os valores usando um `for` tradicional (com índice). Depois, faça o mesmo cálculo usando um `ArrayList<Integer>` com os mesmos 5 valores, usando `for-each`. Critério de pronto: os dois métodos devem imprimir a mesma soma.

**2. Fácil/Médio**  
Dado o array `String[] frutas = {"maçã", "banana", "uva"};`, converta-o para um `ArrayList<String>` (dica: existe um jeito direto de fazer isso, mas cuidado com a armadilha #3 acima). Depois, adicione mais duas frutas à lista e imprima o resultado final. Critério de pronto: a lista final deve ter 5 elementos e não pode lançar exceção.

**3. Médio**  
Escreva um método `int[] adicionarElemento(int[] original, int novoValor)` que recebe um array e um valor, e retorna um **novo** array contendo todos os elementos originais mais o novo valor no final (sem usar `ArrayList` dentro do método — só array puro e cópia manual ou `Arrays.copyOf`). Critério de pronto: chamar `adicionarElemento(new int[]{1,2,3}, 4)` deve retornar `{1,2,3,4}`.

**4. Difícil/Desafio**  
Dada `List<Integer> numeros = new ArrayList<>(List.of(1, 2, 3, 2, 4, 2, 5));`, remova **todas** as ocorrências do valor `2`. Faça de duas formas: (a) usando um `Iterator` explícito com `iterator.remove()`; (b) tentando fazer com um `for-each` comum chamando `numeros.remove(Integer.valueOf(2))` dentro do loop — rode, deixe estourar o erro, e no gabarito explique exatamente por que isso acontece. Critério de pronto: a versão (a) deve funcionar e imprimir `[1, 3, 4, 5]`; a versão (b) deve demonstrar a exceção.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        // Versão com array
        int[] arr = {5, 10, 15, 20, 25};
        int somaArray = 0;
        for (int i = 0; i < arr.length; i++) {
            somaArray += arr[i];
        }
        System.out.println("Soma (array): " + somaArray);

        // Versão com ArrayList
        List<Integer> lista = new ArrayList<>(List.of(5, 10, 15, 20, 25));
        int somaLista = 0;
        for (int valor : lista) {
            somaLista += valor; // aqui acontece unboxing automático: Integer -> int
        }
        System.out.println("Soma (lista): " + somaLista);
    }
}
```

_Raciocínio:_ o objetivo aqui é só fixar a diferença mecânica de acesso — índice explícito no array (`arr.length`, `arr[i]`) vs iteração direta no `for-each` da lista, que já abstrai o índice pra você.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        String[] frutas = {"maçã", "banana", "uva"};

        // new ArrayList<>(...) copia os elementos para uma lista NOVA e mutável
        // (diferente de Arrays.asList(frutas) sozinho, que seria de tamanho fixo)
        List<String> listaFrutas = new ArrayList<>(Arrays.asList(frutas));

        listaFrutas.add("laranja");
        listaFrutas.add("manga");

        System.out.println(listaFrutas); // [maçã, banana, uva, laranja, manga]
    }
}
```

_Raciocínio:_ o pulo do gato é envolver `Arrays.asList(frutas)` com `new ArrayList<>(...)`. Isso cria uma cópia genuinamente mutável, em vez de depender da lista de tamanho fixo que `Arrays.asList` retorna sozinha (armadilha #3).

**Exercício 3**

java

```java
public class Exercicio3 {
    static int[] adicionarElemento(int[] original, int novoValor) {
        int[] novoArray = Arrays.copyOf(original, original.length + 1);
        novoArray[novoArray.length - 1] = novoValor;
        return novoArray;
    }

    public static void main(String[] args) {
        int[] resultado = adicionarElemento(new int[]{1, 2, 3}, 4);
        System.out.println(Arrays.toString(resultado)); // [1, 2, 3, 4]
    }
}
```

_Raciocínio:_ `Arrays.copyOf` cria um array novo, maior, copiando os elementos originais e deixando as posições extras com valor padrão (`0` para `int`). Depois só sobrescrevemos a última posição. Isso ilustra na prática por que arrays são incômodos quando o tamanho precisa mudar — e por que `ArrayList` existe: ela faz exatamente esse tipo de cópia internamente, de forma automática, quando você chama `add()` e a capacidade interna estoura.

**Exercício 4**

java

```java
public class Exercicio4 {
    public static void main(String[] args) {
        // (a) usando Iterator explícito
        List<Integer> numerosA = new ArrayList<>(List.of(1, 2, 3, 2, 4, 2, 5));
        Iterator<Integer> it = numerosA.iterator();
        while (it.hasNext()) {
            int valor = it.next();
            if (valor == 2) {
                it.remove(); // remoção segura, feita através do próprio iterator
            }
        }
        System.out.println(numerosA); // [1, 3, 4, 5]

        // (b) tentando remover dentro de um for-each comum
        List<Integer> numerosB = new ArrayList<>(List.of(1, 2, 3, 2, 4, 2, 5));
        try {
            for (int valor : numerosB) {
                if (valor == 2) {
                    numerosB.remove(Integer.valueOf(2)); // vai estourar
                }
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("Erro capturado: " + e);
        }
    }
}
```

_Raciocínio:_ o `for-each` usa um `Iterator` por trás dos panos. Esse iterator guarda um contador interno (`modCount`) que ele compara a cada `next()` com o `modCount` da lista. Quando você chama `list.remove()` diretamente (não através do iterator), o `modCount` da lista muda, mas o iterator não sabe disso — na próxima chamada de `next()`, ele detecta a divergência e lança `ConcurrentModificationException` como proteção contra inconsistência. É por isso que a única forma seguro de remover elementos durante uma iteração manual é usar `iterator.remove()`, que atualiza os dois contadores em sincronia. (Esse mecanismo de `Iterator` será aprofundado em um tópico futuro deste mesmo bloco.)