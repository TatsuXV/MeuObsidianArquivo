### 1. Teoria

**`Iterator`** é uma interface do Collections Framework que fornece uma forma padronizada de **percorrer** qualquer coleção, um elemento por vez, sem precisar saber os detalhes internos de como aquela coleção está estruturada (array, lista ligada, árvore, hash table...). É a peça que torna possível o `for-each` que você já usou dezenas de vezes até aqui — e agora vamos abrir a caixa preta.

Toda classe que implementa `Iterable` (e toda `Collection` implementa `Iterable`) é obrigada a fornecer um método `iterator()`, que retorna um objeto `Iterator` com três métodos principais:

|Método|O que faz|
|---|---|
|`boolean hasNext()`|Existe um próximo elemento a percorrer?|
|`E next()`|Retorna o próximo elemento e avança o cursor|
|`void remove()`|Remove o **último elemento retornado por `next()`** da coleção|

**Por que isso importa, já que o `for-each` faz isso "sozinho"?** Porque o `for-each` é só **açúcar sintático** — o compilador Java, por baixo dos panos, transforma automaticamente:

java

```java
for (String item : lista) {
    System.out.println(item);
}
```

em algo equivalente a:

java

```java
Iterator<String> it = lista.iterator();
while (it.hasNext()) {
    String item = it.next();
    System.out.println(item);
}
```

Isso explica exatamente por que você **não pode remover elementos dentro de um `for-each`** usando `lista.remove(x)` diretamente (a `ConcurrentModificationException` que apareceu lá no gabarito do exercício 4 de Array vs ArrayList): o `for-each` não te dá acesso ao objeto `Iterator` que ele criou internamente, então você não tem como chamar `it.remove()` — só o `iterator.remove()` sabe atualizar o estado interno da coleção **e** do próprio iterator em sincronia. Remover diretamente da coleção (`lista.remove()`) enquanto o `for-each` está no meio do loop deixa o iterator interno "desatualizado" em relação à coleção, e é exatamente essa inconsistência que a exceção detecta e denuncia.

**`ListIterator`** é uma extensão de `Iterator`, disponível só para `List`, que adiciona: iteração **bidirecional** (`hasPrevious()`, `previous()`), acesso ao índice atual (`nextIndex()`, `previousIndex()`), e a capacidade de **modificar** a lista durante a iteração (`set(e)` para substituir o último elemento retornado, `add(e)` para inserir na posição atual) — coisa que o `Iterator` comum não oferece.

**Não confunda:**

- `Iterator` é sobre **percorrer**; não é uma estrutura de dados em si, é uma "posição/cursor" sobre uma coleção existente.
- `hasNext()` **não avança** o cursor — só consulta. `next()` é quem avança.
- Chamar `remove()` antes de qualquer `next()` (ou duas vezes seguidas sem um `next()` no meio) lança `IllegalStateException` — `remove()` só é válido logo depois de um `next()`.

**Onde aparece no dia a dia de backend:** você raramente vai instanciar um `Iterator` manualmente no dia a dia — o `for-each` cobre 95% dos casos. Mas entender o mecanismo é essencial pra: (1) saber por que `ConcurrentModificationException` acontece e como evitá-la de verdade (removendo com segurança durante iteração), (2) entender como implementar sua própria classe iterável customizada (implementando `Iterable`/`Iterator`), algo que aparece em estruturas de dados customizadas, e (3) entender o próprio Collections Framework por dentro, já que praticamente toda coleção do Java é construída sobre esse contrato.

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class IteratorExample {
    public static void main(String[] args) {
        List<Integer> numeros = new ArrayList<>(List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));

        // Uso explícito e manual de Iterator - equivalente ao que o for-each faz por baixo
        Iterator<Integer> it = numeros.iterator();
        while (it.hasNext()) {
            int valor = it.next();
            System.out.println("Percorrendo: " + valor);
        }

        // O uso real de Iterator no dia a dia: remover com segurança durante a iteração
        List<Integer> numerosParaFiltrar = new ArrayList<>(List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
        Iterator<Integer> itRemocao = numerosParaFiltrar.iterator();
        while (itRemocao.hasNext()) {
            int valor = itRemocao.next();
            if (valor % 2 == 0) {
                itRemocao.remove(); // remove o ÚLTIMO elemento retornado por next() - o "valor" atual
            }
        }
        System.out.println("Só ímpares: " + numerosParaFiltrar); // [1, 3, 5, 7, 9]

        // remove() sem next() antes -> IllegalStateException
        Iterator<Integer> itErro = numerosParaFiltrar.iterator();
        try {
            itErro.remove(); // erro: nenhum next() foi chamado ainda
        } catch (IllegalStateException e) {
            System.out.println("Erro capturado: " + e);
        }

        // ListIterator: bidirecional e com modificação (substituição) durante a iteração
        List<String> nomes = new ArrayList<>(List.of("ana", "bruno", "carla"));
        ListIterator<String> listIt = nomes.listIterator();
        while (listIt.hasNext()) {
            String nome = listIt.next();
            listIt.set(nome.toUpperCase()); // substitui o elemento atual - Iterator comum NÃO tem set()
        }
        System.out.println(nomes); // [ANA, BRUNO, CARLA]

        // Percorrendo de trás pra frente com ListIterator (só é possível chegando ao fim primeiro)
        while (listIt.hasPrevious()) {
            System.out.println("De trás pra frente: " + listIt.previous());
        }
    }
}
```

---

### 3. Armadilhas comuns

1. **Tentar remover da coleção diretamente (`lista.remove(x)`) dentro de um `for-each`** — lança `ConcurrentModificationException`, exatamente como vimos no bloco de `List`. A única forma segura de remover durante a iteração é usando `iterator.remove()` explicitamente (ou, em Java moderno, `collection.removeIf(predicado)`, que resolve isso internamente sem você precisar lidar com `Iterator` na mão).
2. **Chamar `next()` sem checar `hasNext()` antes** — se não houver mais elementos, `next()` lança `NoSuchElementException`. É por isso que todo loop manual com `Iterator` **precisa** vir acompanhado de `hasNext()` como condição.
3. **Chamar `remove()` duas vezes seguidas, sem um `next()` no meio** — cada `remove()` só é válido em relação ao elemento mais recentemente retornado por `next()`. Chamar de novo sem avançar antes lança `IllegalStateException`.
4. **Achar que dá pra modificar a coleção livremente com `ListIterator.add()` no meio do loop sem entender o efeito colateral** — `add()` insere o elemento **antes** da posição que `next()` retornaria em seguida, o que significa que, se você não tomar cuidado, pode acabar processando o elemento recém-inserido de novo (ou criando um loop mais longo do que esperava). Isso é um detalhe sutil — quando for usar, teste com atenção o comportamento no seu caso específico.

---

### 4. Exercícios práticos

**1. Fácil**  
Dada `List<String> frutas = new ArrayList<>(List.of("maçã", "banana", "uva", "manga", "pera"));`, use um `Iterator` explícito (sem `for-each`) para imprimir cada fruta precedida do seu número de ordem, começando em 1 (ex: `"1: maçã"`, `"2: banana"`...). Critério de pronto: a numeração deve ir de 1 a 5, na ordem original da lista.

**2. Fácil/Médio**  
Dada `List<Integer> numeros = new ArrayList<>(List.of(5, 12, 8, 23, 3, 17, 9, 30));`, use um `Iterator` explícito com `remove()` para eliminar da lista todos os valores **maiores que 15**. Critério de pronto: o resultado final deve ser `[5, 12, 8, 3, 9]`, sem lançar nenhuma exceção.

**3. Médio**  
Use um `ListIterator` para substituir, numa `List<Integer> valores = new ArrayList<>(List.of(1, 2, 3, 4, 5));`, cada valor pelo seu dobro (ex: `1` vira `2`, `2` vira `4`, etc.) — sem criar uma lista nova, modificando a lista original no lugar usando `set()`. Critério de pronto: a lista final deve ser `[2, 4, 6, 8, 10]`.

**4. Difícil/Desafio**  
Implemente uma classe customizada `IntervaloNumerico` que representa um intervalo de inteiros (ex: de 1 a 10) e implemente `Iterable<Integer>` nela, de forma que seja possível usar essa classe diretamente num `for-each` (`for (int n : new IntervaloNumerico(1, 5))`). Você vai precisar criar uma classe interna (ou anônima) que implementa `Iterator<Integer>`, controlando `hasNext()` e `next()` manualmente para percorrer do início ao fim do intervalo. Critério de pronto: `for (int n : new IntervaloNumerico(1, 5))` deve imprimir `1, 2, 3, 4, 5`, nessa ordem, sem usar nenhuma `List`/array internamente para armazenar os números — o `Iterator` deve calcular o próximo número sob demanda.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        List<String> frutas = new ArrayList<>(List.of("maçã", "banana", "uva", "manga", "pera"));

        Iterator<String> it = frutas.iterator();
        int numero = 1;
        while (it.hasNext()) {
            String fruta = it.next();
            System.out.println(numero + ": " + fruta);
            numero++;
        }
    }
}
```

_Raciocínio:_ exercício de fixação do padrão básico `while (it.hasNext()) { ... it.next() ... }`, só acrescentando um contador manual pra numeração — o `Iterator` em si não tem noção de índice (diferente do `ListIterator`, que veremos no exercício 3), então precisamos manter esse controle por fora.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        List<Integer> numeros = new ArrayList<>(List.of(5, 12, 8, 23, 3, 17, 9, 30));

        Iterator<Integer> it = numeros.iterator();
        while (it.hasNext()) {
            int valor = it.next();
            if (valor > 15) {
                it.remove(); // remove o elemento atual (o último retornado por next())
            }
        }

        System.out.println(numeros); // [5, 12, 8, 3, 9]
    }
}
```

_Raciocínio:_ esse é o caso de uso mais comum de `Iterator` explícito na prática — filtrar uma coleção removendo elementos que atendem a uma condição, sem lançar `ConcurrentModificationException`. `it.remove()` sabe exatamente qual foi o último elemento retornado (`valor`) e o remove com segurança, mantendo o cursor interno do iterator consistente com o novo estado da lista, coisa que `numeros.remove(valor)` direto não conseguiria garantir no meio do loop.

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        List<Integer> valores = new ArrayList<>(List.of(1, 2, 3, 4, 5));

        ListIterator<Integer> listIt = valores.listIterator();
        while (listIt.hasNext()) {
            int valorAtual = listIt.next();
            listIt.set(valorAtual * 2); // substitui o elemento na posição atual
        }

        System.out.println(valores); // [2, 4, 6, 8, 10]
    }
}
```

_Raciocínio:_ essa é a razão de existir do `ListIterator` em relação ao `Iterator` comum — `set()` permite **substituir** o elemento atual sem removê-lo e reinseri-lo, e sem precisar de acesso por índice manual (`valores.set(i, valores.get(i) * 2)`, que funcionaria também, mas exige controlar `i` você mesmo). O `ListIterator` já sabe "em qual posição" você está a cada `next()`, e `set()` aplica a mudança exatamente ali.

**Exercício 4**

java

```java
public class IntervaloNumerico implements Iterable<Integer> {
    private final int inicio;
    private final int fim;

    public IntervaloNumerico(int inicio, int fim) {
        this.inicio = inicio;
        this.fim = fim;
    }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<Integer>() {
            private int atual = inicio; // estado do cursor, começa no início do intervalo

            @Override
            public boolean hasNext() {
                return atual <= fim;
            }

            @Override
            public Integer next() {
                if (!hasNext()) {
                    throw new NoSuchElementException();
                }
                return atual++; // retorna o valor atual E DEPOIS incrementa
            }
        };
    }

    public static void main(String[] args) {
        for (int n : new IntervaloNumerico(1, 5)) {
            System.out.println(n);
        }
        // Saída: 1, 2, 3, 4, 5
    }
}
```

_Raciocínio:_ esse exercício mostra o mecanismo completo do outro lado — implementando `Iterable<Integer>`, a classe se torna elegível pra sintaxe de `for-each`, porque o compilador consegue chamar `.iterator()` nela. A classe anônima que implementa `Iterator<Integer>` guarda seu próprio estado (`atual`), independente da instância de `IntervaloNumerico` — isso é importante porque, em teoria, você poderia ter **dois `for-each` simultâneos** sobre o mesmo `IntervaloNumerico`, cada um com seu próprio cursor independente, já que cada chamada a `.iterator()` cria um novo objeto Iterator com seu próprio `atual`. Note também que nenhum número é armazenado em lista/array — cada valor é calculado "sob demanda" dentro de `next()`, o que é exatamente o espírito de um `Iterator`: ele representa uma sequência de acesso, não necessariamente uma coleção materializada em memória.