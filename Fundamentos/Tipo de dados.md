### 1. Teoria

**O que são "tipos de dados" em Java?**

Java é uma linguagem **fortemente tipada** (_strongly typed_): toda variável precisa ter um tipo declarado, e esse tipo é fixado em tempo de compilação. Isso é diferente de linguagens como Python ou JavaScript, onde uma variável pode "trocar de tipo" durante a execução. Em Java, se você declara `int idade`, `idade` nunca vai poder guardar um `String` — o compilador barra isso antes mesmo de rodar.

Os tipos em Java se dividem em duas categorias fundamentais:

#### Tipos primitivos (8 no total)

São os tipos "crus", que não são objetos — armazenam o valor diretamente na memória, sem indireção. São 4 grupos:

|Grupo|Tipo|Tamanho|Faixa de valores|
|---|---|---|---|
|Inteiros|`byte`|8 bits|-128 a 127|
||`short`|16 bits|-32.768 a 32.767|
||`int`|32 bits|-2.147.483.648 a 2.147.483.647|
||`long`|64 bits|±9,2 quintilhões (aprox.)|
|Ponto flutuante|`float`|32 bits|precisão simples (IEEE 754)|
||`double`|64 bits|precisão dupla (IEEE 754)|
|Caractere|`char`|16 bits|0 a 65.535 (representa um code unit UTF-16)|
|Lógico|`boolean`|não definido pela spec (depende da JVM)|`true` ou `false`|

Alguns pontos que costumam gerar confusão:

- **`int` é o padrão** para literais inteiros e `double` é o padrão para literais decimais. Se você escrever `long x = 10000000000;` sem o sufixo `L`, o compilador trata `10000000000` como `int` primeiro — e como isso estoura a faixa do `int`, dá **erro de compilação**, não de execução.
- **`char` não é "mini-String"**. É tecnicamente um tipo numérico (representa um número que mapeia pra um caractere Unicode). Por isso `char` participa de contas aritméticas normalmente.
- **Não existe tipo primitivo sem sinal** em Java (exceto `char`, que na prática funciona como um inteiro sem sinal de 16 bits). Isso é diferente de C/C++, que têm `unsigned int`, por exemplo.

#### Tipos de referência (não-primitivos)

Tudo que não é um dos 8 primitivos acima é um **tipo de referência**: `String`, arrays, e qualquer classe (inclusive as classes wrapper, que veremos abaixo). A variável não guarda o valor diretamente — guarda uma referência (endereço) pro objeto na memória. `String`, por exemplo, é uma classe, não um primitivo — por isso `String` começa com maiúscula e `char` com minúscula: essa diferença de capitalização não é estética, é a convenção que sinaliza "isto é uma classe" vs "isto é primitivo".

**Valores padrão (default values)**

Isso só se aplica a **atributos de classe** (campos), não a variáveis locais:

|Tipo|Valor padrão|
|---|---|
|`byte`, `short`, `int`|`0`|
|`long`|`0L`|
|`float`|`0.0f`|
|`double`|`0.0d`|
|`char`|`'\u0000'`|
|`boolean`|`false`|
|Qualquer tipo de referência (`String`, etc.)|`null`|

**Importante:** variável **local** (declarada dentro de um método) **não recebe valor padrão automaticamente**. Se você declarar `int x;` dentro do `main` e tentar usar `x` antes de atribuir um valor, o compilador rejeita com erro de "variable might not have been initialized". Isso é uma regra de segurança do compilador chamada _definite assignment_.

**Onde isso aparece na prática (backend real)**

Escolher o tipo certo importa em produção: usar `int` pra armazenar um ID que pode crescer além de ~2,1 bilhões é um bug real de sistemas legados (é literalmente uma classe de bug conhecida). Usar `double` pra dinheiro é outra armadilha clássica — ponto flutuante binário não representa valores decimais exatos, então cálculo financeiro com `double` acumula erro de arredondamento; o padrão de mercado é usar `BigDecimal` pra dinheiro (isso será aprofundado quando chegarmos em classes/tipos de referência mais avançados).

---

### 2. Exemplo de código comentado

java

```java
public class TiposDeDados {
    public static void main(String[] args) {

        // Inteiros — escolha do tamanho importa
        byte idadeAluno = 25;              // cabe tranquilo em 8 bits (-128 a 127)
        int populacaoCidade = 3_000_000;   // underscore é só separador visual, ignorado pelo compilador (desde Java 7)
        long distanciaEstrelaLuz = 9_460_730_472_580_800L; // precisa do sufixo L: sem ele, é tratado como int e estoura

        // Ponto flutuante
        float precoProduto = 19.90f;       // precisa do sufixo f, senão o compilador trata como double
        double pi = 3.14159265358979;      // double é o padrão pra literais decimais, não precisa de sufixo

        // Caractere — é numérico por baixo dos panos
        char letraInicial = 'A';
        int codigoDaLetra = letraInicial;  // conversão implícita: char vira o código Unicode dele (65 pro 'A')
        System.out.println("Código Unicode de 'A': " + codigoDaLetra);

        // char aceita aritmética direta
        char proximaLetra = (char) (letraInicial + 1); // 'A' + 1 = 66, precisa converter de volta pra char
        System.out.println("Próxima letra: " + proximaLetra); // imprime 'B'

        // Booleano
        boolean maiorDeIdade = idadeAluno >= 18;
        System.out.println("Maior de idade? " + maiorDeIdade);

        // Overflow silencioso — Java NÃO lança exceção aqui, apenas "dá a volta"
        byte limiteByte = 127;
        limiteByte++; // ultrapassa 127, vira -128 silenciosamente
        System.out.println("Overflow de byte: " + limiteByte); // imprime -128

        // Divisão de inteiros trunca, não arredonda
        int resultadoInteiro = 7 / 2;
        System.out.println("7 / 2 como int: " + resultadoInteiro); // imprime 3, não 3.5

        // String não é primitivo — é referência (classe)
        String nome = "Java"; // aprofundado no tópico "Strings and Methods"
    }
}
```

---

### 3. Armadilhas comuns

1. **Overflow silencioso.** Diferente de outras linguagens que lançam exceção em estouro de faixa, Java simplesmente "dá a volta" (wraps around) sem avisar nada. `byte b = 127; b++;` vira `-128` sem erro nenhum — isso é uma fonte real de bug sutil em produção.
2. **Divisão de inteiros truncando ao invés de arredondar.** `int resultado = 5 / 2;` dá `2`, não `2.5`. Pra ter resultado decimal, pelo menos um dos operandos precisa ser `double`/`float` (ex: `5 / 2.0`).
3. **Esquecer o sufixo `L` ou `f`.** `long x = 3000000000;` não compila (estoura `int` antes de virar `long`). `float f = 3.5;` também não compila, porque `3.5` sem sufixo é `double`, e atribuir `double` a `float` exige conversão explícita (perda de precisão).
4. **Usar `double` pra valores monetários.** `0.1 + 0.2` em `double` não dá exatamente `0.3` — dá algo como `0.30000000000000004`, por causa de como ponto flutuante binário representa frações decimais. Isso quebra sistema financeiro de verdade se ninguém perceber a tempo.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Declare uma variável de cada um dos 8 tipos primitivos, com um valor de exemplo plausível para cada um (ex: `idade` como `byte`, `salario` como `double`, etc.). Imprima todas usando `System.out.println`, uma por linha, no formato `"nomeDoTipo: valor"`. Critério de pronto: as 8 linhas aparecem, sem erro de compilação, com os sufixos corretos onde forem necessários.

**Exercício 2 (médio)**  
Escreva um programa que declare `int a = 10;` e `int b = 3;`, e imprima o resultado de `a / b` (divisão inteira) e depois o resultado da divisão "correta" com casas decimais, sem alterar o tipo de `a` e `b` (ou seja, resolva convertendo na hora da operação, não mudando a declaração original). Critério de pronto: primeira saída mostra `3`, segunda mostra `3.3333333333333335` (ou aproximado).

**Exercício 3 (difícil)**  
Demonstre o overflow de `byte` explicitamente: declare `byte contador = 125;` e, num laço simples (`for`, mesmo sem ter estudado o tópico ainda, pode usar `for (int i = 0; i < 5; i++)`), incremente `contador` 5 vezes, imprimindo o valor a cada incremento. Antes de rodar, escreva por escrito qual valor você espera ver na 3ª, 4ª e 5ª iteração — depois rode e confira se acertou seu próprio cálculo mental de overflow.

**Exercício 4 (desafio)**  
Escreva um programa que declare `float valorFloat = 0.1f + 0.2f;` e `double valorDouble = 0.1 + 0.2;`, e imprima os dois. Explique por escrito (comentário no código) por que nenhum dos dois dá exatamente `0.3`, e pesquise (ou responda com o que já sabe da teoria acima) qual seria a alternativa correta em Java pra somar valores monetários com precisão exata — não precisa implementar essa alternativa ainda, só nomear a classe certa e por que ela resolve o problema.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        byte b = 100;
        short s = 20000;
        int i = 1_000_000;
        long l = 5_000_000_000L; // precisa do L, estoura int
        float f = 9.99f;         // precisa do f, senão é double
        double d = 99.999;
        char c = 'J';
        boolean bool = true;

        System.out.println("byte: " + b);
        System.out.println("short: " + s);
        System.out.println("int: " + i);
        System.out.println("long: " + l);
        System.out.println("float: " + f);
        System.out.println("double: " + d);
        System.out.println("char: " + c);
        System.out.println("boolean: " + bool);
    }
}
```

Raciocínio: o ponto central aqui é lembrar dos sufixos `L` e `f` — sem eles, `long` e `float` não compilam quando o literal não cabe (ou não é do tipo certo) por padrão.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        int a = 10;
        int b = 3;

        System.out.println("Divisão inteira: " + (a / b)); // 3

        // Cast explícito de um dos operandos pra double na hora da operação,
        // sem alterar a declaração de a e b (que continuam int)
        double divisaoDecimal = (double) a / b;
        System.out.println("Divisão decimal: " + divisaoDecimal); // 3.3333333333333335
    }
}
```

Raciocínio: `(double) a / b` funciona porque o cast tem precedência maior — primeiro `a` vira `double`, e aí a divisão inteira **não acontece mais**, porque agora um dos operandos não é inteiro. Se você escrevesse `(double) (a / b)`, o cast aconteceria _depois_ da divisão inteira já ter truncado o resultado, e o resultado seria `3.0`, não `3.333...` — essa é uma pegadinha clássica de onde colocar o parêntese do cast.

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        byte contador = 125;
        for (int i = 0; i < 5; i++) {
            contador++;
            System.out.println("Iteração " + i + ": " + contador);
        }
    }
}
```

Saída esperada:

```
Iteração 0: 126
Iteração 1: 127
Iteração 2: -128
Iteração 3: -127
Iteração 4: -126
```

Raciocínio: `byte` vai até `127`. Ao passar desse limite, o valor "dá a volta" pelo lado negativo, recomeçando em `-128` — isso é o comportamento de _overflow_ de tipos inteiros em Java: aritmética modular em complemento de dois, sem lançar exceção.

**Exercício 4**

java

```java
public class Exercicio4 {
    public static void main(String[] args) {
        float valorFloat = 0.1f + 0.2f;
        double valorDouble = 0.1 + 0.2;

        System.out.println("float: " + valorFloat);   // 0.3 (pode variar a exibição, mas internamente é aproximado)
        System.out.println("double: " + valorDouble);  // 0.30000000000000004

        // Nenhum dos dois dá exatamente 0.3 porque float e double representam
        // números em base binária (IEEE 754), e frações decimais como 0.1 e 0.2
        // não têm representação binária finita exata — é o mesmo motivo pelo qual
        // 1/3 não tem representação decimal finita exata em base 10.
        // A alternativa correta pra valores monetários é a classe BigDecimal,
        // que representa números decimais exatamente (sem essa perda de precisão
        // binária), sendo o padrão de mercado pra cálculo financeiro em Java.
    }
}
```

Raciocínio: esse não é um "bug" do Java — é uma limitação universal de qualquer linguagem que usa ponto flutuante binário (IEEE 754), incluindo C, Python, JavaScript etc. A solução não é "usar mais casas decimais", é trocar de representação inteiramente (`BigDecimal`), algo que será aprofundado quando chegarmos em classes/tipos de referência mais adiante no roadmap.