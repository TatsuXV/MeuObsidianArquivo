### 1. Teoria

**`Deque`** (lê-se "deck" — de **D**ouble **E**nded **QUE**ue) é uma interface que estende `Queue` e permite inserir e remover elementos **nas duas pontas**: início e fim. É, na prática, a interface mais completa e flexível de todas as que vimos até aqui — tanto que ela sozinha consegue se comportar como `Queue` (FIFO) e como `Stack` (LIFO), dependendo de quais métodos você usa. Você já usou `Deque` nos dois tópicos anteriores; agora vamos ver a interface **por inteiro**, formalmente.

A API de `Deque` tem métodos explícitos para cada ponta, além de "apelidos" que mapeiam pra semântica de fila ou pilha:

|Operação|Início (head)|Fim (tail)|
|---|---|---|
|Inserir (lança exceção)|`addFirst(e)`|`addLast(e)`|
|Inserir (retorna `false`)|`offerFirst(e)`|`offerLast(e)`|
|Remover (lança exceção)|`removeFirst()`|`removeLast()`|
|Remover (retorna `null`)|`pollFirst()`|`pollLast()`|
|Consultar (lança exceção)|`getFirst()`|`getLast()`|
|Consultar (retorna `null`)|`peekFirst()`|`peekLast()`|

E os "apelidos" que você já usou:

|Apelido|Equivale a|
|---|---|
|`offer(e)`|`offerLast(e)` (comportamento de fila: insere no fim)|
|`poll()`|`pollFirst()` (comportamento de fila: remove do início)|
|`push(e)`|`addFirst(e)` (comportamento de pilha: insere no início)|
|`pop()`|`removeFirst()` (comportamento de pilha: remove do início)|

Isso explica algo que talvez tenha passado despercebido: **`Queue` e `Stack` (via `Deque`) usam pontas diferentes por convenção**, mas a estrutura de dados por baixo (`ArrayDeque`) é exatamente a mesma. `offer`/`poll` operam em pontas opostas (fim/início) pra dar FIFO; `push`/`pop` operam na mesma ponta (início/início) pra dar LIFO.

**Implementações principais:**

- **`ArrayDeque`** — baseada em array redimensionável internamente. Não permite `null`. É a implementação recomendada tanto pra uso como fila quanto como pilha (como já vimos), e geralmente mais eficiente que `LinkedList` pra esses casos.
- **`LinkedList`** — implementa `Deque` também (além de `List`), baseada em lista duplamente ligada. Permite `null`. Mais lenta que `ArrayDeque` na maioria dos cenários por causa do overhead de alocação de nós, mas pode fazer sentido se você já precisa da estrutura de lista ligada por outro motivo.

**Não confunda `Deque` com `Queue` genérico:** todo `Deque` é um `Queue` (por herança de interface), mas nem todo `Queue` é um `Deque` — `PriorityQueue`, por exemplo, implementa `Queue` mas **não** implementa `Deque`, porque não faz sentido "inserir no início" numa fila de prioridade (a ordem é sempre determinada pela prioridade, não por onde você insere).

**Onde aparece no dia a dia de backend:** implementação de estruturas de "janela deslizante" (sliding window — padrão que você vai ver com profundidade na Trilha 2 de DSA), histórico de navegação com undo/redo nas duas direções, buffers circulares, e de forma geral qualquer situação onde você precisa adicionar/remover de ambas as extremidades com eficiência — algo que `ArrayList` não faz bem (remover do início de uma `ArrayList` é O(n), porque desloca todos os elementos).

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class DequeExample {
    public static void main(String[] args) {
        Deque<Integer> deque = new ArrayDeque<>();

        // Inserindo dos dois lados
        deque.addFirst(2);  // [2]
        deque.addLast(3);   // [2, 3]
        deque.addFirst(1);  // [1, 2, 3]
        deque.addLast(4);   // [1, 2, 3, 4]

        System.out.println(deque); // [1, 2, 3, 4]

        // Consultando as duas pontas sem remover
        System.out.println("Primeiro: " + deque.peekFirst()); // 1
        System.out.println("Último: " + deque.peekLast());    // 4

        // Removendo dos dois lados
        int removidoInicio = deque.pollFirst(); // remove 1
        int removidoFim = deque.pollLast();      // remove 4
        System.out.println("Removido do início: " + removidoInicio);
        System.out.println("Removido do fim: " + removidoFim);
        System.out.println("Deque agora: " + deque); // [2, 3]

        // Usando Deque como Queue (FIFO) - offer/poll operam em pontas opostas
        Deque<String> comoFila = new ArrayDeque<>();
        comoFila.offer("A"); // vai pro fim
        comoFila.offer("B");
        comoFila.offer("C");
        System.out.println("Fila: " + comoFila.poll()); // A (removeu do início) - FIFO

        // Usando Deque como Stack (LIFO) - push/pop operam na mesma ponta
        Deque<String> comoPilha = new ArrayDeque<>();
        comoPilha.push("A"); // vai pro início
        comoPilha.push("B");
        comoPilha.push("C");
        System.out.println("Pilha: " + comoPilha.pop()); // C (removeu do início) - LIFO

        // Exemplo prático: janela deslizante de tamanho fixo, guardando os últimos N valores
        Deque<Integer> ultimasTresLeituras = new ArrayDeque<>();
        int[] leiturasSensor = {10, 20, 30, 40, 50};

        for (int leitura : leiturasSensor) {
            ultimasTresLeituras.addLast(leitura);
            if (ultimasTresLeituras.size() > 3) {
                ultimasTresLeituras.pollFirst(); // descarta a leitura mais antiga
            }
            System.out.println("Janela atual: " + ultimasTresLeituras);
        }
        // Última janela impressa: [30, 40, 50] - sempre as 3 mais recentes
    }
}
```

---

### 3. Armadilhas comuns

1. **Confundir qual ponta cada apelido usa** — `push`/`pop` operam no **início**, `offer`/`poll` (sem sufixo) operam em pontas opostas (`offer` = fim, `poll` = início) pra simular FIFO. Misturar `push` com `poll` no mesmo código, achando que são "a mesma coisa", produz um comportamento que parece certo às vezes e errado em outras — é um erro sutil de se debugar.
2. **Usar `LinkedList` como `Deque` por padrão, sem considerar `ArrayDeque`** — mesmo raciocínio de performance que já vimos em `Queue`/`Stack`: `ArrayDeque` costuma ser a escolha mais eficiente, a menos que você já precise da estrutura de lista ligada por outro motivo específico.
3. **Tentar inserir `null` num `ArrayDeque`** — `addFirst(null)` ou `offer(null)` lançam `NullPointerException`. Isso é proposital (diferente de `LinkedList`, que aceita `null`): a documentação usa `null` como valor de retorno especial em `poll`/`peek` pra sinalizar "vazio", então permitir `null` como elemento real criaria ambiguidade — você não saberia se `poll()` retornou `null` porque a fila estava vazia ou porque o elemento em si era `null`.
4. **Esquecer que `Deque` não é indexado** — assim como `Queue`, não existe `get(i)` num `Deque`. Se seu problema precisa de acesso aleatório por posição além de operações nas pontas, `Deque` sozinho não resolve — você provavelmente precisa de `ArrayList` ou de uma estrutura combinada.

---

### 4. Exercícios práticos

**1. Fácil**  
Crie um `Deque<Integer>` vazio. Insira `10` no início, `20` no fim, `5` no início, `30` no fim (usando os métodos explícitos `addFirst`/`addLast`, não os apelidos). Imprima o Deque resultante. Critério de pronto: a saída deve ser exatamente `[5, 10, 20, 30]`.

**2. Fácil/Médio**  
Dado `Deque<Character> deque = new ArrayDeque<>(List.of('r', 'a', 'c', 'e', 'c', 'a', 'r'));`, verifique se a sequência é um **palíndromo** usando exclusivamente as operações de `Deque` — remova e compare o primeiro e o último elemento repetidamente (com `pollFirst()`/`pollLast()`) até sobrar 0 ou 1 elemento. Se em algum momento os elementos removidos forem diferentes, não é palíndromo. Critério de pronto: `"racecar"` deve retornar `true`; `"hello"` deve retornar `false`.

**3. Médio**  
Implemente uma fila de histórico de navegação com **undo e redo**, usando duas estruturas `Deque<String>`: `historicoAnterior` e `historicoProximo`. Métodos: `void visitar(String pagina)` (visita uma página nova, limpando o histórico de "próximo"), `String voltar()` (volta uma página, movendo a atual pro histórico de "próximo"), `String avancar()` (refaz, movendo de volta pro histórico "anterior"). Se não houver pra onde voltar/avançar, retorne `null`. Critério de pronto: visitando `"Home"`, `"Produtos"`, `"Carrinho"` nessa ordem, chamar `voltar()` duas vezes deve retornar `"Produtos"` depois `"Home"`; chamar `avancar()` uma vez depois disso deve retornar `"Produtos"`.

**4. Difícil/Desafio**  
Implemente o algoritmo de **máximo em janela deslizante**: dado um array de inteiros e um tamanho de janela `k`, retorne uma lista com o valor máximo de cada janela consecutiva de tamanho `k` conforme ela "desliza" pelo array. Use um `Deque<Integer>` guardando **índices** (não valores) do array, mantendo a invariante de que os índices na deque estão sempre em ordem decrescente de valor (remova do fim da deque qualquer índice cujo valor seja menor que o valor que está entrando — ele nunca mais será o máximo de nenhuma janela). Critério de pronto: para `int[] valores = {1, 3, -1, -3, 5, 3, 6, 7}` e `k = 3`, o resultado deve ser `[3, 3, 5, 5, 6, 7]`.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        Deque<Integer> deque = new ArrayDeque<>();
        deque.addFirst(10); // [10]
        deque.addLast(20);  // [10, 20]
        deque.addFirst(5);  // [5, 10, 20]
        deque.addLast(30);  // [5, 10, 20, 30]

        System.out.println(deque); // [5, 10, 20, 30]
    }
}
```

_Raciocínio:_ cada `addFirst`/`addLast` afeta só a ponta indicada, sem mexer no resto — seguir a sequência de inserções passo a passo (comentado ao lado de cada linha) é a forma mais direta de confirmar que o resultado bate com o esperado.

**Exercício 2**

java

```java
public class Exercicio2 {
    static boolean ehPalindromo(Deque<Character> deque) {
        // trabalhamos numa cópia para não destruir o Deque original do chamador
        Deque<Character> copia = new ArrayDeque<>(deque);

        while (copia.size() > 1) {
            char inicio = copia.pollFirst();
            char fim = copia.pollLast();
            if (inicio != fim) {
                return false;
            }
        }
        return true; // sobrou 0 ou 1 elemento - sempre "igual" a si mesmo
    }

    public static void main(String[] args) {
        Deque<Character> racecar = new ArrayDeque<>(List.of('r','a','c','e','c','a','r'));
        Deque<Character> hello = new ArrayDeque<>(List.of('h','e','l','l','o'));

        System.out.println(ehPalindromo(racecar)); // true
        System.out.println(ehPalindromo(hello));   // false
    }
}
```

_Raciocínio:_ essa é a razão de existir de um `Deque` — o problema pede comparação simétrica das duas pontas convergindo pro meio, e é exatamente isso que `pollFirst()`/`pollLast()` oferecem nativamente, sem precisar calcular índices manualmente como você faria com um array. Trabalhar numa cópia (`new ArrayDeque<>(deque)`) é uma boa prática aqui: o método não deveria ter o efeito colateral de esvaziar a estrutura que o chamador passou.

**Exercício 3**

java

```java
public class Exercicio3 {
    private Deque<String> historicoAnterior = new ArrayDeque<>();
    private Deque<String> historicoProximo = new ArrayDeque<>();
    private String paginaAtual = null;

    void visitar(String pagina) {
        if (paginaAtual != null) {
            historicoAnterior.push(paginaAtual);
        }
        paginaAtual = pagina;
        historicoProximo.clear(); // visitar página nova invalida o "avançar"
    }

    String voltar() {
        if (historicoAnterior.isEmpty()) {
            return null;
        }
        historicoProximo.push(paginaAtual);
        paginaAtual = historicoAnterior.pop();
        return paginaAtual;
    }

    String avancar() {
        if (historicoProximo.isEmpty()) {
            return null;
        }
        historicoAnterior.push(paginaAtual);
        paginaAtual = historicoProximo.pop();
        return paginaAtual;
    }

    public static void main(String[] args) {
        Exercicio3 navegador = new Exercicio3();
        navegador.visitar("Home");
        navegador.visitar("Produtos");
        navegador.visitar("Carrinho");

        System.out.println(navegador.voltar());   // Produtos
        System.out.println(navegador.voltar());   // Home
        System.out.println(navegador.avancar());  // Produtos
    }
}
```

_Raciocínio:_ cada `Deque` aqui funciona como uma **pilha** (usamos `push`/`pop`, mesma ponta) — `historicoAnterior` empilha pra onde eu posso voltar; `historicoProximo` empilha pra onde eu posso avançar. Voltar uma página empurra a atual pra pilha de "próximo" (pra poder ser refeita depois) e traz de volta o topo de "anterior". Avançar faz o caminho inverso. Visitar uma página nova precisa **limpar** o histórico de "próximo" — essa é a regra de negócio real de qualquer navegador: se você volta e depois visita algo novo, o "avançar" antigo deixa de fazer sentido, porque o futuro mudou.

**Exercício 4**

java

```java
public class Exercicio4 {
    static List<Integer> maximoJanelaDeslizante(int[] valores, int k) {
        List<Integer> resultado = new ArrayList<>();
        Deque<Integer> indices = new ArrayDeque<>(); // guarda ÍNDICES, não valores

        for (int i = 0; i < valores.length; i++) {
            // remove do início se o índice já saiu da janela atual
            if (!indices.isEmpty() && indices.peekFirst() <= i - k) {
                indices.pollFirst();
            }

            // remove do fim todo índice cujo valor é menor que o que está entrando -
            // ele nunca mais será o máximo de nenhuma janela futura
            while (!indices.isEmpty() && valores[indices.peekLast()] < valores[i]) {
                indices.pollLast();
            }

            indices.addLast(i);

            // a partir da janela completa (i >= k - 1), o máximo é o valor no início da deque
            if (i >= k - 1) {
                resultado.add(valores[indices.peekFirst()]);
            }
        }

        return resultado;
    }

    public static void main(String[] args) {
        int[] valores = {1, 3, -1, -3, 5, 3, 6, 7};
        System.out.println(maximoJanelaDeslizante(valores, 3)); // [3, 3, 5, 5, 6, 7]
    }
}
```

_Raciocínio:_ a chave desse algoritmo (um clássico de entrevista, que você vai reencontrar na Trilha 2) é a **invariante mantida na deque**: os índices guardados nela estão sempre em ordem decrescente de _valor_ correspondente. Isso é garantido pelo `while` que remove do fim qualquer índice "dominado" por um valor maior que acabou de chegar — se um valor mais novo é maior que um valor mais antigo ainda dentro da janela, o mais antigo nunca mais vai ser o máximo de nada, então não faz sentido mantê-lo. O primeiro elemento da deque (`peekFirst()`) é sempre o índice do maior valor da janela atual — e o outro `if` no início garante que esse índice não "escapou" da janela (ficou velho demais). É um uso avançado de `Deque` porque ele explora as duas pontas com propósitos diferentes: o fim controla "quem ainda é candidato a máximo", o início controla "quem ainda está dentro da janela".