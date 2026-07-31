#### 1. Teoria

Abstração é o pilar de **esconder complexidade de implementação, expondo só o que é essencial pra quem usa**. Em Java, os dois mecanismos principais pra isso são **classe abstrata** e **interface** — você já usou os dois nas sessões anteriores, mas ainda não olhamos formalmente pra diferença entre eles, que é exatamente o conteúdo de hoje.

**Classe abstrata (`abstract class`)**

- Pode ter métodos abstratos (sem corpo) **e** métodos concretos (com implementação).
- Pode ter atributos com estado (campos de instância).
- Pode ter construtor (chamado via `super()` pela subclasse, embora você nunca instancie a classe abstrata diretamente).
- Uma classe só pode `extends` **uma** abstrata (herança simples).
- Faz sentido quando existe **comportamento compartilhado real** entre as subclasses, além do contrato.

**Interface**

- Até Java 8, só podia ter métodos abstratos (contrato puro). De Java 8 em diante, pode ter métodos `default` (com corpo) e métodos `static`. De Java 9 em diante, pode ter métodos `private` (auxiliares, só usados internamente por métodos `default` da própria interface).
- Não tem estado de instância — só pode ter constantes (`public static final` implícito em qualquer campo declarado).
- Uma classe pode `implements` **várias** interfaces ao mesmo tempo.
- Faz sentido quando você quer definir um **contrato/capacidade** (`Comparable`, `Runnable`, `PaymentMethod` da sessão anterior), sem impor uma hierarquia de tipo real.

**A pergunta que decide qual usar:** "essas classes compartilham _identidade_ (são fundamentalmente o mesmo tipo de coisa, com estado e comportamento em comum) ou só compartilham uma _capacidade_ (conseguem fazer a mesma ação, mas são coisas essencialmente diferentes)?" `Car` e `Motorcycle` compartilhando `Vehicle` é identidade (ambos têm marca, ambos calculam eficiência de combustível do mesmo jeito estrutural). `CreditCardPayment` e `PixPayment` compartilhando `PaymentMethod` é capacidade — não faz sentido dizer que um cartão de crédito "é uma especialização" de Pix, eles só sabem fazer a mesma coisa (processar pagamento).

**Onde isso aparece de verdade no backend:** todo o ecossistema Spring é construído em cima de programar contra interface, não implementação. `List<Produto> produtos` (não `ArrayList<Produto>`), `JpaRepository` (você não sabe nem precisa saber a implementação concreta gerada pelo Spring por trás), `PlatformTransactionManager` — em todos esses casos, o código que você escreve depende só do contrato, o que permite trocar a implementação (ex: trocar de banco, mockar em teste) sem tocar no código que consome.

---

#### 2. Exemplo de código comentado

java

```java
import java.util.List;

// INTERFACE: define uma CAPACIDADE ("consegue notificar"), sem estado,
// sem relação de identidade entre quem vai implementar.
public interface Notifiable {
    void notify(String message);

    // método default (Java 8+): tem corpo, é herdado por quem implementa,
    // mas PODE ser sobrescrito se a implementação quiser um comportamento diferente.
    default void notifyUrgent(String message) {
        notify("URGENT: " + message);
    }

    // método static: pertence à interface em si, não a nenhuma implementação.
    // Chamado como Notifiable.defaultChannel(), nunca por uma instância.
    static String defaultChannel() {
        return "email";
    }
}

// CLASSE ABSTRATA: define IDENTIDADE (toda Employee compartilha nome e salário
// de verdade, não só uma capacidade) + tem estado real.
public abstract class Employee implements Notifiable {
    protected String name;
    protected double salary;

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    // Método abstrato: contrato que toda subclasse concreta deve cumprir.
    public abstract double calculateBonus();

    // Método concreto: comportamento pronto, compartilhado por todas as subclasses.
    public String payslip() {
        return name + " earns " + salary + " + bonus " + calculateBonus();
    }

    @Override
    public void notify(String message) {
        System.out.println("Notifying " + name + ": " + message);
    }
}

public class Developer extends Employee {
    public Developer(String name, double salary) {
        super(name, salary);
    }

    @Override
    public double calculateBonus() {
        return salary * 0.10;
    }
}

public class Main {
    public static void main(String[] args) {
        Developer dev = new Developer("Carlos", 6000);
        System.out.println(dev.payslip());          // usa método concreto herdado
        dev.notify("Meeting at 3pm");                 // implementação própria da interface
        dev.notifyUrgent("Production is down");        // default method, herdado sem reescrever
        System.out.println(Notifiable.defaultChannel()); // static method da interface
    }
}
```

---

#### 3. Armadilhas comuns

- **Usar `abstract class` só porque "parece mais robusto", quando o caso pede interface.** Se as classes envolvidas não compartilham estado nem identidade real, forçar uma classe abstrata só te prende à limitação de herança simples (a classe perde a chance de herdar de outra coisa) sem ganhar nada em troca.
- **Achar que método `default` em interface "quebra" a abstração porque tem implementação.** Não quebra — `default` existe justamente pra permitir evoluir uma interface (adicionar método novo) sem forçar toda implementação existente a quebrar em tempo de compilação. Foi introduzido no Java 8 principalmente pra permitir adicionar métodos como `forEach` nas interfaces de Collections sem quebrar código legado.
- **Esquecer que campo em interface é implicitamente `public static final`.** Se você escrever `int MAX = 10;` dentro de uma interface, isso não é um campo de instância — é uma constante compartilhada, igual pra todo mundo que implementa. Tentar usar isso como estado mutável por implementação é erro conceitual.
- **Confundir abstração com "esconder informação por segurança".** Isso é mais próximo de encapsulamento. Abstração é sobre **reduzir complexidade cognitiva pra quem usa** — mesmo que não houvesse nenhum risco de segurança, você ainda quer abstração pra não obrigar quem consome `PaymentMethod` a entender os detalhes de integração com o Stripe.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie uma interface `Drivable` com um método abstrato `void drive()` e um método `default` `void honk()` que imprime `"Beep beep!"`. Implemente `Car implements Drivable`, sobrescrevendo só `drive()` (não mexa em `honk()`).  
_Critério de pronto:_ `new Car().honk()` funciona mesmo sem `Car` ter escrito nada sobre `honk()`.

**Exercício 2 (Fácil/Médio)**  
Decida e justifique (comentário no código): esse cenário pede **interface** ou **classe abstrata**? Você tem `Dog`, `Cat` e `Robot` — todos precisam "fazer barulho" (`makeSound()`), mas `Dog` e `Cat` compartilham estado real (`name`, `age`, `breed`) e comportamento comum (`sleep()`, `eat()`), enquanto `Robot` não compartilha **nenhum** desses atributos ou comportamentos com os outros dois, só precisa saber fazer barulho também.  
_Critério de pronto:_ seu design usa dois mecanismos diferentes ao mesmo tempo — uma abstração pra identidade compartilhada (`Dog`/`Cat`) e outra pra capacidade isolada (incluindo `Robot`) — e o comentário explica por quê.

**Exercício 3 (Médio/Difícil)**  
Crie uma interface `Discountable` com um método abstrato `double getPrice()` e um método `default` `double getDiscountedPrice(double discountPercent)` que calcula o preço com desconto **usando `getPrice()` internamente** (o `default` não sabe o preço, só sabe pedir pra quem implementa). Implemente em duas classes diferentes, `Book` e `Electronics`, cada uma com sua própria lógica de `getPrice()`.  
_Critério de pronto:_ `getDiscountedPrice()` nunca é reescrito em `Book` nem `Electronics` — só herdado do `default`, mas funciona corretamente pros dois porque cada um fornece seu próprio `getPrice()`.

**Exercício 4 (Desafio)**  
Modele um sistema de "formas exportáveis": você tem `Circle` e `Square` (compartilham `color` como estado, e ambos calculam `area()` de formas estruturalmente parecidas — isso é identidade, pede classe abstrata `Shape`). Além disso, **algumas** formas (não todas) também devem poder ser exportadas pra um arquivo — isso é uma capacidade extra, não algo que toda `Shape` tem. Crie uma interface `Exportable` com `String exportToJson()`, e faça só `Circle` implementá-la (não `Square`), demonstrando que uma classe pode herdar de uma `abstract class` **e** implementar uma `interface` ao mesmo tempo.  
_Critério de pronto:_ `Circle` estende `Shape` (herda `color` e o contrato de `area()`) e implementa `Exportable`; `Square` só estende `Shape`, sem `Exportable`. Um método que recebe `List<Exportable>` só aceita o `Circle`, não o `Square`, provando que a capacidade é realmente opcional/independente da hierarquia de identidade.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public interface Drivable {
    void drive();

    default void honk() {
        System.out.println("Beep beep!");
    }
}

public class Car implements Drivable {
    @Override
    public void drive() {
        System.out.println("Car is driving");
    }
}
```

Simples de propósito: o objetivo é só confirmar na prática que `default` funciona sem exigir nada da classe que implementa — diferente do método abstrato, que é obrigatório sobrescrever.

**Exercício 2**

java

```java
// Dog e Cat compartilham IDENTIDADE real: mesmo estado (name, age, breed),
// mesmo comportamento concreto compartilhado (sleep, eat). Isso pede classe abstrata.
public abstract class Pet {
    protected String name;
    protected int age;
    protected String breed;

    public Pet(String name, int age, String breed) {
        this.name = name;
        this.age = age;
        this.breed = breed;
    }

    public void sleep() {
        System.out.println(name + " is sleeping");
    }

    public void eat() {
        System.out.println(name + " is eating");
    }

    public abstract void makeSound();
}

// Robot não compartilha NADA de estado/comportamento com Dog/Cat — só a
// capacidade isolada de "fazer barulho". Isso pede interface, não herança.
public interface SoundMaker {
    void makeSound();
}

public class Dog extends Pet implements SoundMaker {
    public Dog(String name, int age, String breed) {
        super(name, age, breed);
    }

    @Override
    public void makeSound() {
        System.out.println("Woof");
    }
}

public class Cat extends Pet implements SoundMaker {
    public Cat(String name, int age, String breed) {
        super(name, age, breed);
    }

    @Override
    public void makeSound() {
        System.out.println("Meow");
    }
}

public class Robot implements SoundMaker {
    @Override
    public void makeSound() {
        System.out.println("Beep boop");
    }
}
```

Esse exercício existe pra deixar claro que a decisão não é "escolha um dos dois mecanismos pro sistema inteiro" — é **por relação**. `Dog`/`Cat`/`Robot` compartilham a interface `SoundMaker` (capacidade comum aos três), enquanto só `Dog`/`Cat` compartilham a identidade `Pet`. É perfeitamente normal (e comum em código real) misturar os dois mecanismos na mesma modelagem.

**Exercício 3**

java

```java
public interface Discountable {
    double getPrice();

    default double getDiscountedPrice(double discountPercent) {
        return getPrice() * (1 - discountPercent / 100);
    }
}

public class Book implements Discountable {
    private double price;

    public Book(double price) {
        this.price = price;
    }

    @Override
    public double getPrice() {
        return price;
    }
}

public class Electronics implements Discountable {
    private double price;
    private double warrantyFee;

    public Electronics(double price, double warrantyFee) {
        this.price = price;
        this.warrantyFee = warrantyFee;
    }

    @Override
    public double getPrice() {
        return price + warrantyFee;
    }
}
```

O ponto central: `getDiscountedPrice()` é escrito **uma vez** na interface e funciona corretamente pra qualquer implementação futura de `Discountable`, porque ele depende só do contrato (`getPrice()`), nunca de detalhes internos de `Book` ou `Electronics`. Isso é abstração funcionando de verdade — o método `default` nem sabe que `Electronics` soma uma taxa de garantia no preço, e não precisa saber.

**Exercício 4**

java

```java
public abstract class Shape {
    protected String color;

    public Shape(String color) {
        this.color = color;
    }

    public abstract double area();
}

public interface Exportable {
    String exportToJson();
}

public class Circle extends Shape implements Exportable {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }

    @Override
    public String exportToJson() {
        return "{\"type\":\"circle\",\"color\":\"" + color + "\",\"area\":" + area() + "}";
    }
}

public class Square extends Shape {
    private double side;

    public Square(String color, double side) {
        super(color);
        this.side = side;
    }

    @Override
    public double area() {
        return side * side;
    }
    // sem exportToJson() — Square não implementa Exportable, e não precisa.
}

public class Main {
    public static void main(String[] args) {
        List<Exportable> exportables = List.of(new Circle("red", 3));
        // new Square("blue", 4) NÃO poderia entrar nessa lista — não compila,
        // porque Square não implementa Exportable.

        for (Exportable e : exportables) {
            System.out.println(e.exportToJson());
        }
    }
}
```

Esse é o exercício mais importante da sessão pra fixar a diferença prática: `Shape` resolve **"o que esse objeto é"** (identidade, estado compartilhado), e `Exportable` resolve **"o que esse objeto consegue fazer, opcionalmente"** (capacidade independente da hierarquia). Uma classe real de produção frequentemente tem uma superclasse abstrata **e** várias interfaces ao mesmo tempo — isso não é exceção, é o padrão comum em Java.