#### 1. Teoria

**O que é Injeção de Dependência (DI)?**

Dependency Injection é um padrão de design onde um objeto **não cria** as dependências de que precisa — ele **recebe** essas dependências de fora (via construtor, setter ou campo). Quem monta essas dependências e "entrega" para o objeto é um componente externo, chamado de **container de IoC** (Inversion of Control) — no seu caso, o container do Spring.

Repare no nome: **Inversão de Controle**. Antes do padrão, é a classe que controla quem ela usa (`new MinhaDependencia()` dentro dela mesma). Com DI, esse controle é invertido: quem decide qual implementação usar, quando criar, e como configurar, é o container — a classe só declara "eu preciso disso" e recebe pronto.

**Sem DI (acoplamento forte):**

java

```java
public class UserService {
    private UserRepository userRepository = new UserRepositoryImpl(); 
    // UserService decide e cria a implementação concreta.
    // Se eu quiser trocar por um mock em teste, ou por outra implementação,
    // preciso mudar o código-fonte de UserService.
}
```

**Com DI (acoplamento fraco):**

java

```java
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) { // recebido de fora
        this.userRepository = userRepository;
    }
}
```

`UserService` não sabe, nem precisa saber, qual implementação concreta de `UserRepository` está recebendo. Isso é o que permite trocar a implementação real por um mock no teste, sem tocar em `UserService`.

**Diferenciando de coisas que se confundem:**

- **DI ≠ Inversão de Controle (IoC).** IoC é o princípio geral (quem controla o quê). DI é **uma forma específica** de implementar IoC. Outra forma de IoC é o **Service Locator** (a classe pede a dependência a um registro central, tipo `ServiceLocator.get(UserRepository.class)`) — só que aqui a classe ainda sabe que existe um localizador e depende dele. DI é considerado superior porque a classe não depende de nada além dos seus próprios parâmetros — nem sabe que existe um container.
- **DI ≠ Factory Pattern.** Uma Factory também "cria objetos pra você", mas quem chama a Factory ainda decide _quando_ pedir o objeto. Em DI, você nem chama nada — a dependência já chega pronta antes do seu código rodar.

**As 3 formas de injeção:**

1. **Constructor Injection** — dependência entra via parâmetro do construtor.
2. **Setter Injection** — dependência entra via método setter, depois do objeto já criado.
3. **Field Injection** — dependência é injetada diretamente no campo (via reflection), sem construtor nem setter visível.

Segundo a documentação e as recomendações do time do Spring, **constructor injection é a forma recomendada** — e vou justificar isso na seção de armadilhas.

**Para que serve na prática (backend real):**

Isso é o motor por trás de praticamente toda classe anotada com `@Service`, `@Repository`, `@Component` e `@Controller` que você já viu no Bloco 6 (Spring). Quando você escreve:

java

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

O Spring, no boot da aplicação, escaneia as classes anotadas, monta um grafo de dependências, e instancia tudo na ordem certa, injetando cada uma onde é pedida. Você nunca escreve `new OrderService(new OrderRepositoryImpl())` manualmente — o container faz isso.

---

#### 2. Exemplo de código comentado

java

```java
// ---------- Sem Spring, DI "na mão" (pra entender o conceito puro) ----------

interface PaymentGateway {
    void charge(double amount);
}

class StripeGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Cobrando R$" + amount + " via Stripe");
    }
}

class CheckoutService {
    private final PaymentGateway paymentGateway; // depende da ABSTRAÇÃO, não da implementação concreta

    // Constructor injection manual: quem monta o objeto decide qual implementação entra aqui.
    public CheckoutService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void checkout(double total) {
        paymentGateway.charge(total);
    }
}

class MainSemFramework {
    public static void main(String[] args) {
        // Aqui, EU (o código cliente) monto a dependência e injeto manualmente.
        // Isso já é DI — não precisa de framework nenhum pra ser DI.
        PaymentGateway gateway = new StripeGateway();
        CheckoutService checkout = new CheckoutService(gateway);
        checkout.checkout(150.0);
    }
}
```

java

```java
// ---------- Com Spring, o container faz esse "montar e injetar" pra você ----------

interface PaymentGateway {
    void charge(double amount);
}

@Component // registra essa classe como um bean gerenciado pelo container
class StripeGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Cobrando R$" + amount + " via Stripe");
    }
}

@Service
class CheckoutService {
    private final PaymentGateway paymentGateway;

    // Desde o Spring 4.3, se a classe tem UM ÚNICO construtor,
    // o @Autowired é OPCIONAL — o Spring detecta e injeta automaticamente.
    public CheckoutService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void checkout(double total) {
        paymentGateway.charge(total);
    }
}

// Em uma aplicação Spring Boot real, você nunca chama "new CheckoutService(...)".
// O container escaneia @Component/@Service no classpath, cria os beans,
// resolve que CheckoutService precisa de um PaymentGateway,
// encontra o bean StripeGateway (única implementação da interface) e injeta.
```

---

#### 3. Armadilhas comuns

1. **Usar Field Injection por parecer mais curto.**

java

```java
@Service
class CheckoutService {
    @Autowired
    private PaymentGateway paymentGateway; // funciona, mas é desaconselhado
}
```

Isso funciona, mas tem três problemas reais: (a) o campo não pode ser `final`, então perde imutabilidade; (b) você não consegue instanciar essa classe em um teste unitário puro sem subir o contexto do Spring ou usar reflection, porque não existe construtor pra passar o mock; (c) as dependências da classe ficam escondidas — não aparecem na "assinatura pública" da classe, só analisando os campos. Por isso o time do Spring recomenda constructor injection como padrão.

2. **Ambiguidade quando existe mais de uma implementação da mesma interface.** Se você tiver `StripeGateway` e `PaypalGateway` implementando `PaymentGateway`, e tentar injetar `PaymentGateway` sem indicar qual, o Spring não sabe qual escolher e a aplicação falha ao subir. Resolve-se com `@Primary` (marca uma implementação como padrão) ou `@Qualifier("nomeDoBean")` (especifica explicitamente qual injetar).
3. **Dependência circular (circular dependency).** `ClasseA` depende de `ClasseB` no construtor, e `ClasseB` depende de `ClasseA` no construtor também. Com constructor injection, isso quebra na inicialização — o Spring não consegue decidir quem cria primeiro. Isso costuma ser sintoma de **design ruim** (as duas classes deveriam ser uma só, ou precisa de uma terceira classe mediando). Não é pra "resolver" trocando pra field injection (que mascara o problema deixando ele acontecer em runtime, de forma mais difícil de rastrear) — é pra **refatorar o design**.
4. **Confundir "múltiplos construtores" sem indicar qual usar.** Se a classe tem mais de um construtor, o Spring não sabe automaticamente qual usar pra injeção — você precisa marcar explicitamente com `@Autowired` o construtor que deve ser usado, senão a aplicação falha ao subir.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Você tem esta classe usando acoplamento forte:

java

```java
class NotificationService {
    private EmailSender emailSender = new EmailSender();
    public void notify(String msg) {
        emailSender.send(msg);
    }
}
```

Refatore para usar **constructor injection manual** (sem Spring, só Java puro), criando uma interface `MessageSender` que `EmailSender` implementa. Critério de pronto: `NotificationService` não pode conter nenhum `new` de uma implementação concreta dentro dela.

**Exercício 2 (Médio)**  
Usando anotações do Spring (`@Component`, `@Service`), crie:

- Uma interface `Logger` com método `log(String msg)`.
- Uma implementação `ConsoleLogger`.
- Uma classe `AuditService` que recebe `Logger` via constructor injection (sem usar `@Autowired` explicitamente, aproveitando a regra do construtor único).

Critério de pronto: o código compila e, conceitualmente, o Spring conseguiria montar o grafo de dependência sem ambiguidade.

**Exercício 3 (Difícil)**  
Agora crie **duas** implementações de `Logger`: `ConsoleLogger` e `FileLogger`. Faça `AuditService` injetar especificamente a `FileLogger`, mesmo com as duas implementações registradas como bean. Mostre duas formas de resolver isso (`@Primary` e `@Qualifier`) e explique a diferença de comportamento entre as duas abordagens.

**Exercício 4 (Desafio)**  
Você recebeu este código com um bug de dependência circular:

java

```java
@Service
class OrderService {
    private final InvoiceService invoiceService;
    public OrderService(InvoiceService invoiceService) {
        this.invoiceService = invoiceService;
    }
}

@Service
class InvoiceService {
    private final OrderService orderService;
    public InvoiceService(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

Identifique por que isso quebra a aplicação na subida, e proponha uma **refatoração de design** (não um workaround técnico tipo field injection ou `@Lazy`) que elimine a dependência circular. Justifique sua escolha de design.

---

#### 5. Gabarito comentado

**Exercício 1:**

java

```java
interface MessageSender {
    void send(String msg);
}

class EmailSender implements MessageSender {
    @Override
    public void send(String msg) {
        System.out.println("Enviando email: " + msg);
    }
}

class NotificationService {
    private final MessageSender messageSender;

    public NotificationService(MessageSender messageSender) {
        this.messageSender = messageSender;
    }

    public void notify(String msg) {
        messageSender.send(msg);
    }
}

// Uso:
NotificationService service = new NotificationService(new EmailSender());
```

Raciocínio: o ponto central não é "usar Spring", é que `NotificationService` passou a depender de uma **abstração** (`MessageSender`), e quem decide a implementação concreta é o código que instancia `NotificationService`, não a própria classe. Isso já é DI, com ou sem framework.

**Exercício 2:**

java

```java
interface Logger {
    void log(String msg);
}

@Component
class ConsoleLogger implements Logger {
    @Override
    public void log(String msg) {
        System.out.println("[LOG] " + msg);
    }
}

@Service
class AuditService {
    private final Logger logger;

    public AuditService(Logger logger) { // sem @Autowired: único construtor, Spring 4.3+
        this.logger = logger;
    }
}
```

Raciocínio: como só existe uma implementação de `Logger` registrada (`ConsoleLogger`), não há ambiguidade — o Spring resolve sozinho. E como `AuditService` tem um único construtor, `@Autowired` é redundante (mas não é erro colocar, se você preferir deixar explícito por legibilidade — trade-off de estilo, não de correção).

**Exercício 3:**

java

```java
interface Logger {
    void log(String msg);
}

@Component
class ConsoleLogger implements Logger {
    @Override
    public void log(String msg) {
        System.out.println("[CONSOLE] " + msg);
    }
}

@Primary // opção A: marca essa implementação como a "padrão" quando houver ambiguidade
@Component
class FileLogger implements Logger {
    @Override
    public void log(String msg) {
        System.out.println("[FILE] " + msg);
    }
}

@Service
class AuditService {
    private final Logger logger;

    // Opção B (alternativa a @Primary): @Qualifier aponta o bean pelo nome explicitamente
    public AuditService(@Qualifier("fileLogger") Logger logger) {
        this.logger = logger;
    }
}
```

Trade-off entre as duas abordagens: `@Primary` define uma preferência **global** — toda vez que alguém pedir `Logger` sem especificar, `FileLogger` ganha por padrão, em qualquer classe da aplicação. `@Qualifier` é uma decisão **local**, ponto a ponto — só aquele parâmetro específico pede explicitamente o bean `fileLogger`, e outros pontos de injeção continuam livres para escolher diferente (ou ainda dar erro de ambiguidade, se não especificarem nada e não houver `@Primary`). Use `@Primary` quando existe uma implementação "óbvia padrão" na aplicação toda; use `@Qualifier` quando o mesmo tipo precisa de implementações diferentes em pontos diferentes do sistema.

**Exercício 4:**  
O problema: para o Spring criar `OrderService`, ele primeiro precisa de uma instância pronta de `InvoiceService` (porque é parâmetro do construtor). Mas para criar `InvoiceService`, ele precisa de uma instância pronta de `OrderService`. Nenhum dos dois pode ser criado primeiro — com constructor injection, não existe uma ordem válida, e a aplicação falha ao subir com erro de dependência circular.

Refatoração de design (não workaround): isso geralmente indica que as duas classes têm responsabilidades **misturadas** — cada uma "sabe demais" sobre a outra. Uma solução de design comum é extrair a lógica que realmente precisa das duas pontas para uma terceira classe, que orquestra ambas:

java

```java
@Service
class OrderService {
    // não depende mais de InvoiceService
    public void createOrder() {
        System.out.println("Pedido criado");
    }
}

@Service
class InvoiceService {
    // não depende mais de OrderService
    public void generateInvoice() {
        System.out.println("Nota fiscal gerada");
    }
}

@Service
class OrderProcessingService { // orquestra as duas, sem criar ciclo
    private final OrderService orderService;
    private final InvoiceService invoiceService;

    public OrderProcessingService(OrderService orderService, InvoiceService invoiceService) {
        this.orderService = orderService;
        this.invoiceService = invoiceService;
    }

    public void process() {
        orderService.createOrder();
        invoiceService.generateInvoice();
    }
}
```

Justificativa: o grafo de dependência vira uma **árvore** (`OrderProcessingService` → `OrderService` e `InvoiceService`), não mais um ciclo. Isso é preferível a `@Lazy` (que resolveria o erro tecnicamente, adiando a resolução da dependência) porque `@Lazy` só esconde o sintoma — o acoplamento circular de responsabilidade continua existindo no design, só que sem estourar erro. Refatorar é resolver a causa; `@Lazy` é tratar o sintoma.

#### Exercícios práticos — Rodada 2

**Exercício 1 (Fácil)**  
Refatore esta classe, que usa field injection para duas dependências, para constructor injection:

java

```java
@Service
class ReportService {
    @Autowired
    private ReportRepository reportRepository;
    @Autowired
    private ReportFormatter reportFormatter;
}
```

Critério de pronto: os dois campos precisam virar `final`, injetados via construtor.

**Exercício 2 (Médio)**  
Escreva um teste unitário **sem subir o contexto do Spring** (ou seja, sem `@SpringBootTest`) para uma classe `DiscountService` que depende de `TaxCalculator` via constructor injection. Use um mock simples (pode ser uma implementação manual da interface, tipo um "fake", sem precisar de biblioteca de mock) para provar, na prática, o motivo pelo qual constructor injection facilita teste. Critério de pronto: o teste roda como um `main` comum, criando `new DiscountService(fakeCalculator)` diretamente, sem nenhuma dependência do container do Spring.

**Exercício 3 (Difícil)**  
Você tem um cenário de **dependência opcional**: uma classe `AnalyticsService` que só deve enviar eventos para um `EventTracker` se ele estiver disponível — em alguns ambientes/perfis (`profiles`) essa integração não existe. Implemente isso usando **setter injection** (não constructor injection) e explique, no comentário do código, por que setter injection é apropriado aqui e constructor injection não seria (ou seria pior).

**Exercício 4 (Desafio)**  
Implemente um cenário de **Strategy Pattern via DI**: uma interface `ShippingCalculator` com pelo menos 3 implementações (`StandardShipping`, `ExpressShipping`, `InternationalShipping`). Crie uma classe `ShippingService` que recebe **todas** as implementações injetadas de uma vez (numa única coleção, sem `@Qualifier` individual para cada uma) e escolhe qual usar em tempo de execução com base em um parâmetro de entrada (ex: uma `String tipo`). Critério de pronto: adicionar uma quarta implementação de `ShippingCalculator` não deve exigir nenhuma mudança em `ShippingService`.

---

#### Gabarito comentado

**Exercício 1:**

java

```java
@Service
class ReportService {
    private final ReportRepository reportRepository;
    private final ReportFormatter reportFormatter;

    public ReportService(ReportRepository reportRepository, ReportFormatter reportFormatter) {
        this.reportRepository = reportRepository;
        this.reportFormatter = reportFormatter;
    }
}
```

Raciocínio: com único construtor, `@Autowired` nem precisa aparecer — o Spring detecta sozinho (regra válida desde o Spring 4.3). Ganhamos imutabilidade (`final`) e as dependências ficam visíveis na assinatura pública da classe.

**Exercício 2:**

java

```java
interface TaxCalculator {
    double calculate(double amount);
}

class DiscountService {
    private final TaxCalculator taxCalculator;

    public DiscountService(TaxCalculator taxCalculator) {
        this.taxCalculator = taxCalculator;
    }

    public double applyDiscount(double amount) {
        return amount - taxCalculator.calculate(amount);
    }
}

class DiscountServiceTest {
    public static void main(String[] args) {
        // "Fake" manual, sem framework de mock nenhum
        TaxCalculator fakeCalculator = amount -> amount * 0.1; // sempre 10%, previsível pro teste

        DiscountService service = new DiscountService(fakeCalculator);
        double resultado = service.applyDiscount(100.0);

        assert resultado == 90.0 : "Esperava 90.0, veio " + resultado;
        System.out.println("Teste passou: " + resultado);
    }
}
```

Raciocínio: como `DiscountService` recebe a dependência via construtor, criar uma instância de teste é só `new DiscountService(fakeCalculator)` — nenhuma reflection, nenhum contexto do Spring, nenhuma anotação de teste especial. É exatamente esse ponto que fica impossível (ou muito mais complicado, exigindo reflection) se a dependência estivesse em um campo `@Autowired private`.

**Exercício 3:**

java

```java
@Service
class AnalyticsService {
    private EventTracker eventTracker; // não é final — pode não ser setado nunca

    @Autowired(required = false) // dependência OPCIONAL: se não existir bean de EventTracker, não quebra
    public void setEventTracker(EventTracker eventTracker) {
        this.eventTracker = eventTracker;
    }

    public void track(String event) {
        if (eventTracker != null) {
            eventTracker.send(event);
        }
        // se for null, simplesmente não rastreia, sem erro
    }
}
```

Raciocínio: constructor injection é ótimo justamente porque **força** a dependência a existir antes do objeto ficar pronto — mas isso é exatamente o problema quando a dependência é opcional. Se `EventTracker` fosse parâmetro obrigatório do construtor e não houver bean disponível no perfil ativo, a aplicação falharia ao subir. Setter injection com `@Autowired(required = false)` permite que o objeto exista de forma válida mesmo sem essa dependência, tratando a ausência como caso normal (`if (eventTracker != null)`), não como erro fatal.

**Exercício 4:**

java

```java
interface ShippingCalculator {
    boolean supports(String tipo);
    double calculate(double peso);
}

@Component
class StandardShipping implements ShippingCalculator {
    public boolean supports(String tipo) { return "standard".equals(tipo); }
    public double calculate(double peso) { return peso * 2.0; }
}

@Component
class ExpressShipping implements ShippingCalculator {
    public boolean supports(String tipo) { return "express".equals(tipo); }
    public double calculate(double peso) { return peso * 5.0; }
}

@Component
class InternationalShipping implements ShippingCalculator {
    public boolean supports(String tipo) { return "international".equals(tipo); }
    public double calculate(double peso) { return peso * 10.0; }
}

@Service
class ShippingService {
    private final List<ShippingCalculator> calculators;

    // Spring injeta TODOS os beans que implementam ShippingCalculator nessa lista automaticamente
    public ShippingService(List<ShippingCalculator> calculators) {
        this.calculators = calculators;
    }

    public double calcular(String tipo, double peso) {
        return calculators.stream()
                .filter(c -> c.supports(tipo))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Tipo de frete não suportado: " + tipo))
                .calculate(peso);
    }
}
```

Raciocínio: o Spring, ao ver `List<ShippingCalculator>` como parâmetro, automaticamente coleta **todos** os beans registrados que implementam essa interface e injeta como uma lista — isso é um comportamento nativo, não precisa de configuração extra. O `ShippingService` nunca precisa saber quantas implementações existem nem seus nomes; ele só itera e pergunta "você suporta esse tipo?" (padrão Strategy). Isso cumpre o critério do exercício: adicionar uma `PickupShipping` nova, anotada com `@Component`, é suficiente — `ShippingService` continua igual, sem nenhuma alteração. Isso é o Open/Closed Principle na prática, viabilizado por DI.