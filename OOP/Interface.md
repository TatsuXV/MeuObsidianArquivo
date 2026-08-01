#### 1. Teoria

Interface já apareceu nas últimas sessões como ferramenta de abstração, mas hoje é a sessão formal do tópico — cobrindo regras estruturais que ainda não foram tratadas: herança entre interfaces, implementação múltipla, o "diamond problem" e como Java resolve ele, e o conceito de _functional interface_.

**Uma classe pode implementar várias interfaces ao mesmo tempo.** Isso é a principal diferença estrutural em relação a `extends` de classe (que é limitado a uma só):

java

```java
public class Duck implements Swimmable, Flyable, Quackable { }
```

**Interfaces também podem herdar de outras interfaces, com `extends` — e aqui, diferente de classe, uma interface pode `extends` **várias** interfaces ao mesmo tempo:**

java

```java
public interface Swimmable { void swim(); }
public interface Flyable { void fly(); }
public interface AmphibiousBird extends Swimmable, Flyable { }
```

Uma classe que implementa `AmphibiousBird` precisa fornecer `swim()` e `fly()`, mesmo sem mencionar `Swimmable`/`Flyable` diretamente.

**O "diamond problem" com métodos `default`.** Se uma classe implementa duas interfaces que têm um método `default` **com a mesma assinatura**, existe ambiguidade: qual versão usar? Java resolve isso de um jeito estrito — **não escolhe automaticamente**. O código **não compila** até você resolver a ambiguidade explicitamente, sobrescrevendo o método na classe e decidindo (ou combinando) qual versão usar:

java

```java
interface A { default String greet() { return "A"; } }
interface B { default String greet() { return "B"; } }

class C implements A, B {
    @Override
    public String greet() {
        return A.super.greet() + B.super.greet(); // sintaxe especial pra chamar o default de uma interface específica
    }
}
```

Isso é diferente do diamond problem clássico de C++ (que motivou Java a proibir herança múltipla de classe): aqui não existe ambiguidade de **estado** (interfaces não têm campos de instância), só de **comportamento**, e o compilador força a resolução manual em vez de adivinhar.

**Constantes em interface** já foi citado na sessão de Abstraction: qualquer campo declarado numa interface é implicitamente `public static final`. Isso significa que, ao contrário de métodos `default`, campos **nunca** geram ambiguidade de diamond — se duas interfaces têm uma constante de mesmo nome, você só precisa qualificar com o nome da interface (`A.CONST`) ao usar, não existe conflito de "qual versão", porque ambas coexistem como constantes independentes.

**Functional Interface** — uma interface com **exatamente um** método abstrato (métodos `default`/`static` não contam). É o que permite usar lambdas e method references (tópicos futuros do seu checklist, no Bloco 12) no lugar de uma implementação anônima verbosa. A anotação `@FunctionalInterface` é opcional, mas recomendada: ela não muda comportamento nenhum, só faz o compilador **verificar** e recusar compilar se a interface deixar de ter exatamente um método abstrato — é uma rede de segurança contra quebrar o contrato sem perceber. `Runnable`, `Comparable<T>` e `Comparator<T>` da própria API do Java são exemplos de functional interfaces.

**Onde isso aparece de verdade no backend:** múltiplas interfaces implementadas na mesma classe é o padrão comum em Spring (`@RestController` frequentemente implementa uma interface de contrato de API **e** talvez `ErrorController`, por exemplo). Functional interfaces são a base de toda a API de Streams (Bloco 12) e de callbacks assíncronos.

---

#### 2. Exemplo de código comentado

java

```java
// Duas interfaces independentes, cada uma com seu próprio default method
// de mesma assinatura — configurando o cenário de diamond problem.
public interface Loggable {
    default String describe() {
        return "Loggable component";
    }
}

public interface Auditable {
    default String describe() {
        return "Auditable component";
    }
}

// Interface herdando de OUTRA interface (não de classe).
public interface Trackable extends Loggable {
    void track(String event);
}

public class Transaction implements Auditable, Trackable {

    // OBRIGATÓRIO sobrescrever describe(): sem isso, o código não compila,
    // porque Auditable.describe() e Loggable.describe() (herdado via Trackable)
    // colidem, e o compilador se recusa a escolher um dos dois sozinho.
    @Override
    public String describe() {
        // Sintaxe especial InterfaceName.super.metodo() pra escolher explicitamente
        return Auditable.super.describe() + " / " + Loggable.super.describe();
    }

    @Override
    public void track(String event) {
        System.out.println("Tracking: " + event);
    }
}

// FUNCTIONAL INTERFACE: exatamente um método abstrato.
// @FunctionalInterface é opcional, mas o compilador valida a regra se presente.
@FunctionalInterface
public interface Validator<T> {
    boolean isValid(T value);

    // default methods NÃO contam pra regra de "um método abstrato" —
    // Validator continua sendo functional interface mesmo com isso aqui.
    default Validator<T> negate() {
        return value -> !isValid(value);
    }
}

public class Main {
    public static void main(String[] args) {
        Transaction t = new Transaction();
        System.out.println(t.describe()); // "Auditable component / Loggable component"
        t.track("payment processed");

        // Implementação anônima de Validator (antes de Lambda ser seu tópico formal,
        // é assim que isso seria escrito sem a sintaxe curta):
        Validator<String> notEmpty = new Validator<String>() {
            @Override
            public boolean isValid(String value) {
                return value != null && !value.isEmpty();
            }
        };
        System.out.println(notEmpty.isValid(""));  // false
    }
}
```

---

#### 3. Armadilhas comuns

- **Achar que implementar duas interfaces com o mesmo método `default` compila e "escolhe uma automaticamente".** Não compila. Java força você a resolver explicitamente com `@Override`, mesmo que as duas implementações fossem idênticas.
- **Confundir herança de interface (`interface B extends A`) com implementação (`class B implements A`).** Sintaxe parecida (`extends` em vez de `implements`), mas o significado é diferente: interface estendendo interface é composição de **contratos**, sempre permite múltipla (`interface C extends A, B`); classe usa `extends` só pra uma classe, e `implements` (sempre plural possível) pra interfaces.
- **Anotar `@FunctionalInterface` numa interface com dois métodos abstratos e esperar que funcione.** O compilador recusa compilar com uma mensagem de erro explícita — a anotação existe justamente pra pegar esse erro cedo, em vez de descobrir só na hora de tentar usar uma lambda e o compilador reclamar de um jeito menos claro.
- **Esquecer que método `private` de interface (Java 9+) não pode ser chamado de fora, nem por quem implementa.** Ele só serve como auxiliar interno de outros métodos `default`/`static` da própria interface — é encapsulamento aplicado dentro da própria interface, não um contrato pra ninguém implementar ou chamar diretamente.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie duas interfaces, `Drawable` (com `void draw()`) e `Resizable` (com `void resize(double factor)`). Crie uma classe `Shape2D` que implementa **as duas** ao mesmo tempo, fornecendo ambas as implementações.  
_Critério de pronto:_ `Shape2D` compila implementando os dois contratos sem herdar de nenhuma classe.

**Exercício 2 (Fácil/Médio)**  
Crie uma interface `Identifiable` com `String getId()`. Crie uma interface `Auditable extends Identifiable` que adiciona `String getLastModifiedBy()`. Implemente uma classe `Document implements Auditable`.  
_Critério de pronto:_ `Document` é obrigada a implementar **os dois** métodos (`getId()` e `getLastModifiedBy()`), mesmo só declarando `implements Auditable` — prove isso comentando o que aconteceria se você esquecesse de implementar `getId()`.

**Exercício 3 (Médio/Difícil)**  
Reproduza o diamond problem de propósito: crie `Printable` e `Exportable`, cada uma com um `default String summary()` retornando textos diferentes (`"Printable summary"` e `"Exportable summary"`). Crie uma classe `Report implements Printable, Exportable` **sem** sobrescrever `summary()` primeiro — mostre (em comentário) o erro de compilação esperado. Depois, corrija adicionando `@Override` e usando `Printable.super.summary()` e `Exportable.super.summary()` pra combinar as duas em uma string só.  
_Critério de pronto:_ o comentário descreve corretamente por que o código sem `@Override` não compila, e a versão corrigida combina as duas mensagens.

**Exercício 4 (Desafio)**  
Crie uma `@FunctionalInterface` chamada `PriceCalculator<T>` com um único método abstrato `double calculate(T item)`. Adicione um método `default` `PriceCalculator<T> withTax(double taxRate)` que retorna um **novo** `PriceCalculator<T>` (implementação anônima) cujo `calculate()` chama o `calculate()` original e aplica o imposto por cima. Implemente uma instância concreta (anônima) que calcula o preço de um `String` (assuma que a String é um código de produto e retorne, por simplicidade, `10.0` fixo), depois combine com `withTax(0.1)` e mostre que o resultado final já vem com imposto aplicado.  
_Critério de pronto:_ `PriceCalculator<String> base = ...; PriceCalculator<String> withTax = base.withTax(0.1); withTax.calculate("ABC")` retorna `11.0`, sem que `withTax` reimplemente a lógica de cálculo original — só o `default` compõe em cima do que já existe.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public interface Drawable {
    void draw();
}

public interface Resizable {
    void resize(double factor);
}

public class Shape2D implements Drawable, Resizable {
    @Override
    public void draw() {
        System.out.println("Drawing shape");
    }

    @Override
    public void resize(double factor) {
        System.out.println("Resizing by factor " + factor);
    }
}
```

Nada de diamond aqui — as duas interfaces são completamente independentes, sem métodos de mesmo nome, então implementar as duas é só "cumprir dois contratos separados" sem nenhuma ambiguidade.

**Exercício 2**

java

```java
public interface Identifiable {
    String getId();
}

public interface Auditable extends Identifiable {
    String getLastModifiedBy();
}

public class Document implements Auditable {
    private String id;
    private String lastModifiedBy;

    public Document(String id, String lastModifiedBy) {
        this.id = id;
        this.lastModifiedBy = lastModifiedBy;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public String getLastModifiedBy() {
        return lastModifiedBy;
    }
}
// Se getId() fosse omitido, o código NÃO compilaria: "Document is not abstract
// and does not override abstract method getId() in Identifiable" — porque
// Auditable extends Identifiable herda o contrato inteiro, não só o próprio.
```

Isso confirma que herança entre interfaces é cumulativa: `Document` precisa satisfazer o contrato completo da cadeia (`Auditable` + tudo que `Auditable` herdou de `Identifiable`), não só o que está escrito diretamente em `Auditable`.

**Exercício 3**

java

```java
public interface Printable {
    default String summary() {
        return "Printable summary";
    }
}

public interface Exportable {
    default String summary() {
        return "Exportable summary";
    }
}

// VERSÃO COM ERRO (comentada, não compila):
// public class Report implements Printable, Exportable { }
// Erro esperado: "class Report inherits unrelated defaults for summary()
// from types Printable and Exportable" — o compilador se recusa a escolher
// um dos dois automaticamente, porque são implementações DIFERENTES colidindo.

// VERSÃO CORRIGIDA:
public class Report implements Printable, Exportable {
    @Override
    public String summary() {
        return Printable.super.summary() + " + " + Exportable.super.summary();
    }
}
```

A sintaxe `Printable.super.summary()` é exclusiva pra esse cenário (chamar o `default` de uma interface específica) — é diferente de `super.metodo()` de herança de classe, que sempre se refere à superclasse direta sem precisar nomear.

**Exercício 4**

java

```java
@FunctionalInterface
public interface PriceCalculator<T> {
    double calculate(T item);

    default PriceCalculator<T> withTax(double taxRate) {
        // 'this' aqui se refere à implementação original (base),
        // capturada pela implementação anônima retornada abaixo.
        PriceCalculator<T> original = this;
        return new PriceCalculator<T>() {
            @Override
            public double calculate(T item) {
                double basePrice = original.calculate(item);
                return basePrice * (1 + taxRate);
            }
        };
    }
}

public class Main {
    public static void main(String[] args) {
        PriceCalculator<String> base = new PriceCalculator<String>() {
            @Override
            public double calculate(String item) {
                return 10.0;
            }
        };

        PriceCalculator<String> withTax = base.withTax(0.1);
        System.out.println(withTax.calculate("ABC")); // 11.0
    }
}
```

Esse exercício é uma prévia direta do que Streams e composição funcional (Bloco 12) vão explorar muito mais a fundo: um `default` method que **decora** o comportamento original, retornando uma nova implementação que "embrulha" a antiga, sem duplicar a lógica de cálculo em nenhum lugar. É o mesmo princípio do `Discountable.getDiscountedPrice()` da sessão de Abstraction, só que aqui o próprio `default` retorna outra instância da interface, em vez de só um valor calculado.