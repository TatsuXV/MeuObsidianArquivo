### 1. Teoria

**O que é um array?**

Um array é uma estrutura que armazena uma **coleção de elementos do mesmo tipo**, em posições contíguas de memória, com **tamanho fixo** definido no momento da criação. Uma vez criado, você não pode "adicionar" ou "remover" posições — o tamanho é imutável (isso é diferente de `ArrayList`, que veremos no bloco de Coleções, e que resolve exatamente essa limitação).

Java trata array como um **tipo de referência** (mesmo array de primitivos), ou seja, a variável guarda uma referência ao bloco de memória, não os valores diretamente.

**Declaração e criação**

java

```java
// Declaração + criação em uma linha
int[] numeros = new int[5]; // array de 5 inteiros, todos inicializados com 0 (valor padrão do tipo)

// Declaração + inicialização com valores literais
int[] idades = {18, 25, 30, 42};

// Forma alternativa de declaração (válida, mas menos usada)
int numeros2[] = new int[5]; // colchete depois do nome — herança de sintaxe do C, evite em código novo
```

Pontos importantes:

- **Índices começam em 0.** Um array de tamanho 5 tem índices válidos de `0` a `4`. Acessar o índice `5` lança `ArrayIndexOutOfBoundsException` em tempo de execução — o compilador não pega esse erro, porque índice geralmente vem de uma variável/cálculo, não é conhecido em tempo de compilação.
- **Tamanho fixo desde a criação.** `numeros.length` (sem parênteses — é um atributo, não um método, diferente de `String.length()`) retorna o tamanho, mas não existe forma de "redimensionar" um array depois de criado. Pra "crescer", você cria um array novo e copia os elementos (ou usa `ArrayList`).
- **Valor padrão em array recém-criado com `new tipo[tamanho]`:** segue a mesma tabela de valores padrão de atributos — `0` para numéricos, `false` para `boolean`, `null` para tipos de referência (incluindo `String[]`).

**Arrays multidimensionais**

Java não tem array multidimensional "nativo" — o que existe é **array de arrays**:

java

```java
int[][] matriz = new int[3][4]; // 3 linhas, 4 colunas
matriz[0][0] = 1;
matriz[2][3] = 99;

// Inicialização literal
int[][] tabuleiro = {
    {1, 2, 3},
    {4, 5, 6}
};
```

Isso permite, inclusive, arrays "irregulares" (jagged arrays), onde cada linha tem um tamanho diferente — algo que não existe em linguagens com matriz verdadeira como array 2D contíguo.

**Percorrendo um array**

java

```java
// for tradicional — quando você precisa do índice
for (int i = 0; i < numeros.length; i++) {
    System.out.println(numeros[i]);
}

// for-each (enhanced for) — quando você só precisa do valor, não do índice
for (int numero : numeros) {
    System.out.println(numero);
}
```

O `for-each` é preferível quando você não precisa do índice, porque elimina a chance de erro de off-by-one (`<=` em vez de `<`, por exemplo) — mas ele não permite modificar o array original nem saber a posição atual sem uma variável contadora extra.

**Classe utilitária `Arrays`**

java

```java
import java.util.Arrays;

int[] nums = {5, 2, 8, 1};
Arrays.sort(nums);                        // ordena in-place: [1, 2, 5, 8]
System.out.println(Arrays.toString(nums)); // imprime "[1, 2, 5, 8]" — println direto em array imprime o hashcode, não o conteúdo
int[] copia = Arrays.copyOf(nums, 6);      // copia pra um array maior, preenchendo o resto com 0
boolean igual = Arrays.equals(nums, copia); // compara conteúdo, não referência (diferente de ==)
```

**Onde isso aparece na prática (backend real)**

Array puro aparece menos no dia a dia de backend Spring do que `List`/`ArrayList` — a maioria dos retornos de repository, por exemplo, são coleções, não arrays. Mas array continua aparecendo em: parâmetros de métodos utilitários (`String[] args` do `main`, que você já viu desde o primeiro tópico), resultado de `String.split(String)`, manipulação de bytes (`byte[]` em criptografia, leitura de arquivo), e como base interna de estruturas de dados que você vai implementar na Trilha de DSA (heap, hash table). Entender array bem é pré-requisito direto pra entender como `ArrayList` funciona por baixo.

---

### 2. Exemplo de código comentado

java

```java
import java.util.Arrays;

public class ArraysExemplo {
    public static void main(String[] args) {

        // Criação com tamanho fixo — valores começam em 0 (valor padrão de int)
        int[] notas = new int[4];
        notas[0] = 8;
        notas[1] = 6;
        notas[2] = 10;
        notas[3] = 7;
        // notas[4] = 5; // isso lançaria ArrayIndexOutOfBoundsException — índice válido só vai até 3

        // Criação com valores literais direto
        String[] nomes = {"Ana", "Bruno", "Carla"};

        // .length é atributo, sem parênteses (diferente de String.length())
        System.out.println("Quantidade de notas: " + notas.length);

        // for tradicional — usado aqui porque preciso do índice pra montar a mensagem
        for (int i = 0; i < nomes.length; i++) {
            System.out.println("Posição " + i + ": " + nomes[i]);
        }

        // for-each — preferível quando só preciso do valor, sem o índice
        int soma = 0;
        for (int nota : notas) {
            soma += nota;
        }
        double media = (double) soma / notas.length; // cast pra evitar divisão inteira truncada
        System.out.println("Média: " + media);

        // Array multidimensional (array de arrays)
        int[][] matriz = {
            {1, 2, 3},
            {4, 5, 6}
        };
        System.out.println("Elemento [1][2]: " + matriz[1][2]); // acessa linha 1, coluna 2 → imprime 6

        // Classe utilitária Arrays
        int[] desordenado = {9, 3, 7, 1};
        Arrays.sort(desordenado); // ordena in-place — modifica o array original
        System.out.println("Ordenado: " + Arrays.toString(desordenado));

        // Println direto em array NÃO imprime o conteúdo, imprime referência/hashcode
        System.out.println("Sem Arrays.toString: " + desordenado); // algo como [I@1b6d3586
    }
}
```

---

### 3. Armadilhas comuns

1. **`ArrayIndexOutOfBoundsException`.** O erro clássico de off-by-one: usar `<=` em vez de `<` no `for` tradicional (`for (int i = 0; i <= array.length; i++)`) tenta acessar um índice que não existe, porque o último índice válido é `length - 1`, não `length`.
2. **Confundir `.length` (array) com `.length()` (String).** Array usa atributo sem parênteses; `String` usa método com parênteses. Trocar um pelo outro é erro de compilação direto, mas ainda assim é a confusão mais comum de quem está começando com os dois ao mesmo tempo.
3. **Usar `System.out.println(array)` esperando ver o conteúdo.** Isso imprime algo como `[I@1b6d3586` (o tipo e o hashcode), não os valores. É preciso `Arrays.toString(array)` pra arrays 1D, ou `Arrays.deepToString(array)` pra arrays multidimensionais.
4. **Achar que dá pra "aumentar" um array existente.** `array[5] = 10;` num array de tamanho 5 não "expande" nada — lança exceção. Pra crescer, é preciso criar um array novo (com `Arrays.copyOf`, por exemplo) ou, melhor ainda na prática, usar `ArrayList` desde o início se o tamanho não for fixo e conhecido de antemão.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Crie um array de `int` com 6 números escolhidos por você (literal, direto na declaração). Usando um `for` tradicional, imprima cada elemento no formato `"Índice X: valor"`. Depois, usando um `for-each`, calcule e imprima a soma de todos os elementos. Critério de pronto: as duas formas de percorrer o array aparecem no mesmo programa, cada uma usada de forma apropriada ao que ela resolve melhor.

**Exercício 2 (médio)**  
Crie um array de `String` com pelo menos 5 nomes. Sem usar nenhum método pronto de ordenação (nada de `Arrays.sort`), escreva a lógica manualmente pra encontrar e imprimir o nome que vem primeiro em ordem alfabética (dica: `String` tem um método `.compareTo(String)` que retorna negativo/zero/positivo — pesquise o comportamento exato se não tiver certeza). Critério de pronto: funciona corretamente independente da ordem em que os nomes foram colocados no array original.

**Exercício 3 (difícil)**  
Crie uma matriz `int[][] tabuleiro` de 3 linhas por 3 colunas (como um tabuleiro de jogo da velha), preenchida com valores `0`. Usando dois `for` aninhados, preencha a diagonal principal (posições `[0][0]`, `[1][1]`, `[2][2]`) com o valor `1`, e a diagonal secundária (`[0][2]`, `[1][1]`, `[2][0]`) com o valor `2` — a posição `[1][1]` deve ficar com o valor da diagonal que for processada por último no seu código (decida qual e explique por comentário). Depois, imprima a matriz inteira formatada, linha por linha. Critério de pronto: a saída mostra visualmente as duas diagonais preenchidas corretamente.

**Exercício 4 (desafio)**  
Implemente, **sem usar `Arrays.sort` nem nenhum outro método pronto de ordenação**, o algoritmo _Bubble Sort_ pra ordenar um array de `int` em ordem crescente. Depois de ordenar manualmente, use `Arrays.equals()` pra comparar o resultado do seu algoritmo com o resultado de `Arrays.sort()` aplicado a uma cópia do array original, e imprima se os dois bateram (`true`/`false`). Critério de pronto: a comparação final imprime `true`, confirmando que sua implementação manual está correta. (Isso é só um aquecimento — o algoritmo em si, com análise de complexidade formal, será aprofundado na Trilha de DSA.)

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        int[] numeros = {12, 45, 3, 67, 21, 9};

        for (int i = 0; i < numeros.length; i++) {
            System.out.println("Índice " + i + ": " + numeros[i]);
        }

        int soma = 0;
        for (int numero : numeros) {
            soma += numero;
        }
        System.out.println("Soma: " + soma);
    }
}
```

Raciocínio: usei `for` tradicional na primeira parte porque a mensagem exige o índice explicitamente; usei `for-each` na segunda parte porque a soma só precisa do valor, não da posição — escolher a ferramenta certa pra cada necessidade, e não usar sempre a mesma por hábito.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        String[] nomes = {"Carla", "Ana", "Eduardo", "Bruno", "Daniela"};

        String primeiroAlfabeticamente = nomes[0]; // assume o primeiro como candidato inicial

        for (int i = 1; i < nomes.length; i++) {
            if (nomes[i].compareTo(primeiroAlfabeticamente) < 0) {
                // compareTo retorna negativo quando "nomes[i]" vem ANTES alfabeticamente
                primeiroAlfabeticamente = nomes[i];
            }
        }

        System.out.println("Primeiro em ordem alfabética: " + primeiroAlfabeticamente);
    }
}
```

Raciocínio: a lógica é a mesma de "encontrar o mínimo" num array numérico, só trocando `<` numérico por `.compareTo(...) < 0`. Comecei assumindo o primeiro elemento como candidato e só troco quando encontro algo que vem "antes" dele — dessa forma, um único loop resolve, sem precisar ordenar o array inteiro pra descobrir só o primeiro elemento (ordenar tudo seria desperdício de trabalho pra essa pergunta específica).

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        int[][] tabuleiro = new int[3][3]; // já nasce todo com 0 (valor padrão de int)

        // Diagonal principal: linha == coluna
        for (int i = 0; i < 3; i++) {
            tabuleiro[i][i] = 1;
        }

        // Diagonal secundária: coluna = (tamanho - 1 - linha)
        // Processada DEPOIS da principal, então [1][1] fica com valor 2 (a secundária "vence" o conflito)
        for (int i = 0; i < 3; i++) {
            tabuleiro[i][2 - i] = 2;
        }

        for (int i = 0; i < tabuleiro.length; i++) {
            for (int j = 0; j < tabuleiro[i].length; j++) {
                System.out.print(tabuleiro[i][j] + " ");
            }
            System.out.println();
        }
    }
}
```

Saída:

```
1 0 2 
0 2 0 
2 0 1
```

Raciocínio: a fórmula da diagonal secundária (`coluna = tamanho - 1 - linha`) é o ponto-chave — pra uma matriz 3x3, isso dá `(0,2)`, `(1,1)`, `(2,0)`, exatamente os pontos da diagonal que vai "de trás pra frente". Como a diagonal secundária roda por último no meu código, ela sobrescreve o `1` que estava em `[1][1]`, resultando em `2` — é uma decisão de ordem de execução, não uma regra fixa; se você tivesse invertido a ordem dos loops, o resultado em `[1][1]` seria `1`.

**Exercício 4**

java

```java
import java.util.Arrays;

public class Exercicio4 {
    public static void main(String[] args) {
        int[] original = {5, 2, 9, 1, 5, 6};
        int[] paraOrdenarManualmente = Arrays.copyOf(original, original.length);
        int[] paraOrdenarComSort = Arrays.copyOf(original, original.length);

        // Bubble Sort manual
        for (int i = 0; i < paraOrdenarManualmente.length - 1; i++) {
            for (int j = 0; j < paraOrdenarManualmente.length - 1 - i; j++) {
                if (paraOrdenarManualmente[j] > paraOrdenarManualmente[j + 1]) {
                    // troca (swap) usando variável auxiliar
                    int temp = paraOrdenarManualmente[j];
                    paraOrdenarManualmente[j] = paraOrdenarManualmente[j + 1];
                    paraOrdenarManualmente[j + 1] = temp;
                }
            }
        }

        Arrays.sort(paraOrdenarComSort);

        System.out.println("Manual: " + Arrays.toString(paraOrdenarManualmente));
        System.out.println("Arrays.sort: " + Arrays.toString(paraOrdenarComSort));
        System.out.println("Resultados batem? " + Arrays.equals(paraOrdenarManualmente, paraOrdenarComSort));
    }
}
```

Raciocínio: Bubble Sort funciona comparando pares adjacentes e trocando quando estão fora de ordem, "borbulhando" o maior valor pro final a cada passada externa — por isso o loop interno vai até `length - 1 - i`: a cada passada `i`, o último `i` elementos já estão garantidamente no lugar certo, então não precisam ser comparados de novo. Usei `Arrays.copyOf` duas vezes pra ter duas cópias independentes do array original — se eu tivesse ordenado o mesmo array duas vezes, a segunda ordenação não provaria nada, porque o array já estaria ordenado desde a primeira vez.