#### 1. Teoria

`Record` é um recurso de linguagem introduzido oficialmente no **Java 16** (depois de passar por preview no 14 e 15 — vale mencionar isso porque, se você ver código ou tutorial usando `record` em versões anteriores ao 16 com flag de preview, não é incompatibilidade sua, é a feature ainda não finalizada). Ele existe pra resolver um problema específico: **classes de dados imutáveis** (o padrão _Value Object_ que você já construiu manualmente no exercício de `Money`, na sessão de Encapsulamento) exigiam uma quantidade grande de código repetitivo — construtor, getters, `equals()`, `hashCode()`, `toString()` — só pra modelar "aqui estão alguns dados que andam juntos".

java

```java
public record Point(int x, int y) { }
```

Essa única linha gera automaticamente:

- Um construtor que recebe `x` e `y` (chamado _canonical constructor_).
- Campos `private final` pra cada componente.
- Métodos de acesso **sem prefixo `get`** — não `getX()`, mas `x()` (e `y()`).
- `equals()` e `hashCode()` comparando **todos** os componentes.
- `toString()` no formato `Point[x=5, y=10]`.

**Records são implicitamente `final`** — não podem ser estendidos por outra classe (embora possam implementar interfaces). Isso é intencional: o objetivo é modelar dados imutáveis simples, não criar uma nova hierarquia de herança.

**Todos os campos de um record são implicitamente `private final`.** Não existe jeito de declarar um componente mutável dentro de um record — se você precisa de estado mutável, `record` não é a ferramenta certa, volte pra `class` normal.

**Você pode customizar o construtor canônico pra validar (compact constructor):**

java

```java
public record Point(int x, int y) {
    public Point {
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException("Coordinates cannot be negative");
        }
        // não precisa escrever this.x = x; this.y = y; — o compilador faz isso
        // automaticamente DEPOIS desse bloco, num compact constructor.
    }
}
```

**Você também pode adicionar métodos extras** (mas não campos de instância novos além dos componentes declarados no cabeçalho):

java

```java
public record Point(int x, int y) {
    public double distanceFromOrigin() {
        return Math.sqrt(x * x + y * y);
    }
}
```

**Onde isso aparece de verdade no backend:** DTOs (Data Transfer Objects) — que você já viu mencionados na sessão de Encapsulamento como um caso legítimo de classe "burra" sem regra de negócio pesada — são o caso de uso mais comum de `record` em Spring moderno: um corpo de requisição/resposta de API REST (`record UserResponse(Long id, String name, String email) { }`) é quase sempre mais idiomático como record do que como classe tradicional com Lombok ou boilerplate manual, justamente porque é imutável por natureza e não precisa de identidade além dos próprios dados.

---

#### 2. Exemplo de código comentado

java

```java
// Record básico: gera construtor, getters sem "get", equals, hashCode, toString.
public record Point(int x, int y) { }

// Record com compact constructor validando invariante — mesma ideia do
// BankAccount da sessão de Encapsulamento, só que com sintaxe mais enxuta.
public record Percentage(double value) {
    public Percentage {
        if (value < 0 || value > 100) {
            throw new IllegalArgumentException("Percentage must be between 0 and 100");
        }
    }
}

// Record implementando interface — permitido, mesmo sendo implicitamente final.
public interface Shape2D {
    double area();
}

public record Rectangle(double width, double height) implements Shape2D {
    @Override
    public double area() {
        return width * height;
    }

    // Método extra, além dos componentes do cabeçalho — permitido.
    public boolean isSquare() {
        return width == height;
    }
}

public class Main {
    public static void main(String[] args) {
        Point p1 = new Point(3, 4);
        Point p2 = new Point(3, 4);

        System.out.println(p1.x());        // 3 — getter sem "get"
        System.out.println(p1);            // Point[x=3, y=4] — toString automático
        System.out.println(p1.equals(p2)); // true — equals compara os componentes, não a referência
        System.out.println(p1 == p2);       // false — objetos diferentes na memória

        Rectangle r = new Rectangle(5, 5);
        System.out.println(r.area());       // 25.0
        System.out.println(r.isSquare());   // true

        // Percentage(150) lançaria IllegalArgumentException — testando o compact constructor:
        try {
            new Percentage(150);
        } catch (IllegalArgumentException e) {
            System.out.println("Rejected: " + e.getMessage());
        }
    }
}
```

---

#### 3. Armadilhas comuns

- **Achar que o getter se chama `getX()`.** É `x()`, sem prefixo — diferente da convenção JavaBean tradicional (`getName()`/`setName()`). Isso pode confundir ferramentas antigas construídas em cima da convenção JavaBean (algumas libs de serialização mais antigas esperavam `getX`/`isX`), mas o ecossistema moderno (Jackson, por exemplo) já reconhece o padrão de record nativamente.
- **Tentar adicionar um campo de instância extra fora dos componentes do cabeçalho.** `record Point(int x, int y) { private int z; }` não compila — records só podem ter os campos declarados no cabeçalho (mais campos `static`, que são permitidos, já que não fazem parte do estado de instância).
- **Esquecer que `equals()`/`hashCode()` gerados comparam por _valor_, e assumir que isso sempre é o comportamento desejado.** Pra um DTO simples, isso é exatamente o que você quer. Mas se o record contém um campo que é, por exemplo, uma lista mutável, o `equals()` gerado compara a lista por conteúdo — o que pode ser surpreendente se dois records "iguais no momento da criação" divergirem depois porque alguém mutou a lista compartilhada por fora (o mesmo problema de vazamento de encapsulamento da sessão de Encapsulamento ainda se aplica a records, componentes mutáveis dentro de um record não ficam magicamente protegidos).
- **Tentar herdar de um record, ou fazer um record herdar de uma classe.** Nenhum dos dois é permitido — record é implicitamente `final`, e só pode "estender" `java.lang.Record` implicitamente (isso é reservado pra própria linguagem). Se você precisa de hierarquia de tipo com estado compartilhado, isso pede classe abstrata normal (sessão de Abstraction), não record.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie um record `Coordinate(double latitude, double longitude)`. No `main`, crie duas instâncias com os mesmos valores e imprima o resultado de `equals()` entre elas, e o resultado de `toString()` de uma delas.  
_Critério de pronto:_ `equals()` retorna `true` mesmo sendo objetos diferentes na memória; `toString()` mostra os dois campos automaticamente, sem você escrever esse método.

**Exercício 2 (Fácil/Médio)**  
Crie um record `Age(int years)` com um **compact constructor** que valida que `years` está entre `0` e `150` (inclusive), lançando `IllegalArgumentException` fora desse intervalo. Adicione um método extra `boolean isAdult()` que retorna `true` se `years >= 18`.  
_Critério de pronto:_ `new Age(-5)` e `new Age(200)` lançam exceção; `new Age(25).isAdult()` retorna `true`; `new Age(10).isAdult()` retorna `false`.

**Exercício 3 (Médio/Difícil)**  
Crie uma interface `Priced` com `double totalPrice()`. Crie um record `OrderLine(String productName, double unitPrice, int quantity) implements Priced`, cujo `totalPrice()` calcula `unitPrice * quantity`. Crie uma classe comum (não record) `Order` que guarda uma `List<OrderLine>` e tem um método `double grandTotal()` que soma `totalPrice()` de todas as linhas.  
_Critério de pronto:_ uma `Order` com 3 `OrderLine`s diferentes calcula o total corretamente, e você demonstra que `OrderLine` (record) pode implementar interface normalmente, como qualquer classe.

**Exercício 4 (Desafio)**  
Esse exercício é sobre a armadilha de componente mutável dentro de record. Crie um record `Team(String name, List<String> members)`, **sem** nenhuma proteção especial primeiro — mostre no `main` que é possível "vazar" a lista mutável (igual o vazamento de encapsulamento da sessão anterior: pegar `team.members()` e chamar `.add(...)` nela, alterando o record "por fora", mesmo ele sendo teoricamente imutável). Depois, corrija usando um **compact constructor** que envolve `members` numa cópia defensiva imutável (`List.copyOf(members)`), e prove que a mesma tentativa de mutação agora lança exceção (`List.copyOf` retorna uma lista que não aceita `.add()`).  
_Critério de pronto:_ a versão sem proteção demonstra o vazamento funcionando (a lista muda); a versão corrigida lança `UnsupportedOperationException` ao tentar `.add()` na lista retornada por `members()`.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public record Coordinate(double latitude, double longitude) { }

public class Main {
    public static void main(String[] args) {
        Coordinate c1 = new Coordinate(-15.7801, -47.9292);
        Coordinate c2 = new Coordinate(-15.7801, -47.9292);

        System.out.println(c1.equals(c2)); // true
        System.out.println(c1);            // Coordinate[latitude=-15.7801, longitude=-47.9292]
    }
}
```

Compare mentalmente com uma `class` tradicional equivalente: pra ter esse mesmo `equals()`/`toString()` sem record, você precisaria escrever manualmente (ou gerar via IDE) o `equals()`, `hashCode()` e `toString()` inteiros — é exatamente esse boilerplate que record elimina, sem trade-off algum quando o objeto é, de fato, só um conjunto de dados imutáveis.

**Exercício 2**

java

```java
public record Age(int years) {
    public Age {
        if (years < 0 || years > 150) {
            throw new IllegalArgumentException("Age must be between 0 and 150");
        }
    }

    public boolean isAdult() {
        return years >= 18;
    }
}
```

Repare que o compact constructor **não tem parênteses** (`public Age { ... }`, não `public Age(int years) { ... }`) — essa é a sintaxe especial de compact constructor, diferente de um construtor normal. Se você escrever com parênteses e reatribuir `this.years = years` manualmente lá dentro, isso também funciona (vira um construtor canônico explícito completo), mas a forma compacta é preferida quando você só quer validar, deixando a atribuição automática pro compilador.

**Exercício 3**

java

```java
public interface Priced {
    double totalPrice();
}

public record OrderLine(String productName, double unitPrice, int quantity) implements Priced {
    @Override
    public double totalPrice() {
        return unitPrice * quantity;
    }
}

public class Order {
    private final List<OrderLine> lines = new ArrayList<>();

    public void addLine(OrderLine line) {
        lines.add(line);
    }

    public double grandTotal() {
        double total = 0;
        for (OrderLine line : lines) {
            total += line.totalPrice();
        }
        return total;
    }
}

public class Main {
    public static void main(String[] args) {
        Order order = new Order();
        order.addLine(new OrderLine("Book", 50.0, 2));
        order.addLine(new OrderLine("Pen", 5.0, 10));
        order.addLine(new OrderLine("Notebook", 20.0, 3));

        System.out.println(order.grandTotal()); // 210.0
    }
}
```

Esse exercício mostra um padrão comum e saudável: **records pra dados imutáveis simples** (`OrderLine`, que não muda depois de criado) coexistindo com **classes tradicionais pra objetos com identidade e comportamento mais complexo** (`Order`, que acumula estado ao longo do tempo via `addLine`). Não é "record substitui classe" — é "record é a ferramenta certa quando o objeto é só dado imutável".

**Exercício 4**

java

```java
// VERSÃO SEM PROTEÇÃO:
public record Team(String name, List<String> members) { }

public class MainLeaky {
    public static void main(String[] args) {
        List<String> members = new ArrayList<>();
        members.add("Ana");
        members.add("Bruno");

        Team team = new Team("Backend", members);
        team.members().add("Carlos"); // "vaza" e muta a lista interna do record
        System.out.println(team.members()); // [Ana, Bruno, Carlos] — o record "imutável" mudou!
    }
}

// VERSÃO CORRIGIDA:
public record Team(String name, List<String> members) {
    public Team {
        members = List.copyOf(members); // cópia defensiva IMUTÁVEL
    }
}

public class MainFixed {
    public static void main(String[] args) {
        List<String> members = new ArrayList<>();
        members.add("Ana");

        Team team = new Team("Backend", members);

        try {
            team.members().add("Carlos");
        } catch (UnsupportedOperationException e) {
            System.out.println("Rejected: cannot mutate an immutable team roster");
        }
    }
}
```

Esse exercício é o ponto mais importante da sessão: **`record` garante que os componentes não podem ser _reatribuídos_, mas não garante, por si só, que o _conteúdo_ de um componente mutável (como uma `List`) seja protegido.** É exatamente o mesmo princípio do vazamento de encapsulamento visto na sessão de Encapsulamento (`Playlist.getSongs()`), só que agora aplicado a um record — a "imutabilidade automática" que o `record` promete é sobre a referência ao componente, não uma garantia profunda (_deep immutability_) do objeto inteiro. `List.copyOf()` é a ferramenta padrão da biblioteca pra fechar essa brecha quando o componente é uma coleção.