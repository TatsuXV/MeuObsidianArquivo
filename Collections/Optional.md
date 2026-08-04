### 1. Teoria

**`Optional<T>`** é uma classe (não uma interface, e tecnicamente nem faz parte do "Collections Framework" propriamente dito, mas está no mesmo pacote `java.util` e é tratada junto por convenção) introduzida no Java 8 pra resolver um problema específico: **representar explicitamente a ausência de um valor**, sem usar `null`.

**O problema que `Optional` resolve:** antes dele, um método que "podia não ter resultado" simplesmente retornava `null` — e não havia nenhuma indicação no tipo de retorno de que isso era possível. Quem chamava o método precisava _lembrar_ de checar `null`, e esquecer isso é a causa mais comum de `NullPointerException` em Java (tão comum que ganhou o apelido de "bilion dollar mistake", cunhado pelo próprio criador do conceito de referência nula). `Optional<T>` transforma essa possibilidade de ausência em parte **visível do contrato do método** — o tipo de retorno `Optional<Usuario>` já avisa: "esse método pode não encontrar um usuário, trate esse caso".

`Optional` é essencialmente um "envelope" que pode estar vazio ou conter um valor:

|Método|O que faz|
|---|---|
|`Optional.of(valor)`|Cria um Optional com um valor **não-nulo** garantido (lança `NullPointerException` se você passar `null` aqui)|
|`Optional.ofNullable(valor)`|Cria um Optional que pode ou não ter valor — se `valor` for `null`, cria um Optional vazio, sem lançar exceção|
|`Optional.empty()`|Cria um Optional explicitamente vazio|
|`isPresent()`|Retorna `true` se tem valor|
|`isEmpty()`|Retorna `true` se **não** tem valor (Java 11+)|
|`get()`|Retorna o valor — lança `NoSuchElementException` se estiver vazio|
|`orElse(padrao)`|Retorna o valor, ou o `padrao` se estiver vazio|
|`orElseGet(supplier)`|Retorna o valor, ou executa uma função (lazy) se estiver vazio|
|`orElseThrow(...)`|Retorna o valor, ou lança uma exceção customizada se estiver vazio|
|`ifPresent(consumer)`|Executa uma ação **só se** houver valor|
|`map(funcao)`|Transforma o valor, se presente (retorna outro `Optional`)|
|`filter(predicado)`|Mantém o valor só se atender à condição, senão vira vazio|

**Não confunda `orElse` com `orElseGet`:** `orElse(calcularPadrao())` **sempre executa** `calcularPadrao()`, mesmo quando o Optional tem valor (porque o argumento é avaliado antes de ser passado, como qualquer chamada de método em Java). `orElseGet(() -> calcularPadrao())` só executa a lambda **se realmente precisar** do valor padrão. Se calcular o padrão for uma operação custosa (ex: consultar banco de dados), essa diferença importa de verdade em performance.

**Um princípio de design importante (frequentemente mal utilizado na prática):** `Optional` foi desenhado principalmente para ser usado como **tipo de retorno** de método, sinalizando "isso pode não existir". A documentação oficial e a comunidade **desaconselham fortemente** usar `Optional` como: tipo de campo de classe, tipo de parâmetro de método, ou dentro de coleções (`List<Optional<T>>`). Nesses casos, existem alternativas melhores (valor padrão, sobrecarga de método, simplesmente não incluir o elemento na coleção).

**Onde aparece no dia a dia de backend:** é _extremamente_ comum em Spring Data JPA — métodos como `findById(id)` em repositórios retornam `Optional<Entidade>`, exatamente porque buscar por ID pode não encontrar nada, e o framework força você a lidar com esse caso explicitamente em vez de arriscar um `null` esquecido se propagando pelo código.

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class OptionalExample {

    // Simula um "repositório" que pode ou não encontrar um usuário - padrão real do Spring Data JPA
    static Optional<String> buscarUsuarioPorId(int id) {
        Map<Integer, String> banco = Map.of(1, "Ana", 2, "Bruno", 3, "Carla");
        // ofNullable: se banco.get(id) retornar null (id não existe), o Optional fica vazio
        return Optional.ofNullable(banco.get(id));
    }

    public static void main(String[] args) {
        // Caso 1: usuário existe
        Optional<String> usuario1 = buscarUsuarioPorId(1);
        System.out.println("Presente? " + usuario1.isPresent()); // true

        // orElse: fornece um valor padrão se estiver vazio
        String nome1 = usuario1.orElse("Usuário desconhecido");
        System.out.println(nome1); // Ana

        // Caso 2: usuário NÃO existe
        Optional<String> usuario2 = buscarUsuarioPorId(99);
        String nome2 = usuario2.orElse("Usuário desconhecido");
        System.out.println(nome2); // Usuário desconhecido

        // ifPresent: executa uma ação só se houver valor - sem precisar checar isPresent() antes
        buscarUsuarioPorId(2).ifPresent(nome -> System.out.println("Encontrado: " + nome));
        buscarUsuarioPorId(99).ifPresent(nome -> System.out.println("Não vai imprimir isso"));

        // map: transforma o valor, se presente, encadeando de forma segura
        Optional<Integer> tamanhoDoNome = buscarUsuarioPorId(1).map(String::length);
        System.out.println(tamanhoDoNome.orElse(0)); // 3 (tamanho de "Ana")

        // orElseThrow: lança exceção customizada se vazio - comum em regras de negócio obrigatórias
        try {
            String nomeObrigatorio = buscarUsuarioPorId(99)
                .orElseThrow(() -> new NoSuchElementException("Usuário não encontrado"));
        } catch (NoSuchElementException e) {
            System.out.println("Erro capturado: " + e);
        }

        // get() sem checar antes - ARMADILHA, evite isso (ver seção de armadilhas)
        try {
            String perigoso = buscarUsuarioPorId(99).get();
        } catch (NoSuchElementException e) {
            System.out.println("get() em Optional vazio: " + e);
        }

        // orElse vs orElseGet - diferença de quando o "padrão" é executado
        System.out.println("--- orElse sempre avalia o argumento ---");
        usuario1.orElse(metodoCaro()); // metodoCaro() RODA, mesmo que usuario1 tenha valor

        System.out.println("--- orElseGet só avalia se necessário ---");
        usuario1.orElseGet(() -> metodoCaro()); // metodoCaro() NÃO roda, porque usuario1 tem valor
    }

    static String metodoCaro() {
        System.out.println("Calculando valor padrão caro...");
        return "padrão";
    }
}
```

---

### 3. Armadilhas comuns

1. **Chamar `.get()` sem checar `isPresent()`/`isEmpty()` antes** — isso reproduz exatamente o mesmo problema que `Optional` foi criado pra evitar: se o Optional estiver vazio, `.get()` lança `NoSuchElementException`. Usar `.get()` direto, sem tratamento, é basicamente voltar a programar como se fosse `null`, só que com uma exceção diferente. Prefira sempre `orElse`, `orElseGet`, `orElseThrow`, `ifPresent`, ou `map`.
2. **Usar `Optional` como tipo de campo de classe ou parâmetro de método** — vai contra a intenção de design da própria API (mencionado na Teoria). `Optional` não implementa `Serializable`, por exemplo, o que já causa problemas se você tentar usá-lo como campo de uma entidade JPA. Use `Optional` como **tipo de retorno**, não como forma geral de representar "valor opcional" em qualquer contexto.
3. **Confundir `orElse(valorPadrao)` com `orElseGet(() -> valorPadrao)` quando o cálculo do padrão tem efeito colateral ou é custoso** — como vimos no exemplo, `orElse` sempre executa o argumento (é avaliação "eager"), mesmo quando o valor já está presente e o padrão nunca vai ser usado. Isso pode causar chamadas desnecessárias (ex: consulta a banco de dados) mesmo em caminhos onde o Optional já tinha valor.
4. **Fazer `if (optional.isPresent()) { optional.get()... }`** — funciona, mas é um "code smell": você voltou a escrever o padrão imperativo de checagem manual que `Optional` tenta te tirar. Na maioria dos casos, `map`, `ifPresent`, ou `orElseGet` expressam a mesma lógica de forma mais direta e menos propensa a erro (esquecer o `isPresent()` antes do `get()` em outro lugar do código, por exemplo).

---

### 4. Exercícios práticos

**1. Fácil**  
Escreva um método `static Optional<Integer> dividir(int a, int b)` que retorna o resultado da divisão inteira `a / b` dentro de um `Optional`, mas retorna `Optional.empty()` se `b` for zero (em vez de lançar `ArithmeticException`). Teste chamando com `dividir(10, 2)` e `dividir(10, 0)`, usando `orElse(-1)` para imprimir o resultado (ou `-1` no caso de divisão por zero). Critério de pronto: `dividir(10, 2)` deve resultar em `5`; `dividir(10, 0)` deve resultar em `-1`, sem lançar exceção.

**2. Fácil/Médio**  
Dado `Map<String, String> config = Map.of("timeout", "30", "modoDebug", "true");`, escreva um método `static Optional<Integer> buscarTimeout(Map<String, String> config)` que busca a chave `"timeout"`, e se encontrar, converte o valor de `String` pra `Integer` usando `.map()` (não usando `if/else` manual). Use `ifPresent` para imprimir `"Timeout configurado: X"` se existir, e não imprima nada se não existir. Critério de pronto: com a chave presente, deve imprimir `"Timeout configurado: 30"`; testando com uma chave inexistente (`"retries"`), nada deve ser impresso.

**3. Médio**  
Escreva um método `static Optional<String> validarEmail(String email)` que retorna o próprio email dentro de um `Optional` **somente se** ele contiver o caractere `"@"` **e** tiver mais de 5 caracteres (use `.filter()` encadeado, não `if`). Teste com um email válido, um email sem `@`, e um email válido mas muito curto (ex: `"a@b"`). Use `orElseThrow` para lançar uma `IllegalArgumentException` com a mensagem `"Email inválido"` quando a validação falhar, e capture essa exceção no teste pra confirmar que funciona. Critério de pronto: `"usuario@teste.com"` deve passar; `"usuarioteste.com"` e `"a@b"` devem lançar a exceção.

**4. Difícil/Desafio**  
Simule um cenário de "busca em cascata": você tem três "fontes de dados" hipotéticas, cada uma representada por um método `Optional<String> buscarNoCacheLocal(String chave)`, `Optional<String> buscarNoBancoDados(String chave)`, `Optional<String> buscarNaApiExterna(String chave)` (implemente cada um retornando um valor fixo pra algumas chaves de teste e `Optional.empty()` para as demais, simulando "encontrou"/"não encontrou"). Escreva um método `static Optional<String> buscarComFallback(String chave)` que tenta as três fontes **em ordem**, retornando o primeiro resultado não-vazio encontrado (dica: existe um método de `Optional` chamado `or()`, do Java 9+, feito exatamente pra encadear fallbacks assim — pesquise a assinatura dele se não tiver certeza, e verifique a partir de qual versão ele existe antes de usar). Critério de pronto: se a chave existir só na "API externa", o método deve retornar esse valor mesmo com cache e banco vazios; se não existir em nenhuma fonte, deve retornar `Optional.empty()`.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    static Optional<Integer> dividir(int a, int b) {
        if (b == 0) {
            return Optional.empty();
        }
        return Optional.of(a / b);
    }

    public static void main(String[] args) {
        System.out.println(dividir(10, 2).orElse(-1)); // 5
        System.out.println(dividir(10, 0).orElse(-1)); // -1
    }
}
```

_Raciocínio:_ esse é o caso de uso mais simples e direto de `Optional` — transformar uma operação que poderia lançar exceção (`ArithmeticException` de divisão por zero) numa ausência de valor explícita e tratável, sem try/catch no lado de quem chama. Usamos `Optional.of(...)` (não `ofNullable`) porque sabemos, dentro do próprio método, que o resultado de `a / b` nunca é `null` (é um `int` primitivo, virando `Integer` por autoboxing) — a única forma de "não ter resultado" aqui é o caso `b == 0`, que tratamos explicitamente antes.

**Exercício 2**

java

```java
public class Exercicio2 {
    static Optional<Integer> buscarTimeout(Map<String, String> config) {
        return Optional.ofNullable(config.get("timeout"))
                        .map(Integer::parseInt);
    }

    public static void main(String[] args) {
        Map<String, String> config = Map.of("timeout", "30", "modoDebug", "true");

        buscarTimeout(config).ifPresent(t -> System.out.println("Timeout configurado: " + t));
        // Timeout configurado: 30

        Map<String, String> configSemTimeout = Map.of("modoDebug", "true");
        buscarTimeout(configSemTimeout).ifPresent(t -> System.out.println("Não deveria imprimir"));
        // nada é impresso
    }
}
```

_Raciocínio:_ `Optional.ofNullable(config.get("timeout"))` lida com o `null` que `Map.get()` retornaria pra chave ausente, transformando isso num `Optional` vazio de forma segura. `.map(Integer::parseInt)` só executa a conversão **se** houver valor — encadeando a transformação sem precisar de um `if` explícito checando presença antes de converter. Isso é o poder de `map`: ele "atravessa" o Optional, aplicando a função só quando faz sentido, e propaga o vazio automaticamente se não houver nada pra transformar.

**Exercício 3**

java

```java
public class Exercicio3 {
    static Optional<String> validarEmail(String email) {
        return Optional.of(email)
                        .filter(e -> e.contains("@"))
                        .filter(e -> e.length() > 5);
    }

    public static void main(String[] args) {
        String[] testes = {"usuario@teste.com", "usuarioteste.com", "a@b"};

        for (String email : testes) {
            try {
                String valido = validarEmail(email)
                    .orElseThrow(() -> new IllegalArgumentException("Email inválido"));
                System.out.println("Válido: " + valido);
            } catch (IllegalArgumentException e) {
                System.out.println("Rejeitado (" + email + "): " + e.getMessage());
            }
        }
    }
}
```

_Raciocínio:_ `.filter()` encadeado é o equivalente a múltiplas condições `&&`, mas expresso "atravessando" o Optional — cada `.filter()` mantém o valor se a condição for verdadeira, ou transforma o Optional em vazio se for falsa (e uma vez vazio, os `.filter()` seguintes não fazem mais nada, o vazio simplesmente propaga). É por isso que dois `.filter()` seguidos funcionam como um "E" lógico: só sobrevive até o `orElseThrow` final quem passou pelas duas condições. `"a@b"` tem `@` mas só 3 caracteres, então falha no segundo filtro; `"usuarioteste.com"` não tem `@`, falha no primeiro.

**Exercício 4**

java

```java
public class Exercicio4 {
    static Optional<String> buscarNoCacheLocal(String chave) {
        return "chaveNoCache".equals(chave) ? Optional.of("valorDoCache") : Optional.empty();
    }

    static Optional<String> buscarNoBancoDados(String chave) {
        return "chaveNoBanco".equals(chave) ? Optional.of("valorDoBanco") : Optional.empty();
    }

    static Optional<String> buscarNaApiExterna(String chave) {
        return "chaveNaApi".equals(chave) ? Optional.of("valorDaApi") : Optional.empty();
    }

    static Optional<String> buscarComFallback(String chave) {
        return buscarNoCacheLocal(chave)
                .or(() -> buscarNoBancoDados(chave))
                .or(() -> buscarNaApiExterna(chave));
    }

    public static void main(String[] args) {
        System.out.println(buscarComFallback("chaveNaApi"));      // Optional[valorDaApi]
        System.out.println(buscarComFallback("chaveNoBanco"));    // Optional[valorDoBanco]
        System.out.println(buscarComFallback("chaveInexistente")); // Optional.empty
    }
}
```

_Raciocínio:_ confirmei antes de usar: `Optional.or(Supplier<? extends Optional<? extends T>>)` foi introduzido no **Java 9** — se o Optional que chama `or()` já tiver valor, ele é retornado imediatamente e o `Supplier` passado **nem é executado** (avaliação lazy, igual `orElseGet`); se estiver vazio, o `Supplier` é executado e seu resultado (outro `Optional`) é retornado no lugar. Encadear `.or(...)` várias vezes cria exatamente a cascata de fallback pedida: tenta cache, se vazio tenta banco, se vazio tenta API — cada fonte só é consultada se as anteriores realmente falharam, o que é importante numa cascata real (você não quer bater na API externa se já achou no cache, por exemplo).