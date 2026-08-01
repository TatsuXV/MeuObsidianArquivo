#### 1. Teoria

**Enum** (abreviação de _enumeration_) é um tipo especial que representa um conjunto fixo e finito de constantes. Foi introduzido no Java 5 (JDK 1.5).

A diferença fundamental para outras linguagens: em Java, um enum **não é** um simples inteiro disfarçado (como em C). Cada constante de um enum é, por baixo dos panos, uma **instância singleton** da própria classe enum. Isso significa que um enum pode ter atributos, construtor, métodos, e até implementar interfaces — é uma classe completa, só que com um conjunto fixo de instâncias pré-definidas.

**Diferença de "Enum" vs. int/String como constante:**

- Com `int` ou `String` como constante (ex: `public static final int STATUS_PENDENTE = 0;`), o compilador não impede você de passar qualquer inteiro/string inválido pra um método que espera esse "status". Não há segurança de tipo.
- Com enum, o compilador só aceita valores que existem no enum. É impossível representar um estado inválido — isso é chamado de _type safety_.

**Diferença de Enum vs. Classe Abstrata:**

- Uma classe abstrata pode ter infinitas subclasses/instâncias em tempo de execução.
- Um enum tem um conjunto **fechado e conhecido em tempo de compilação** de instâncias — você não pode criar uma nova constante em runtime, nem instanciar um enum com `new` (o construtor é implicitamente `private`).

**Onde aparece no dia a dia de backend Java/Spring:**

- Representar status de entidade: `OrderStatus.PENDING`, `PAID`, `SHIPPED`, `CANCELED`.
- Representar papéis/permissões: `Role.ADMIN`, `Role.USER`.
- Persistência com JPA: `@Enumerated(EnumType.STRING)` numa entidade.
- `switch` exaustivo (o compilador força você a tratar todos os casos, ou usar `default`).
- Estruturas otimizadas como `EnumMap` e `EnumSet`, muito mais eficientes que `HashMap`/`HashSet` quando a chave é um enum.

---

#### 2. Exemplo de código comentado

java

```java
// Exemplo 1: enum simples, sem estado extra
public enum OrderStatus {
    PENDING,
    PAID,
    SHIPPED,
    DELIVERED,
    CANCELED
}
```

java

```java
// Exemplo 2: enum com estado (atributos) e comportamento (métodos)
public enum Planet {
    // Cada constante chama o construtor do enum passando seus próprios valores
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6);

    // Atributos de instância normais — cada constante tem os seus próprios valores
    private final double mass;   // em kg
    private final double radius; // em metros

    // Construtor de enum é implicitamente private (não compila se você marcar public)
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    private static final double G = 6.67300E-11;

    // Método normal, disponível em todas as constantes
    public double surfaceGravity() {
        return G * mass / (radius * radius);
    }
}
```

java

```java
// Exemplo 3: enum com comportamento DIFERENTE por constante (constant-specific method)
// Útil quando cada constante precisa de uma lógica própria — evita if/else ou switch espalhado
public enum Operation {
    ADD {
        @Override
        public int apply(int a, int b) {
            return a + b;
        }
    },
    SUBTRACT {
        @Override
        public int apply(int a, int b) {
            return a - b;
        }
    },
    MULTIPLY {
        @Override
        public int apply(int a, int b) {
            return a * b;
        }
    };

    // Método abstrato: obriga CADA constante a fornecer sua própria implementação
    public abstract int apply(int a, int b);
}

// Uso:
// Operation.ADD.apply(2, 3);       -> 5
// Operation.MULTIPLY.apply(2, 3);  -> 6
```

java

```java
// Exemplo 4: métodos utilitários que todo enum ganha de graça
OrderStatus status = OrderStatus.PAID;

status.name();      // "PAID" — nome exato como declarado, final, não pode ser sobrescrito
status.ordinal();   // 1 — posição na declaração, começando em 0 (PENDING=0, PAID=1...)
OrderStatus.values();          // array com todas as constantes, na ordem de declaração
OrderStatus.valueOf("PAID");   // retorna OrderStatus.PAID; lança IllegalArgumentException se não existir
```

---

#### 3. Armadilhas comuns

1. **Usar `ordinal()` para persistir no banco.** Se você salvar o número da posição (`ordinal()`) numa coluna do banco e depois **reordenar ou inserir uma constante no meio** do enum, todos os dados antigos passam a apontar pro status errado. Regra: nunca persista `ordinal()`. Persista o `name()` (String) ou um código explícito.
2. **`@Enumerated(EnumType.ORDINAL)` no JPA.** É o mesmo problema do item acima, só que dentro do Spring Data JPA — é a armadilha clássica de quem está aprendendo. Praticamente sempre você quer `@Enumerated(EnumType.STRING)`, mesmo custando um pouco mais de espaço no banco.
3. **Achar que precisa comparar enum com `.equals()`.** Como cada constante é um singleton, `==` é seguro, mais rápido e é o padrão da comunidade Java para comparar enums (diferente de String, onde `==` é perigoso). Usar `.equals()` não é errado, mas é redundante.
4. **Colocar estado mutável dentro do enum.** Como as constantes são singletons compartilhados por toda a aplicação, um campo mutável (não-`final`, com setter) vira estado global compartilhado entre threads — risco real de bug de concorrência. Enums devem ser tratados como imutáveis.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie um enum `DayOfWeek` com os 7 dias da semana. Escreva um método separado (fora do enum, ex: numa classe `Main`) que recebe um `DayOfWeek` e imprime `"Dia útil"` se for de segunda a sexta, ou `"Fim de semana"` se for sábado ou domingo, usando `switch`.  
_Critério de pronto:_ compila, roda para os 7 dias e imprime a categoria correta para cada um.

**Exercício 2 (Médio)**  
Crie um enum `Currency` (moeda) com pelo menos 3 constantes (`USD`, `BRL`, `EUR`), cada uma armazenando um atributo `symbol` (String, ex: "","R", "R ","R", "€") e um atributo `conversionRateToUSD` (double). Adicione um método `toUSD(double amount)` que converte um valor da moeda para dólar usando a taxa armazenada.  
_Critério de pronto:_ `BRL.toUSD(100)` retorna o valor correto de acordo com a taxa que você definir; o método usa os atributos da própria constante, sem `if/else` externo verificando qual moeda é.

**Exercício 3 (Difícil)**  
Crie um enum `TrafficLight` (semáforo) representando `RED`, `YELLOW`, `GREEN`. Cada constante deve implementar um método abstrato `next()` que retorna a **próxima cor do ciclo** (RED → GREEN → YELLOW → RED), usando o padrão de _constant-specific method_ (igual ao Exemplo 3 acima) — sem usar `switch` nem `if`.  
_Critério de pronto:_ chamar `.next()` encadeado várias vezes reproduz o ciclo correto do semáforo indefinidamente.

**Exercício 4 (Desafio)**  
Crie um enum `CardSuit` (naipe de carta: `HEARTS`, `DIAMONDS`, `CLUBS`, `SPADES`) que implemente uma interface `Colorable` com um método `String getColor()`. Naipes de copas/ouros retornam `"RED"`; paus/espadas retornam `"BLACK"`. Depois, use um `EnumMap<CardSuit, Integer>` para contar quantas cartas de cada naipe existem numa lista de exemplo (`List<CardSuit>`) que você mesmo cria com valores repetidos.  
_Critério de pronto:_ o enum implementa a interface corretamente; o `EnumMap` é populado dinamicamente iterando a lista (nada de contar na mão) e imprime a contagem final por naipe.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public enum DayOfWeek {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

public class Main {
    public static void classify(DayOfWeek day) {
        // switch clássico funciona bem aqui porque são só duas categorias de saída
        switch (day) {
            case SATURDAY:
            case SUNDAY:
                System.out.println("Fim de semana");
                break;
            default:
                System.out.println("Dia útil");
                break;
        }
    }

    public static void main(String[] args) {
        for (DayOfWeek d : DayOfWeek.values()) {
            System.out.print(d + ": ");
            classify(d);
        }
    }
}
```

_Raciocínio:_ uso `default` para os dias úteis em vez de listar os 5 `case`s — menos código, mesma clareza, já que "dia útil" é a regra geral e "fim de semana" é a exceção.

**Exercício 2**

java

```java
public enum Currency {
    USD("$", 1.0),
    BRL("R$", 0.19),   // taxa fictícia de exemplo
    EUR("€", 1.08);

    private final String symbol;
    private final double conversionRateToUSD;

    Currency(String symbol, double conversionRateToUSD) {
        this.symbol = symbol;
        this.conversionRateToUSD = conversionRateToUSD;
    }

    public double toUSD(double amount) {
        return amount * conversionRateToUSD;
    }

    public String getSymbol() {
        return symbol;
    }
}
```

_Raciocínio:_ a taxa de conversão fica **dentro** de cada constante, não numa tabela externa ou num `if (currency == BRL)`. Isso é o ganho real de usar enum com estado: a lógica "qual taxa usar" nunca escapa do próprio enum, então adicionar uma moeda nova (`JPY`, por exemplo) não exige tocar em nenhum outro lugar do código.

**Exercício 3**

java

```java
public enum TrafficLight {
    RED {
        @Override
        public TrafficLight next() {
            return GREEN;
        }
    },
    GREEN {
        @Override
        public TrafficLight next() {
            return YELLOW;
        }
    },
    YELLOW {
        @Override
        public TrafficLight next() {
            return RED;
        }
    };

    public abstract TrafficLight next();
}

// Teste:
// TrafficLight atual = TrafficLight.RED;
// atual = atual.next(); // GREEN
// atual = atual.next(); // YELLOW
// atual = atual.next(); // RED
```

_Raciocínio:_ essa é a alternativa "elegante" ao `switch`. Uma solução alternativa válida seria um único método `next()` fora do enum usando `switch(this)` — funciona igual, mas centraliza a lógica de transição fora do enum, o que é pior quando o número de estados cresce (você teria que lembrar de atualizar o `switch` toda vez que adicionar uma cor nova). O trade-off do _constant-specific method_ é: mais verboso pra poucas constantes, mas escala melhor e o compilador te obriga a implementar `next()` em toda constante nova.

**Exercício 4**

java

```java
public interface Colorable {
    String getColor();
}

public enum CardSuit implements Colorable {
    HEARTS {
        @Override
        public String getColor() { return "RED"; }
    },
    DIAMONDS {
        @Override
        public String getColor() { return "RED"; }
    },
    CLUBS {
        @Override
        public String getColor() { return "BLACK"; }
    },
    SPADES {
        @Override
        public String getColor() { return "BLACK"; }
    };
}

import java.util.EnumMap;
import java.util.List;

public class Main {
    public static void main(String[] args) {
        List<CardSuit> hand = List.of(
            CardSuit.HEARTS, CardSuit.SPADES, CardSuit.HEARTS,
            CardSuit.CLUBS, CardSuit.DIAMONDS, CardSuit.HEARTS
        );

        // EnumMap é mais eficiente que HashMap quando a chave é um enum:
        // internamente usa um array indexado pelo ordinal(), não hashing.
        EnumMap<CardSuit, Integer> count = new EnumMap<>(CardSuit.class);

        for (CardSuit suit : hand) {
            count.merge(suit, 1, Integer::sum);
            // merge: se a chave não existe, usa 1; se existe, soma 1 ao valor atual
        }

        count.forEach((suit, total) ->
            System.out.println(suit + " (" + suit.getColor() + "): " + total)
        );
    }
}
```

_Raciocínio:_ `EnumMap` foi escolhido em vez de `HashMap` porque a chave é um enum — é a estrutura correta e mais performática pra esse caso (mencionado na Teoria). O método `merge` evita o clássico `if (map.containsKey(suit)) ... else ...` pra incrementar contador.