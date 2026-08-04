### 1. Teoria

**`Map`** é uma interface do Collections Framework que representa uma coleção de pares **chave-valor**. Diferente de `List` e `Set`, `Map` **não estende `Collection`** — é uma interface separada (motivo histórico + conceitual: você não itera diretamente sobre um `Map`, você itera sobre suas chaves, valores ou entradas).

Cada chave é única (se você inserir uma chave já existente, o valor antigo é **sobrescrito**, não duplicado). Os valores podem se repetir livremente.

As três implementações principais:

|Implementação|Ordem|Performance (get/put/remove)|Permite chave/valor `null`?|
|---|---|---|---|
|`HashMap`|Nenhuma garantida|O(1) médio|1 chave `null` permitida, valores `null` sim|
|`LinkedHashMap`|Ordem de inserção|O(1) médio|Mesmo que HashMap|
|`TreeMap`|Ordenada por chave (natural ou `Comparator`)|O(log n)|Chave `null` não permitida|

**Mecanismo interno (importante de verdade entender, não decorar):** um `HashMap` usa o `hashCode()` da chave pra calcular em qual "bucket" (posição interna do array) aquele par vai morar. Quando você faz `get(chave)`, ele recalcula o hash, vai direto no bucket certo, e usa `equals()` pra confirmar qual entrada, dentre as que colidiram naquele bucket, é a certa. É exatamente o mesmo mecanismo do `HashSet` — não é coincidência: **`HashSet` é implementado internamente usando um `HashMap`**, onde cada elemento do Set vira uma chave do Map (com um valor-sentinela fixo por trás).

**Diferenças de List/Set:**

- Não tem `add()` — tem `put(chave, valor)`.
- Não tem `contains()` sozinho — tem `containsKey()` e `containsValue()` (que são bem diferentes em custo: `containsKey` é O(1), `containsValue` é O(n), porque precisa varrer todos os valores).
- Iteração precisa de `.keySet()`, `.values()` ou `.entrySet()`.

**Onde aparece no dia a dia de backend:** é uma das estruturas mais usadas de todas — cache em memória (`Map<Long, Usuario>` guardando usuário por ID), contagem/agrupamento (`Map<String, Integer>` contando ocorrências), configuração (`Map<String, String>` de properties), resposta de endpoints simples (`Map<String, Object>` como JSON ad-hoc), e por baixo de praticamente todo mecanismo de cache do Spring (`@Cacheable` usa isso conceitualmente).

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class MapExample {
    public static void main(String[] args) {
        // HashMap: sem ordem garantida
        Map<String, Integer> idadePorNome = new HashMap<>();
        idadePorNome.put("Ana", 28);
        idadePorNome.put("Bruno", 35);
        idadePorNome.put("Carla", 22);

        // put com chave já existente SOBRESCREVE o valor
        idadePorNome.put("Ana", 29); // Ana agora tem 29, não duas entradas

        System.out.println(idadePorNome);
        System.out.println("Tamanho: " + idadePorNome.size()); // 3

        // get: retorna null se a chave não existir (cuidado com NullPointerException depois)
        Integer idadeDaAna = idadePorNome.get("Ana"); // 29
        Integer idadeDoInexistente = idadePorNome.get("Zeca"); // null, sem exceção

        // getOrDefault: forma segura de evitar null inesperado
        int idadeSegura = idadePorNome.getOrDefault("Zeca", 0); // 0

        // containsKey vs containsValue - custo bem diferente (O(1) vs O(n))
        boolean temAna = idadePorNome.containsKey("Ana"); // true, O(1)
        boolean temIdade35 = idadePorNome.containsValue(35); // true, mas varre tudo, O(n)

        // Iterando: três formas
        System.out.println("--- Só chaves ---");
        for (String nome : idadePorNome.keySet()) {
            System.out.println(nome);
        }

        System.out.println("--- Só valores ---");
        for (Integer idade : idadePorNome.values()) {
            System.out.println(idade);
        }

        System.out.println("--- Chave e valor juntos (forma mais eficiente) ---");
        for (Map.Entry<String, Integer> entry : idadePorNome.entrySet()) {
            System.out.println(entry.getKey() + " tem " + entry.getValue() + " anos");
        }

        // merge: útil pra padrão de "contar ocorrências"
        Map<String, Integer> contagem = new HashMap<>();
        List<String> palavras = List.of("java", "python", "java", "go", "java", "python");
        for (String palavra : palavras) {
            // se a chave não existe, usa 1 como valor inicial (o "1" do put)
            // se já existe, soma 1 ao valor atual (a lambda recebe valorAtual, e retorna valorAtual + 1)
            contagem.merge(palavra, 1, Integer::sum);
        }
        System.out.println(contagem); // {java=3, python=2, go=1} (ordem pode variar)
    }
}
```

---

### 3. Armadilhas comuns

1. **Chamar `get()` numa chave que não existe e assumir que nunca vai dar `null`** — `get()` retorna `null` silenciosamente se a chave não existe (sem lançar exceção). Se você fizer `map.get("chaveErrada").metodoQualquer()`, o `NullPointerException` estoura ali, não no `get()`. Prefira `getOrDefault()` quando fizer sentido ter um valor padrão.
2. **Usar objeto customizado como chave sem sobrescrever `equals()`/`hashCode()`** — exatamente o mesmo problema do `HashSet`: sem isso, duas chaves "logicamente iguais" (mesmos dados) são tratadas como chaves diferentes, e você nunca encontra o valor que esperava.
3. **Confundir `containsValue()` com `containsKey()` em termos de custo** — usar `containsValue()` num `HashMap` grande dentro de um loop é um jeito silencioso de criar um gargalo O(n²) sem perceber.
4. **Modificar o Map enquanto itera com `for-each` direto** — igual ao que vimos em `List`/`Set`: fazer `map.remove(chave)` dentro de um `for (String chave : map.keySet())` lança `ConcurrentModificationException`. A forma segura é iterar sobre `entrySet()` com um `Iterator` explícito e chamar `iterator.remove()`, ou usar `map.entrySet().removeIf(...)`.

---

### 4. Exercícios práticos

**1. Fácil**  
Crie um `HashMap<String, String>` representando capitais de países: `"Brasil" -> "Brasília"`, `"França" -> "Paris"`, `"Japão" -> "Tóquio"`. Depois, tente buscar a capital de `"Alemanha"` (que não está no map) usando `getOrDefault`, com valor padrão `"Desconhecida"`. Imprima o resultado. Critério de pronto: deve imprimir `"Desconhecida"` sem lançar exceção.

**2. Fácil/Médio**  
Dada a frase `String texto = "o rato roeu a roupa do rei de roma";`, conte quantas vezes cada palavra aparece, guardando o resultado num `Map<String, Integer>`. Imprima o resultado ordenado alfabeticamente pela palavra (dica: pense em qual implementação de `Map` já ordena por chave automaticamente). Critério de pronto: `"roma"` deve aparecer com contagem 1 e `"roeu"` com contagem 1, e a saída deve estar em ordem alfabética.

**3. Médio**  
Você tem um `Map<String, List<String>>` representando departamentos e a lista de funcionários de cada um (ex: `"TI" -> ["Ana", "Bruno"]`, `"RH" -> ["Carla"]`). Escreva um método `void adicionarFuncionario(Map<String, List<String>> departamentos, String depto, String funcionario)` que adiciona o funcionário à lista do departamento — mas se o departamento ainda não existir no Map, ele deve **criar a lista automaticamente** antes de adicionar (sem lançar `NullPointerException`, e sem usar `if/else` explícito — dica: existe um método do `Map` feito exatamente pra esse padrão). Critério de pronto: chamar `adicionarFuncionario(deptos, "Financeiro", "Diego")` num Map onde `"Financeiro"` não existe ainda deve criar a entrada e funcionar sem erro.

**4. Difícil/Desafio**  
Dado `Map<String, Integer> vendas = Map.of("Ana", 15000, "Bruno", 22000, "Carla", 18000, "Diego", 22000, "Eva", 9000);` (vendedor → total vendido), encontre o(s) vendedor(es) com o **maior valor de vendas** — pode haver empate, então o resultado deve ser uma `List<String>` com todos os nomes empatados no topo (não assuma que só existe um vencedor). Não use bibliotecas de Stream/Collectors ainda (isso é assunto de um bloco futuro) — resolva só com estruturas de Map/List e loop. Critério de pronto: para o Map de exemplo acima, o resultado deve ser uma lista contendo exatamente `"Bruno"` e `"Diego"` (em qualquer ordem entre eles).

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        Map<String, String> capitais = new HashMap<>();
        capitais.put("Brasil", "Brasília");
        capitais.put("França", "Paris");
        capitais.put("Japão", "Tóquio");

        String capitalAlemanha = capitais.getOrDefault("Alemanha", "Desconhecida");
        System.out.println(capitalAlemanha); // Desconhecida
    }
}
```

_Raciocínio:_ `getOrDefault` evita o padrão repetitivo de `if (map.containsKey(x)) {...} else {...}` e, mais importante, evita esquecer de tratar o caso de chave ausente — o que levaria a um `null` se passando por aí até explodir em `NullPointerException` em outro lugar do código, longe da causa real.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        String texto = "o rato roeu a roupa do rei de roma";
        String[] palavras = texto.split(" ");

        // TreeMap ordena automaticamente pela chave (ordem natural de String = alfabética)
        Map<String, Integer> contagem = new TreeMap<>();

        for (String palavra : palavras) {
            contagem.merge(palavra, 1, Integer::sum);
        }

        for (Map.Entry<String, Integer> entry : contagem.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }
    }
}
```

_Raciocínio:_ a escolha de `TreeMap` em vez de `HashMap` resolve a ordenação "de graça" — não precisamos ordenar manualmente depois, porque `TreeMap` mantém a ordem das chaves sempre que você insere. `merge(chave, 1, Integer::sum)` é o padrão idiomático pra contagem: na primeira ocorrência da palavra, usa `1` como valor inicial; nas seguintes, soma `1` ao valor que já está lá, usando a função `Integer::sum` (que é só uma referência de método equivalente a `(a, b) -> a + b`).

**Exercício 3**

java

```java
public class Exercicio3 {
    static void adicionarFuncionario(Map<String, List<String>> departamentos, String depto, String funcionario) {
        // computeIfAbsent: se a chave "depto" não existir, cria a entrada usando a lambda
        // (aqui, uma ArrayList nova) e retorna essa lista - pronta pra usar
        departamentos.computeIfAbsent(depto, k -> new ArrayList<>()).add(funcionario);
    }

    public static void main(String[] args) {
        Map<String, List<String>> deptos = new HashMap<>();
        deptos.put("TI", new ArrayList<>(List.of("Ana", "Bruno")));
        deptos.put("RH", new ArrayList<>(List.of("Carla")));

        adicionarFuncionario(deptos, "Financeiro", "Diego"); // departamento novo
        adicionarFuncionario(deptos, "TI", "Eva"); // departamento existente

        System.out.println(deptos);
    }
}
```

_Raciocínio:_ `computeIfAbsent` é exatamente o método feito pra esse padrão comum ("map de listas"): ele verifica se a chave existe; se não existir, executa a lambda pra criar o valor, guarda no Map, e **retorna esse valor** (seja o recém-criado ou o que já existia) pronto pra você usar na sequência — nesse caso, encadeando o `.add()` direto. Isso evita o código repetitivo de checar `containsKey`, criar a lista manualmente, dar `put`, e só depois dar `add`.

**Exercício 4**

java

```java
public class Exercicio4 {
    static List<String> vendedoresComMaiorVenda(Map<String, Integer> vendas) {
        int maiorValor = Integer.MIN_VALUE;

        // Primeira passada: descobre qual é o maior valor de venda
        for (int valor : vendas.values()) {
            if (valor > maiorValor) {
                maiorValor = valor;
            }
        }

        // Segunda passada: coleta todos os nomes que batem com esse valor
        List<String> vencedores = new ArrayList<>();
        for (Map.Entry<String, Integer> entry : vendas.entrySet()) {
            if (entry.getValue() == maiorValor) {
                vencedores.add(entry.getKey());
            }
        }

        return vencedores;
    }

    public static void main(String[] args) {
        Map<String, Integer> vendas = Map.of(
            "Ana", 15000, "Bruno", 22000, "Carla", 18000, "Diego", 22000, "Eva", 9000
        );

        System.out.println(vendedoresComMaiorVenda(vendas)); // [Bruno, Diego] (ordem pode variar)
    }
}
```

_Raciocínio:_ a abordagem de **duas passadas** é a mais direta pra lidar com empate: numa passada só, você teria que decidir "no meio do caminho" se reseta a lista de vencedores ou adiciona a mais um — dá pra fazer numa passada só, mas fica mais propenso a erro. Aqui, a primeira passada isola o problema "qual é o maior valor" (usando só `.values()`, já que nome não importa nessa etapa), e a segunda passada isola o problema "quem tem esse valor" (usando `.entrySet()`, porque agora precisamos do par chave-valor). Separar em duas responsabilidades deixa o código mais fácil de ler e de garantir que está correto.