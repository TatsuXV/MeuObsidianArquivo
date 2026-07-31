### 1. Teoria

**O que é `String` em Java?**

`String` é uma classe (não um primitivo) que representa uma sequência de caracteres. Apesar de você usá-la o tempo todo com sintaxe de literal (`String nome = "Ana";`, parecendo um primitivo), por baixo é um objeto — e isso tem consequências importantes de comportamento que vamos destrinchar agora.

#### Imutabilidade

**A propriedade mais importante de `String`: uma vez criada, ela nunca muda.** Todo método que "parece" alterar uma `String` na verdade **retorna uma nova `String`**, deixando a original intacta.

java

```java
String original = "java";
original.toUpperCase(); // isso NÃO altera "original"
System.out.println(original); // ainda imprime "java", minúsculo

String maiuscula = original.toUpperCase(); // preciso capturar o RETORNO
System.out.println(maiuscula); // agora sim, "JAVA"
```

Isso é uma pegadinha constante pra quem vem de outras linguagens (ou não presta atenção): chamar o método sozinho, sem atribuir o resultado a algo, não tem efeito nenhum visível.

#### String Pool

Java otimiza `String` literais com uma área especial de memória chamada **String Pool**. Quando você escreve `String a = "java";`, o Java verifica se já existe uma `String` com esse conteúdo exato no pool — se existir, reaproveita a mesma referência, em vez de criar um objeto novo.

java

```java
String a = "java";
String b = "java";
System.out.println(a == b); // true! Ambos apontam pro MESMO objeto no pool

String c = new String("java"); // "new" força criação de objeto NOVO, fora do pool
System.out.println(a == c); // false — mesmo conteúdo, objetos DIFERENTES na memória
System.out.println(a.equals(c)); // true — mesmo conteúdo, é isso que .equals() compara
```

**Essa é a razão pela qual `==` nunca deve ser usado pra comparar conteúdo de `String`.** `==` compara referência (é o mesmo objeto na memória?); `.equals()` compara conteúdo (o texto é igual, caractere por caractere?). Como já vimos de relance em Conditionals, essa armadilha aparece direto na prática.

#### Métodos principais de `String`

java

```java
String texto = "  Olá, Mundo!  ";

texto.length()              // 15 — quantidade de caracteres, incluindo espaços
texto.trim()                 // remove espaços do início e fim (não do meio) → "Olá, Mundo!"
texto.strip()                 // similar ao trim(), mas Unicode-aware (mais moderno, Java 11+)
texto.toUpperCase()           // tudo maiúsculo
texto.toLowerCase()           // tudo minúsculo
texto.charAt(2)                // caractere na posição 2 (0-indexado)
texto.substring(2, 6)          // parte da String, do índice 2 (incluso) até 6 (exclusivo)
texto.indexOf("Mundo")         // posição onde "Mundo" começa, ou -1 se não encontrar
texto.contains("Olá")          // true/false, verifica se contém a sub-String
texto.replace("Mundo", "Java") // substitui todas as ocorrências
texto.split(",")               // divide em array de Strings, usando "," como delimitador
texto.isEmpty()                // true se length() == 0
texto.isBlank()                // true se vazia OU só espaços em branco (Java 11+)
texto.equals(outraString)      // compara conteúdo
texto.equalsIgnoreCase(outra)  // compara conteúdo, ignorando maiúsculo/minúsculo
```

#### Concatenação e `StringBuilder`

java

```java
String resultado = "a" + "b" + "c"; // concatenação simples, ok pra poucos casos
```

Cada `+` entre `String`s, dentro de um loop, **cria um objeto `String` novo a cada iteração** (porque `String` é imutável — não dá pra "modificar" a existente, então uma nova é criada toda vez). Em loops com muitas iterações, isso é ineficiente. A ferramenta certa pra concatenação repetida é `StringBuilder`, que é **mutável**:

java

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 5; i++) {
    sb.append(i).append(", "); // modifica o MESMO objeto internamente, sem criar String nova a cada passo
}
String resultado = sb.toString(); // converte pro resultado final quando terminar
```

#### Formatação de String

java

```java
String formatado = String.format("Nome: %s, Idade: %d", "Ana", 25);
// %s = String, %d = inteiro, %f = decimal (ex: %.2f para 2 casas decimais)

String comPrintf = "Preço: %.2f".formatted(19.9); // forma alternativa, Java 15+
```

**Onde isso aparece na prática (backend real)**

`String` é provavelmente o tipo mais usado em qualquer backend: parâmetros de request, mensagens de erro, queries SQL montadas dinamicamente (com os riscos de SQL Injection que isso implica, quando feito errado — algo que a Trilha de SQL e o uso correto de JDBC/JPA evitam), logs, serialização JSON. Entender imutabilidade evita bug sutil ("por que essa String não mudou?"), e saber quando trocar `+` por `StringBuilder` é o tipo de detalhe que aparece em code review de time sênior.

---

### 2. Exemplo de código comentado

java

```java
public class StringsExemplo {
    public static void main(String[] args) {

        // Imutabilidade — o método NÃO altera a String original
        String frase = "aprendendo java";
        frase.toUpperCase(); // resultado descartado, sem efeito nenhum
        System.out.println("Original inalterada: " + frase); // ainda minúsculo

        String fraseMaiuscula = frase.toUpperCase(); // agora sim, capturando o retorno
        System.out.println("Nova String: " + fraseMaiuscula);

        // String Pool vs new String()
        String a = "backend";
        String b = "backend";
        String c = new String("backend");

        System.out.println("a == b (pool, mesmo objeto): " + (a == b));       // true
        System.out.println("a == c (new, objeto diferente): " + (a == c));    // false
        System.out.println("a.equals(c) (conteúdo igual): " + a.equals(c));   // true

        // Métodos principais
        String textoComEspacos = "  Java é ótimo  ";
        System.out.println("Trim: [" + textoComEspacos.trim() + "]");
        System.out.println("Length original: " + textoComEspacos.length());
        System.out.println("Substring(2, 6): " + textoComEspacos.substring(2, 6)); // "Java"
        System.out.println("IndexOf 'ótimo': " + textoComEspacos.indexOf("ótimo"));
        System.out.println("Replace: " + textoComEspacos.replace("ótimo", "poderoso"));

        // Split — muito comum pra processar entrada tipo CSV
        String csv = "Ana,25,Brasília";
        String[] partes = csv.split(",");
        for (String parte : partes) {
            System.out.println("Parte: " + parte);
        }

        // Concatenação simples vs StringBuilder em loop
        String concatenado = "";
        for (int i = 0; i < 5; i++) {
            concatenado = concatenado + i; // cria uma String NOVA a cada iteração — ineficiente em escala
        }
        System.out.println("Concatenado com +: " + concatenado);

        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 5; i++) {
            sb.append(i); // modifica o MESMO objeto internamente — eficiente
        }
        System.out.println("Concatenado com StringBuilder: " + sb.toString());

        // Formatação
        String mensagem = String.format("Usuário: %s | Idade: %d | Saldo: R$%.2f", "Carlos", 30, 1500.5);
        System.out.println(mensagem);
    }
}
```

---

### 3. Armadilhas comuns

1. **Chamar um método de `String` e esperar que ele altere a variável original.** `texto.trim();` sozinho não faz nada visível — é preciso `texto = texto.trim();` pra capturar o resultado, porque `String` é imutável.
2. **Comparar `String` com `==` em vez de `.equals()`.** Funciona "por acidente" com literais (por causa do String Pool), mas quebra silenciosamente com `new String(...)` ou `String`s vindas de processamento (como `.substring()`, `.concat()`, entrada de usuário) — que raramente vêm do pool.
3. **Erro de índice em `substring()`.** `texto.substring(2, 6)` pega do índice 2 (incluso) até 6 (**exclusivo**) — é fácil errar por um a mais ou a menos, já que o segundo parâmetro não é "o último índice incluído", é "onde parar antes de incluir".
4. **Concatenar String dentro de loop grande com `+`.** Funciona, mas cria um objeto novo a cada iteração — em loops de milhares/milhões de iterações isso vira problema real de performance. `StringBuilder` resolve isso sendo mutável internamente.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Declare uma `String frase = " Programação em Java é excelente ";`. Sem alterar a variável original em nenhum momento (ou seja, sempre capturando em novas variáveis), imprima: a frase com `trim()` aplicado, a mesma frase toda em maiúsculas, e o `length()` da frase original (com os espaços). Critério de pronto: a `String` original impressa por último ainda mostra os espaços intactos, provando a imutabilidade.

**Exercício 2 (médio)**  
Declare `String email = "usuario@dominio.com";`. Usando `indexOf` e `substring`, extraia e imprima separadamente a parte antes do `@` (o "usuário") e a parte depois do `@` (o "domínio"), sem usar `split` (pratique com `indexOf`/`substring` mesmo, já que `split` você já viu no exemplo). Critério de pronto: imprime `"usuario"` numa linha e `"dominio.com"` em outra, calculado dinamicamente (funcionaria com outro e-mail também, não só hardcoded pra esse valor específico).

**Exercício 3 (difícil)**  
Declare um array `String[] palavras = {"Java", "é", "uma", "linguagem", "robusta"};`. Usando `StringBuilder` (não use `+` de String), monte uma única frase juntando todas as palavras separadas por espaço, mas **sem espaço extra no final** (dica: você pode checar se é a última posição do array antes de adicionar o espaço, ou adicionar o espaço só antes de cada palavra exceto a primeira). Imprima o resultado final via `.toString()`. Critério de pronto: a saída é exatamente `"Java é uma linguagem robusta"`, sem espaço sobrando no início ou fim.

**Exercício 4 (desafio)**  
Escreva um programa que receba uma `String frase = "arara";` fixa e determine, **sem usar nenhum método pronto de inversão de String** (nada de reverse de biblioteca), se ela é um **palíndromo** (lê-se igual de trás pra frente). Implemente manualmente comparando caracteres (dica: `charAt(i)` te dá acesso a um caractere específico; pense em comparar posições simétricas: primeira com última, segunda com penúltima, etc., até o meio). Teste também com `"java"` (não é palíndromo) pra confirmar que sua lógica identifica os dois casos corretamente. Critério de pronto: imprime `true` pra "arara" e `false` pra "java", com a lógica de comparação feita manualmente, sem usar `StringBuilder.reverse()` ou similar.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        String frase = "  Programação em Java é excelente  ";

        String semEspacos = frase.trim();
        System.out.println("Trim: [" + semEspacos + "]");

        String maiuscula = frase.toUpperCase();
        System.out.println("Maiúscula: " + maiuscula);

        System.out.println("Length original: " + frase.length());
        System.out.println("Original ainda intacta: [" + frase + "]");
    }
}
```

Raciocínio: cada método (`trim()`, `toUpperCase()`) é chamado sobre `frase`, mas o resultado é sempre capturado numa variável nova (`semEspacos`, `maiuscula`) — a variável `frase` original nunca é reatribuída, então continua com os espaços intactos até o final do programa, demonstrando a imutabilidade na prática.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        String email = "usuario@dominio.com";

        int posicaoArroba = email.indexOf("@");

        String usuario = email.substring(0, posicaoArroba);
        String dominio = email.substring(posicaoArroba + 1); // substring com 1 argumento vai até o fim da String

        System.out.println(usuario);
        System.out.println(dominio);
    }
}
```

Raciocínio: `indexOf("@")` encontra dinamicamente a posição do `@`, então a lógica funciona pra qualquer e-mail, não só pra esse valor fixo. `substring(0, posicaoArroba)` pega tudo antes do `@` (o índice do `@` é exclusivo, então ele mesmo não entra). `substring(posicaoArroba + 1)` usa a versão de um único argumento, que pega da posição indicada até o final da String — pulei o índice do `@` somando `+1`, pra não incluir o próprio `@` no resultado do domínio.

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        String[] palavras = {"Java", "é", "uma", "linguagem", "robusta"};

        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < palavras.length; i++) {
            if (i > 0) {
                sb.append(" "); // só adiciona espaço ANTES de palavras que não são a primeira
            }
            sb.append(palavras[i]);
        }

        System.out.println(sb.toString());
    }
}
```

Raciocínio: a condição `if (i > 0)` é o que evita o espaço extra — em vez de adicionar espaço depois de cada palavra (o que deixaria um espaço sobrando no final), adiciono o espaço **antes** de cada palavra, exceto a primeira (índice 0). Essa é a abordagem mais comum pra esse tipo de problema de "separador entre itens, sem sobra nas pontas".

**Exercício 4**

java

```java
public class Exercicio4 {
    public static void main(String[] args) {
        System.out.println(ehPalindromo("arara")); // true
        System.out.println(ehPalindromo("java"));   // false
    }

    static boolean ehPalindromo(String texto) {
        int inicio = 0;
        int fim = texto.length() - 1;

        while (inicio < fim) {
            if (texto.charAt(inicio) != texto.charAt(fim)) {
                return false; // encontrou par de caracteres diferentes — não é palíndromo, para aqui
            }
            inicio++;
            fim--;
        }

        return true; // percorreu tudo sem achar diferença — é palíndromo
    }
}
```

Raciocínio: uso dois "ponteiros" (índices `inicio` e `fim`) que se movem em direção ao centro da String simultaneamente — `inicio` avança da esquerda pra direita, `fim` recua da direita pra esquerda. Comparo `charAt(inicio)` com `charAt(fim)` a cada passo: se algum par não bater, já sei que não é palíndromo e posso retornar `false` imediatamente, sem precisar checar o resto. Se o loop terminar inteiro sem nenhuma diferença (quando `inicio` e `fim` se cruzam ou se encontram no meio), a String é palíndromo. Essa técnica de "dois ponteiros" é, inclusive, um dos padrões que você vai formalizar na Trilha 2 (DSA) — aqui já é uma prévia natural do conceito.