#### 1. Teoria

OOP (Programação Orientada a Objetos) é um paradigma onde você organiza o código em **objetos**: unidades que combinam **estado** (atributos/dados) e **comportamento** (métodos) numa coisa só. Isso é diferente de programação procedural, onde dados e funções ficam separados e qualquer função pode mexer em qualquer dado.

O paradigma se sustenta em **4 pilares**. Aqui vai a visão geral de cada um — cada um vira uma sessão própria depois no roadmap, então hoje é só o mapa, não o território:

**Encapsulamento** — proteger o estado interno do objeto, expondo só o que é necessário através de métodos controlados (getters/setters, ou melhor ainda, métodos que fazem sentido de negócio). O objetivo não é "usar `private`" por regra — é garantir que o objeto nunca fique num estado inválido porque alguém de fora mexeu direto no dado.

**Abstração** — expor _o que_ um objeto faz, escondendo _como_ ele faz. Uma interface `Repository` te diz "eu salvo e busco dados" sem você precisar saber se por trás é PostgreSQL, MongoDB ou uma lista em memória.

**Herança** — uma classe reaproveita e especializa comportamento de outra, numa relação "é um" (`Circle` **é uma** `Shape`). Serve pra evitar duplicação quando existe uma hierarquia real de conceitos.

**Polimorfismo** — o mesmo método, chamado da mesma forma, se comporta diferente dependendo do objeto real por trás. É isso que permite escrever `shape.calculateArea()` sem saber se `shape` é um círculo ou um retângulo, e o Java resolver isso em tempo de execução (dynamic dispatch).

**Onde isso aparece de verdade no backend:** no Spring, você quase sempre depende de **interfaces**, não de implementações concretas (abstração). Um `@Service` que recebe um `PaymentGateway` no construtor não sabe (nem precisa saber) se é Stripe ou PagSeguro por trás — isso é abstração + polimorfismo trabalhando juntos, e é a base de como Injeção de Dependência funciona (isso será aprofundado no Bloco 8).

Uma diferença importante que gera confusão: **Encapsulamento ≠ Abstração**. Encapsulamento protege o _estado_ (dados). Abstração esconde a _complexidade de implementação_. Uma interface Java é abstração pura — ela nem tem estado pra encapsular.

---

#### 2. Exemplo de código comentado

java

```java
import java.util.ArrayList;
import java.util.List;

// ABSTRAÇÃO: classe abstrata define O QUE todo Shape faz (tem cor, calcula área),
// sem dizer COMO cada forma específica calcula essa área.
public abstract class Shape {

    // ENCAPSULAMENTO: campo privado, só acessível via método controlado abaixo.
    // Ninguém de fora consegue fazer shape.color = null, por exemplo.
    private final String color;

    public Shape(String color) {
        this.color = color;
    }

    public String getColor() {
        return color;
    }

    // Método abstrato: não tem corpo aqui, é um "contrato" que toda subclasse
    // é OBRIGADA a cumprir. Isso é abstração forçada pelo compilador.
    public abstract double calculateArea();
}

// HERANÇA: Circle "é um" Shape, herda getColor() e o construtor via super().
public class Circle extends Shape {
    private final double radius;

    public Circle(String color, double radius) {
        super(color); // chama o construtor da classe pai
        this.radius = radius;
    }

    // POLIMORFISMO: sobrescreve o método abstrato com a lógica específica de círculo.
    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle extends Shape {
    private final double width;
    private final double height;

    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }

    @Override
    public double calculateArea() {
        return width * height;
    }
}

public class Main {
    public static void main(String[] args) {
        List<Shape> shapes = new ArrayList<>();
        shapes.add(new Circle("red", 5));
        shapes.add(new Rectangle("blue", 4, 6));

        // POLIMORFISMO EM AÇÃO: a mesma chamada shape.calculateArea() executa
        // um código diferente dependendo se o objeto real é Circle ou Rectangle.
        // O código aqui não sabe (nem precisa saber) qual é qual.
        for (Shape shape : shapes) {
            System.out.println(shape.getColor() + " area: " + shape.calculateArea());
        }
    }
}
```

---

#### 3. Armadilhas comuns

- **Achar que "usar `private`" já é encapsulamento de verdade.** Se você coloca `private` em tudo mas cria um `setColor(String color)` público sem nenhuma validação, você só reescreveu o campo público com passos a mais. Encapsulamento de verdade protege _invariantes_ (regras que nunca podem ser quebradas), não é só sintaxe.
- **Confundir herança com "reaproveitar código".** Herdar de uma classe só porque ela já tem um método que você quer usar, sem existir relação "é um" real, gera hierarquias artificiais que quebram fácil. Quando não há relação de tipo genuína, o padrão preferido é **composição** (a classe _tem um_ objeto do outro tipo, em vez de _ser um_) — isso será aprofundado mais pra frente.
- **Misturar Overloading com Overriding** (isso tem sessão própria depois, mas evita confusão agora: Overloading é mesmo nome, parâmetros diferentes, resolvido em tempo de compilação; Overriding é redefinir um método herdado, resolvido em tempo de execução — são mecanismos completamente diferentes apesar do nome parecido).
- **Esquecer o `@Override`.** Tecnicamente opcional, mas sem essa anotação, se você errar a assinatura do método (typo, parâmetro errado), o Java aceita como um método novo sem avisar nada — você achava que estava sobrescrevendo e na verdade criou outro método do zero.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie uma classe `Animal` com um atributo privado `name` (String), um construtor que recebe o nome, e um getter `getName()`. Adicione um método `makeSound()` que retorna a String `"Some generic sound"`.  
_Critério de pronto:_ a classe compila, e um `main` consegue criar um `Animal`, imprimir o nome e o som.

**Exercício 2 (Fácil/Médio)**  
Transforme `Animal` numa classe **abstrata**, com `makeSound()` sendo um método abstrato (sem implementação). Crie duas subclasses, `Dog` e `Cat`, cada uma sobrescrevendo `makeSound()` com um som próprio (`"Woof"`, `"Meow"`).  
_Critério de pronto:_ uma `List<Animal>` contendo um `Dog` e um `Cat` deve imprimir sons diferentes ao chamar `makeSound()` em loop, sem nenhum `if`/`instanceof` no loop.

**Exercício 3 (Médio/Difícil)**  
Crie uma interface `PaymentMethod` com um método `processPayment(double amount)` que retorna `String` (uma mensagem de confirmação). Implemente `CreditCardPayment` e `PixPayment`, cada uma com uma mensagem de confirmação diferente. Crie uma classe `Checkout` que **recebe um `PaymentMethod` no construtor** (não cria a dependência internamente) e tem um método `finalizePurchase(double amount)` que delega pro `PaymentMethod` recebido.  
_Critério de pronto:_ você consegue trocar de `CreditCardPayment` pra `PixPayment` no `main` sem alterar uma linha sequer da classe `Checkout`. (Esse padrão — receber a dependência de fora em vez de criar internamente — é exatamente a ideia por trás de Injeção de Dependência, que vem no Bloco 8.)

**Exercício 4 (Desafio)**  
Refatore o código abaixo, que resolve o problema de forma puramente procedural, aplicando os 4 pilares (encapsulamento, abstração, herança, polimorfismo) onde fizer sentido — sem inventar complexidade desnecessária, só o suficiente pra resolver os problemas reais do código:

java

```java
public class EmployeeUtils {
    public static double calculateSalary(String type, double baseSalary, int yearsWorked) {
        if (type.equals("MANAGER")) {
            return baseSalary + (baseSalary * 0.20) + (yearsWorked * 100);
        } else if (type.equals("DEVELOPER")) {
            return baseSalary + (baseSalary * 0.10) + (yearsWorked * 150);
        } else if (type.equals("INTERN")) {
            return baseSalary;
        }
        throw new IllegalArgumentException("Unknown type");
    }
}
```

_Critério de pronto:_ adicionar um novo tipo de funcionário não pode exigir alterar nenhuma classe existente, só criar uma nova.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Animal {
    private final String name;

    public Animal(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public String makeSound() {
        return "Some generic sound";
    }
}
```

Simples: o único pilar exercitado aqui é encapsulamento (campo privado + getter). Não tem herança nem polimorfismo ainda porque não existe variação de comportamento — só uma classe.

**Exercício 2**

java

```java
public abstract class Animal {
    private final String name;

    public Animal(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public abstract String makeSound();
}

public class Dog extends Animal {
    public Dog(String name) {
        super(name);
    }

    @Override
    public String makeSound() {
        return "Woof";
    }
}

public class Cat extends Animal {
    public Cat(String name) {
        super(name);
    }

    @Override
    public String makeSound() {
        return "Meow";
    }
}
```

O ponto central aqui é: o loop que chama `makeSound()` **não sabe nem precisa saber** se está lidando com `Dog` ou `Cat`. Se você sentiu vontade de escrever `if (animal instanceof Dog)` dentro do loop pra decidir o som, isso é sinal de que o polimorfismo não foi aplicado — o `if` é exatamente o que o método abstrato + override elimina.

**Exercício 3**

java

```java
public interface PaymentMethod {
    String processPayment(double amount);
}

public class CreditCardPayment implements PaymentMethod {
    @Override
    public String processPayment(double amount) {
        return "Paid R$" + amount + " with credit card";
    }
}

public class PixPayment implements PaymentMethod {
    @Override
    public String processPayment(double amount) {
        return "Paid R$" + amount + " with Pix";
    }
}

public class Checkout {
    private final PaymentMethod paymentMethod;

    public Checkout(PaymentMethod paymentMethod) {
        this.paymentMethod = paymentMethod;
    }

    public String finalizePurchase(double amount) {
        return paymentMethod.processPayment(amount);
    }
}

// uso:
Checkout checkout = new Checkout(new PixPayment());
System.out.println(checkout.finalizePurchase(150.0));
```

Repare que usei `interface`, não `abstract class`, dessa vez — quando não há **nenhum** estado ou comportamento comum pra compartilhar entre as implementações (só o contrato do método), interface é a escolha mais idiomática em Java. `abstract class` faz mais sentido quando as subclasses compartilham código de verdade (como o `getName()` do Exercício 2). Essa diferença — quando usar cada um — é o próprio tópico "Interfaces" que vem mais adiante no roadmap, então por enquanto fica só o sinal de alerta.

**Exercício 4**

java

```java
public abstract class Employee {
    private final double baseSalary;
    private final int yearsWorked;

    public Employee(double baseSalary, int yearsWorked) {
        this.baseSalary = baseSalary;
        this.yearsWorked = yearsWorked;
    }

    protected double getBaseSalary() {
        return baseSalary;
    }

    protected int getYearsWorked() {
        return yearsWorked;
    }

    public abstract double calculateSalary();
}

public class Manager extends Employee {
    public Manager(double baseSalary, int yearsWorked) {
        super(baseSalary, yearsWorked);
    }

    @Override
    public double calculateSalary() {
        return getBaseSalary() + (getBaseSalary() * 0.20) + (getYearsWorked() * 100);
    }
}

public class Developer extends Employee {
    public Developer(double baseSalary, int yearsWorked) {
        super(baseSalary, yearsWorked);
    }

    @Override
    public double calculateSalary() {
        return getBaseSalary() + (getBaseSalary() * 0.10) + (getYearsWorked() * 150);
    }
}

public class Intern extends Employee {
    public Intern(double baseSalary, int yearsWorked) {
        super(baseSalary, yearsWorked);
    }

    @Override
    public double calculateSalary() {
        return getBaseSalary();
    }
}
```

O código original tem um cheiro clássico de "falta de polimorfismo": uma cadeia de `if/else` checando um campo `type` como String. Isso é frágil (typo no texto quebra silenciosamente em runtime) e viola o critério pedido — adicionar `Diretor` significaria editar o método `calculateSalary` existente, arriscando quebrar a lógica de `Manager` e `Developer` que já funcionava. Com herança + polimorfismo, `Diretor` vira só uma classe nova que estende `Employee`; nada existente é tocado.

Vale notar uma alternativa válida: em vez de `protected` nos getters, algumas pessoas preferem passar `baseSalary` e `yearsWorked` como parâmetros do método `calculateSalary(double baseSalary, int yearsWorked)` em vez de guardar como estado do objeto. O trade-off é: guardar como estado (como fiz acima) é mais "OOP puro" e funciona bem se `Employee` também vai ter outros métodos que usam esses dados depois; passar como parâmetro evita duplicar dado em memória se o `Employee` for só uma casca fina só pra esse cálculo. Pra esse exercício, tanto faz — a decisão importa mais conforme a classe cresce.