#### 1. Teoria

Este é um dos tópicos que mais gera debate (e confusão) entre desenvolvedores Java — inclusive gente com anos de experiência discute isso incorretamente às vezes. Vamos ser categóricos: **Java é estritamente _pass by value_, sempre, sem exceção.** Não existe _pass by reference_ em Java. O que confunde as pessoas é entender **o que exatamente** é o "valor" que é passado quando lidamos com objetos.

**O que "pass by value" significa, precisamente:**  
Quando você chama um método passando um argumento, Java **copia o valor** da variável e entrega essa cópia para o parâmetro do método. O método recebe uma cópia, nunca a variável original em si. Qualquer reatribuição feita **dentro** do método, ao parâmetro, não afeta a variável original de quem chamou.

**A parte que confunde: tipos primitivos vs. referências**

- **Tipos primitivos** (`int`, `double`, `boolean`, `char`, etc.): a variável **contém o valor diretamente**. Copiar a variável copia o valor em si. Isso é intuitivo pra praticamente todo mundo.
- **Objetos** (qualquer coisa que não seja primitivo — `String`, `ArrayList`, sua própria classe, etc.): a variável **não contém o objeto**. Ela contém uma **referência** (um endereço de memória, conceitualmente, apontando para onde o objeto vive na heap). Quando você "passa um objeto" para um método, você está, na verdade, copiando o **valor da referência** — ou seja, copiando o "endereço". As duas variáveis (a original e o parâmetro do método) passam a apontar para **o mesmo objeto** na memória, mas são **duas cópias independentes do endereço**.

Isso explica os dois comportamentos que parecem contraditórios até você entender o mecanismo:

1. **Mutação do objeto através do parâmetro AFETA o objeto original** — porque ambas as referências (cópia e original) apontam pro mesmo objeto na heap. Se o método chama `objeto.setAlgumCampo(...)`, ele está mexendo no objeto real, visível por qualquer referência que aponte pra ele.
2. **Reatribuir o parâmetro para um NOVO objeto NÃO afeta a variável original** — porque isso só muda para onde a **cópia local da referência** aponta, sem tocar na variável original, que continua apontando pro objeto de sempre.

**Resumindo com uma analogia útil:** pense numa referência como um **papel com um endereço de casa escrito**. Se eu te dou uma **cópia** desse papel (isso é o "pass by value" da referência):

- Se você for até a casa e pintar a parede (mutar o objeto), a casa real mudou — eu vou ver a parede pintada também, porque é a mesma casa.
- Se você **rasurar o seu papel** e escrever o endereço de outra casa (reatribuir o parâmetro), isso não muda nada no **meu** papel — o meu continua apontando pra casa original.

**Diferença de linguagens que TÊM pass by reference de verdade (ex: C++ com `&`):**  
Nessas linguagens, seria possível o método reatribuir a variável do chamador para apontar para outro objeto completamente diferente, e essa mudança **se refletir** de volta pra fora do método. Isso é **impossível** em Java — é exatamente o experimento que separa as duas coisas (ver Exemplo 2 abaixo).

**Onde aparece no dia a dia de backend Java/Spring:**

- Entender isso evita um bug clássico: passar uma entidade JPA pra um método esperando que reatribuí-la lá dentro "resete" o objeto pra quem chamou — não funciona, e quem não entende o mecanismo perde tempo debugando algo que é comportamento esperado da linguagem.
- Métodos que **mutam** objetos recebidos como parâmetro (ex: preencher uma `List` passada como argumento) são um padrão legítimo e comum, mas devem ser usados com WCA — sem documentação clara, é uma fonte de _side effect_ inesperado que dificulta entender o fluxo do código só lendo a assinatura do método.
- Isso é a base pra entender por que, mais adiante, você vai ver a diferença entre criar um **novo objeto imutável** (functional style, comum em Streams) versus **mutar um objeto existente** — dois estilos de programação com implicações bem diferentes de rastreabilidade de bug.

---

#### 2. Exemplo de código comentado

java

```java
// Exemplo 1: tipo primitivo — a cópia é do VALOR em si
public class ExemploPrimitivo {

    public static void tentarDobrar(int numero) {
        numero = numero * 2; // muda só a cópia LOCAL do parâmetro
        System.out.println("Dentro do método: " + numero);
    }

    public static void main(String[] args) {
        int valor = 5;
        tentarDobrar(valor);
        System.out.println("Fora do método: " + valor);
    }
}
// Saída:
// Dentro do método: 10
// Fora do método: 5   <- não mudou! 'valor' e o parâmetro 'numero' são cópias independentes
```

java

```java
// Exemplo 2: objeto — MUTAÇÃO afeta o original, REATRIBUIÇÃO não
public class Contador {
    int total = 0;
}

public class ExemploObjeto {

    // Este método MUTA o objeto recebido (chama um "setter" via campo público
    // aqui só por brevidade didática — em código real seria um método)
    public static void incrementar(Contador c) {
        c.total = c.total + 1; // isso AFETA o objeto original
    }

    // Este método tenta REATRIBUIR o parâmetro para um objeto novo
    public static void tentarSubstituir(Contador c) {
        c = new Contador();     // 'c' agora aponta pra um objeto DIFERENTE,
        c.total = 999;          // mas isso só afeta a cópia LOCAL da referência
    }

    public static void main(String[] args) {
        Contador meuContador = new Contador();

        incrementar(meuContador);
        System.out.println("Após incrementar: " + meuContador.total);
        // Após incrementar: 1  <- MUDOU, porque é o MESMO objeto sendo mutado

        tentarSubstituir(meuContador);
        System.out.println("Após tentar substituir: " + meuContador.total);
        // Após tentar substituir: 1  <- NÃO MUDOU! meuContador continua
        // apontando pro objeto original, com total=1. A reatribuição dentro
        // de tentarSubstituir só afetou a cópia LOCAL da referência 'c'.
    }
}
```

java

```java
// Exemplo 3: caso clássico que confunde — String "parece" mudar, mas não muda
public class ExemploString {

    public static void tentarModificar(String texto) {
        texto = texto + " modificado"; // cria uma NOVA String e reatribui 'texto' local
    }

    public static void main(String[] args) {
        String original = "original";
        tentarModificar(original);
        System.out.println(original); // "original" <- sem alteração
    }
}
// Isso confunde iniciantes porque parece que "String não é mutável mesmo por
// referência", mas na verdade é o MESMO mecanismo do Exemplo 2 (reatribuição
// de parâmetro local não afeta fora) combinado com o fato de String ser
// IMUTÁVEL por design (concatenação sempre cria uma nova String, nunca
// altera a existente) — dois conceitos empilhados no mesmo exemplo.
```

---

#### 3. Armadilhas comuns

1. **Chamar isso de "pass by reference" porque objetos "parecem" ser passados por referência.** Essa é a confusão mais comum e mais debatida. O termo tecnicamente correto para o que acontece com objetos em Java é "**pass by value of the reference**" (passagem por valor da referência) — a referência em si é copiada por valor. Chamar de "pass by reference" está tecnicamente errado e é a origem de praticamente toda confusão sobre o assunto.
2. **Escrever um método esperando "resetar" ou "trocar" um objeto do chamador através de reatribuição no parâmetro.** Ex: escrever um método `limpar(List<String> lista) { lista = new ArrayList<>(); }` esperando que a lista do chamador fique vazia — isso não funciona. Se você quer de fato limpar o conteúdo, precisa **mutar** o objeto existente (`lista.clear();`), não reatribuir o parâmetro.
3. **Assumir que passar um objeto "protege" contra mutação, como se fosse uma cópia defensiva automática.** Java não faz cópia do objeto ao passá-lo como argumento — só copia a referência. Se você precisa garantir que um método não altere o objeto original, isso é responsabilidade **sua**, geralmente fazendo uma cópia explícita antes de passar, ou usando/desenhando um objeto imutável.
4. **Achar que arrays se comportam diferente de outros objetos nesse aspecto.** Arrays em Java são objetos (mesmo arrays de primitivos, como `int[]`), então seguem exatamente a mesma regra: mutar elementos dentro do array afeta o array original; reatribuir a variável do array dentro do método (`array = new int[10];`) não afeta a variável de fora.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Escreva um método `dobrarValor(int n)` que multiplica o parâmetro por 2 e o imprime dentro do método. No `main`, declare uma variável `int x = 7;`, chame o método, e depois imprima `x` de novo. Comente no código, em uma frase, por que o valor de `x` fora do método não muda.  
_Critério de pronto:_ a saída mostra 14 dentro do método e 7 fora, com o comentário explicando corretamente o motivo (cópia de valor primitivo).

**Exercício 2 (Médio)**  
Crie uma classe `Caixa` com um campo `int quantidade`. Escreva dois métodos estáticos: `adicionarItem(Caixa caixa)` que faz `caixa.quantidade++`, e `trocarCaixa(Caixa caixa)` que faz `caixa = new Caixa();` seguido de `caixa.quantidade = 100;`. No `main`, crie uma `Caixa`, chame os dois métodos em sequência, e imprima o resultado final, comentando por que só um dos dois métodos realmente afeta o objeto original.  
_Critério de pronto:_ a saída demonstra que `adicionarItem` alterou a caixa original, mas `trocarCaixa` não teve nenhum efeito visível de fora.

**Exercício 3 (Difícil)**  
Escreva um método `dobrarTodosOsElementos(int[] array)` que percorre um array de `int` e dobra cada elemento **no lugar** (mutando o array recebido). Depois, escreva um segundo método `tentarSubstituirArray(int[] array)` que tenta reatribuir o parâmetro para um array totalmente novo (`array = new int[]{999, 999, 999};`). No `main`, crie um array, chame ambos os métodos em sequência, e imprima o array final, explicando em comentário a diferença de resultado entre os dois métodos — ligando explicitamente ao fato de array ser objeto em Java.  
_Critério de pronto:_ o array final reflete a mutação do primeiro método, mas não é afetado pelo segundo.

**Exercício 4 (Desafio)**  
Crie uma classe `Pessoa` com um campo mutável `List<String> apelidos` (inicializado vazio no construtor). Escreva um método `adicionarApelido(Pessoa pessoa, String apelido)` que adiciona um apelido à lista da pessoa recebida. Depois, escreva um método `criarCopiaDefensiva(Pessoa original)` que **não modifica** `original`, mas retorna uma **nova** `Pessoa` com uma cópia independente da lista de apelidos (de forma que adicionar um apelido na cópia não afete a lista da pessoa original, e vice-versa). Demonstre no `main` que modificar a cópia não afeta o original, provando que sua implementação de cópia defensiva está correta (não é só copiar a referência da lista por engano).  
_Critério de pronto:_ após modificar a lista de apelidos da cópia, a lista de apelidos da pessoa original permanece com o conteúdo anterior, inalterado.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Main {
    public static void dobrarValor(int n) {
        n = n * 2;
        System.out.println("Dentro do método: " + n);
    }

    public static void main(String[] args) {
        int x = 7;
        dobrarValor(x);
        System.out.println("Fora do método: " + x);
        // x continua 7 porque 'n' recebeu uma CÓPIA do valor de x (7),
        // e multiplicar 'n' por 2 só altera essa cópia local, sem
        // nenhuma ligação de volta com a variável 'x' do main.
    }
}
```

_Raciocínio:_ este é o caso mais simples e serve de base pra entender por que o caso com objetos, embora pareça diferente à primeira vista, segue exatamente o mesmo princípio — só que o "valor" copiado, no caso de objetos, é uma referência em vez de um número.

**Exercício 2**

java

```java
public class Caixa {
    int quantidade = 0;
}

public class Main {
    public static void adicionarItem(Caixa caixa) {
        caixa.quantidade++;
        // MUTA o objeto que 'caixa' aponta — como é o MESMO objeto
        // que a variável do chamador aponta, o efeito é visível fora.
    }

    public static void trocarCaixa(Caixa caixa) {
        caixa = new Caixa();
        caixa.quantidade = 100;
        // Isso só reatribui a cópia LOCAL da referência 'caixa' para um
        // objeto novo. A variável do chamador nunca soube dessa troca —
        // ela continua apontando pro objeto original, intocado por aqui.
    }

    public static void main(String[] args) {
        Caixa minhaCaixa = new Caixa();

        adicionarItem(minhaCaixa);
        System.out.println("Após adicionarItem: " + minhaCaixa.quantidade); // 1

        trocarCaixa(minhaCaixa);
        System.out.println("Após trocarCaixa: " + minhaCaixa.quantidade); // ainda 1
    }
}
```

_Raciocínio:_ esse exercício isola, lado a lado, os dois comportamentos descritos na Teoria — mutação (afeta) vs. reatribuição (não afeta) — usando o mesmo objeto e o mesmo tipo de parâmetro, o que deixa claro que a diferença está inteiramente em **o que o método faz com a referência recebida**, não em alguma diferença de "tipo de passagem".

**Exercício 3**

java

```java
public class Main {
    public static void dobrarTodosOsElementos(int[] array) {
        for (int i = 0; i < array.length; i++) {
            array[i] = array[i] * 2;
        }
        // Muta o array recebido, posição por posição — o array em si
        // (o objeto na heap) é o MESMO que o chamador possui.
    }

    public static void tentarSubstituirArray(int[] array) {
        array = new int[]{999, 999, 999};
        // Reatribui a variável LOCAL 'array' para apontar pra um array
        // novo e diferente. Isso não tem nenhum efeito sobre a variável
        // de array do chamador, pelo mesmo motivo do Exercício 2.
    }

    public static void main(String[] args) {
        int[] numeros = {1, 2, 3};

        dobrarTodosOsElementos(numeros);
        System.out.println(java.util.Arrays.toString(numeros)); // [2, 4, 6]

        tentarSubstituirArray(numeros);
        System.out.println(java.util.Arrays.toString(numeros)); // ainda [2, 4, 6]
    }
}
```

_Raciocínio:_ array reforça o ponto da Armadilha 4 — mesmo sendo uma estrutura "de baixo nível" que muita gente trata mentalmente como caso especial, ele é um objeto normal em Java e segue exatamente as mesmas regras de qualquer outro objeto nesse aspecto.

**Exercício 4**

java

```java
import java.util.ArrayList;
import java.util.List;

public class Pessoa {
    private final String nome;
    private final List<String> apelidos;

    public Pessoa(String nome) {
        this.nome = nome;
        this.apelidos = new ArrayList<>();
    }

    // Construtor auxiliar usado internamente pela cópia defensiva
    private Pessoa(String nome, List<String> apelidosCopiados) {
        this.nome = nome;
        this.apelidos = apelidosCopiados;
    }

    public List<String> getApelidos() {
        return apelidos;
    }

    public String getNome() {
        return nome;
    }
}

public class Main {

    public static void adicionarApelido(Pessoa pessoa, String apelido) {
        pessoa.getApelidos().add(apelido);
        // Muta a lista existente dentro do objeto Pessoa recebido —
        // afeta o objeto real, porque é a mesma lista referenciada.
    }

    public static Pessoa criarCopiaDefensiva(Pessoa original) {
        // O ponto CRÍTICO deste exercício: 'new ArrayList<>(original.getApelidos())'
        // cria uma lista NOVA, com os MESMOS elementos copiados pra dentro dela —
        // não é a mesma referência de lista. Se eu simplesmente fizesse
        // 'original.getApelidos()' sem envolver em 'new ArrayList<>(...)',
        // a cópia e o original compartilhariam a MESMA lista por baixo,
        // e essa "cópia defensiva" estaria quebrada (só copiaria a referência
        // da lista, não o conteúdo — o mesmo erro raiz do resto do tópico).
        List<String> apelidosCopiados = new ArrayList<>(original.getApelidos());
        return new Pessoa(original.getNome(), apelidosCopiados);
    }

    public static void main(String[] args) {
        Pessoa joao = new Pessoa("João");
        adicionarApelido(joao, "Jotinha");

        Pessoa copia = criarCopiaDefensiva(joao);
        adicionarApelido(copia, "Apelido só da cópia");

        System.out.println("Original: " + joao.getApelidos());   // [Jotinha]
        System.out.println("Cópia: " + copia.getApelidos());     // [Jotinha, Apelido só da cópia]
        // O apelido adicionado na cópia NÃO vazou pro original, provando
        // que são listas fisicamente diferentes, não a mesma referência.
    }
}
```

_Raciocínio:_ este exercício conecta diretamente a Teoria com a Armadilha 3 — a única forma de garantir que um objeto passado como parâmetro não seja afetado por mutação externa é fazer uma **cópia real do conteúdo**, não confiar que "passar de novo" já protege automaticamente. `new ArrayList<>(listaExistente)` é o idiom padrão em Java para isso: cria uma nova lista, copiando os elementos da lista de origem para dentro dela — as duas listas passam a existir de forma completamente independente na memória, mesmo que os elementos (nesse caso, `String`s, que já são imutáveis) sejam compartilhados entre elas sem problema.