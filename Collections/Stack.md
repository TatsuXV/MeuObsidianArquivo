### 1. Teoria

**`Stack`** representa uma coleção organizada pra processamento **LIFO** (Last In, First Out) — o último elemento a entrar é o primeiro a sair. Pense numa pilha de pratos: você só consegue tirar o de cima, que foi o último colocado.

Aqui tem uma pegadinha histórica importante: existe uma **classe `Stack`** (`java.util.Stack`), mas ela é considerada **legada** e a documentação oficial da Oracle recomenda **não usá-la** em código novo. Os motivos:

- `Stack` estende `Vector`, uma classe antiga e **sincronizada** (thread-safe), o que traz overhead de performance desnecessário se você não precisa de sincronização.
- Por herdar de `Vector` (que é uma `List`), `Stack` acaba expondo métodos de lista como `get(index)`, `add(index, elemento)` — que quebram a disciplina LIFO que uma pilha deveria garantir. Nada te impede de inserir no meio, o que não faz sentido conceitual pra uma pilha.

**A recomendação atual é usar `Deque` (via `ArrayDeque`) como pilha**, chamando os métodos específicos de pilha que `Deque` oferece:

|Operação de pilha|Método em `Deque`|
|---|---|
|Empilhar (push)|`push(e)`|
|Desempilhar (pop)|`pop()`|
|Olhar o topo sem remover|`peek()`|

Por baixo dos panos, `push()` insere no início do `Deque` e `pop()`/`peek()` operam sobre esse mesmo início — então o "topo da pilha" é sempre a cabeça do `Deque`.

**Não confunda com `Queue`:** a diferença central entre `Stack` e `Queue` é só a ordem de saída — LIFO vs FIFO. A mesma classe `ArrayDeque` serve pra implementar as duas coisas, dependendo de quais métodos você usa (`offer`/`poll` para fila; `push`/`pop` para pilha).

**Onde aparece no dia a dia de backend:** validação de parênteses/expressões balanceadas, implementação de "desfazer" (undo), controle de call stack em parsers/interpretadores, e principalmente: entender pilha é pré-requisito direto pra entender **recursão** (cada chamada de função empilha um frame na call stack do próprio Java) — assunto que será aprofundado na Trilha 2 de DSA.

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class StackExample {
    public static void main(String[] args) {
        // Deque como Stack - a forma recomendada atualmente (NÃO usar java.util.Stack)
        Deque<String> pilhaHistorico = new ArrayDeque<>();

        // push: empilha no topo
        pilhaHistorico.push("Página Inicial");
        pilhaHistorico.push("Produtos");
        pilhaHistorico.push("Carrinho");

        System.out.println(pilhaHistorico); // [Carrinho, Produtos, Página Inicial] - topo primeiro

        // peek: olha o topo, sem remover
        System.out.println("Página atual: " + pilhaHistorico.peek()); // Carrinho

        // pop: remove e retorna o topo (LIFO)
        String paginaAnterior = pilhaHistorico.pop();
        System.out.println("Voltando de: " + paginaAnterior); // Carrinho
        System.out.println("Pilha agora: " + pilhaHistorico); // [Produtos, Página Inicial]

        // pop numa pilha vazia LANÇA exceção (diferente de poll() de Queue, que retorna null)
        Deque<String> pilhaVazia = new ArrayDeque<>();
        try {
            pilhaVazia.pop();
        } catch (NoSuchElementException e) {
            System.out.println("Erro capturado: " + e);
        }

        // Exemplo clássico: validar parênteses balanceados usando pilha
        System.out.println(parentesesBalanceados("(a(b)c)")); // true
        System.out.println(parentesesBalanceados("(a(b)c"));  // false
    }

    static boolean parentesesBalanceados(String expressao) {
        Deque<Character> pilha = new ArrayDeque<>();

        for (char c : expressao.toCharArray()) {
            if (c == '(') {
                pilha.push(c); // empilha toda abertura
            } else if (c == ')') {
                // se fecha um parêntese mas a pilha está vazia, não há abertura correspondente
                if (pilha.isEmpty()) {
                    return false;
                }
                pilha.pop(); // fechamento "consome" a abertura mais recente
            }
        }

        // se sobrou algo na pilha, tem abertura sem fechamento
        return pilha.isEmpty();
    }
}
```

---

### 3. Armadilhas comuns

1. **Usar `java.util.Stack` em código novo** — funciona, mas é legado, sincronizado (overhead desnecessário na maioria dos casos) e permite operações de `List` que quebram a semântica de pilha. A recomendação atual (inclusive da documentação oficial) é `Deque`/`ArrayDeque`.
2. **Confundir `pop()` (lança exceção em pilha vazia) com `poll()` de Queue (retorna `null`)** — são interfaces diferentes com contratos diferentes. Se você usa `Deque` como pilha, `pop()`/`push()` seguem o padrão "exception-throwing"; não existe um "poll de pilha" com o mesmo nome — o equivalente seria checar `isEmpty()` antes, ou usar `pollFirst()` diretamente se quiser a versão segura.
3. **Achar que imprimir a pilha (`System.out.println(pilha)`) mostra a ordem de desempilhamento visualmente invertida** — quando você imprime um `ArrayDeque` usado como pilha, o elemento mais recente (o topo) aparece **primeiro** na saída, porque `push()` insere na cabeça do Deque. Isso é intuitivo uma vez que você sabe, mas confunde quem espera ver o topo como "último da lista".
4. **Esquecer de checar `isEmpty()` antes de `pop()`/`peek()` quando o esvaziamento é uma condição possível e esperada** (como no algoritmo de parênteses balanceados acima) — chamar `pop()` numa pilha que pode estar vazia sem checar antes é a causa mais comum de `NoSuchElementException` inesperado nesse tipo de algoritmo.

---

### 4. Exercícios práticos

**1. Fácil**  
Crie uma pilha (`Deque<Integer>` usando `ArrayDeque`) e empilhe os valores `1, 2, 3, 4, 5` nessa ordem (usando `push`). Depois, desempilhe todos os valores um a um (usando `pop`) e imprima cada um. Critério de pronto: a ordem de saída deve ser `5, 4, 3, 2, 1` — exatamente o inverso da ordem de inserção.

**2. Fácil/Médio**  
Usando uma pilha, inverta uma `String` sem usar `StringBuilder.reverse()` nem nenhum método pronto de inversão — empilhe cada caractere da string original, depois desempilhe tudo formando a string resultante. Critério de pronto: `inverter("java")` deve retornar `"avaj"`.

**3. Médio**  
Implemente um verificador de expressões balanceadas mais completo que o do exemplo — dessa vez, considerando **três tipos de delimitadores**: `()`, `[]` e `{}`. A expressão só é válida se todos os delimitadores estiverem corretamente abertos e fechados, **na ordem certa** (ex: `"([)]"` deve ser inválido, mesmo tendo a mesma quantidade de cada delimitador, porque a ordem de fechamento está errada). Critério de pronto: `"{[a(b)c]d}"` → `true`; `"([)]"` → `false`; `"(a]"` → `false`.

**4. Difícil/Desafio**  
Implemente uma pilha que, além das operações normais (`push`, `pop`, `peek`), também tenha um método `int minimo()` que retorna o **menor valor atualmente na pilha em tempo O(1)** (sem percorrer a pilha inteira toda vez que `minimo()` é chamado). Dica: você vai precisar de uma segunda estrutura auxiliar (outra pilha) que acompanha, em paralelo, qual é o mínimo em cada "nível" da pilha principal. Critério de pronto: empilhando `5, 2, 8, 1, 9` nessa ordem, `minimo()` deve retornar `1`; depois de um `pop()` (removendo o `9`)... espere, `9` foi o último empilhado, então o `pop()` remove o `9` e `minimo()` continua `1`; faça mais um `pop()` (removendo o `1`) e `minimo()` deve voltar a retornar `2`.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        Deque<Integer> pilha = new ArrayDeque<>();
        for (int i = 1; i <= 5; i++) {
            pilha.push(i);
        }

        while (!pilha.isEmpty()) {
            System.out.println(pilha.pop());
        }
        // Saída: 5, 4, 3, 2, 1
    }
}
```

_Raciocínio:_ essa é a demonstração mais direta do comportamento LIFO — o último `push()` (valor `5`) é necessariamente o primeiro `pop()`, porque cada `push` empilha por cima do anterior, e `pop` sempre tira do topo.

**Exercício 2**

java

```java
public class Exercicio2 {
    static String inverter(String texto) {
        Deque<Character> pilha = new ArrayDeque<>();

        for (char c : texto.toCharArray()) {
            pilha.push(c);
        }

        StringBuilder resultado = new StringBuilder();
        while (!pilha.isEmpty()) {
            resultado.append(pilha.pop());
        }

        return resultado.toString();
    }

    public static void main(String[] args) {
        System.out.println(inverter("java")); // avaj
    }
}
```

_Raciocínio:_ empilhar cada caractere na ordem original e depois desempilhar tudo produz naturalmente a ordem inversa — é uma ilustração direta de por que pilha é a estrutura clássica pra esse tipo de problema de inversão. (Usamos `StringBuilder` só para _construir_ a string de saída de forma eficiente, não pra inverter — a inversão em si é feita inteiramente pela pilha.)

**Exercício 3**

java

```java
public class Exercicio3 {
    static boolean delimitadoresBalanceados(String expressao) {
        Deque<Character> pilha = new ArrayDeque<>();
        Map<Character, Character> pares = Map.of(
            ')', '(',
            ']', '[',
            '}', '{'
        );

        for (char c : expressao.toCharArray()) {
            if (c == '(' || c == '[' || c == '{') {
                pilha.push(c); // qualquer abertura vai pra pilha
            } else if (pares.containsKey(c)) {
                // c é um fechamento - precisa bater com a abertura correspondente no topo
                if (pilha.isEmpty() || pilha.pop() != pares.get(c)) {
                    return false;
                }
            }
            // qualquer outro caractere (letras, etc.) é ignorado
        }

        return pilha.isEmpty(); // se sobrou abertura sem fechamento, é inválido
    }

    public static void main(String[] args) {
        System.out.println(delimitadoresBalanceados("{[a(b)c]d}")); // true
        System.out.println(delimitadoresBalanceados("([)]"));        // false
        System.out.println(delimitadoresBalanceados("(a]"));         // false
    }
}
```

_Raciocínio:_ a ideia central é: toda abertura vai pra pilha; todo fechamento precisa corresponder **exatamente** à abertura que está no topo da pilha naquele momento (não só existir em algum lugar da pilha). O `Map<Character, Character>` associa cada fechamento ao seu par de abertura esperado, então `pilha.pop() != pares.get(c)` verifica: "o que eu tirei do topo é realmente a abertura que esse fechamento espera?" — se não for (como em `"([)]"`, onde o `)` fecha esperando `(` mas o topo da pilha é `[`), a expressão é inválida. É esse detalhe — comparar contra o topo, não contra "existe em algum lugar" — que resolve o caso de ordem errada que uma simples contagem de parênteses não pegaria.

**Exercício 4**

java

```java
public class Exercicio4 {
    private Deque<Integer> pilhaPrincipal = new ArrayDeque<>();
    private Deque<Integer> pilhaMinimos = new ArrayDeque<>();

    void push(int valor) {
        pilhaPrincipal.push(valor);
        // empilha na pilha auxiliar o menor valor "até agora"
        if (pilhaMinimos.isEmpty() || valor <= pilhaMinimos.peek()) {
            pilhaMinimos.push(valor);
        } else {
            pilhaMinimos.push(pilhaMinimos.peek()); // repete o mínimo atual
        }
    }

    int pop() {
        pilhaMinimos.pop(); // desempilha em sincronia com a principal
        return pilhaPrincipal.pop();
    }

    int minimo() {
        return pilhaMinimos.peek(); // O(1) - já está pronto no topo da auxiliar
    }

    public static void main(String[] args) {
        Exercicio4 pilha = new Exercicio4();
        pilha.push(5);
        pilha.push(2);
        pilha.push(8);
        pilha.push(1);
        pilha.push(9);

        System.out.println(pilha.minimo()); // 1

        pilha.pop(); // remove o 9
        System.out.println(pilha.minimo()); // 1

        pilha.pop(); // remove o 1
        System.out.println(pilha.minimo()); // 2
    }
}
```

_Raciocínio:_ o truque é manter as duas pilhas **sempre do mesmo tamanho, em sincronia** — pra cada `push` na principal, empilhamos também na `pilhaMinimos` (seja o novo valor, se ele for o novo mínimo, seja uma repetição do mínimo atual, se não for). Isso garante que, em qualquer momento, o topo da `pilhaMinimos` reflete exatamente qual é o mínimo da pilha principal **naquele nível específico**. Quando fazemos `pop()`, desempilhamos as duas juntas — então o novo topo de `pilhaMinimos` automaticamente "volta" pro mínimo correto de antes daquele push, sem precisar recalcular nada. É essa sincronia que garante o O(1) em `minimo()`: nunca precisamos percorrer a pilha inteira, só consultar o topo da auxiliar.