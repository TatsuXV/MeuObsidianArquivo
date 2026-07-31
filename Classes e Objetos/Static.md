#### 1. Teoria

`static` marca um membro (atributo, método, bloco ou classe aninhada) como pertencente **à classe em si**, não a uma instância específica. Isso muda duas coisas na prática:

- **Atributo `static`**: existe **uma única cópia** compartilhada entre todos os objetos daquela classe — se um objeto altera o valor, todos "veem" a mudança, porque não é "do objeto", é da classe.
- **Método `static`**: pode ser chamado **sem precisar criar um objeto** (`NomeDaClasse.metodo()`), e por isso **não tem acesso a `this`** nem a atributos/métodos de instância diretamente — só a outros membros `static`.

Não confunda com:

- **Access specifiers** (`private`/`public`/etc.): já vimos que é um eixo diferente — controla _quem enxerga_, `static` controla _a quem pertence_ (classe vs. instância). Um membro pode ser `private static`, `public static`, e por aí vai.
- **Final**: `final` impede reatribuição de valor. `static final` juntos formam o padrão de **constante** (ex: `public static final double PI = 3.14159`), mas são conceitos independentes — `static` sozinho não impede que o valor mude.
- **Singleton (padrão de projeto)**: às vezes gente confunde "método static" com "objeto único" — não é a mesma coisa. `static` é recurso da linguagem; Singleton é um padrão de design que _usa_ `static` como uma das peças, mas envolve mais decisões (construtor privado, controle de instanciação) que não vamos ver agora.

**Onde isso aparece no dia a dia de backend**: classes utilitárias inteiras costumam ser só métodos `static` — é o caso de `Math.sqrt()`, `Collections.sort()`, e é um padrão comum você mesmo escrever uma classe `Utils` ou `Constantes` assim. Em Spring, é raro usar `static` em beans/serviços (porque o framework gerencia instâncias via injeção de dependência — Bloco 8), mas para métodos auxiliares puros (sem estado, sem depender de nada injetado) `static` ainda é comum.

#### 2. Exemplo de código comentado

java

```java
public class ContadorDeUsuarios {

    private static int totalDeUsuarios = 0; // uma única cópia, compartilhada por TODOS os objetos
    private String nome;                     // atributo de instância, cada objeto tem o seu

    public ContadorDeUsuarios(String nome) {
        this.nome = nome;
        totalDeUsuarios++; // incrementa o contador compartilhado toda vez que um objeto é criado
    }

    public static int getTotalDeUsuarios() { // método static: chamado sem precisar de instância
        return totalDeUsuarios;
        // repare: aqui NÃO poderíamos acessar "this.nome" — método static não tem "this"
    }
}
```

java

```java
ContadorDeUsuarios u1 = new ContadorDeUsuarios("Ana");
ContadorDeUsuarios u2 = new ContadorDeUsuarios("Bruno");

System.out.println(ContadorDeUsuarios.getTotalDeUsuarios()); // 2
// chamado direto na CLASSE, sem precisar de u1 ou u2
```

#### 3. Armadilhas comuns

1. **Tentar acessar atributo/método de instância dentro de um método `static`** — não compila, porque método `static` não sabe "de qual objeto" ele deveria pegar esse valor (não existe `this` ali).
2. **Abusar de `static` para "economizar" criação de objetos** — usar `static` em tudo pode parecer prático, mas quebra a ideia de estado independente por objeto, e em código com múltiplas threads (Bloco 10) atributos `static` mutáveis são fonte clássica de bug de concorrência.
3. **Esquecer que atributo `static` é compartilhado entre TODAS as instâncias, inclusive em testes** — um erro comum em testes unitários (Trilha "Testes", Bloco 15) é um atributo `static` "vazar" estado de um teste para o outro, porque o valor não reseta sozinho entre execuções.
4. **Confundir `static` com "constante"** — `static` sozinho só define que o membro pertence à classe; se você quiser que o valor também seja imutável, precisa combinar com `final` (`static final`).

#### 4. Exercícios práticos

**1. Fácil** — Crie uma classe `Circulo` com um atributo `static final double PI = 3.14159` e um atributo de instância `raio`. Implemente um método de instância `calcularArea()` que usa o `PI` estático. Critério de pronto: criar dois círculos com raios diferentes e mostrar que ambos usam o mesmo valor de `PI`, mas calculam áreas diferentes por causa do `raio` (que é de instância).

**2. Médio** — Crie uma classe `GeradorDeId` com um atributo `private static int proximoId = 1` e um método `public static int gerarProximoId()` que retorna o valor atual de `proximoId` e depois o incrementa. Critério de pronto: chamar `gerarProximoId()` três vezes seguidas (sem criar nenhum objeto de `GeradorDeId`) e mostrar que os valores retornados são sequenciais (1, 2, 3).

**3. Difícil** — Crie uma classe `Produto` com atributos de instância `nome` e `preco`, e um atributo `static double totalVendasEmValor = 0`. Implemente um método de instância `venderProduto()` que soma o `preco` deste produto ao total estático. Adicione também um método `static` `getTotalVendas()`. Critério de pronto: criar três produtos diferentes, "vender" todos, e mostrar que `getTotalVendas()` reflete a soma correta — e explicar em um comentário por que esse desenho (estado compartilhado mutável) seria arriscado em um sistema com múltiplos usuários simultâneos (não precisa implementar solução, só identificar o risco).

**4. Desafio** — Implemente uma classe `Configuracao` que representa configurações globais de um sistema fictício (ex: `nomeAplicacao`, `versaoApi`). Ela deve ter: um construtor `private` (para impedir que qualquer código externo crie uma instância diretamente), um atributo `private static Configuracao instanciaUnica`, e um método `public static Configuracao getInstancia()` que cria a instância na primeira chamada e retorna sempre a mesma instância nas chamadas seguintes. Critério de pronto: chamar `Configuracao.getInstancia()` duas vezes em pontos diferentes do código e provar (ex: comparando com `==`) que é o **mesmo objeto** nas duas vezes.

#### 5. Gabarito comentado

**1. Circulo**

java

```java
public class Circulo {
    private static final double PI = 3.14159;
    private double raio;

    public Circulo(double raio) {
        this.raio = raio;
    }

    public double calcularArea() {
        return PI * raio * raio; // PI é compartilhado, raio é individual
    }
}

// Uso:
Circulo c1 = new Circulo(2);
Circulo c2 = new Circulo(5);
System.out.println(c1.calcularArea()); // ~12.566
System.out.println(c2.calcularArea()); // ~78.54
```

Raciocínio: `PI` não faz sentido variar por objeto — é uma constante matemática, então faz total sentido ser `static final`. `raio`, por outro lado, é característica própria de cada círculo, por isso é atributo de instância.

**2. GeradorDeId**

java

```java
public class GeradorDeId {
    private static int proximoId = 1;

    public static int gerarProximoId() {
        int idAtual = proximoId;
        proximoId++;
        return idAtual;
    }
}

// Uso, sem instanciar nada:
System.out.println(GeradorDeId.gerarProximoId()); // 1
System.out.println(GeradorDeId.gerarProximoId()); // 2
System.out.println(GeradorDeId.gerarProximoId()); // 3
```

Raciocínio: isso só funciona porque `proximoId` é `static` — se fosse atributo de instância, cada chamada precisaria de um objeto `GeradorDeId` diferente, e cada um começaria do zero, o que quebraria a ideia de sequência global de IDs.

**3. Produto**

java

```java
public class Produto {
    private String nome;
    private double preco;
    private static double totalVendasEmValor = 0;

    public Produto(String nome, double preco) {
        this.nome = nome;
        this.preco = preco;
    }

    public void venderProduto() {
        totalVendasEmValor += this.preco;
    }

    public static double getTotalVendas() {
        return totalVendasEmValor;
    }
}

// Uso:
Produto p1 = new Produto("Teclado", 150.0);
Produto p2 = new Produto("Mouse", 80.0);
Produto p3 = new Produto("Monitor", 900.0);
p1.venderProduto();
p2.venderProduto();
p3.venderProduto();
System.out.println(Produto.getTotalVendas()); // 1130.0

// Risco: se dois usuários "vendessem" produtos ao mesmo tempo (múltiplas threads),
// o incremento "totalVendasEmValor += preco" não é uma operação atômica —
// pode haver race condition e o total final ficar incorreto. Isso será
// aprofundado no Bloco 10 (Concorrência / Java Memory Model).
```

Raciocínio: o exercício força você a perceber que estado `static` mutável é conveniente, mas é exatamente o tipo de desenho que quebra em ambiente com concorrência real — vale só reconhecer o risco agora, a solução técnica (sincronização) é assunto de bloco futuro.

**4. Configuracao (Singleton simples)**

java

```java
public class Configuracao {
    private static Configuracao instanciaUnica;

    private String nomeAplicacao;
    private String versaoApi;

    private Configuracao() { // construtor private: ninguém de fora cria isso diretamente
        this.nomeAplicacao = "MeuApp";
        this.versaoApi = "1.0";
    }

    public static Configuracao getInstancia() {
        if (instanciaUnica == null) {
            instanciaUnica = new Configuracao(); // só cria na primeira chamada
        }
        return instanciaUnica;
    }
}

// Uso:
Configuracao c1 = Configuracao.getInstancia();
Configuracao c2 = Configuracao.getInstancia();
System.out.println(c1 == c2); // true — é o mesmo objeto
```

Raciocínio: o construtor `private` é o que impede `new Configuracao()` de fora da classe — a única forma de obter uma instância é via `getInstancia()`, que controla o ciclo de vida. Isso é uma primeira pincelada do padrão Singleton, que você vai reconhecer bastante em frameworks (o Spring, por exemplo, trata a maioria dos seus beans como singleton por padrão — mas isso é aprofundado no Bloco 8, Dependency Injection, não é o mesmo mecanismo que fizemos aqui na mão).

**Nota sobre esse exercício 4**: essa implementação de Singleton, do jeito que está, **não é thread-safe** (dois threads chamando `getInstancia()` ao mesmo tempo, na primeira vez, poderiam teoricamente criar duas instâncias). Isso é aprofundado no Bloco 10; por enquanto o objetivo era só entender o mecanismo de `static` guardando estado único da classe.