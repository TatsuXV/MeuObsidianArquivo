#### 1. Teoria

Herança é o mecanismo que permite uma classe (subclasse) reutilizar e especializar o estado e comportamento de outra (superclasse), numa relação **"é um"**. Em Java, isso se declara com `extends`:

java

```java
public class Dog extends Animal { }
```

Alguns pontos estruturais que fazem toda a diferença na prática:

**Herança simples de classe.** Java só permite `extends` de **uma** classe por vez (diferente de C++). Isso é deliberado — evita o "problema do diamante" (ambiguidade quando duas superclasses têm um método com o mesmo nome). Pra ganhar múltipla "herança" de comportamento, Java usa **interfaces** (uma classe pode `implements` várias), que você já viu na sessão anterior.

**Toda classe herda de `Object`, mesmo sem dizer.** Se você não escrever `extends`, o compilador assume `extends Object` implicitamente. É por isso que todo objeto Java tem `toString()`, `equals()`, `hashCode()` disponíveis mesmo sem você definir nada — eles vêm de `Object`.

**Encadeamento de construtores (`super()`).** O construtor da subclasse **sempre** chama, direta ou indiretamente, um construtor da superclasse — antes de qualquer outra coisa. Se você não escrever `super(...)` explicitamente, o Java insere um `super()` (sem argumentos) automaticamente como primeira linha. Isso só funciona se a superclasse **tiver** um construtor sem argumentos; se não tiver, você é **obrigado** a chamar `super(algumArgumento)` explicitamente, ou o código não compila.

**`protected` existe por causa de herança.** É o nível de acesso intermediário entre `private` e `public`: visível pra própria classe, pro mesmo pacote, e pras subclasses (mesmo em pacote diferente). Isso permite que a subclasse acesse algo da superclasse que não deveria ser público pro resto do mundo.

**Membros `private` não são "invisíveis" na subclasse — são inacessíveis por nome.** Um campo `private` da superclasse existe no objeto (ocupa memória), mas a subclasse não pode se referir a ele diretamente. Se a subclasse precisa desse dado, a superclasse precisa expor um getter/setter `protected` ou `public`.

**Onde isso aparece de verdade no backend:** exceptions customizadas quase sempre usam herança (`class ResourceNotFoundException extends RuntimeException`), e classes base abstratas em camadas de serviço (ex: um `BaseEntity` com `id`, `createdAt`, `updatedAt` que toda entidade JPA estende) são um uso legítimo — porque ali existe uma relação "é um" genuína e comportamento compartilhado real, não um atalho pra evitar copiar código.

---

#### 2. Exemplo de código comentado

java

```java
public class Employee {
    protected String name;      // protected: subclasse acessa direto, resto do mundo não
    private double baseSalary;  // private: só Employee mexe nisso diretamente

    // Employee NÃO tem construtor sem argumentos — só este aqui.
    public Employee(String name, double baseSalary) {
        this.name = name;
        this.baseSalary = baseSalary;
    }

    public double getBaseSalary() {
        return baseSalary;
    }

    public double calculateBonus() {
        return baseSalary * 0.05;
    }

    // Método estático: NÃO participa de polimorfismo (mais sobre isso nas armadilhas).
    public static String describeRole() {
        return "Generic employee";
    }
}

public class Manager extends Employee {
    private int teamSize;

    public Manager(String name, double baseSalary, int teamSize) {
        // OBRIGATÓRIO chamar super(...) explicitamente aqui, porque Employee
        // não tem construtor sem argumentos — o compilador não tem o que inserir sozinho.
        super(name, baseSalary);
        this.teamSize = teamSize;
    }

    // Override de verdade: mesma assinatura, resolvido em tempo de execução.
    @Override
    public double calculateBonus() {
        // reaproveita a lógica da superclasse via super.metodo(), em vez de duplicar
        double base = super.calculateBonus();
        return base + (teamSize * 50);
    }

    // Isso NÃO é override, é "method hiding" — mais sobre isso nas armadilhas.
    public static String describeRole() {
        return "Manager (leads a team)";
    }

    public String greet() {
        // 'name' é acessível aqui porque é protected na superclasse, não private
        return "Hi, I'm " + name + ", managing " + teamSize + " people.";
    }
}

public class Main {
    public static void main(String[] args) {
        Employee e = new Manager("Ana", 5000, 4);
        System.out.println(e.calculateBonus()); // 250.0 + 200 = 450.0 — chama a versão de Manager
        System.out.println(((Manager) e).greet()); // precisa de downcast pra acessar método exclusivo de Manager
    }
}
```

---

#### 3. Armadilhas comuns

- **Método estático "sobrescrito" não é polimórfico — isso se chama _hiding_, não _overriding_.** Métodos estáticos são resolvidos em tempo de **compilação**, com base no tipo da variável, não no tipo do objeto real. No exemplo acima, `Employee.describeRole()` chamado através de uma variável do tipo `Employee` (mesmo apontando pra um `Manager`) sempre chama a versão de `Employee`. Isso engana muita gente que assume que estático se comporta igual a instância.
- **Esquecer que `super()` precisa ser a primeira linha do construtor.** Você não pode fazer nenhuma lógica antes de chamar `super(...)` — nem um `if`, nem atribuir outro campo. O compilador acusa erro.
- **Usar herança quando a relação não é genuinamente "é um".** Já foi tratado na sessão anterior (`Wallet extends Logger`), mas em Inheritance especificamente o erro mais comum é herdar só pra reaproveitar um método específico de uma classe grande, criando uma hierarquia que não reflete o domínio real.
- **Reduzir a visibilidade de um método sobrescrito.** Se `calculateBonus()` é `public` na superclasse, a subclasse não pode sobrescrever como `protected` ou `private` — o Java não permite enfraquecer o contrato. Pode manter ou aumentar a visibilidade, nunca diminuir.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie `Vehicle` com atributo `protected String model` e construtor que recebe `model`. Crie `Truck extends Vehicle` com um atributo próprio `double cargoCapacity`, com construtor que chama `super(model)` e inicializa `cargoCapacity`. Adicione um método `describe()` em `Truck` que usa `model` (herdado) e `cargoCapacity` (próprio).  
_Critério de pronto:_ compila e imprime corretamente combinando dado herdado com dado próprio.

**Exercício 2 (Fácil/Médio)**  
Crie `Shape` com um método `double area()` que retorna `0` (não abstrato — implementação "burra" de propósito, sem `abstract`, pra esse exercício). Crie `Square extends Shape`, sobrescrevendo `area()` pra calcular a área de verdade a partir de um atributo `side`. No método `area()` de `Square`, chame `super.area()` primeiro (mesmo sabendo que retorna 0) só pra praticar a sintaxe, e depois retorne o cálculo real.  
_Critério de pronto:_ `new Square(4).area()` retorna `16.0`, e seu código de fato contém uma chamada a `super.area()`.

**Exercício 3 (Médio/Difícil)**  
Reproduza o cenário da armadilha de _method hiding_: crie `Animal` com um método **estático** `String category()` retornando `"Generic animal"`, e `Bird extends Animal` com um método estático de mesma assinatura retornando `"Bird"`. No `main`, crie `Animal a = new Bird();` e imprima o resultado de `Animal.category()` chamado assim: `((Animal) a)` não é possível pra estático, então chame diretamente `Animal.category()` e separadamente `Bird.category()`. Escreva um comentário explicando por que os dois retornam valores diferentes mesmo `a` sendo, em runtime, um `Bird`.  
_Critério de pronto:_ seu comentário explica corretamente que métodos estáticos são resolvidos pelo tipo da referência/classe, não pelo objeto real, e por isso não são polimórficos.

**Exercício 4 (Desafio)**  
Crie uma pequena hierarquia de 3 níveis: `Person` → `Employee extends Person` → `Manager extends Employee`. Cada nível adiciona um atributo próprio (`Person` tem `name`; `Employee` tem `salary`; `Manager` tem `teamSize`) e cada construtor deve encadear corretamente via `super(...)` até a raiz. Adicione um método `summary()` em cada classe que **chama `super.summary()` e concatena** a informação própria daquele nível (não reescreva do zero em cada classe — cada nível só acrescenta a sua parte).  
_Critério de pronto:_ `new Manager("Ana", 5000, 3).summary()` retorna uma string com as três informações combinadas, e nenhuma classe duplica a lógica de formatação das outras.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Vehicle {
    protected String model;

    public Vehicle(String model) {
        this.model = model;
    }
}

public class Truck extends Vehicle {
    private double cargoCapacity;

    public Truck(String model, double cargoCapacity) {
        super(model);
        this.cargoCapacity = cargoCapacity;
    }

    public String describe() {
        return model + " can carry " + cargoCapacity + " tons";
    }
}
```

Repare que `describe()` usa `model` diretamente, sem getter — isso só é possível porque `model` é `protected`, não `private`. Se fosse `private` na superclasse, esse código não compilaria, e `Truck` precisaria de um `getModel()` público na `Vehicle`.

**Exercício 2**

java

```java
public class Shape {
    public double area() {
        return 0;
    }
}

public class Square extends Shape {
    private double side;

    public Square(double side) {
        this.side = side;
    }

    @Override
    public double area() {
        double base = super.area(); // retorna 0, mas demonstra a sintaxe
        return base + (side * side);
    }
}
```

`super.area()` é útil de verdade quando a superclasse tem lógica parcial que a subclasse quer **complementar**, não substituir por completo — como no exemplo do `Manager.calculateBonus()` da seção de teoria, onde o bônus do gestor é "o bônus base + um extra", não um cálculo do zero.

**Exercício 3**

java

```java
public class Animal {
    public static String category() {
        return "Generic animal";
    }
}

public class Bird extends Animal {
    public static String category() {
        return "Bird";
    }
}

public class Main {
    public static void main(String[] args) {
        Animal a = new Bird();
        System.out.println(Animal.category()); // "Generic animal"
        System.out.println(Bird.category());   // "Bird"

        // Métodos estáticos são associados à CLASSE em tempo de compilação,
        // não ao objeto em tempo de execução. A variável 'a' é declarada como
        // tipo Animal — então Animal.category() (mesmo chamado através de uma
        // referência que aponta pra um Bird em runtime) usa a versão de Animal.
        // Isso é o oposto do que acontece com métodos de instância (overriding),
        // onde o tipo REAL do objeto decide qual versão roda.
    }
}
```

Esse exercício existe porque é um erro sutil e comum: gente assume que "herança + mesma assinatura = polimorfismo" sempre, mas isso só vale pra métodos de **instância**. Estático nunca participa de dynamic dispatch.

**Exercício 4**

java

```java
public class Person {
    protected String name;

    public Person(String name) {
        this.name = name;
    }

    public String summary() {
        return "Name: " + name;
    }
}

public class Employee extends Person {
    protected double salary;

    public Employee(String name, double salary) {
        super(name);
        this.salary = salary;
    }

    @Override
    public String summary() {
        return super.summary() + ", Salary: " + salary;
    }
}

public class Manager extends Employee {
    private int teamSize;

    public Manager(String name, double salary, int teamSize) {
        super(name, salary);
        this.teamSize = teamSize;
    }

    @Override
    public String summary() {
        return super.summary() + ", Team size: " + teamSize;
    }
}
```

`new Manager("Ana", 5000, 3).summary()` chama `Manager.summary()`, que chama `super.summary()` (o de `Employee`), que por sua vez chama `super.summary()` (o de `Person`) — formando uma cadeia. Resultado: `"Name: Ana, Salary: 5000.0, Team size: 3"`. Esse padrão (cada nível só cuida da sua parte e delega o resto pro `super`) é o que evita duplicar a lógica de formatação em três lugares diferentes — é uma aplicação direta de "não repita o que a superclasse já resolve".