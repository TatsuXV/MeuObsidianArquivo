#### 1. Teoria

Encapsulamento já apareceu de raspão nas três sessões anteriores (o `Book.readPages()`, o `currentPage` protegido por `Math.min`), mas hoje é o tópico central: **proteger o estado interno de um objeto, garantindo que ele nunca fique num estado inválido, expondo só o que é necessário através de uma interface controlada**.

A ideia central não é "colocar `private` em tudo" — é sobre **invariantes**: regras que precisam ser verdadeiras durante toda a vida do objeto. Um `BankAccount` nunca pode ter saldo negativo (se essa for a regra de negócio); um `Percentage` nunca pode ser menor que 0 ou maior que 100. Encapsulamento é o mecanismo que garante isso, escondendo o campo e controlando _todo_ acesso a ele através de métodos que aplicam a regra.

**Os quatro níveis de acesso em Java, do mais restrito ao mais aberto:**

- `private` — só a própria classe.
- _(default, sem modificador)_ — a própria classe + mesmo pacote.
- `protected` — mesmo pacote + subclasses (mesmo em pacote diferente), como você viu em Inheritance.
- `public` — qualquer lugar.

**Getters/setters não são "boa prática por padrão"** — são uma ferramenta. Um getter que só retorna o campo sem lógica nenhuma (`public String getName() { return name; }`) tecnicamente encapsula (o campo está `private`), mas do ponto de vista de design não adiciona proteção nenhuma além de sintaxe — é indistinguível de um campo público na prática, exceto que você pode adicionar lógica ali no futuro sem quebrar quem já chama `getName()`. O valor real do encapsulamento aparece quando o setter (ou método equivalente) **valida** ou **transforma** antes de aceitar o novo valor, ou quando você nem expõe um setter — só um método de negócio (`deposit(amount)`, não `setBalance(amount)`).

**Imutabilidade é a forma mais forte de encapsulamento.** Se um campo é `final` e só é atribuído no construtor, não existe nenhum jeito de alguém corromper esse estado depois — nem por engano, nem em código concorrente (multi-thread). Você já viu isso nos exemplos anteriores (`private final String name`). Em Java moderno, `record` (que já apareceu no seu checklist, mais adiante) é basicamente um atalho de linguagem pra criar classes imutáveis encapsuladas com muito menos código boilerplate.

**Onde isso aparece de verdade no backend:** entidades JPA que só permitem mudar `status` de um pedido através de um método `cancel()` ou `ship()` (que valida a transição de estado — não dá pra cancelar um pedido já entregue), em vez de um `setStatus(String status)` genérico que aceita qualquer string, são encapsulamento de invariante de negócio. DTOs (Data Transfer Objects), por outro lado, costumam ser propositalmente "burros" (só getters/setters sem lógica) porque o objetivo deles é só carregar dado entre camadas, não proteger regra de negócio — é um caso legítimo onde encapsulamento "forte" não é a prioridade.

---

#### 2. Exemplo de código comentado

java

```java
public class BankAccount {

    // private: ninguém de fora consegue setar balance = -500 direto.
    private double balance;

    // final: o número da conta nunca muda depois de criada.
    private final String accountNumber;

    public BankAccount(String accountNumber, double initialBalance) {
        if (initialBalance < 0) {
            // Validação JÁ no construtor: o objeto nunca chega a existir
            // num estado inválido, nem por um instante.
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    // Sem setBalance(double) público. O ÚNICO jeito de alterar o saldo é
    // através de métodos que representam ações de negócio reais.
    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (amount > balance) {
            // Invariante protegida: saldo nunca fica negativo, e o motivo
            // fica explícito na exceção em vez de silenciosamente "dar errado".
            throw new IllegalStateException("Insufficient funds");
        }
        this.balance -= amount;
    }

    // Getter simples aqui: expor o saldo pra leitura é seguro, porque
    // não existe jeito de usar esse valor de retorno pra CORROMPER o objeto.
    public double getBalance() {
        return balance;
    }

    public String getAccountNumber() {
        return accountNumber;
    }
}
```

---

#### 3. Armadilhas comuns

- **Retornar uma referência mutável direto de um getter (o "vazamento de encapsulamento").** Se um campo é `private List<String> items` e o getter faz `return items;`, quem chamar `getItems().clear()` **apaga a lista de dentro do objeto original**, mesmo sem nenhum setter público. O campo está tecnicamente `private`, mas o encapsulamento foi furado por fora. A correção comum é retornar uma cópia (`return new ArrayList<>(items);`) ou uma view imutável (`return Collections.unmodifiableList(items);`).
- **Validar só no setter e esquecer o construtor (ou vice-versa).** Se `setBalance()` valida saldo negativo mas o construtor aceita `initialBalance` sem checar nada, existe uma porta lateral pra criar o objeto já num estado inválido. Toda invariante precisa ser protegida em **todo** ponto de entrada do estado, não só nos métodos "óbvios".
- **Gerar getter/setter pra tudo automaticamente (via IDE) sem pensar se deveria existir.** IDEs geram `getX`/`setX` pra cada campo com um clique, mas isso frequentemente recria o problema de "campo público disfarçado". A pergunta certa antes de gerar um setter é: "existe alguma regra que esse valor precisa respeitar? Se sim, esse setter deveria validar, ou nem deveria existir — só um método de negócio."
- **Confundir `final` de campo com objeto imutável.** `private final List<String> items = new ArrayList<>();` significa que a **referência** `items` nunca vai apontar pra outra lista — mas a lista em si continua totalmente mutável (`items.add(...)` funciona livremente de dentro da classe, e se vazar pelo getter, de fora também). `final` só trava a variável, não o conteúdo do objeto que ela aponta.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie uma classe `Temperature` com um campo privado `double celsius`. O construtor deve validar que a temperatura não é menor que `-273.15` (zero absoluto) — se for, lança `IllegalArgumentException`. Adicione `getCelsius()` e `getFahrenheit()` (calculado a partir de `celsius`, sem campo próprio pra Fahrenheit).  
_Critério de pronto:_ `new Temperature(-300)` lança exceção; `new Temperature(25).getFahrenheit()` retorna `77.0`.

**Exercício 2 (Fácil/Médio)**  
Crie uma classe `Playlist` com um campo privado `List<String> songs`, inicializado vazio no construtor. Adicione `addSong(String song)` e um getter `getSongs()`. Sem usar `Collections.unmodifiableList` nem criar cópia ainda — só implemente do jeito "ingênuo" primeiro (retornando `songs` direto). Depois, escreva um teste manual no `main` que **prova** que existe um vazamento de encapsulamento (chame `getSongs().clear()` e mostre que a playlist original fica vazia). Por fim, corrija `getSongs()` pra retornar uma cópia, e prove no `main` que o mesmo `clear()` agora não afeta mais o objeto original.  
_Critério de pronto:_ você tem as duas versões (com o bug e sem o bug) e uma prova no `main` de que a correção funciona.

**Exercício 3 (Médio/Difícil)**  
Crie uma classe `Order` com um campo privado `String status`, que só pode ser um destes valores: `"CREATED"`, `"SHIPPED"`, `"DELIVERED"`, `"CANCELLED"`. Comece sempre em `"CREATED"`. Em vez de um `setStatus(String)` genérico, crie métodos de negócio: `ship()` (só permitido se `status == "CREATED"`), `deliver()` (só permitido se `status == "SHIPPED"`), `cancel()` (só permitido se `status` for `"CREATED"` ou `"SHIPPED"`, nunca depois de `"DELIVERED"`). Cada transição inválida deve lançar `IllegalStateException` com uma mensagem clara.  
_Critério de pronto:_ tentar `order.deliver()` numa order recém-criada (ainda `"CREATED"`) lança exceção; o fluxo válido `ship()` → `deliver()` funciona; `cancel()` depois de `deliver()` lança exceção.

**Exercício 4 (Desafio)**  
Modele uma classe `Money` **imutável** que representa um valor monetário com `amount` (double) e `currency` (String, ex: `"BRL"`). Todos os campos são `private final`. Em vez de um método que "altera" o valor, qualquer operação (`add(Money other)`, `subtract(Money other)`) deve **retornar uma nova instância de `Money`**, sem nunca modificar `this`. `add`/`subtract` devem lançar exceção se as moedas forem diferentes (não faz sentido somar BRL com USD diretamente).  
_Critério de pronto:_ `Money a = new Money(100, "BRL"); Money b = a.add(new Money(50, "BRL"));` — depois disso, `a.getAmount()` ainda é `100` (não mudou), e `b.getAmount()` é `150`. Tentar `a.add(new Money(10, "USD"))` lança exceção.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Temperature {
    private final double celsius;

    public Temperature(double celsius) {
        if (celsius < -273.15) {
            throw new IllegalArgumentException("Temperature cannot be below absolute zero");
        }
        this.celsius = celsius;
    }

    public double getCelsius() {
        return celsius;
    }

    public double getFahrenheit() {
        return celsius * 9 / 5 + 32;
    }
}
```

Note que `getFahrenheit()` **calcula** em vez de guardar em outro campo. Isso é uma escolha de encapsulamento também: se você guardasse `fahrenheit` como campo separado, teria duas fontes de verdade que podem ficar dessincronizadas caso alguém altere uma sem atualizar a outra. Derivar sempre do dado único elimina essa classe inteira de bug.

**Exercício 2**

java

```java
public class Playlist {
    private List<String> songs = new ArrayList<>();

    public void addSong(String song) {
        songs.add(song);
    }

    // VERSÃO COM BUG (vazamento de encapsulamento):
    public List<String> getSongsLeaky() {
        return songs;
    }

    // VERSÃO CORRIGIDA:
    public List<String> getSongs() {
        return new ArrayList<>(songs);
    }
}

public class Main {
    public static void main(String[] args) {
        Playlist playlist = new Playlist();
        playlist.addSong("Song A");
        playlist.addSong("Song B");

        // Provando o vazamento:
        playlist.getSongsLeaky().clear();
        System.out.println(playlist.getSongsLeaky().size()); // 0 — a playlist original foi apagada!

        // Provando a correção:
        Playlist playlist2 = new Playlist();
        playlist2.addSong("Song C");
        playlist2.getSongs().clear(); // opera só na cópia
        System.out.println(playlist2.getSongs().size()); // 1 — original intacto
    }
}
```

Esse é um dos bugs mais sutis em Java porque o compilador não avisa nada — `getSongsLeaky()` está sintaticamente perfeito, `private` está lá, mas a proteção é ilusória. É um erro real e comum em código de produção, especialmente em APIs REST que retornam listas de entidades JPA diretamente.

**Exercício 3**

java

```java
public class Order {
    private String status;

    public Order() {
        this.status = "CREATED";
    }

    public void ship() {
        if (!status.equals("CREATED")) {
            throw new IllegalStateException("Cannot ship an order with status: " + status);
        }
        this.status = "SHIPPED";
    }

    public void deliver() {
        if (!status.equals("SHIPPED")) {
            throw new IllegalStateException("Cannot deliver an order with status: " + status);
        }
        this.status = "DELIVERED";
    }

    public void cancel() {
        if (status.equals("DELIVERED") || status.equals("CANCELLED")) {
            throw new IllegalStateException("Cannot cancel an order with status: " + status);
        }
        this.status = "CANCELLED";
    }

    public String getStatus() {
        return status;
    }
}
```

Isso é um exemplo direto de **máquina de estados encapsulada num objeto**. Comparado a expor `setStatus(String)` genérico (onde qualquer código externo poderia fazer `order.setStatus("DELIVERED")` pulando `SHIPPED`), esse design torna transições inválidas **impossíveis de expressar**, não apenas "proibidas por convenção". Vale mencionar: em Java moderno esse `status` frequentemente seria um `enum` em vez de `String` (evita erro de digitação tipo `"CANCELED"` vs `"CANCELLED"`) — isso é o próprio tópico **Enums**, que vem mais adiante no seu checklist, então por enquanto fica só o aviso.

**Exercício 4**

java

```java
public class Money {
    private final double amount;
    private final String currency;

    public Money(double amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public double getAmount() {
        return amount;
    }

    public String getCurrency() {
        return currency;
    }

    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(this.amount + other.amount, this.currency);
    }

    public Money subtract(Money other) {
        validateSameCurrency(other);
        return new Money(this.amount - other.amount, this.currency);
    }

    private void validateSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Cannot operate on different currencies: " + this.currency + " and " + other.currency);
        }
    }
}
```

Esse padrão se chama **Value Object imutável**, e é extremamente comum em sistemas financeiros de produção — justamente porque `Money` sendo imutável elimina uma categoria inteira de bugs em código concorrente (duas threads nunca podem "brigar" pra modificar o mesmo `Money` ao mesmo tempo, porque nenhuma delas consegue modificá-lo, só criar um novo). Note que `validateSameCurrency` é `private`: é um método auxiliar interno, não faz sentido nenhum ele ser público, porque só existe pra servir `add`/`subtract` — isso também é encapsulamento, aplicado a **comportamento**, não só a dado.