#### 1. Teoria

Static vs Dynamic Binding é, na verdade, o **nome formal e mais amplo** do mecanismo que você já viu funcionando em três sessões diferentes — overloading, overriding, e method hiding. Hoje é a sessão que amarra tudo isso num conceito único e dá vocabulário preciso pra ele.

**Binding** é o processo de decidir **qual implementação de método efetivamente vai rodar** quando uma chamada é feita. Existem dois momentos possíveis pra essa decisão:

**Static Binding (early binding)** — a decisão acontece em **tempo de compilação**, com base no **tipo declarado** da referência (o tipo que aparece na declaração da variável), não no objeto real que ela aponta em runtime. Isso se aplica a:

- Métodos `private`
- Métodos `static`
- Métodos `final`
- Overloading (a escolha de _qual_ overload, entre vários com o mesmo nome)

O que esses quatro têm em comum: em nenhum dos casos existe possibilidade de uma subclasse "interceptar" a chamada com uma versão diferente — `private` não é herdado, `static` não participa de hierarquia de instância, `final` proíbe explicitamente ser sobrescrito, e overload já foi decidido antes mesmo de pensar em herança.

**Dynamic Binding (late binding)** — a decisão acontece em **tempo de execução**, com base no **tipo real do objeto** (o que foi de fato instanciado com `new`), não no tipo da variável. Isso se aplica exclusivamente a:

- Métodos de instância que **não** são `private`, `static` ou `final` — ou seja, métodos elegíveis a overriding.

Esse é o mecanismo por trás de **todo** o polimorfismo de overriding que você já praticou nas últimas sessões (`Shape s = new Circle(...); s.calculateArea();` chamando a versão de `Circle`).

**Como a JVM implementa isso, em alto nível (sem prometer detalhe de bytecode que eu não tenha certeza absoluta):** métodos com dynamic binding são despachados através de uma tabela de métodos virtuais (comumente chamada de _vtable_, um conceito geral de linguagens OOP) associada à classe real do objeto, consultada em runtime. Métodos com static binding são resolvidos direto, sem essa indireção, porque não há ambiguidade a resolver depois de compilado. Se você quiser o detalhe exato de como a JVM implementa isso (invokevirtual vs invokestatic vs invokespecial no bytecode), recomendo verificar a JVM Specification oficial (docs.oracle.com/javase/specs) — isso é implementação interna da JVM, não Java como linguagem, e prefiro não afirmar detalhe de bytecode sem confirmar.

**Onde isso aparece de verdade no backend:** entender essa diferença é o que evita bugs sutis como o do Exercício 3 da sessão anterior (`calculateArea(int precision)` sem `@Override`) — e também explica por que frameworks como Spring, Hibernate e Mockito (que você vai usar mais adiante) frequentemente **exigem que métodos não sejam `final`**: essas ferramentas criam proxies/subclasses dinâmicas em runtime pra interceptar chamadas (ex: abrir uma transação antes de um método `@Transactional` rodar), e isso só é possível através de dynamic binding — um método `final` bloqueia esse mecanismo completamente, porque não pode ser sobrescrito pelo proxy.

---

#### 2. Exemplo de código comentado

java

```java
public class Base {
    // STATIC BINDING: método estático, resolvido pelo tipo da referência.
    public static String staticMethod() {
        return "Base.staticMethod";
    }

    // STATIC BINDING: método final, não pode ser sobrescrito, resolvido direto.
    public final String finalMethod() {
        return "Base.finalMethod";
    }

    // STATIC BINDING: método private, sequer visível/herdável pela subclasse.
    private String privateMethod() {
        return "Base.privateMethod";
    }

    // DYNAMIC BINDING: método de instância normal, elegível a override.
    public String instanceMethod() {
        return "Base.instanceMethod";
    }

    // Chama privateMethod() internamente — importante ver o que acontece
    // quando Derived "tenta" sobrescrever isso.
    public String callPrivate() {
        return privateMethod();
    }
}

public class Derived extends Base {
    // Isso NÃO é override — é method hiding, porque staticMethod é static.
    public static String staticMethod() {
        return "Derived.staticMethod";
    }

    // Isso NÃO é override, NEM hiding — é um método NOVO e independente.
    // privateMethod() de Base nem é visível aqui; esse "private" é outro método,
    // sem nenhuma relação com o da superclasse.
    private String privateMethod() {
        return "Derived.privateMethod";
    }

    // Isso É override de verdade — dynamic binding em ação.
    @Override
    public String instanceMethod() {
        return "Derived.instanceMethod";
    }
}

public class Main {
    public static void main(String[] args) {
        Base b = new Derived();

        System.out.println(b.instanceMethod()); // "Derived.instanceMethod" — dynamic binding, olha o objeto REAL
        System.out.println(b.finalMethod());    // "Base.finalMethod" — final, static binding
        System.out.println(Base.staticMethod()); // "Base.staticMethod" — chamado pela classe, sem ambiguidade
        System.out.println(b.callPrivate());
        // "Base.privateMethod" — e não "Derived.privateMethod"!
        // callPrivate() está DEFINIDO em Base, então a chamada interna a
        // privateMethod() dentro dele é resolvida em static binding, olhando
        // o que existe em Base — Derived.privateMethod() é invisível pra Base,
        // são dois métodos completamente não relacionados que só têm o mesmo nome.
    }
}
```

---

#### 3. Armadilhas comuns

- **Achar que `b.callPrivate()` vai "enxergar" o `privateMethod()` de `Derived`.** Esse é o erro mais contraintuitivo da sessão: como `callPrivate()` está definido em `Base` e chama `privateMethod()` internamente, essa chamada é resolvida com static binding **no contexto de `Base`**, mesmo que o objeto real seja um `Derived`. `Derived.privateMethod()` não é uma sobrescrita — é um método totalmente separado que só coincidentemente tem o mesmo nome e assinatura.
- **Confundir "não pode ser sobrescrito" (`final`) com "não pode ser chamado por subclasse".** Um método `final` da superclasse **é herdado normalmente** e pode ser chamado pela subclasse — só não pode ser **redefinido**. Isso ainda é static binding (não existe ambiguidade de qual versão rodar, porque só existe uma).
- **Achar que campos (atributos) participam de dynamic binding igual métodos.** Eles não. Se uma subclasse declara um campo com o mesmo nome de um campo da superclasse ("field hiding"), o acesso a esse campo é sempre resolvido pelo **tipo da referência declarada**, nunca pelo tipo real do objeto — campos nunca são polimórficos em Java, só métodos de instância não-`private`/`static`/`final`.
- **Achar que construtor participa de dynamic binding.** Construtores não são herdados e não são "sobrescritos" — cada classe tem os seus próprios, encadeados via `super()`. A pergunta "qual construtor roda" não é uma questão de binding polimórfico, é simplesmente qual `new` você chamou.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Reproduza o exemplo de field hiding citado na teoria: crie `Base` com um campo público `String label = "Base label"` e `Derived extends Base` com **outro** campo público, também chamado `label`, valendo `"Derived label"`. No `main`, crie `Base ref = new Derived();` e imprima `ref.label`. Depois, crie `Derived ref2 = new Derived();` e imprima `ref2.label`. Escreva um comentário explicando por que os dois resultados são diferentes mesmo os dois objetos sendo, em runtime, `Derived`.  
_Critério de pronto:_ seu comentário identifica que campos são resolvidos por static binding (tipo da referência), diferente de métodos de instância.

**Exercício 2 (Fácil/Médio)**  
Crie `Vehicle` com um método `final String type()` retornando `"Generic vehicle"`, e um método de instância normal `String description()` retornando `"A vehicle"`. Crie `Car extends Vehicle` sobrescrevendo `description()` (não tente sobrescrever `type()` — tente, veja o erro de compilação, e depois remova pra deixar o código compilando, comentando o que aconteceu).  
_Critério de pronto:_ o código final compila, `description()` demonstra dynamic binding funcionando, e existe um comentário relatando o erro de compilação que você teve ao tentar sobrescrever `type()`.

**Exercício 3 (Médio/Difícil)**  
Reproduza fielmente o cenário do `callPrivate()` da teoria, mas com um caso novo: crie `Notifier` com um método público `void sendAll()` que chama internamente um método `private String channel()` retornando `"default channel"`, imprimindo `"Sending via: " + channel()`. Crie `EmailNotifier extends Notifier` com seu **próprio** método `private String channel()` retornando `"email channel"` (sem `@Override` — nem seria possível, já que é `private`). No `main`, crie `Notifier n = new EmailNotifier(); n.sendAll();`.  
_Critério de pronto:_ seu comentário prevê corretamente, **antes de rodar**, se a saída vai ser `"Sending via: default channel"` ou `"Sending via: email channel"`, e explica por quê usando o conceito de binding.

**Exercício 4 (Desafio)**  
Modele um cenário que mostra a motivação prática de `final` citada na teoria (frameworks que criam proxies): crie uma interface `Interceptable` com um método `void execute()`. Crie uma classe `Service` que implementa `Interceptable`, com `execute()` **não-final**, imprimindo `"Executing real logic"`. Crie uma classe `LoggingProxy implements Interceptable` que **recebe um `Service` no construtor** (composição, não herança) e, no seu `execute()`, imprime `"Before execution"`, chama `service.execute()`, e imprime `"After execution"`. Depois, escreva em comentário: se `Service.execute()` fosse `final`, isso impediria esse padrão de proxy por composição funcionar? Justifique.  
_Critério de pronto:_ o proxy funciona corretamente encadeando log + execução real; o comentário responde corretamente que `final` **não** impediria esse proxy específico (porque é composição, não herança/override) — mas impediria uma abordagem alternativa de proxy baseada em **subclassing** (criar uma subclasse dinâmica de `Service` que sobrescreve `execute()`), que é como ferramentas como Mockito/Spring frequentemente implementam proxies por baixo dos panos.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Base {
    public String label = "Base label";
}

public class Derived extends Base {
    public String label = "Derived label"; // field hiding, não override — campos não são polimórficos
}

public class Main {
    public static void main(String[] args) {
        Base ref = new Derived();
        System.out.println(ref.label); // "Base label"

        Derived ref2 = new Derived();
        System.out.println(ref2.label); // "Derived label"

        // Os dois objetos são, em runtime, EXATAMENTE o mesmo Derived.
        // A diferença de resultado vem exclusivamente do tipo DECLARADO da
        // referência (Base vs Derived) — porque acesso a campo é sempre
        // static binding, nunca olha o tipo real do objeto como métodos fazem.
    }
}
```

Esse comportamento é uma das razões práticas pelas quais "programar contra a interface/tipo mais genérico possível" pode ter efeitos colaterais sutis se você depender de campos públicos — mais um motivo pra preferir encapsulamento com métodos (que são polimórficos) a campos públicos (que não são).

**Exercício 2**

java

```java
public class Vehicle {
    public final String type() {
        return "Generic vehicle";
    }

    public String description() {
        return "A vehicle";
    }
}

public class Car extends Vehicle {
    // Tentativa que NÃO compila (comentada):
    // @Override
    // public String type() {
    //     return "Car";
    // }
    // Erro de compilação: "type() in Car cannot override type() in Vehicle;
    // overridden method is final" — o compilador bloqueia explicitamente
    // qualquer tentativa de redefinir um método final.

    @Override
    public String description() {
        return "A car, which is a type of vehicle";
    }
}
```

O erro de compilação real do `javac` tem essa forma geral (parafraseando o conteúdo, não a mensagem exata, que pode variar levemente entre versões do JDK) — o ponto é que isso é **erro de compilação**, não warning, não exceção em runtime: o Java simplesmente não permite que esse código exista.

**Exercício 3**

java

```java
public class Notifier {
    public void sendAll() {
        System.out.println("Sending via: " + channel());
    }

    private String channel() {
        return "default channel";
    }
}

public class EmailNotifier extends Notifier {
    // Método NOVO, sem relação com o de Notifier (que é private, invisível aqui).
    // Não é override — nem poderia ser, private não participa de binding dinâmico.
    private String channel() {
        return "email channel";
    }
}

public class Main {
    public static void main(String[] args) {
        Notifier n = new EmailNotifier();
        n.sendAll();
        // Previsão CORRETA: "Sending via: default channel"
        // sendAll() está definido em Notifier, e a chamada interna a channel()
        // dentro dele é resolvida por static binding NO CONTEXTO DE NOTIFIER —
        // é irrelevante que o objeto real seja um EmailNotifier, porque
        // channel() é private e portanto nunca participa de dynamic dispatch.
    }
}
```

Se alguém espera `"Sending via: email channel"` aqui, geralmente é porque está aplicando a intuição de overriding (que já é automática depois de tantas sessões) num caso onde ela simplesmente não se aplica — `private` quebra a cadeia de polimorfismo completamente.

**Exercício 4**

java

```java
public interface Interceptable {
    void execute();
}

public class Service implements Interceptable {
    @Override
    public void execute() {
        System.out.println("Executing real logic");
    }
}

public class LoggingProxy implements Interceptable {
    private final Service service; // composição: proxy TEM UM Service

    public LoggingProxy(Service service) {
        this.service = service;
    }

    @Override
    public void execute() {
        System.out.println("Before execution");
        service.execute();
        System.out.println("After execution");
    }
}

public class Main {
    public static void main(String[] args) {
        Interceptable proxy = new LoggingProxy(new Service());
        proxy.execute();
        // Before execution
        // Executing real logic
        // After execution
    }
}

// Se Service.execute() fosse final, esse proxy específico continuaria
// funcionando perfeitamente — LoggingProxy nunca tenta SOBRESCREVER
// execute() de Service, ele só CHAMA o método a partir de uma referência
// própria (composição). final só bloqueia herança/override, não chamada.
//
// O que final QUEBRARIA é uma abordagem diferente de proxy, onde o
// framework cria uma SUBCLASSE dinâmica de Service em runtime
// (ex: Service$$EnhancerBySpring) que sobrescreve execute() pra injetar
// lógica antes/depois automaticamente, sem você escrever LoggingProxy
// manualmente. Essa é a técnica que Spring AOP e Mockito frequentemente
// usam por baixo dos panos, e é exatamente por isso que classes/métodos
// final são um problema conhecido ao usar essas ferramentas.
```

Esse exercício conecta o conceito abstrato de binding com uma decisão de design real: existem dois jeitos de fazer proxy (composição manual, como `LoggingProxy`, vs subclassing dinâmico automático, como frameworks fazem) — e só o segundo depende de dynamic binding via herança, o que explica por que a recomendação "evite `final` em classes/métodos que o Spring vai gerenciar" existe na prática, sem você precisar decorar a regra sem entender o motivo.