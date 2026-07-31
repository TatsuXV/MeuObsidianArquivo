**O que é type casting?**

Type casting é o processo de **converter um valor de um tipo pra outro**. Já vimos isso de forma pontual em tópicos anteriores (`(double) a / b`, `(char)(letraInicial + 1)`) — agora vamos formalizar as regras completas.

Existem duas categorias bem distintas: casting entre **tipos primitivos** e casting entre **tipos de referência**. São mecanismos diferentes, com regras diferentes.

#### Casting entre primitivos numéricos

**Widening (conversão implícita, "alargamento")**

Acontece automaticamente, sem precisar escrever nada, quando você converte de um tipo "menor" pra um "maior" — não há risco de perda de dado:

```
byte → short → int → long → float → double
```

(`char` também entra nessa cadeia, convertendo implicitamente pra `int` e adiante)

java

```java
int numeroInt = 100;
double numeroDouble = numeroInt; // widening automático, sem cast explícito
```

**Narrowing (conversão explícita, "estreitamento")**

Precisa de cast explícito com `(tipo)`, porque **pode perder informação** — o compilador te obriga a declarar que você sabe do risco:

java

```java
double valor = 9.99;
int valorInt = (int) valor; // narrowing explícito — trunca a parte decimal, valorInt = 9 (não arredonda!)

int numeroGrande = 300;
byte numeroByte = (byte) numeroGrande; // narrowing — 300 não cabe em byte (-128 a 127), resultado é imprevisível pro leigo (na verdade seguem regras de overflow/wraparound, dá 44)
```

**Ponto crítico: narrowing de `double`/`float` pra tipo inteiro trunca, não arredonda.** `(int) 9.99` dá `9`, não `10`. Se você quer arredondar de verdade, precisa de `Math.round()` antes do cast (aprofundado em Math Operations).

#### Casting entre tipos de referência (upcasting e downcasting)

Isso só existe entre tipos que têm relação de herança/implementação (aprofundado de verdade em Inheritance/Polymorphism — aqui é só a mecânica da sintaxe de cast em si):

java

```java
Object obj = "Uma String"; // upcasting implícito — String é subtipo de Object, sempre seguro
String texto = (String) obj; // downcasting explícito — exige cast, pode falhar em tempo de execução
```

- **Upcasting** (subtipo → supertipo): sempre seguro, acontece implicitamente, sem precisar de `(tipo)`.
- **Downcasting** (supertipo → subtipo): precisa de cast explícito, e **pode lançar `ClassCastException`** em tempo de execução se o objeto não for realmente daquele tipo por baixo dos panos.

java

```java
Object valor = "texto";
Integer numero = (Integer) valor; // compila! Mas lança ClassCastException em tempo de execução,
                                    // porque o objeto real por trás de "valor" é uma String, não um Integer
```

O compilador só verifica se a conversão é _sintaticamente plausível_ (existe alguma relação entre os tipos) — ele não sabe, em tempo de compilação, o que realmente está guardado dentro da variável `Object` em tempo de execução.

#### Conversão entre `String` e tipos primitivos (não é "cast" tecnicamente, mas é convertido junto no dia a dia)

java

```java
// primitivo → String
int numero = 42;
String texto1 = String.valueOf(numero);   // forma recomendada
String texto2 = "" + numero;              // funciona, mas menos explícito/legível

// String → primitivo
String entrada = "123";
int valorConvertido = Integer.parseInt(entrada); // lança NumberFormatException se "entrada" não for um número válido
double valorDouble = Double.parseDouble("3.14");
```

Isso **não é cast** no sentido técnico da palavra (`(int) "123"` não compila — `String` e `int` não têm relação de tipo nenhuma) — é conversão via método de classe utilitária. Mas é tão comum confundir os dois conceitos que vale deixar a distinção clara aqui.

**Onde isso aparece na prática (backend real)**

Narrowing e downcasting mal feitos são fontes reais de bug de produção: `ClassCastException` acontece direto quando se trabalha com coleções genéricas mal tipadas ou APIs que retornam `Object`. `Integer.parseInt` sem tratamento de exceção quebra qualquer endpoint que recebe entrada de usuário como texto e espera número — é por isso que validação de entrada em controllers Spring é levada tão a sério.

---

### 2. Exemplo de código comentado

java

```java
public class TypeCastingExemplo {
    public static void main(String[] args) {

        // WIDENING — automático, sem risco de perda
        int idade = 25;
        long idadeLonga = idade;      // int → long, implícito
        double idadeDouble = idadeLonga; // long → double, implícito
        System.out.println("Widening: " + idade + " → " + idadeLonga + " → " + idadeDouble);

        // NARROWING — explícito, com risco de perda
        double preco = 49.99;
        int precoTruncado = (int) preco; // trunca, não arredonda: 49, não 50
        System.out.println("Narrowing com truncamento: " + precoTruncado);

        // Narrowing com overflow real (fora da faixa do tipo destino)
        int numeroGrande = 130;
        byte numeroConvertido = (byte) numeroGrande; // 130 estoura o limite de byte (127)
        System.out.println("Overflow no narrowing: " + numeroConvertido); // dá -126, por wraparound

        // Upcasting — implícito, sempre seguro
        Object objGenerico = "Isso é uma String"; // String "sobe" pra Object sem cast

        // Downcasting — explícito, pode falhar em runtime
        String textoDeVolta = (String) objGenerico; // funciona, porque objGenerico REALMENTE é uma String
        System.out.println("Downcasting bem-sucedido: " + textoDeVolta);

        // Downcasting que FALHA em runtime — comentado pra não quebrar a execução do resto do programa
        // Object outroObjeto = Integer.valueOf(10);
        // String vaiFalhar = (String) outroObjeto; // ClassCastException aqui

        // Conversão String <-> primitivo (não é cast tecnicamente)
        String textoNumero = "150";
        int numeroConvertidoDeString = Integer.parseInt(textoNumero);
        System.out.println("String pra int: " + numeroConvertidoDeString);

        int valorOriginal = 99;
        String valorComoTexto = String.valueOf(valorOriginal);
        System.out.println("Int pra String: " + valorComoTexto);

        // Arredondar de verdade (em vez de truncar) — precisa de Math.round ANTES do cast
        double valorParaArredondar = 9.7;
        int arredondado = (int) Math.round(valorParaArredondar); // Math.round retorna long, por isso o cast pra int aqui
        System.out.println("Arredondado corretamente: " + arredondado); // 10, não 9
    }
}
```

---

### 3. Armadilhas comuns

1. **Achar que narrowing de decimal pra inteiro arredonda.** `(int) 9.99` dá `9`, sempre trunca (descarta a parte decimal), nunca arredonda. Pra arredondar de verdade, é preciso `Math.round()` antes do cast.
2. **Downcasting sem checar o tipo real antes.** `(String) objeto` compila sempre que existe relação de herança plausível entre os tipos — mas só funciona em tempo de execução se o objeto realmente for daquele tipo. O jeito seguro é checar antes com `instanceof` (será aprofundado, mas já vale adiantar o padrão: `if (objeto instanceof String) { ... }`).
3. **Esquecer que `Integer.parseInt` lança exceção com entrada inválida.** `Integer.parseInt("abc")` ou `Integer.parseInt("")` lança `NumberFormatException` — isso quebra o programa se não for tratado (Exception Handling será visto em bloco dedicado, mas já é bom saber que essa conversão é um ponto de risco).
4. **Cast explícito "forçando" um narrowing perigoso sem entender a consequência.** `(byte) 200` não dá erro de compilação nem de execução — dá silenciosamente um valor errado (`-56`, por overflow), porque cast explícito é a forma que você tem de dizer ao compilador "eu sei o que estou fazendo, deixa passar" — e o compilador confia em você, mesmo quando o resultado é matematicamente "errado" pra intenção original.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Declare uma variável `double valor = 15.75;`. Imprima o valor convertido pra `int` usando cast direto (mostrando o truncamento) e, em seguida, o valor arredondado corretamente usando `Math.round()` + cast pra `int`. Critério de pronto: a primeira impressão mostra `15`, a segunda mostra `16`.

**Exercício 2 (médio)**  
Declare uma `String entrada = "42";` e converta pra `int` usando `Integer.parseInt`. Depois, declare uma segunda `String entradaInvalida = "quarenta e dois";` e tente convertê-la também — **sem usar try/catch ainda** (isso será visto em Exception Handling), apenas escreva um comentário no código explicando qual exceção você espera que seja lançada e por quê, sem executar essa segunda conversão de fato (comente a linha ou coloque num bloco separado que você não roda). Critério de pronto: a primeira conversão funciona e imprime `42`; o comentário sobre a segunda está tecnicamente correto (nome da exceção certo).

**Exercício 3 (difícil)**  
Declare `int numeroGrande = 1000;` e converta para `byte` usando cast explícito. Antes de rodar, calcule manualmente por escrito qual deveria ser o resultado do overflow (dica: `byte` vai de -128 a 127, e o comportamento é aritmética modular — pesquise "cálculo de overflow por complemento de dois" se quiser entender o mecanismo exato, mas o importante aqui é você tentar prever o resultado antes de só rodar e ver). Depois rode e confira se sua previsão bateu.

**Exercício 4 (desafio)**  
Escreva um programa que declare `Object valor1 = "Uma String de verdade";` e `Object valor2 = Integer.valueOf(100);`. Usando `instanceof` (pesquise a sintaxe exata se não tiver certeza — é um operador que retorna `boolean` verificando o tipo real do objeto em tempo de execução) **antes** de cada downcasting, escreva uma lógica que só faz o cast pra `String` (ou `Integer`) se o `instanceof` confirmar que é seguro, evitando `ClassCastException` mesmo ao tentar castar `valor2` (que não é `String`) incorretamente. Critério de pronto: o programa roda sem lançar nenhuma exceção, mesmo tentando (de forma segura, verificada) castar os dois valores para tipos diferentes dos que realmente são.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        double valor = 15.75;

        int truncado = (int) valor;
        System.out.println("Truncado: " + truncado); // 15

        int arredondado = (int) Math.round(valor);
        System.out.println("Arredondado: " + arredondado); // 16
    }
}
```

Raciocínio: cast direto `(int) valor` simplesmente descarta a parte decimal, sem considerar se ela é maior ou menor que 0.5 — por isso `15.75` vira `15`, não `16`, mesmo estando mais perto de `16`. `Math.round()` aplica a regra matemática de arredondamento de verdade antes da conversão, e é por isso que o resultado muda pra `16`.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        String entrada = "42";
        int numeroConvertido = Integer.parseInt(entrada);
        System.out.println("Convertido: " + numeroConvertido); // 42

        String entradaInvalida = "quarenta e dois";
        // int vaiFalhar = Integer.parseInt(entradaInvalida);
        // Isso lançaria NumberFormatException, porque "quarenta e dois" não é
        // uma sequência de dígitos válida — Integer.parseInt só aceita Strings
        // que representam números inteiros no formato numérico padrão (com
        // sinal opcional de - na frente, sem espaços, sem palavras).
    }
}
```

Raciocínio: `NumberFormatException` é uma subclasse de `IllegalArgumentException`, lançada especificamente quando a `String` passada não segue o formato esperado de número — o método tenta interpretar caractere por caractere e falha assim que encontra algo que não é dígito válido no contexto.

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        int numeroGrande = 1000;
        byte numeroConvertido = (byte) numeroGrande;
        System.out.println("Resultado: " + numeroConvertido);
    }
}
```

Saída: `Resultado: -24`

Raciocínio (cálculo manual): `byte` usa 8 bits, faixa de -128 a 127 (256 valores possíveis no total). O narrowing pega só os últimos 8 bits da representação binária de `1000` e reinterpreta esses bits como um `byte` com sinal. Uma forma prática de calcular sem entender bit a bit: `1000 mod 256 = 232`. Como `232` é maior que `127` (o máximo positivo de um `byte`), ele "estoura" pro lado negativo: `232 - 256 = -24`. Essa é a mesma lógica de aritmética modular que vimos no overflow de `byte++` no tópico de Data Types, só que aplicada de uma vez, via cast, em vez de incremento repetido.

**Exercício 4**

java

```java
public class Exercicio4 {
    public static void main(String[] args) {
        Object valor1 = "Uma String de verdade";
        Object valor2 = Integer.valueOf(100);

        // Tentando castar valor1 pra String — deve funcionar, valor1 REALMENTE é String
        if (valor1 instanceof String) {
            String texto = (String) valor1;
            System.out.println("valor1 é String: " + texto);
        } else {
            System.out.println("valor1 não é String, cast evitado");
        }

        // Tentando castar valor2 pra String — NÃO deve funcionar, valor2 é Integer
        if (valor2 instanceof String) {
            String texto = (String) valor2;
            System.out.println("valor2 é String: " + texto);
        } else {
            System.out.println("valor2 não é String, cast evitado com segurança"); // este é o caminho executado
        }

        // Tentando castar valor2 pra Integer — deve funcionar, valor2 REALMENTE é Integer
        if (valor2 instanceof Integer) {
            Integer numero = (Integer) valor2;
            System.out.println("valor2 é Integer: " + numero);
        }
    }
}
```

Raciocínio: `instanceof` verifica, em tempo de execução, o tipo _real_ do objeto por trás da referência — não o tipo declarado da variável (`Object`), mas o que foi de fato instanciado (`String` ou `Integer`). Ao colocar o cast **dentro** do bloco `if` que já confirmou o tipo com `instanceof`, elimino completamente o risco de `ClassCastException`: o cast só roda quando já sei, com certeza, que é seguro. Essa combinação `instanceof` + cast é exatamente o padrão defensivo usado em código de produção sempre que se trabalha com tipos genéricos demais (`Object`, ou hierarquias de herança complexas).