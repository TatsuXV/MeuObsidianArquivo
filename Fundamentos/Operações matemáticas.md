### 1. Teoria

**Operações matemáticas em Java**

Java oferece operadores aritméticos básicos embutidos na linguagem, além da classe utilitária `Math`, que cobre operações mais avançadas que não têm operador dedicado (potência, raiz, arredondamento, trigonometria, etc.).

#### Operadores aritméticos básicos

|Operador|Nome|Exemplo|
|---|---|---|
|`+`|Soma|`5 + 3` → `8`|
|`-`|Subtração|`5 - 3` → `2`|
|`*`|Multiplicação|`5 * 3` → `15`|
|`/`|Divisão|`5 / 3` → `1` (inteiros) ou `1.666...` (decimais)|
|`%`|Módulo (resto da divisão)|`5 % 3` → `2`|

Já vimos boa parte disso indiretamente em tópicos anteriores (divisão inteira truncando, `%` extraindo dígitos), mas aqui formalizamos as regras.

**Divisão: comportamento depende do tipo dos operandos**

java

```java
int resultado1 = 7 / 2;       // 3 — divisão inteira, trunca a parte decimal
double resultado2 = 7 / 2;    // 3.0 — ainda faz divisão INTEIRA primeiro (7/2=3), DEPOIS converte pra double!
double resultado3 = 7.0 / 2;  // 3.5 — agora sim, um operando já é double, então a divisão é decimal desde o início
```

Isso é uma das pegadinhas mais comuns: o tipo da variável que **recebe** o resultado não muda como a operação é calculada — o que importa é o tipo dos **operandos** envolvidos na operação em si.

**Módulo (`%`) com números negativos**

java

```java
System.out.println(7 % 3);   // 1
System.out.println(-7 % 3);  // -1 — o sinal do resultado segue o sinal do DIVIDENDO (o primeiro operando)
System.out.println(7 % -3);  // 1
```

Diferente de matemática pura (onde módulo geralmente é sempre não-negativo), o operador `%` em Java é tecnicamente o **resto** da divisão, e carrega o sinal do dividendo — isso é relevante quando você usa `%` pra lógica que assume sempre resultado positivo (ex: indexar um array circular).

#### Operadores de incremento/decremento

java

```java
int x = 5;
x++;  // pós-incremento: x vira 6
x--;  // pós-decremento: x vira 5 de novo
++x;  // pré-incremento: x vira 6
--x;  // pré-decremento: x vira 5
```

A diferença entre pré e pós só importa quando o incremento é usado **dentro de uma expressão maior**, no mesmo statement:

java

```java
int a = 5;
int b = a++; // b recebe o valor de "a" ANTES de incrementar → b = 5, depois a vira 6
int c = 5;
int d = ++c; // c incrementa PRIMEIRO, "d" recebe o valor JÁ incrementado → c = 6, d = 6
```

#### Operadores de atribuição composta

java

```java
int x = 10;
x += 5;  // equivale a x = x + 5  → 15
x -= 3;  // equivale a x = x - 3  → 12
x *= 2;  // equivale a x = x * 2  → 24
x /= 4;  // equivale a x = x / 4  → 6
x %= 4;  // equivale a x = x % 4  → 2
```

Detalhe importante: `x += 5` faz um cast implícito quando necessário — `byte b = 10; b += 5;` compila mesmo que `byte b = b + 5;` (sem o operador composto) **não compilaria** sem cast explícito, porque `b + 5` é promovido a `int` pela regra de promoção numérica, e atribuir `int` de volta a `byte` normalmente exigiria narrowing explícito. O operador composto "esconde" um cast automático — outra pegadinha sutil que vale saber que existe.

#### Precedência de operadores

```
() → parênteses, sempre avaliados primeiro
++ -- (unário) → incremento/decremento, negação unária
* / % → multiplicação, divisão, módulo
+ - → soma, subtração
```

Na dúvida sobre ordem de avaliação numa expressão complexa, **use parênteses explicitamente** — é mais legível e elimina qualquer ambiguidade, mesmo quando tecnicamente desnecessário pela regra de precedência.

#### Classe `Math`

java

```java
Math.pow(2, 10);       // 1024.0 — potência (retorna double, sempre)
Math.sqrt(16);          // 4.0 — raiz quadrada
Math.abs(-7);            // 7 — valor absoluto
Math.max(5, 9);          // 9
Math.min(5, 9);          // 5
Math.round(9.5);         // 10 — arredonda pro inteiro mais próximo (retorna long para double, int para float)
Math.floor(9.7);         // 9.0 — sempre arredonda pra BAIXO (retorna double)
Math.ceil(9.2);          // 10.0 — sempre arredonda pra CIMA (retorna double)
Math.random();            // double aleatório entre 0.0 (inclusivo) e 1.0 (exclusivo)
Math.PI;                  // constante, 3.14159...
Math.E;                   // constante, 2.71828... (número de Euler)
```

**Gerando número aleatório num intervalo específico**, usando `Math.random()`:

java

```java
int min = 1, max = 10;
int aleatorio = min + (int) (Math.random() * (max - min + 1)); // número entre 1 e 10, inclusive
```

**Onde isso aparece na prática (backend real)**

Cálculo de paginação (`Math.ceil` pra saber quantas páginas totais dado um total de itens e itens por página), cálculo de percentual/desconto em regras de negócio, validação de faixas de valor, e geração de identificadores/tokens temporários com `Math.random()` (embora, para segurança real como tokens de autenticação, o correto seja `SecureRandom`, não `Math.random()` — algo que será visto em Cryptography, mais adiante no roadmap). Overflow de `int` também é um bug real de produção quando cálculos financeiros ou contadores crescem além do esperado sem ninguém prever isso a tempo.

---

### 2. Exemplo de código comentado

java

```java
public class MathOperationsExemplo {
    public static void main(String[] args) {

        // Operadores aritméticos básicos
        System.out.println("7 / 2 (int): " + (7 / 2));         // 3
        System.out.println("7.0 / 2 (double): " + (7.0 / 2));  // 3.5
        System.out.println("7 % 3: " + (7 % 3));                // 1
        System.out.println("-7 % 3: " + (-7 % 3));               // -1, sinal segue o dividendo

        // Pré vs pós incremento
        int a = 5;
        int b = a++; // b pega o valor ANTES do incremento
        System.out.println("a++ : a=" + a + " b=" + b); // a=6, b=5

        int c = 5;
        int d = ++c; // c incrementa ANTES de ser atribuído
        System.out.println("++c : c=" + c + " d=" + d); // c=6, d=6

        // Atribuição composta com cast implícito escondido
        byte contador = 10;
        contador += 5; // funciona sem cast explícito, mesmo sendo narrowing por baixo dos panos
        System.out.println("Contador byte: " + contador); // 15

        // Precedência — sempre use parênteses quando não tiver 100% de certeza
        int resultado = 2 + 3 * 4;       // multiplicação primeiro: 2 + 12 = 14
        int resultadoComParenteses = (2 + 3) * 4; // agora a soma vem primeiro: 5 * 4 = 20
        System.out.println("Sem parênteses: " + resultado);
        System.out.println("Com parênteses: " + resultadoComParenteses);

        // Classe Math
        System.out.println("2^10: " + Math.pow(2, 10));
        System.out.println("Raiz de 16: " + Math.sqrt(16));
        System.out.println("Abs(-7): " + Math.abs(-7));
        System.out.println("Max(5,9): " + Math.max(5, 9));
        System.out.println("Round(9.5): " + Math.round(9.5));  // 10
        System.out.println("Floor(9.7): " + Math.floor(9.7));   // 9.0
        System.out.println("Ceil(9.2): " + Math.ceil(9.2));     // 10.0

        // Número aleatório num intervalo específico (1 a 10, inclusive)
        int min = 1, max = 10;
        int aleatorio = min + (int) (Math.random() * (max - min + 1));
        System.out.println("Aleatório entre 1 e 10: " + aleatorio);
    }
}
```

---

### 3. Armadilhas comuns

1. **Achar que atribuir o resultado a `double` muda como a divisão é calculada.** `double d = 7 / 2;` ainda faz divisão **inteira** primeiro (`3`), e só depois converte pra `double` (`3.0`) na atribuição — não vira `3.5`. É preciso que pelo menos um dos **operandos** já seja decimal.
2. **Confundir pré e pós-incremento dentro da mesma expressão.** `int x = 5; int y = x++ + ++x;` é um exemplo clássico de pegadinha de entrevista/prova — funciona, mas é ilegível e propenso a erro de leitura; em código profissional, evite combinar incrementos múltiplos numa expressão só, mesmo sabendo a regra.
3. **Esperar que `%` sempre retorne resultado positivo.** Como visto, `-7 % 3` dá `-1`, não `2`. Se você precisa de um resto sempre positivo (comum em lógica de índice circular), a fórmula segura é `((valor % modulo) + modulo) % modulo`.
4. **Não considerar overflow em cálculo com `int`.** `int resultado = 2_000_000_000 + 2_000_000_000;` estoura o limite de `int` (~2,1 bilhões) e "dá a volta" silenciosamente pro negativo, sem lançar exceção nem aviso — mesma lógica de overflow já vista em Data Types, mas reaparecendo aqui em contexto de cálculo real.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Declare `int a = 17;` e `int b = 5;`. Imprima o resultado de todas as 5 operações aritméticas básicas (`+`, `-`, `*`, `/`, `%`) entre eles, cada uma numa linha, com rótulo indicando qual operação é. Critério de pronto: os 5 resultados batem com o cálculo manual esperado (confira antes de rodar).

**Exercício 2 (médio)**  
Escreva um programa que calcule a média de 3 notas (`double nota1 = 7.5, nota2 = 8.0, nota3 = 6.5;`), usando `Math.round()` para exibir a média arredondada para o inteiro mais próximo, **junto** com a média exata sem arredondar (com casas decimais). Critério de pronto: as duas versões (arredondada e exata) aparecem, e a arredondada realmente corresponde à regra de arredondamento matemático (não truncamento).

**Exercício 3 (difícil)**  
Escreva um programa que simule uma calculadora de paginação: dado um `int totalDeItens = 47;` e um `int itensPorPagina = 10;`, calcule o número total de páginas necessárias usando `Math.ceil()` (dica: você vai precisar converter os `int` pra `double` antes da divisão, senão a divisão inteira trunca antes do `Math.ceil` ter chance de fazer algo). Teste também com um caso onde a divisão é exata (ex: `totalDeItens = 50`, `itensPorPagina = 10`) pra confirmar que não sobra uma página vazia indevidamente. Critério de pronto: `47/10` deve dar `5` páginas (não `4`, mesmo truncado dando 4 — a última página parcial ainda conta), e `50/10` deve dar exatamente `5`, sem página extra.

**Exercício 4 (desafio)**  
Implemente um simulador de dado de 6 lados que "rola" 10 vezes seguidas, usando `Math.random()` (sem usar a classe `Random` — isso será visto em outro contexto). Para cada rolagem, imprima o valor sorteado (entre 1 e 6, inclusive). Ao final, conte e imprima quantas vezes cada valor (1 a 6) saiu, usando um array `int[] contagem = new int[6];` como acumulador (dica: o valor sorteado, menos 1, dá o índice certo no array de contagem). Critério de pronto: a soma de todas as contagens no array final bate exatamente com 10 (o total de rolagens), e nenhum valor sorteado sai fora da faixa 1-6.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        int a = 17;
        int b = 5;

        System.out.println("Soma: " + (a + b));       // 22
        System.out.println("Subtração: " + (a - b));  // 12
        System.out.println("Multiplicação: " + (a * b)); // 85
        System.out.println("Divisão: " + (a / b));     // 3
        System.out.println("Módulo: " + (a % b));      // 2
    }
}
```

Raciocínio: nada além da aplicação direta de cada operador — o ponto de atenção é lembrar que `a / b` com dois `int` trunca (`17/5 = 3.4`, vira `3`), e `a % b` dá o resto dessa mesma divisão (`17 = 3*5 + 2`, resto `2`).

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        double nota1 = 7.5, nota2 = 8.0, nota3 = 6.5;

        double mediaExata = (nota1 + nota2 + nota3) / 3;
        long mediaArredondada = Math.round(mediaExata);

        System.out.println("Média exata: " + mediaExata);         // 7.333333333333333
        System.out.println("Média arredondada: " + mediaArredondada); // 7
    }
}
```

Raciocínio: `Math.round(double)` retorna `long` (não `int`), por isso a variável `mediaArredondada` é `long` — usar `int` exigiria cast explícito adicional. A divisão `(nota1 + nota2 + nota3) / 3` já é decimal desde o início porque os três operandos da soma são `double`, então o `3` (int) é automaticamente promovido a `double` na divisão — sem precisar de cast manual aqui, diferente da armadilha vista na teoria (que só acontece quando o dividendo original já seria `int`).

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        int totalDeItens = 47;
        int itensPorPagina = 10;

        int totalPaginas = (int) Math.ceil((double) totalDeItens / itensPorPagina);
        System.out.println("47 itens, 10 por página: " + totalPaginas + " páginas"); // 5

        int totalDeItens2 = 50;
        int totalPaginas2 = (int) Math.ceil((double) totalDeItens2 / itensPorPagina);
        System.out.println("50 itens, 10 por página: " + totalPaginas2 + " páginas"); // 5
    }
}
```

Raciocínio: o cast `(double) totalDeItens` é essencial **antes** da divisão — sem ele, `totalDeItens / itensPorPagina` já trunca pra `4` (divisão inteira) antes mesmo do `Math.ceil` entrar em ação, e arredondar pra cima um número que já foi truncado errado não resolve o problema (`Math.ceil(4.0)` continua sendo `4.0`, resultado errado). Com o cast aplicado antes, `47.0 / 10 = 4.7`, e `Math.ceil(4.7) = 5.0` — o resultado correto, contando a página parcial. No caso exato (`50/10 = 5.0`), `Math.ceil` de um número já inteiro não adiciona página extra nenhuma, porque `ceil` de um valor que já é "redondo" retorna ele mesmo.

**Exercício 4**

java

```java
public class Exercicio4 {
    public static void main(String[] args) {
        int[] contagem = new int[6]; // índices 0 a 5, representando os valores 1 a 6

        for (int i = 0; i < 10; i++) {
            int valorSorteado = 1 + (int) (Math.random() * 6); // 1 a 6, inclusive
            System.out.println("Rolagem " + (i + 1) + ": " + valorSorteado);
            contagem[valorSorteado - 1]++; // valor 1 vai pro índice 0, valor 6 vai pro índice 5
        }

        System.out.println("--- Contagem final ---");
        int somaTotal = 0;
        for (int i = 0; i < contagem.length; i++) {
            System.out.println("Valor " + (i + 1) + ": " + contagem[i] + " vez(es)");
            somaTotal += contagem[i];
        }
        System.out.println("Total de rolagens contadas: " + somaTotal); // deve bater com 10
    }
}
```

Raciocínio: `Math.random()` retorna um `double` entre `0.0` (inclusive) e `1.0` (exclusivo). Multiplicar por `6` dá um valor entre `0.0` e `5.999...`; o cast pra `int` trunca isso pra um inteiro entre `0` e `5`; somar `1` desloca a faixa final pra `1` a `6`, exatamente o que um dado real produz. O array `contagem` usa `valorSorteado - 1` como índice porque arrays são zero-indexados, mas os valores do dado começam em `1` — esse "desvio de 1" entre valor de domínio (1-6) e índice de array (0-5) é um padrão recorrente que vale internalizar, porque aparece em várias situações parecidas.