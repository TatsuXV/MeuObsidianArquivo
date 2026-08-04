### 1. Teoria

**`Queue`** é uma interface do Collections Framework que representa uma coleção organizada pra processamento **FIFO** (First In, First Out) — o primeiro elemento a entrar é o primeiro a sair. Pense numa fila de banco: quem chegou primeiro é atendido primeiro.

`Queue` estende `Collection`, mas adiciona um conjunto de operações específicas pra funcionar como fila:

|Operação|Lança exceção se falhar|Retorna valor especial (`null`/`false`) se falhar|
|---|---|---|
|Inserir|`add(e)`|`offer(e)`|
|Remover (e retornar a cabeça)|`remove()`|`poll()`|
|Consultar a cabeça (sem remover)|`element()`|`peek()`|

Essa tabela é o coração da interface: pra cada operação, existem **duas versões**. A versão "exception-throwing" (`add`, `remove`, `element`) é útil quando falhar é um erro de verdade no seu programa. A versão "special-value" (`offer`, `poll`, `peek`) é mais usada na prática, porque tratar `null`/`false` costuma ser mais simples que capturar exceção — especialmente em fila que pode legitimamente ficar vazia com frequência.

**Implementações principais:**

- **`LinkedList`** — implementa tanto `List` quanto `Queue` (e `Deque`). É a implementação mais genérica, baseada em lista duplamente ligada.
- **`ArrayDeque`** — implementa `Deque` (que estende `Queue`). Geralmente **preferível a `LinkedList`** pra uso como fila pura, porque tem melhor performance (menos overhead de alocação de nós) e menor consumo de memória. Não permite `null`.
- **`PriorityQueue`** — não é FIFO! A "cabeça" da fila é sempre o menor elemento (ordem natural ou via `Comparator`), não o que entrou primeiro. É uma fila de prioridade, não uma fila comum — vale destacar isso porque o nome confunde muita gente.

**Não confunda com `Deque`:** `Deque` (double-ended queue) é uma sub-interface de `Queue` que permite inserir/remover **dos dois lados** (início e fim). Isso será o próximo tópico do checklist — aqui o foco é só o comportamento FIFO padrão de `Queue`.

**Onde aparece no dia a dia de backend:** filas de processamento assíncrono (antes de chegar numa fila de verdade tipo RabbitMQ/Kafka, a lógica interna costuma usar `Queue` em memória), implementação de BFS (Breadth-First Search) em grafos/árvores — que vamos ver a fundo na Trilha 2 —, buffers de tarefas a processar, e `PriorityQueue` aparece em cenários de "processar por prioridade" (ex: fila de atendimento com urgência).

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class QueueExample {
    public static void main(String[] args) {
        // ArrayDeque como Queue - a implementação recomendada pra fila pura
        Queue<String> filaAtendimento = new ArrayDeque<>();

        // offer: insere no fim da fila (versão "segura", sem exceção)
        filaAtendimento.offer("Ana");
        filaAtendimento.offer("Bruno");
        filaAtendimento.offer("Carla");

        System.out.println(filaAtendimento); // [Ana, Bruno, Carla]

        // peek: olha quem é o próximo, SEM remover
        System.out.println("Próximo a ser atendido: " + filaAtendimento.peek()); // Ana

        // poll: remove e retorna a cabeça da fila (FIFO)
        String atendido = filaAtendimento.poll();
        System.out.println("Atendendo: " + atendido); // Ana
        System.out.println("Fila agora: " + filaAtendimento); // [Bruno, Carla]

        // poll numa fila vazia retorna null, não lança exceção
        Queue<String> filaVazia = new ArrayDeque<>();
        String resultado = filaVazia.poll();
        System.out.println("Poll em fila vazia: " + resultado); // null

        // remove() numa fila vazia LANÇA exceção - diferença chave em relação a poll()
        try {
            filaVazia.remove();
        } catch (NoSuchElementException e) {
            System.out.println("Erro capturado: " + e);
        }

        // PriorityQueue: NÃO é FIFO - a "cabeça" é sempre o menor valor
        Queue<Integer> filaPrioridade = new PriorityQueue<>();
        filaPrioridade.offer(30);
        filaPrioridade.offer(10);
        filaPrioridade.offer(20);

        // repare: a ordem de saída é 10, 20, 30 - não é a ordem de inserção (30, 10, 20)
        System.out.println("Ordem de saída da PriorityQueue:");
        while (!filaPrioridade.isEmpty()) {
            System.out.println(filaPrioridade.poll());
        }
    }
}
```

---

### 3. Armadilhas comuns

1. **Achar que `PriorityQueue` mantém ordem de inserção** — o nome "Queue" engana. `PriorityQueue` reorganiza os elementos internamente (usando uma estrutura de heap) e `poll()` sempre retorna o menor elemento disponível, não o mais antigo inserido. Se você imprimir a `PriorityQueue` diretamente (`System.out.println(filaPrioridade)`), a ordem impressa também **não** é garantida como a ordem de prioridade — só `poll()` sucessivo garante a ordem correta.
2. **Confundir `remove()`/`element()` (lançam exceção) com `poll()`/`peek()` (retornam `null`)** — usar `remove()` numa fila que pode legitimamente estar vazia, sem tratar a exceção, é um jeito fácil de derrubar seu processo de um jeito inesperado. Na maioria dos casos de fila em produção, `poll()`/`peek()` são as escolhas mais seguras e naturais.
3. **Usar `LinkedList` como `Queue` por hábito, sem considerar `ArrayDeque`** — funciona, mas na prática `ArrayDeque` costuma performar melhor pra esse uso específico, e é a recomendação atual da própria documentação do Java pra fila/pilha simples (sem necessidade de acesso indexado por posição, que só `List` oferece).
4. **Esquecer que `Queue` não permite acesso por índice** — como `Queue` não estende `List`, não existe `get(i)` nela. Se você precisa acessar elementos no meio da fila sem removê-los, `Queue` não é a ferramenta certa — você provavelmente quer `List` ou `Deque`.

---

### 4. Exercícios práticos

**1. Fácil**  
Crie uma `Queue<String>` usando `ArrayDeque`, simulando uma fila de impressão de documentos: adicione `"relatorio.pdf"`, `"contrato.docx"`, `"planilha.xlsx"` (nessa ordem, usando `offer`). Depois, processe (remova e imprima) os documentos um a um usando `poll()` até a fila ficar vazia, mostrando a mensagem `"Imprimindo: [nome do arquivo]"` para cada um. Critério de pronto: a ordem de impressão deve ser exatamente a ordem de inserção (FIFO).

**2. Fácil/Médio**  
Usando a mesma fila do exercício 1 (recriada), use `peek()` para verificar qual é o próximo documento **sem removê-lo** e imprima uma mensagem `"Próximo na fila: [nome]"`. Depois confirme que o `.size()` da fila não mudou após o `peek()`. Critério de pronto: o tamanho da fila antes e depois do `peek()` deve ser idêntico.

**3. Médio**  
Implemente uma simulação de atendimento com **duas filas separadas**: `Queue<String> filaNormal` e `Queue<String> filaPrioritaria`. Escreva um método `String proximoAtendimento(Queue<String> normal, Queue<String> prioritaria)` que retorna (via `poll()`) o próximo nome a ser atendido — **sempre priorizando a fila prioritária primeiro**, e só indo pra fila normal se a prioritária estiver vazia. Se as duas estiverem vazias, retorne `null` sem lançar exceção. Critério de pronto: com `filaPrioritaria = ["Zeca"]` e `filaNormal = ["Ana", "Bruno"]`, a primeira chamada deve retornar `"Zeca"`; a segunda chamada (já com a prioritária vazia) deve retornar `"Ana"`.

**4. Difícil/Desafio**  
Use uma `PriorityQueue<Integer>` para implementar a lógica de "encontrar os 3 menores valores" de uma lista de inteiros `List<Integer> valores = List.of(15, 3, 42, 8, 23, 1, 9, 30);`, sem ordenar a lista inteira manualmente (nem usar `Collections.sort`) — a ideia é usar o comportamento da `PriorityQueue` para extrair os menores um a um. Imprima os 3 menores valores, na ordem em que saem da fila. Critério de pronto: a saída deve ser `1, 3, 8` (nessa ordem).

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        Queue<String> filaImpressao = new ArrayDeque<>();
        filaImpressao.offer("relatorio.pdf");
        filaImpressao.offer("contrato.docx");
        filaImpressao.offer("planilha.xlsx");

        while (!filaImpressao.isEmpty()) {
            String documento = filaImpressao.poll();
            System.out.println("Imprimindo: " + documento);
        }
    }
}
```

_Raciocínio:_ `while (!filaImpressao.isEmpty())` é o padrão idiomático pra "processar tudo até esvaziar" — mais direto do que tentar controlar um índice manualmente, já que `Queue` nem tem índice. Cada `poll()` já remove e retorna o elemento, então não precisamos de uma chamada extra de remoção separada.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        Queue<String> filaImpressao = new ArrayDeque<>();
        filaImpressao.offer("relatorio.pdf");
        filaImpressao.offer("contrato.docx");
        filaImpressao.offer("planilha.xlsx");

        int tamanhoAntes = filaImpressao.size();
        String proximo = filaImpressao.peek();
        int tamanhoDepois = filaImpressao.size();

        System.out.println("Próximo na fila: " + proximo); // relatorio.pdf
        System.out.println("Tamanho antes: " + tamanhoAntes + " | Tamanho depois: " + tamanhoDepois); // 3 | 3
    }
}
```

_Raciocínio:_ o objetivo aqui é fixar na prática a diferença conceitual entre "consultar" (`peek`) e "consumir" (`poll`) — `peek()` é uma operação de leitura pura, não tem efeito colateral sobre o estado da fila, por isso o tamanho se mantém idêntico antes e depois.

**Exercício 3**

java

```java
public class Exercicio3 {
    static String proximoAtendimento(Queue<String> normal, Queue<String> prioritaria) {
        if (!prioritaria.isEmpty()) {
            return prioritaria.poll();
        }
        return normal.poll(); // se normal também estiver vazia, poll() retorna null - sem exceção
    }

    public static void main(String[] args) {
        Queue<String> filaNormal = new ArrayDeque<>(List.of("Ana", "Bruno"));
        Queue<String> filaPrioritaria = new ArrayDeque<>(List.of("Zeca"));

        System.out.println(proximoAtendimento(filaNormal, filaPrioritaria)); // Zeca
        System.out.println(proximoAtendimento(filaNormal, filaPrioritaria)); // Ana (prioritária já vazia)
    }
}
```

_Raciocínio:_ a checagem `if (!prioritaria.isEmpty())` resolve a regra de negócio de priorização de forma direta — sempre tenta a fila prioritária primeiro, e só recorre à normal como fallback. Usar `poll()` em vez de `remove()` nos dois casos garante que, se as duas filas estiverem vazias, o método retorna `null` graciosamente em vez de lançar `NoSuchElementException`, como o enunciado pede.

**Exercício 4**

java

```java
public class Exercicio4 {
    public static void main(String[] args) {
        List<Integer> valores = List.of(15, 3, 42, 8, 23, 1, 9, 30);

        // PriorityQueue com ordem natural: poll() sempre retorna o MENOR valor disponível
        Queue<Integer> filaPrioridade = new PriorityQueue<>(valores);

        for (int i = 0; i < 3; i++) {
            System.out.println(filaPrioridade.poll());
        }
        // Saída: 1, 3, 8
    }
}
```

_Raciocínio:_ o construtor de `PriorityQueue` aceita uma `Collection` inteira e já organiza tudo internamente (usando uma estrutura de heap binário, que veremos com mais profundidade na Trilha 2 de DSA). Cada chamada de `poll()` remove e retorna o menor valor **atual** da estrutura, reorganizando o heap internamente pra que o próximo `poll()` já saiba qual é o novo menor — é exatamente esse comportamento que permite extrair "os N menores" sem precisar ordenar a coleção inteira de uma vez, o que é mais eficiente quando você só precisa de alguns dos menores/maiores valores, não da lista toda ordenada.