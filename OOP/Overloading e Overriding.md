#### 1. Teoria

Overloading e Overriding têm nomes parecidos em português (ambos viram "sobrecarga"/"sobrescrita"), mas são mecanismos **completamente diferentes**, resolvidos em momentos diferentes pelo compilador/JVM. Isso já foi citado como armadilha nas sessões 1 e 3 — agora é a vez de aprofundar de verdade.

**Overloading (sobrecarga)** — mesma classe, **mesmo nome de método**, mas **assinatura diferente** (quantidade, tipo ou ordem dos parâmetros diferente). Não tem relação nenhuma com herança.

java

```java
public void print(String s) { }
public void print(int i) { }
public void print(String s, int i) { }
```

Isso é resolvido em **tempo de compilação**: o compilador olha os argumentos que você passou na chamada e decide, ali mesmo, qual versão do método vai ser chamada. Por isso overloading é chamado de **polimorfismo estático** (ou _compile-time polymorphism_) — apesar de tecnicamente não ser o mesmo tipo de polimorfismo do pilar OOP (aquele é sempre sobre dynamic dispatch em hierarquia).

**Overriding (sobrescrita)** — uma subclasse redefine um método **herdado**, com a **mesma assinatura exata** (mesmo nome, mesmos parâmetros, tipo de retorno igual ou covariante). Isso _depende_ de herança — sem `extends`/`implements`, não existe overriding.

java

```java
class Animal { String sound() { return "..."; } }
class Dog extends Animal { @Override String sound() { return "Woof"; } }
```

Isso é resolvido em **tempo de execução**, com base no tipo **real** do objeto (não no tipo da variável) — é o dynamic dispatch que você já viu funcionando nas sessões anteriores. É por isso que overriding é o polimorfismo "de verdade" do pilar OOP.

**Regras que o compilador força no overriding:**

- Mesma assinatura (nome + tipos e ordem de parâmetros).
- Tipo de retorno igual, ou **covariante** (um subtipo do retorno original — ex: superclasse retorna `Animal`, subclasse pode retornar `Dog`).
- Não pode reduzir a visibilidade (já visto na sessão de Inheritance).
- Não pode lançar uma exception checada **mais ampla** que a declarada no método original (pode lançar a mesma, uma mais específica, ou nenhuma).

**Regras que decidem qual overload é chamado (resolução de sobrecarga):** o compilador procura, nessa ordem de preferência: (1) match exato de tipo, (2) match via widening primitivo (`int` → `long` → `double`, sem perda de dado), (3) match via autoboxing (`int` → `Integer`), (4) match via varargs (`String...`). Isso importa na prática porque **ambiguidade entre overloads é erro de compilação**, não de execução — se dois overloads "empatam" na prioridade pra uma chamada específica, o código nem compila.

**Onde isso aparece de verdade no backend:** overloading aparece o tempo todo em construtores (`new BigDecimal(String)` vs `new BigDecimal(double)` — e não são equivalentes, isso inclusive é uma armadilha clássica documentada na própria Javadoc do `BigDecimal`, mas não afirmo detalhe de comportamento aqui sem confirmar — se precisar do detalhe exato eu busco na documentação). Overriding é a base de qualquer camada de serviço no Spring: um `@Service` implementando uma interface de `Repository` customizada, ou uma exception customizada sobrescrevendo `getMessage()`.

---

#### 2. Exemplo de código comentado

java

```java
public class Calculator {

    // OVERLOADING: mesmo nome "add", assinaturas diferentes.
    // Resolvido em tempo de COMPILAÇÃO, com base nos tipos dos argumentos.
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }

    // Varargs: forma especial de overload, aceita 0 ou mais argumentos.
    public int add(int... numbers) {
        int sum = 0;
        for (int n : numbers) sum += n;
        return sum;
    }
}

public class Base {
    public String describe() {
        return "I am a Base";
    }

    // Retorno covariante será demonstrado abaixo por Derived.create()
    public Base create() {
        return new Base();
    }
}

public class Derived extends Base {
    // OVERRIDING: mesma assinatura de describe(), resolvido em tempo de EXECUÇÃO.
    @Override
    public String describe() {
        return "I am a Derived, and my parent says: " + super.describe();
    }

    // Retorno covariante: Base.create() retorna Base, aqui retornamos Derived
    // (um subtipo de Base) — isso é permitido, não é overload nem erro.
    @Override
    public Derived create() {
        return new Derived();
    }
}

public class Main {
    public static void main(String[] args) {
        Calculator calc = new Calculator();
        System.out.println(calc.add(2, 3));         // chama add(int, int) — decidido em compilação
        System.out.println(calc.add(2.5, 3.5));      // chama add(double, double)
        System.out.println(calc.add(1, 2, 3));       // chama add(int, int, int)

        Base b = new Derived();
        System.out.println(b.describe());  // "I am a Derived..." — decidido em RUNTIME pelo tipo real
    }
}
```

---

#### 3. Armadilhas comuns

- **Achar que mudar só o tipo de retorno é overload válido.** `int add(int a, int b)` e `double add(int a, int b)` **não compilam juntos** — o compilador não consegue diferenciar qual chamar baseado só no retorno, já que a assinatura (nome + parâmetros) é idêntica. Overload exige diferença nos **parâmetros**, nunca só no retorno.
- **Ambiguidade em overload com autoboxing/widening.** Se você tem `print(long l)` e `print(Integer i)`, chamar `print(5)` (um `int` literal) pode gerar comportamento inesperado: o compilador prefere widening (`int`→`long`) a autoboxing (`int`→`Integer`) quando os dois são candidatos — então `print(5)` chama a versão `long`, não `Integer`, o que surpreende muita gente.
- **Esquecer `@Override` e criar overload sem querer.** Isso já foi citado na sessão 1, mas vale reforçar aqui porque agora você tem o vocabulário certo: se você tenta sobrescrever `calculateArea()` mas digita `calculateArea(int precision)` por engano, o Java **não acusa erro nenhum** — ele só criou um overload novo, e o método original da superclasse continua lá, intacto e não sobrescrito. `@Override` faz o compilador **verificar** que existe mesmo um método com essa assinatura exata pra sobrescrever, e falha a compilação se não existir.
- **Confundir "overriding de método estático" com overriding de verdade.** Method hiding (visto na sessão de Inheritance) tem sintaxe parecida com overriding, mas é resolvido em compile-time — é, na prática, mais parecido com overload em termos de quando é decidido, apesar de "parecer" overriding.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie uma classe `Greeter` com três overloads de um método `greet`: `greet()` (sem parâmetro, retorna `"Hello!"`), `greet(String name)` (retorna `"Hello, " + name + "!"`), e `greet(String name, String language)` (retorna uma saudação diferente dependendo do idioma — pelo menos `"en"` e `"pt"`).  
_Critério de pronto:_ as três chamadas (`greet()`, `greet("Ana")`, `greet("Ana", "pt")`) produzem saídas diferentes e corretas.

**Exercício 2 (Fácil/Médio)**  
Crie `Shape` com um método `void render()` que imprime `"Rendering a shape"`. Crie `Circle extends Shape` sobrescrevendo `render()` pra imprimir `"Rendering a circle"`. Depois, em `Circle`, adicione **também** um overload `void render(String style)` (não uma sobrescrita — um método novo, com parâmetro extra) que imprime `"Rendering a circle with style: " + style`. No `main`, chame os três: `new Shape().render()`, `new Circle().render()`, `new Circle().render("outline")`.  
_Critério de pronto:_ seu código deixa explícito, em comentário, qual dos dois `render` em `Circle` é overload e qual é override, e por quê.

**Exercício 3 (Médio/Difícil)**  
Reproduza a armadilha do `@Override` esquecido de propósito: crie `Shape` com `double calculateArea()` retornando `0.0`. Crie `Circle extends Shape` com um método `double calculateArea(int precision)` (**sem** `@Override`, com um parâmetro a mais por engano) que deveria ter sido um override, mas virou overload sem querer. No `main`, crie `Shape s = new Circle(5);` e chame `s.calculateArea()` — mostre no comentário por que o resultado é `0.0` mesmo `s` sendo, em runtime, um `Circle`. Depois, corrija o bug (remova o parâmetro extra, adicione `@Override`) e mostre que agora `s.calculateArea()` retorna o valor certo.  
_Critério de pronto:_ seu comentário explica que sem `@Override`, o compilador não detectou que a intenção era sobrescrever, e por isso `calculateArea()` (sem parâmetro) chamado em `s` continua executando a versão "burra" de `Shape`.

**Exercício 4 (Desafio)**  
Crie uma classe `Repository<T>` (genérica — se `Generics` ainda não foi seu tópico formal, use só o necessário aqui: `class Repository<T> { }`) com um método `void save(T item)`. Crie `UserRepository extends Repository<User>` (assuma uma classe `User` simples com `id` e `name`) que sobrescreve `save(User item)` — **mas** adicione também um overload `void save(User item, boolean validate)` que, se `validate` for `true`, checa se `item.getName()` não é nulo/vazio antes de delegar pro `save(User item)` sobrescrito (chame o overload de 1 parâmetro de dentro do de 2, não duplique a lógica de salvar). Escreva um comentário identificando qual dos métodos em `UserRepository` é override de `Repository<T>` e qual é overload próprio, novo, que não existe na superclasse genérica.  
_Critério de pronto:_ `save(item, true)` com nome vazio lança exceção; `save(item, true)` com nome válido delega corretamente pro overload de 1 parâmetro.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Greeter {
    public String greet() {
        return "Hello!";
    }

    public String greet(String name) {
        return "Hello, " + name + "!";
    }

    public String greet(String name, String language) {
        if (language.equals("pt")) {
            return "Olá, " + name + "!";
        }
        return "Hello, " + name + "!";
    }
}
```

Os três `greet` coexistem tranquilamente porque a assinatura (número e tipo de parâmetros) é diferente em cada um — é exatamente isso que overload exige. O compilador escolhe qual chamar olhando só pra chamada, sem nenhuma relação com herança.

**Exercício 2**

java

```java
public class Shape {
    public void render() {
        System.out.println("Rendering a shape");
    }
}

public class Circle extends Shape {
    // OVERRIDE: mesma assinatura de Shape.render(), redefine comportamento herdado.
    // Resolvido em runtime pelo tipo real do objeto.
    @Override
    public void render() {
        System.out.println("Rendering a circle");
    }

    // OVERLOAD: método NOVO, não existe em Shape com essa assinatura.
    // Não tem relação com o render() herdado, é só um método adicional de Circle.
    public void render(String style) {
        System.out.println("Rendering a circle with style: " + style);
    }
}
```

A diferença fica clara ao pensar "esse método já existia na superclasse com essa exata assinatura?" — pra `render()` sim (override), pra `render(String)` não (overload, método próprio novo que só `Circle` tem).

**Exercício 3**

java

```java
public class Shape {
    public double calculateArea() {
        return 0.0;
    }
}

// VERSÃO COM BUG:
public class Circle {
    private double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    // SEM @Override, com um parâmetro a mais — isso é overload sem querer,
    // não sobrescreve calculateArea() de Shape.
    public double calculateArea(int precision) {
        return Math.PI * radius * radius;
    }
}
```

Espera — pra reproduzir o bug de verdade, `Circle` precisa `extends Shape`:

java

```java
public class Circle extends Shape {
    private double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    public double calculateArea(int precision) { // deveria ser calculateArea(), sem parâmetro
        return Math.PI * radius * radius;
    }
}

public class Main {
    public static void main(String[] args) {
        Shape s = new Circle(5);
        System.out.println(s.calculateArea());
        // Resultado: 0.0. Mesmo 's' sendo um Circle em runtime, calculateArea()
        // (sem parâmetro) chama a versão de Shape, porque Circle.calculateArea(int)
        // é um método DIFERENTE (overload por engano), não uma sobrescrita.
        // O compilador não avisou nada porque, sintaticamente, criar um overload
        // novo é uma operação 100% válida — só não era a intenção.
    }
}
```

Correção:

java

```java
public class Circle extends Shape {
    private double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double calculateArea() { // sem parâmetro extra, com @Override
        return Math.PI * radius * radius;
    }
}
// Agora s.calculateArea() retorna o valor correto, porque @Override forçou
// o compilador a verificar que existe, de fato, um método com essa assinatura
// exata na superclasse pra sobrescrever — se não existisse, o código não compilaria.
```

Esse exercício é a demonstração mais concreta possível de por que `@Override` não é só estilo — ele transforma um bug silencioso (comportamento errado, sem nenhum erro de compilação) num erro de compilação explícito, caso a assinatura não bata.

**Exercício 4**

java

```java
public class Repository<T> {
    public void save(T item) {
        System.out.println("Saving generic item: " + item);
    }
}

public class User {
    private String name;

    public User(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

public class UserRepository extends Repository<User> {
    // OVERRIDE: Repository<User> espera save(User item) — essa é a versão
    // "especializada" (o T virou User de verdade) do método da superclasse genérica.
    @Override
    public void save(User item) {
        System.out.println("Saving user: " + item.getName());
    }

    // OVERLOAD: método novo, não existe em Repository<T> com essa assinatura.
    // Adiciona validação e DELEGA pro override acima, sem duplicar a lógica de salvar.
    public void save(User item, boolean validate) {
        if (validate && (item.getName() == null || item.getName().isEmpty())) {
            throw new IllegalArgumentException("User name cannot be empty");
        }
        save(item); // chama o overload de 1 parâmetro
    }
}
```

Esse padrão — um overload que só adiciona uma responsabilidade extra (validação) e delega o trabalho de verdade pro método principal — é comum em código de produção, e evita duplicar a lógica de "salvar" em dois lugares. Vale notar que `save(User item)` aqui é tecnicamente um override de um método genérico — em bytecode, o Java gera também um método "bridge" pra fazer isso funcionar com type erasure, mas esse é um detalhe de implementação do compilador que só vale a pena aprofundar quando `Generics` virar seu tópico formal mais adiante.