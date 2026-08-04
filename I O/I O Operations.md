### 1. Teoria

#### O que é I/O em Java

I/O (Input/Output) é como seu programa troca dados com o mundo fora da JVM: teclado, tela, arquivos, rede, memória. Em Java isso é modelado através de **streams** — um fluxo sequencial de dados que você lê (input) ou escreve (output), um pedaço de cada vez, sem precisar carregar tudo na memória de uma vez.

Hoje o foco é o modelo de streams do pacote `java.io` (o "clássico"). **File Operations** — a API `Path`/`Files` mais moderna (`java.nio.file`) — é o próximo item do bloco, então vou citar `File`/`Path` só quando for inevitável para contextualizar, sem entrar em detalhe.

#### As duas hierarquias

Java separa I/O em dois grandes grupos, dependendo do tipo de dado:

**Streams de byte** — para dados binários (imagens, PDFs, qualquer arquivo não-texto):

- `InputStream` (abstrata) → leitura de bytes
- `OutputStream` (abstrata) → escrita de bytes

**Streams de caractere** — para texto, já lidando com encoding (charset):

- `Reader` (abstrata) → leitura de caracteres
- `Writer` (abstrata) → escrita de caracteres

Essa separação existe porque texto não é só "bytes crus" — depende de um charset (UTF-8, ISO-8859-1, etc.) pra converter byte em caractere corretamente. Se você usa stream de byte pra ler texto, você é responsável por fazer essa conversão manualmente; se usa `Reader`/`Writer`, a conversão já é tratada pra você.

#### O padrão Decorator (isso é o que mais confunde iniciante)

As classes de I/O em Java são combináveis: você pega uma stream "crua" (que sabe ler de um arquivo, por exemplo) e **envolve** ela com outra stream que adiciona um comportamento (buffer, conversão de encoding, etc.), sem que a classe original precise saber disso. Isso é o padrão de projeto _Decorator_.

Exemplo mental: `FileInputStream` sabe ler bytes de um arquivo, um byte de cada vez — lento se você chamar `read()` byte a byte, porque cada chamada pode envolver uma operação de sistema. Você então "decora" ela com `BufferedInputStream`, que lê um bloco grande de uma vez pra memória e entrega os bytes daí — muito mais rápido.

Por isso você frequentemente vê construções encadeadas como:

java

```java
new BufferedReader(new InputStreamReader(new FileInputStream("arquivo.txt")))
```

Cada camada adiciona uma responsabilidade: `FileInputStream` lê bytes do arquivo → `InputStreamReader` converte bytes em caracteres (usando um charset) → `BufferedReader` adiciona buffer e o método conveniente `readLine()`.

#### Fechamento de recursos: try-with-resources

Toda stream que você abre precisa ser fechada — senão você vaza handle de arquivo/socket no sistema operacional. Antes do Java 7 isso exigia `try/finally` manual e verboso. Desde o Java 7, o try-with-resources fecha automaticamente qualquer recurso que implemente `AutoCloseable` (interface que `Closeable`, implementada por praticamente toda stream, estende), simplificando o uso correto de recursos que implementam java.lang.AutoCloseable, incluindo os que implementam java.io.Closeable. A partir do Java 9, se a variável do recurso já for final ou efetivamente final, você nem precisa redeclará-la dentro do `try` — pode referenciar uma variável já existente. [educative](https://www.educative.io/answers/what-is-try-with-resources-in-java)

java

```java
try (BufferedReader br = new BufferedReader(new FileReader("arquivo.txt"))) {
    // usa br
} // fechado automaticamente aqui, mesmo se der exceção
```

#### `System.in`, `System.out`, `System.err`

Três streams padrão sempre disponíveis, herdadas do processo do SO:

- `System.in` — um `InputStream` (entrada padrão, ex: teclado)
- `System.out` / `System.err` — `PrintStream` (que estende `OutputStream`), pra saída padrão e saída de erro

#### Onde isso aparece num backend real

Você raramente lê arquivo de texto na mão num Spring Boot moderno (isso costuma ser resolvido por `Resource`/`ClassPathResource` do Spring, ou bibliotecas específicas). Mas o conceito de stream aparece o tempo todo por baixo: upload de arquivo em endpoint REST (`MultipartFile` do Spring encapsula um `InputStream`), leitura de resposta de uma chamada HTTP externa, processamento de CSV/JSON grande sem estourar memória, leitura de arquivo de configuração, logging (que no fundo escreve numa stream). Entender o modelo de stream é o que te permite não travar quando um desses componentes não abstrai tudo pra você.

### 2. Exemplo de código comentado

java

```java
import java.io.*;

public class LeituraDeArquivo {

    public static void main(String[] args) {
        String caminho = "dados.txt";

        // try-with-resources: br é fechado automaticamente ao sair do bloco,
        // mesmo se ocorrer exceção durante a leitura.
        try (BufferedReader br = new BufferedReader(new FileReader(caminho))) {

            String linha;
            // readLine() retorna null quando chega ao fim do arquivo (EOF)
            while ((linha = br.readLine()) != null) {
                System.out.println("Lida: " + linha);
            }

        } catch (FileNotFoundException e) {
            // Subclasse de IOException, específica para arquivo inexistente.
            // Pegar ela separadamente permite mensagem de erro mais precisa.
            System.err.println("Arquivo não encontrado: " + caminho);

        } catch (IOException e) {
            // Qualquer outro erro de I/O (permissão, disco cheio, etc.)
            System.err.println("Erro ao ler o arquivo: " + e.getMessage());
        }
    }
}
```

E o inverso, escrevendo:

java

```java
import java.io.*;

public class EscritaEmArquivo {

    public static void main(String[] args) {
        // BufferedWriter decora FileWriter, adicionando buffer de escrita
        // (evita uma operação de I/O física a cada chamada de write).
        try (BufferedWriter bw = new BufferedWriter(new FileWriter("saida.txt"))) {

            bw.write("Primeira linha");
            bw.newLine(); // quebra de linha portável entre sistemas operacionais
            bw.write("Segunda linha");
            // Não precisa chamar flush() ou close() manualmente:
            // o try-with-resources cuida disso ao sair do bloco.

        } catch (IOException e) {
            System.err.println("Erro ao escrever: " + e.getMessage());
        }
    }
}
```

### 3. Armadilhas comuns

1. **Esquecer de fechar a stream (ou fechar na ordem errada manualmente).** Antes de conhecer try-with-resources, é comum abrir stream em `try` e esquecer o `close()` no `finally`, causando vazamento de recursos do SO. Solução: sempre use try-with-resources quando a classe implementar `AutoCloseable`.
2. **Ler byte a byte (ou linha a linha) sem buffer.** Chamar `read()` diretamente num `FileInputStream` sem envolver em `BufferedInputStream` funciona, mas é lento — cada leitura pode custar uma chamada de sistema. Sempre que for ler algo maior que trivial, decore com buffer.
3. **Misturar stream de byte com stream de caractere sem querer.** Usar `FileInputStream` para ler texto diretamente (sem passar por `InputStreamReader`) obriga você a converter bytes em `String` manualmente e lidar com encoding "na unha" — fonte comum de texto corrompido com acento (`á`, `ç`, etc.) quando o encoding usado não bate com o do arquivo.
4. **Confundir `IOException` genérica com o motivo real do erro.** Capturar só `catch (IOException e)` e mostrar uma mensagem genérica dificulta debug. Vale diferenciar subclasses relevantes (como `FileNotFoundException`) quando o tratamento precisar ser diferente.

### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Escreva um programa que leia um arquivo de texto `entrada.txt` (crie esse arquivo você mesmo com 3-4 linhas de qualquer conteúdo) e imprima cada linha no console, precedida do número da linha. Exemplo de saída:

```
1: primeira linha do arquivo
2: segunda linha do arquivo
```

Critério de pronto: roda sem erro com o arquivo existindo, e trata graciosamente (sem stack trace cru) o caso do arquivo não existir.

**Exercício 2 (Médio)**  
Escreva um programa que leia `entrada.txt` e escreva em `saida.txt` apenas as linhas que **não estão vazias**, removendo espaços em branco extras no início/fim de cada linha (`trim()`). Use try-with-resources para os dois recursos (leitura e escrita) no mesmo bloco `try`.  
Critério de pronto: `saida.txt` é gerado corretamente e nenhuma linha em branco do original aparece nele.

**Exercício 3 (Difícil)**  
Escreva um programa que conte quantas palavras existem em `entrada.txt` (considere "palavra" como qualquer sequência de caracteres separada por espaço em branco) e quantas linhas têm mais de 5 palavras. Ao final, imprima um resumo:

```
Total de linhas: X
Total de palavras: Y
Linhas com mais de 5 palavras: Z
```

Critério de pronto: os números batem manualmente conferindo um arquivo de teste pequeno que você mesmo cria.

**Exercício 4 (Desafio)**  
Escreva um programa que copie um arquivo binário (por exemplo, uma imagem `.jpg` ou `.png` pequena) de um caminho para outro, usando `InputStream`/`OutputStream` de byte (não `Reader`/`Writer` — o arquivo não é texto). Implemente a cópia lendo em blocos (não byte a byte, e não carregando o arquivo inteiro na memória de uma vez com `readAllBytes()` — o objetivo aqui é praticar leitura em blocos manual com buffer).  
Critério de pronto: o arquivo copiado abre normalmente e tem o mesmo tamanho em bytes que o original.

### 5. Gabarito comentado

#### Exercício 1 (Fácil)

java

```java
import java.io.*;

public class Exercicio1 {

    public static void main(String[] args) {
        String caminho = "entrada.txt";

        try (BufferedReader br = new BufferedReader(new FileReader(caminho))) {

            String linha;
            int numeroLinha = 1;

            while ((linha = br.readLine()) != null) {
                System.out.println(numeroLinha + ": " + linha);
                numeroLinha++;
            }

        } catch (FileNotFoundException e) {
            System.err.println("Arquivo não encontrado: " + caminho);
        } catch (IOException e) {
            System.err.println("Erro ao ler o arquivo: " + e.getMessage());
        }
    }
}
```

**Raciocínio:** `readLine()` retorna `null` quando chega ao fim do arquivo — esse é o sinal de parada do loop, não uma exceção. Um contador manual (`numeroLinha`) resolve a numeração sem precisar de nenhuma estrutura extra. Separar `FileNotFoundException` de `IOException` genérica é o que o critério de "tratar graciosamente" pede: sem isso, um arquivo ausente gera stack trace cru no console em vez de mensagem clara.

---

#### Exercício 2 (Médio)

java

```java
import java.io.*;

public class Exercicio2 {

    public static void main(String[] args) {
        String entrada = "entrada.txt";
        String saida = "saida.txt";

        // Dois recursos no mesmo try-with-resources, separados por ';'.
        // Ambos são fechados automaticamente, na ordem inversa da declaração.
        try (BufferedReader br = new BufferedReader(new FileReader(entrada));
             BufferedWriter bw = new BufferedWriter(new FileWriter(saida))) {

            String linha;
            while ((linha = br.readLine()) != null) {
                String linhaLimpa = linha.trim();

                if (!linhaLimpa.isEmpty()) {
                    bw.write(linhaLimpa);
                    bw.newLine();
                }
            }

            System.out.println("Arquivo processado com sucesso.");

        } catch (IOException e) {
            System.err.println("Erro ao processar arquivos: " + e.getMessage());
        }
    }
}
```

**Raciocínio:** o try-with-resources aceita múltiplos recursos separados por `;` — não precisa de um `try` aninhado dentro do outro. Isso importa porque, se você abrisse os dois recursos em `try`s separados (um dentro do outro) e a abertura do segundo falhasse, teria que garantir manualmente que o primeiro também fosse fechado; com múltiplos recursos no mesmo `try`, a JVM cuida disso, fechando na ordem inversa (o último aberto é o primeiro fechado). `trim()` remove espaço em branco das pontas, e o teste `isEmpty()` depois do trim (não antes) é o que garante que uma linha só com espaços também seja tratada como vazia.

---

#### Exercício 3 (Difícil)

java

```java
import java.io.*;

public class Exercicio3 {

    public static void main(String[] args) {
        String caminho = "entrada.txt";

        int totalLinhas = 0;
        int totalPalavras = 0;
        int linhasComMaisDe5Palavras = 0;

        try (BufferedReader br = new BufferedReader(new FileReader(caminho))) {

            String linha;
            while ((linha = br.readLine()) != null) {
                totalLinhas++;

                String linhaLimpa = linha.trim();
                if (linhaLimpa.isEmpty()) {
                    continue; // linha vazia não tem palavra nenhuma
                }

                // split por um ou mais espaços em branco (\\s+)
                String[] palavras = linhaLimpa.split("\\s+");
                int quantidadeNestaLinha = palavras.length;

                totalPalavras += quantidadeNestaLinha;

                if (quantidadeNestaLinha > 5) {
                    linhasComMaisDe5Palavras++;
                }
            }

            System.out.println("Total de linhas: " + totalLinhas);
            System.out.println("Total de palavras: " + totalPalavras);
            System.out.println("Linhas com mais de 5 palavras: " + linhasComMaisDe5Palavras);

        } catch (IOException e) {
            System.err.println("Erro ao processar arquivo: " + e.getMessage());
        }
    }
}
```

**Raciocínio:** o ponto de atenção aqui é o `split("\\s+")` em vez de `split(" ")` — se a linha tiver mais de um espaço entre palavras (comum em texto digitado por humano), `split(" ")` geraria strings vazias no array, inflando a contagem. `\\s+` trata qualquer sequência de espaços/tabs como um único separador. Também é importante fazer `trim()` antes do `split`: se a linha começar ou terminar com espaço, `split` pode gerar um elemento vazio na primeira posição do array.

**Alternativa:** dá pra fazer isso com Stream API (`Files.lines(...).flatMap(...)`), mas isso é conteúdo do Bloco 12 (Programação Funcional) e do próximo item (File Operations com `java.nio.file`) — fica pra quando você chegar lá, a versão com `BufferedReader` é a correta pro escopo de hoje.

---

#### Exercício 4 (Desafio)

java

```java
import java.io.*;

public class Exercicio4 {

    public static void main(String[] args) {
        String origem = "foto.jpg";
        String destino = "foto_copia.jpg";

        // FileInputStream/FileOutputStream trabalham com bytes brutos —
        // corretos aqui porque o arquivo não é texto.
        try (InputStream in = new FileInputStream(origem);
             OutputStream out = new FileOutputStream(destino)) {

            byte[] buffer = new byte[4096]; // bloco de 4KB por leitura
            int bytesLidos;

            // read(buffer) devolve quantos bytes realmente leu nessa chamada,
            // ou -1 quando chega ao fim do arquivo.
            while ((bytesLidos = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesLidos);
            }

            System.out.println("Arquivo copiado com sucesso.");

        } catch (IOException e) {
            System.err.println("Erro ao copiar arquivo: " + e.getMessage());
        }
    }
}
```

**Raciocínio:** a parte que costuma confundir é o `out.write(buffer, 0, bytesLidos)` — não pode ser `out.write(buffer)` sozinho. Isso porque a última leitura do arquivo normalmente preenche só uma parte do buffer (ex: arquivo tem 4100 bytes, buffer é 4096 — na segunda iteração só 4 bytes são lidos, mas o array `buffer` ainda tem 4096 posições, as outras 4092 são "lixo" da leitura anterior). Escrever `buffer` inteiro sem o parâmetro de tamanho gravaria esse lixo no arquivo de destino, corrompendo os últimos bytes. Passar `bytesLidos` como terceiro argumento garante que só os bytes realmente lidos nessa iteração sejam escritos.

**Por que não usar `readAllBytes()`:** existe (desde o `InputStream` moderno), mas carrega o arquivo inteiro na memória de uma vez — funciona bem para arquivo pequeno, mas não escala para arquivo grande (um vídeo de 2GB estouraria memória). Ler em blocos, como fizemos aqui, é o padrão que se sustenta em produção independente do tamanho do arquivo — por isso pedi explicitamente essa abordagem no enunciado.