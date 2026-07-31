**O que é "sintaxe" em Java?**

Sintaxe é o conjunto de regras que o compilador exige pra aceitar seu código como um programa Java válido — independente de o programa fazer sentido lógico ou não. Você pode escrever algo com lógica perfeita, mas se violar a sintaxe, o compilador nem chega a tentar entender a lógica: ele já rejeita antes.

**Estrutura mínima de um programa Java**

Todo arquivo `.java` que você for executar diretamente segue este esqueleto:

java

```java
public class NomeDaClasse {
    public static void main(String[] args) {
        // código vai aqui
    }
}
```

Alguns pontos estruturais importantes:

- **Nome do arquivo = nome da classe pública.** Se a classe se chama `NomeDaClasse`, o arquivo _precisa_ se chamar `NomeDaClasse.java`. Isso não é convenção, é regra do compilador — só existe uma exceção rara (classes locais/anônimas dentro de outro arquivo, algo que você não vai usar agora).
- **`main` é o ponto de entrada.** Quando você roda `java NomeDaClasse`, a JVM procura especificamente um método com essa assinatura exata: `public static void main(String[] args)`. Se a assinatura estiver diferente (por exemplo, sem `static`, ou com letra maiúscula `Main`), a JVM simplesmente não encontra o ponto de entrada.
- **Blocos de código** ficam entre `{ }`. Toda chave aberta precisa de uma fechada correspondente — e a indentação (embora não seja exigida pelo compilador) é o que te ajuda a não se perder em blocos aninhados.
- **Cada instrução (statement) termina com `;`**. Isso é diferente de linguagens como Python, onde a quebra de linha já delimita o comando.
- **Java é case-sensitive.** `String`, `string` e `STRING` são três identificadores diferentes pro compilador.

**Comentários**

Existem três formas:

java

```java
// comentário de uma linha

/* comentário
   de várias linhas */

/**
 * Javadoc — gera documentação e aparece no autocomplete da IDE.
 * Usado geralmente acima de classes e métodos públicos.
 */
```

**Identificadores (nomes de variáveis, métodos, classes)**

Regras do compilador (não convenção, regra mesmo):

- Podem conter letras, dígitos, `_` e `$`.
- Não podem começar com dígito.
- Não podem ser uma _palavra reservada_ (`class`, `public`, `int`, `if`, etc.).

Convenção da indústria (o compilador aceita se você não seguir, mas ninguém em nenhuma empresa vai aceitar seu código assim):

- Classes: `PascalCase` (`ClienteRepository`).
- Variáveis e métodos: `camelCase` (`nomeCliente`, `calcularTotal()`).
- Constantes: `UPPER_SNAKE_CASE` (`TAXA_MAXIMA`).

**Onde isso aparece na prática (backend real)**

Toda classe Spring Boot que você vai escrever — um `@RestController`, um `@Service`, uma entidade JPA — usa exatamente essa estrutura de base (`public class`, chaves, ponto e vírgula, convenção de nomes). As anotações do Spring (`@Service`, `@Autowired`) são só "decoração" em cima dessa sintaxe fundamental. Se a base não estiver sólida, qualquer erro de sintaxe vai te fazer perder tempo tentando entender "por que o Spring não sobe", quando na verdade é um erro de estrutura básica do arquivo.

---

### 2. Exemplo de código comentado

java

```java
// Arquivo: BoasVindas.java
// O nome do arquivo TEM que ser igual ao nome da classe pública abaixo.

public class BoasVindas {

    // Ponto de entrada do programa. A JVM procura exatamente esta assinatura.
    public static void main(String[] args) {

        // println adiciona quebra de linha no final; print não adiciona.
        System.out.println("Bem-vindo ao estudo de Java!");
        System.out.print("Este texto ");
        System.out.println("fica na mesma linha do anterior.");

        // Escape sequences dentro de String literal
        System.out.println("Ele disse: \"vamos programar\"."); // aspas escapadas
        System.out.println("Colunas:\tAlinhadas\tcom Tab");     // \t = tab
        System.out.println("Quebra\nde\nlinha manual");          // \n = nova linha

        /*
         * Bloco de comentário de múltiplas linhas.
         * Útil pra explicar trechos maiores de raciocínio.
         */
        int idade = 25; // declaração de variável — tipo será aprofundado no próximo tópico
        System.out.println("Idade: " + idade); // concatenação de String com int
    } // fecha o main

} // fecha a classe — toda chave aberta tem uma fechada correspondente
```

---

### 3. Armadilhas comuns

1. **Nome do arquivo ≠ nome da classe pública.** Se você chamar o arquivo de `Main.java` mas a classe for `public class Teste`, o compilador rejeita com `class Teste is public, should be declared in a file named Teste.java`.
2. **Esquecer o `;` no final da instrução.** É o erro sintático mais comum de quem está começando — e a mensagem de erro do compilador às vezes aponta pra linha _seguinte_, o que confunde iniciante.
3. **Confundir `main` com `Main`, ou esquecer `static`.** Sem a assinatura exata (`public static void main(String[] args)`), o programa compila mas não roda — erro tipo "Main method not found".
4. **Chaves desbalanceadas.** Esquecer de fechar uma `{` gera uma cascata de erros de compilação que parecem não ter relação com a causa real — sempre confira o balanceamento primeiro quando o compilador cuspir uma lista grande de erros estranhos.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Crie um programa Java que imprima seu nome e, na linha seguinte, a frase "Estou aprendendo Java". Adicione um comentário de uma linha acima do `main` explicando o que o programa faz. Critério de pronto: compila sem erro e imprime exatamente 2 linhas.

**Exercício 2 (médio)**  
Crie um programa que use `System.out.print` e `System.out.println` combinados para montar, na saída do console, a seguinte formatação exata (preste atenção em tabs e quebras):

```
Nome:	João
Idade:	30
Cidade:	"Brasília"
```

(As aspas em `"Brasília"` devem aparecer literalmente no output — isso exige escape sequence.) Critério de pronto: output bate exatamente com o esperado, incluindo o tab entre rótulo e valor.

**Exercício 3 (difícil)**  
Crie um arquivo com **duas classes**: uma `public class Programa` (que vai conter o `main`) e uma segunda classe `Ajudante` (sem `public`) no mesmo arquivo. Dentro do `main`, imprima uma mensagem fixa e, depois, chame um método qualquer da classe `Ajudante` que também imprima algo (pode ser um método `static` simples, mesmo sem entender ainda todo o resto de OOP — o objetivo aqui é só entender a regra sintática de "só uma classe pública por arquivo, e o nome do arquivo segue a pública"). Critério de pronto: compila com as duas classes no mesmo arquivo, sem erro de nomeação.

**Exercício 4 (desafio)**  
Abaixo está um trecho de código com **5 erros de sintaxe** intencionais. Sem rodar no compilador ainda, identifique cada um dos 5 erros por escrito (linha e motivo), e só depois disso, corrija o código:

java

```java
public class saudacao {
    public static void Main(String[] args) {
        String nome = "Ana"
        System.out.println("Olá, " + nome + "!");
        System.out.println("Bem-vinda ao curso.")
    
}
```

Critério de pronto: você lista os 5 erros com a explicação de por que cada um quebra a compilação, antes de simplesmente colar a versão corrigida.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
// Programa simples de apresentação pessoal
public class Apresentacao {
    public static void main(String[] args) {
        System.out.println("Maria");
        System.out.println("Estou aprendendo Java");
    }
}
```

Raciocínio: nada além do esqueleto básico + duas chamadas de `println`, já que cada uma quebra linha automaticamente — não precisa de `\n` manual aqui.

**Exercício 2**

java

```java
public class DadosPessoais {
    public static void main(String[] args) {
        System.out.println("Nome:\tJoão");
        System.out.println("Idade:\t30");
        System.out.println("Cidade:\t\"Brasília\"");
    }
}
```

Raciocínio: `\t` insere um caractere de tabulação real (não é "quatro espaços", é um caractere só, que o terminal renderiza como alinhamento). As aspas em `"Brasília"` precisam ser escapadas com `\"` porque aspas duplas sem escape fechariam a String literal antes da hora — o compilador entenderia `"Cidade:\tBrasília` e `"` como dois tokens quebrados, gerando erro.

**Exercício 3**

java

```java
public class Programa {
    public static void main(String[] args) {
        System.out.println("Início do programa");
        Ajudante.mostrarMensagem();
    }
}

class Ajudante {
    static void mostrarMensagem() {
        System.out.println("Mensagem vinda da classe Ajudante");
    }
}
```

Raciocínio: a regra é "no máximo uma classe `public` por arquivo, e o nome do arquivo segue essa classe pública" — mas você pode ter quantas classes _não-públicas_ quiser no mesmo arquivo. Chamei o método com `Ajudante.mostrarMensagem()` porque ele é `static`: não precisei criar um objeto da classe pra usá-lo (isso será aprofundado quando chegarmos em OOP/Static Keyword — aqui é só pra você ver a regra de arquivo/classe funcionando, não pra dominar o conceito de static ainda).

**Exercício 4 — os 5 erros:**

1. `public class saudacao` — nome de classe deveria seguir PascalCase (`Saudacao`), mas isso é _convenção_, não erro de compilação por si só. O erro real de compilação está ligado a isso: se o arquivo se chama `saudacao.java` (minúsculo) mas depois você declarar `Saudacao` com S maiúsculo, o nome bate errado com o arquivo — dependendo de como o arquivo foi salvo, dá erro de "class não encontrada" ao rodar. Trato como ponto de atenção mesmo não sendo, isoladamente, erro de sintaxe.
2. `public static void Main(String[] args)` — `Main` com M maiúsculo. Java é case-sensitive; a JVM procura `main`, minúsculo. Isso compila (é um método válido), mas a JVM não acha o ponto de entrada ao rodar — erro em tempo de execução, não de compilação.
3. `String nome = "Ana"` — falta o `;` no final da linha. Erro de compilação direto.
4. `System.out.println("Bem-vinda ao curso.")` — mesma coisa, falta `;` no final.
5. Chave de fechamento do método `main` está faltando — só a chave da classe (`}`) foi fechada no final, mas o bloco do `main` nunca fecha. Isso gera erro de "reached end of file while parsing".

**Versão corrigida:**

java

```java
public class Saudacao {
    public static void main(String[] args) {
        String nome = "Ana";
        System.out.println("Olá, " + nome + "!");
        System.out.println("Bem-vinda ao curso.");
    }
}
```