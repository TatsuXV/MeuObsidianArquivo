### 1. Teoria

**O que é "escopo" de uma variável?**

Escopo é a região do código onde uma variável **existe e pode ser acessada**. Fora dessa região, a variável simplesmente não é reconhecida pelo compilador — não é uma questão de "valor inválido", é como se ela não existisse ali.

Java define escopo principalmente pelos **blocos de código** (delimitados por `{ }`). A regra geral: uma variável só é visível dentro do bloco onde foi declarada, e em blocos aninhados _dentro_ dele — nunca em blocos irmãos ou no bloco pai.

#### Tipos de variável por escopo

**Variáveis locais**

java

```java
public void metodo() {
    int x = 10; // variável local, existe só dentro deste método
} // x deixa de existir aqui
```

- Declaradas dentro de um método, construtor, ou bloco (`{ }`).
- **Não recebem valor padrão** — precisam ser explicitamente inicializadas antes do primeiro uso (o compilador barra o uso de variável local não inicializada, como já vimos em Data Types).
- Vivem só durante a execução daquele bloco; depois disso, deixam de existir (a memória é liberada).

**Atributos de instância (instance fields)**

java

```java
public class Pessoa {
    private String nome; // atributo de instância — cada objeto Pessoa tem o seu próprio

    public void imprimirNome() {
        System.out.println(nome); // acessível em qualquer método da classe
    }
}
```

- Declarados diretamente no corpo da classe, fora de qualquer método.
- **Recebem valor padrão automaticamente** (tabela vista em Data Types: `0`, `false`, `null`, etc.) — diferente de variável local.
- Cada instância (objeto) da classe tem sua própria cópia independente.
- Acessíveis em qualquer método não-estático da classe, sem precisar passar como parâmetro.

**Atributos estáticos (static fields)**

java

```java
public class Contador {
    static int totalDeInstancias = 0; // compartilhado entre TODAS as instâncias
}
```

- Pertencem à classe, não a uma instância específica — existe **uma única cópia**, compartilhada por todos os objetos daquela classe.
- Será aprofundado no tópico "Static Keyword", mais adiante — aqui é só pra você já reconhecer a diferença de escopo/vida em relação a atributo de instância.

**Parâmetros de método**

java

```java
public void saudar(String nome) { // "nome" é um parâmetro — escopo limitado ao corpo do método
    System.out.println("Olá, " + nome);
}
```

- Se comportam como variável local: existem só durante a execução do método, inicializados com o valor passado na chamada.

#### Regras de sombreamento (shadowing) e visibilidade

java

```java
public class Exemplo {
    int valor = 100; // atributo de instância

    public void metodo() {
        int valor = 5; // variável local com MESMO NOME do atributo — "esconde" o atributo dentro deste método
        System.out.println(valor);       // imprime 5 (a local "ganha" da de instância, dentro deste escopo)
        System.out.println(this.valor);  // imprime 100 — "this." força o acesso ao atributo de instância
    }
}
```

Isso é chamado de **shadowing**: a variável mais "próxima" (de escopo mais interno) sempre tem prioridade sobre uma de mesmo nome em escopo mais externo. `this` é a palavra-chave que permite acessar explicitamente o atributo de instância quando há ambiguidade de nome — será aprofundado quando chegarmos em classes/objetos com mais profundidade, mas é essencial já entender essa regra de resolução de nome agora.

**Uma regra importante que trava muita gente:** dentro de um mesmo bloco (ou blocos aninhados diretamente relacionados), você **não pode** declarar duas variáveis locais com o mesmo nome — isso é erro de compilação, diferente do caso acima onde uma é atributo e outra é local (contextos diferentes, permitido).

java

```java
public void metodo() {
    int x = 1;
    if (true) {
        int x = 2; // ERRO DE COMPILAÇÃO: "variable x is already defined"
    }
}
```

**Escopo de bloco dentro de estruturas de controle**

java

```java
for (int i = 0; i < 5; i++) {
    int quadrado = i * i; // escopo: só dentro deste for
}
// i e quadrado não existem mais aqui
```

Cada iteração do `for` tecnicamente recria as variáveis declaradas dentro dele — isso é relevante mais adiante quando lidarmos com lambdas capturando variáveis (Programação Funcional), mas por ora o importante é: nada declarado dentro de `{ }` "vaza" pra fora desse bloco.

**Onde isso aparece na prática (backend real)**

Em uma classe `@Service` do Spring, você vai ter atributos de instância (geralmente dependências injetadas, como um `Repository`) que ficam disponíveis para todos os métodos da classe, e variáveis locais dentro de cada método que existem só durante aquela chamada específica. Entender essa diferença é o que evita, por exemplo, guardar estado de uma requisição específica num atributo de instância de um `@Service` — isso é um bug real e sério em aplicação com múltiplas threads (o `@Service` é compartilhado entre requisições por padrão), algo que será aprofundado em Threads/Java Memory Model, mas a raiz do problema já está aqui: confundir o que devia ser variável local com o que virou atributo de instância.

---

### 2. Exemplo de código comentado

java

```java
public class VariaveisEscopos {

    // Atributo de instância — cada objeto desta classe tem o seu próprio
    // Recebe valor padrão automaticamente (0), mesmo sem inicializar aqui
    int contadorDeChamadas;

    // Atributo estático — compartilhado por TODAS as instâncias da classe
    static int totalDeObjetosCriados = 0;

    String nome; // outro atributo de instância, valor padrão é null (tipo de referência)

    public VariaveisEscopos(String nome) {
        this.nome = nome; // "this.nome" é o atributo; "nome" (parâmetro) é a variável local do construtor
        totalDeObjetosCriados++; // incrementa a cópia compartilhada, não uma cópia por objeto
    }

    public void metodoComEscopoLocal() {
        contadorDeChamadas++; // acessa o atributo de instância diretamente, sem precisar de "this." (não há ambiguidade aqui)

        int valorTemporario = 42; // variável local — existe só durante esta chamada de método
        System.out.println("Valor temporário: " + valorTemporario);

        if (valorTemporario > 40) {
            int mensagemCodigo = 1; // escopo: só dentro deste bloco if
            System.out.println("Código da mensagem: " + mensagemCodigo);
        }
        // mensagemCodigo não existe mais aqui fora do if

        for (int i = 0; i < 3; i++) {
            int quadrado = i * i; // recriado a cada iteração, escopo local ao for
            System.out.println(i + "² = " + quadrado);
        }
        // i e quadrado não existem mais aqui fora do for
    }

    public void demonstrarShadowing() {
        int contadorDeChamadas = 999; // variável LOCAL com mesmo nome do atributo — sombreamento
        System.out.println("Local: " + contadorDeChamadas);       // imprime 999
        System.out.println("Atributo: " + this.contadorDeChamadas); // imprime o valor real do atributo, via "this."
    }

    public static void main(String[] args) {
        VariaveisEscopos obj1 = new VariaveisEscopos("Primeiro");
        VariaveisEscopos obj2 = new VariaveisEscopos("Segundo");

        obj1.metodoComEscopoLocal();
        obj1.metodoComEscopoLocal(); // chamando de novo, no MESMO objeto

        System.out.println("Contador de chamadas do obj1: " + obj1.contadorDeChamadas); // 2 — específico deste objeto
        System.out.println("Contador de chamadas do obj2: " + obj2.contadorDeChamadas); // 0 — obj2 nunca chamou o método

        // Estático é compartilhado — reflete os DOIS objetos criados, não é "por objeto"
        System.out.println("Total de objetos criados: " + totalDeObjetosCriados); // 2

        obj1.demonstrarShadowing();
    }
}
```

---

### 3. Armadilhas comuns

1. **Achar que variável local recebe valor padrão, como atributo.** `int x; System.out.println(x);` dentro de um método **não compila** — "variable x might not have been initialized". Isso já foi visto em Data Types, mas é literalmente uma regra de escopo: só atributo tem valor padrão automático.
2. **Confundir atributo de instância com atributo estático numa classe com múltiplos objetos.** Esperar que `objeto1.contador` e `objeto2.contador` sejam independentes quando `contador` é `static` — nesse caso os dois "compartilham" a mesma variável, e incrementar em um objeto afeta o valor visto pelo outro.
3. **Tentar usar variável declarada dentro de um bloco (`if`, `for`, `while`) fora dele.** É erro de compilação direto: "cannot find symbol" — a variável simplesmente deixou de existir assim que o bloco fechou.
4. **Esquecer o `this.` quando há shadowing intencional ou acidental.** Em construtores, é comum o parâmetro ter o mesmo nome do atributo (`this.nome = nome;`) — esquecer o `this.` faz o atributo nunca ser de fato atribuído (o parâmetro só atribui a ele mesmo, e o atributo real fica com o valor padrão, geralmente `null`), um bug silencioso porque o compilador não acusa erro nenhum.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Crie uma classe `ContaBancaria` com um atributo de instância `double saldo`. Escreva um método `depositar(double valor)` que soma `valor` ao `saldo` (use `this.saldo` explicitamente, mesmo sem ter parâmetro de nome conflitante, só pra praticar a sintaxe). No `main`, crie dois objetos `ContaBancaria` diferentes, deposite valores diferentes em cada um, e imprima o saldo dos dois separadamente, provando que são independentes. Critério de pronto: os dois saldos impressos são diferentes e correspondem exatamente ao que foi depositado em cada objeto.

**Exercício 2 (médio)**  
Reescreva a classe do Exercício 1, mas agora adicione um atributo **estático** `static int totalDeContasAbertas`, incrementado no construtor toda vez que uma nova conta é criada. Crie 3 objetos `ContaBancaria` no `main` e imprima `ContaBancaria.totalDeContasAbertas` (acessando pela classe, não por um objeto específico — isso é convenção de como se acessa membro estático) depois de criar os 3. Critério de pronto: o valor impresso é `3`, mesmo sendo objetos diferentes.

**Exercício 3 (difícil)**  
Escreva uma classe `Contador` com um atributo de instância `int valor = 0;` e um método `incrementar()` que faz `int valor = this.valor + 1;` seguido de `System.out.println(valor);` (propositalmente criando uma variável local com o mesmo nome do atributo, sem nunca atribuir de volta ao atributo). Chame `incrementar()` três vezes seguidas no `main`, no mesmo objeto. Antes de rodar, escreva por escrito o que você espera ver impresso nas 3 chamadas, e explique por que o atributo `valor` do objeto nunca muda de fato (dica: pense em qual variável está sendo lida e qual está sendo escrita a cada linha). Depois rode e confirme.

**Exercício 4 (desafio)**  
O código abaixo tem um bug clássico de shadowing em construtor — o atributo `idade` nunca é de fato atribuído com o valor recebido. Sem rodar ainda, explique por escrito por que `pessoa.idade` vai imprimir `0` mesmo passando `25` no construtor, depois corrija o bug (existem duas formas válidas de corrigir: uma usando `this.`, outra renomeando o parâmetro — implemente as duas versões separadamente):

java

```java
public class Pessoa {
    int idade;

    public Pessoa(int idade) {
        idade = idade; // bug proposital aqui
    }

    public static void main(String[] args) {
        Pessoa pessoa = new Pessoa(25);
        System.out.println(pessoa.idade);
    }
}
```

Critério de pronto: você explica corretamente por que o bug acontece (em termos de qual variável está sendo lida/escrita) antes de mostrar as duas versões corrigidas, e ambas imprimem `25`.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        ContaBancaria conta1 = new ContaBancaria();
        ContaBancaria conta2 = new ContaBancaria();

        conta1.depositar(500.0);
        conta2.depositar(1200.0);

        System.out.println("Saldo conta1: " + conta1.saldo);
        System.out.println("Saldo conta2: " + conta2.saldo);
    }
}

class ContaBancaria {
    double saldo; // valor padrão 0.0, atributo de instância

    void depositar(double valor) {
        this.saldo = this.saldo + valor;
    }
}
```

Raciocínio: cada objeto (`conta1`, `conta2`) tem sua própria cópia independente de `saldo`, porque é atributo de instância, não estático — por isso depositar em um não afeta o outro. O `this.saldo` do lado esquerdo poderia ser só `saldo` aqui (não há ambiguidade de nome nesse método, já que o parâmetro se chama `valor`), mas usei explicitamente pra deixar claro qual variável está sendo modificada, como pedido no enunciado.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        new ContaBancaria();
        new ContaBancaria();
        new ContaBancaria();

        System.out.println("Total de contas abertas: " + ContaBancaria.totalDeContasAbertas);
    }
}

class ContaBancaria {
    double saldo;
    static int totalDeContasAbertas = 0; // uma única cópia, compartilhada por todas as instâncias

    ContaBancaria() {
        totalDeContasAbertas++;
    }
}
```

Raciocínio: `totalDeContasAbertas` é incrementado toda vez que o construtor roda, e como é `static`, o incremento afeta a **mesma** variável independente de qual objeto está sendo criado — por isso, depois de 3 construções, o valor acumulado é `3`. Acessei via `ContaBancaria.totalDeContasAbertas` (nome da classe) em vez de por um objeto específico, porque isso deixa explícito no código que o dado pertence à classe como um todo, não a um objeto — é a convenção correta, mesmo o compilador tecnicamente aceitando acessar `static` por uma referência de objeto também.

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        Contador c = new Contador();
        c.incrementar();
        c.incrementar();
        c.incrementar();
    }
}

class Contador {
    int valor = 0;

    void incrementar() {
        int valor = this.valor + 1; // variável LOCAL nova, calculada a partir do atributo atual
        System.out.println(valor);
        // this.valor nunca é reatribuído — o atributo do objeto continua 0 para sempre
    }
}
```

Saída esperada: `1`, `1`, `1` (não `1, 2, 3`).

Raciocínio: a cada chamada de `incrementar()`, a linha `int valor = this.valor + 1;` **lê** o atributo (`this.valor`, que é sempre `0`, porque nunca é alterado), soma 1, e guarda esse resultado numa variável **local** nova chamada `valor` — que só existe durante aquela chamada e desaparece depois. Como nada é escrito de volta em `this.valor`, o atributo do objeto fica travado em `0` para sempre, e cada chamada recalcula `0 + 1 = 1` do zero, sempre imprimindo `1`. Esse é exatamente o tipo de bug que shadowing descuidado provoca — o código parece incrementar, mas na verdade só lê e descarta.

**Exercício 4**

Explicação do bug: dentro do construtor `Pessoa(int idade)`, a linha `idade = idade;` está lendo e escrevendo na **mesma variável**: o **parâmetro local** `idade` (que sombreia o atributo `idade` da classe, porque têm o mesmo nome). Não existe, nessa linha, nenhuma referência ao atributo de instância — `this.idade` nunca é tocado. Resultado: o parâmetro recebe `25`, atribui `25` a si mesmo (operação sem efeito real), e o atributo `idade` do objeto permanece com o valor padrão `0`, porque `int` é primitivo e tem valor padrão `0` quando não é explicitamente inicializado.

**Versão corrigida 1 — usando `this.`:**

java

```java
public class Pessoa {
    int idade;

    public Pessoa(int idade) {
        this.idade = idade; // esquerda: atributo (via this.) | direita: parâmetro local
    }

    public static void main(String[] args) {
        Pessoa pessoa = new Pessoa(25);
        System.out.println(pessoa.idade); // 25
    }
}
```

**Versão corrigida 2 — renomeando o parâmetro:**

java

```java
public class Pessoa {
    int idade;

    public Pessoa(int idadeRecebida) {
        idade = idadeRecebida; // sem ambiguidade de nome, não precisa de this.
    }

    public static void main(String[] args) {
        Pessoa pessoa = new Pessoa(25);
        System.out.println(pessoa.idade); // 25
    }
}
```

Raciocínio: as duas resolvem o mesmo problema por caminhos diferentes — a primeira mantém os nomes iguais (convenção mais comum em construtores profissionais, porque deixa claro que o parâmetro corresponde diretamente ao atributo) e resolve a ambiguidade explicitamente com `this.`; a segunda elimina a ambiguidade de raiz, dando nomes diferentes, então não há shadowing algum pra se preocupar. Na prática de mercado, a primeira forma (`this.` com nomes iguais) é mais comum em código Java profissional.