### 1. Teoria

#### Onde isso se encaixa (diferença do que vimos em I/O Operations)

Em I/O Operations o foco era **conteúdo**: ler e escrever dados através de streams, byte a byte ou caractere a caractere. Em File Operations o foco é o **sistema de arquivos em si**: existe esse arquivo? é diretório ou arquivo? posso criar, mover, deletar, listar o conteúdo de uma pasta? quais são os metadados (tamanho, data de modificação, permissões)? São perguntas que uma stream sozinha não responde — você precisa de uma API que converse com o sistema operacional sobre a estrutura de arquivos, não só sobre o conteúdo de um arquivo específico.

#### Duas APIs — e por que você precisa saber das duas

**`java.io.File`** — a API original, disponível desde o começo do Java. Funciona, mas tem limitações conhecidas: métodos que retornam `boolean` em vez de lançar exceção quando algo falha (dificultando saber _por que_ falhou), suporte fraco a links simbólicos, e API pouco intuitiva pra operações comuns como copiar um arquivo (não existe `File.copy()`).

**`java.nio.file` (Path + Files)** — pacote introduzido no Java 7 (NIO.2), que oferece uma API mais moderna e eficiente para trabalhar com arquivos, diretórios e sistemas de arquivos, substituindo várias limitações da antiga classe java.io.File. Essa é a que você deve usar em código novo. Os dois pilares dela: [GeeksforGeeks](https://www.geeksforgeeks.org/java/read-and-write-files-using-the-new-i-o-nio-2-api-in-java/)

- **`Path`** — representa um caminho no sistema de arquivos (arquivo ou diretório), de forma imutável e independente de sistema operacional (`/` no Linux/Mac, `\` no Windows — `Path` abstrai isso pra você).
- **`Files`** — classe utilitária com métodos estáticos que operam sobre um `Path`: `Files.exists()`, `Files.createFile()`, `Files.delete()`, `Files.copy()`, `Files.move()`, `Files.readAllLines()`, `Files.write()`, `Files.list()` (lista o conteúdo de um diretório), entre outros.

Você ainda vai encontrar `File` em código legado (é bem comum em projeto antigo, e algumas APIs mais velhas do próprio Java ainda pedem `File` como parâmetro), então reconhecer não é opcional — mas para código que você escreve do zero hoje, `Path`/`Files` é o padrão.

#### Diferença de filosofia de erro

`File` tende a devolver `false`/`null` quando algo dá errado (`file.delete()` retorna `false` se falhar, sem dizer o motivo). `Files` tende a lançar exceção específica (`Files.delete()` lança `NoSuchFileException` se o arquivo não existe, `DirectoryNotEmptyException` se você tentar deletar um diretório não-vazio, etc.) — isso facilita muito debugar o motivo real da falha.

#### Onde aparece num backend real

Verificar se um diretório de upload existe antes de gravar nele (e criar se não existir), limpar arquivos temporários depois de processar um job, listar arquivos de um diretório de configuração, mover um arquivo de uma pasta "processando" pra uma pasta "concluído" num pipeline de processamento em lote. É código de infraestrutura de aplicação, não regra de negócio, mas aparece com frequência em qualquer sistema que lida com arquivo real (não só banco de dados).

### 2. Exemplo de código comentado

java

```java
import java.io.IOException;
import java.nio.file.*;
import java.util.List;

public class OperacoesComArquivo {

    public static void main(String[] args) throws IOException {

        Path diretorio = Paths.get("dados");
        Path arquivo = diretorio.resolve("relatorio.txt"); // combina caminhos de forma segura

        // Cria o diretório só se ele ainda não existir.
        // createDirectories (com 's') cria também diretórios pai que faltarem.
        if (Files.notExists(diretorio)) {
            Files.createDirectories(diretorio);
        }

        // Escreve uma lista de linhas de uma vez, criando o arquivo se não existir
        // e sobrescrevendo o conteúdo se já existir (comportamento padrão).
        List<String> linhas = List.of("Linha 1", "Linha 2", "Linha 3");
        Files.write(arquivo, linhas);

        // Lê todas as linhas de uma vez pra uma List<String>.
        // Bom para arquivo pequeno; para arquivo grande, prefira Files.lines()
        // (retorna um Stream, lido sob demanda, sem carregar tudo na memória).
        List<String> lidas = Files.readAllLines(arquivo);
        lidas.forEach(System.out::println);

        // Metadados básicos do arquivo.
        System.out.println("Tamanho em bytes: " + Files.size(arquivo));
        System.out.println("É diretório? " + Files.isDirectory(arquivo));
        System.out.println("Última modificação: " + Files.getLastModifiedTime(arquivo));

        // Copiar, substituindo se o destino já existir.
        Path copia = diretorio.resolve("relatorio_copia.txt");
        Files.copy(arquivo, copia, StandardCopyOption.REPLACE_EXISTING);
    }
}
```

### 3. Armadilhas comuns

1. **Usar `File.delete()` ou `File.mkdir()` e ignorar o retorno `boolean`.** Como esses métodos não lançam exceção em caso de falha, é fácil escrever `arquivo.delete();` sem checar se realmente deletou, e o programa segue como se nada tivesse acontecido. Com `Files.delete()`, uma falha vira exceção — mais difícil de ignorar silenciosamente.
2. **Confundir `Files.delete()` com `Files.deleteIfExists()`.** `Files.delete()` lança `NoSuchFileException` se o arquivo não existir; `Files.deleteIfExists()` simplesmente não faz nada (retorna `false`) nesse caso. Usar o primeiro quando você não tem certeza se o arquivo existe é fonte comum de exceção inesperada.
3. **Esquecer que `Files.copy()`/`Files.move()` não sobrescrevem por padrão.** Sem passar `StandardCopyOption.REPLACE_EXISTING`, uma tentativa de copiar/mover para um caminho que já tem arquivo lança `FileAlreadyExistsException`.
4. **Usar caminho absoluto "hardcoded" (`"C:\\Users\\joao\\arquivo.txt"`) em vez de caminho relativo ou configurável.** Isso quebra assim que o código roda em outra máquina ou em produção (container Docker, servidor Linux, etc.). Prefira caminhos relativos ao projeto ou vindos de configuração (`application.properties` no caso de Spring), e deixe `Path` cuidar da diferença de separador entre sistemas operacionais.

### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Escreva um programa que verifique se um diretório chamado `saida` existe no diretório atual. Se não existir, crie-o. Depois, dentro dele, crie um arquivo `status.txt` contendo o texto `"Processamento iniciado"`. Rode o programa duas vezes seguidas e confirme que na segunda vez ele não falha (mesmo com diretório e arquivo já existentes).

**Exercício 2 (Médio)**  
Escreva um programa que receba (via array de `String` fixo no código, simulando uma lista de arquivos) os nomes de 3 arquivos de texto que devem existir dentro de `dados/`. Para cada um: se existir, imprima seu tamanho em bytes e a data da última modificação; se não existir, imprima uma mensagem clara dizendo qual arquivo está faltando — sem lançar exceção não tratada.

**Exercício 3 (Difícil)**  
Escreva um programa que liste todos os arquivos `.txt` dentro de um diretório (`Files.list()` ou `Files.newDirectoryStream()`) e mova cada um deles para um subdiretório `processados/` (criando esse subdiretório se não existir). Ao final, imprima quantos arquivos foram movidos. Trate o caso de já existir um arquivo com o mesmo nome no destino (decida e justifique: sobrescreve, pula, ou renomeia — não deixe estourar exceção sem tratamento).

**Exercício 4 (Desafio)**  
Escreva um programa que percorra um diretório e todos os seus subdiretórios (recursivamente) e some o tamanho total em bytes de todos os arquivos `.log` encontrados, imprimindo o total ao final. Pesquise o método `Files.walk()` (ou `Files.walkFileTree()`) — não é obrigatório eu já ter mostrado ele antes; parte do exercício é você ler a documentação oficial (docs.oracle.com/en/java) e entender a assinatura antes de usar.

### 5. Gabarito comentado

#### Exercício 1 (Fácil)

java

```java
import java.io.IOException;
import java.nio.file.*;

public class Exercicio1 {

    public static void main(String[] args) throws IOException {
        Path diretorio = Paths.get("saida");

        if (Files.notExists(diretorio)) {
            Files.createDirectories(diretorio);
            System.out.println("Diretório criado.");
        } else {
            System.out.println("Diretório já existia.");
        }

        Path arquivo = diretorio.resolve("status.txt");

        // write() cria o arquivo se não existir, e sobrescreve se já existir —
        // por isso rodar duas vezes não quebra: a segunda vez simplesmente
        // sobrescreve o conteúdo com o mesmo texto.
        Files.writeString(arquivo, "Processamento iniciado");

        System.out.println("Arquivo escrito em: " + arquivo.toAbsolutePath());
    }
}
```

**Raciocínio:** o ponto central do exercício é perceber que `Files.createDirectories()` **não lança exceção** se o diretório já existir — diferente de `Files.createDirectory()` (sem "s"), que lançaria `FileAlreadyExistsException` nesse caso. Como o enunciado pede rodar duas vezes sem falhar, `createDirectories` já resolve isso sozinho, sem precisar de `if/else` para a criação (o `if` que fiz aqui é só pra imprimir uma mensagem diferente, não é necessário para a lógica funcionar). Usei `Files.writeString()` (atalho para escrever uma única `String` direto, sem precisar envolver em `List.of(...)`) — funciona a partir do Java 11; se você estiver numa versão anterior, `Files.write(arquivo, "texto".getBytes())` resolve o mesmo problema.

---

#### Exercício 2 (Médio)

java

```java
import java.io.IOException;
import java.nio.file.*;

public class Exercicio2 {

    public static void main(String[] args) throws IOException {
        Path diretorioDados = Paths.get("dados");
        String[] nomesArquivos = {"relatorio.txt", "log.txt", "config.txt"};

        for (String nome : nomesArquivos) {
            Path arquivo = diretorioDados.resolve(nome);

            if (Files.exists(arquivo)) {
                long tamanho = Files.size(arquivo);
                Object dataModificacao = Files.getLastModifiedTime(arquivo);

                System.out.println(nome + " -> " + tamanho + " bytes, modificado em " + dataModificacao);
            } else {
                System.out.println("AVISO: arquivo não encontrado -> " + nome);
            }
        }
    }
}
```

**Raciocínio:** a decisão chave é usar `Files.exists()` como um **portão de checagem antes** de chamar qualquer outro método (`size()`, `getLastModifiedTime()`), em vez de tentar chamá-los direto e capturar a exceção que eles lançariam para arquivo inexistente (`NoSuchFileException`). As duas abordagens funcionam, mas checar antes com `exists()` deixa o fluxo de controle explícito no código — fica claro, lendo o `if/else`, que "arquivo não existe" é um caminho esperado do programa, não um erro excepcional. Isso importa porque usar exceção para controle de fluxo normal (em vez de erro genuinamente excepcional) é considerado má prática — aqui, "arquivo pode não existir" é uma situação totalmente esperada, não uma falha do sistema.

**Alternativa válida:** capturar `NoSuchFileException` num `try/catch` por arquivo, dentro do loop. Funciona igual, mas fica mais verboso para um cenário que `exists()` resolve de forma mais direta.

---

#### Exercício 3 (Difícil)

java

```java
import java.io.IOException;
import java.nio.file.*;
import java.util.stream.Stream;

public class Exercicio3 {

    public static void main(String[] args) throws IOException {
        Path origem = Paths.get("dados");
        Path destino = origem.resolve("processados");

        if (Files.notExists(destino)) {
            Files.createDirectories(destino);
        }

        int contador = 0;

        // try-with-resources é obrigatório aqui: Files.list() retorna um Stream
        // que mantém um handle de diretório aberto no SO — precisa ser fechado.
        try (Stream<Path> arquivos = Files.list(origem)) {

            for (Path arquivo : arquivos.toList()) {

                // Pula diretórios (como "processados") e arquivos que não são .txt
                if (Files.isDirectory(arquivo) || !arquivo.toString().endsWith(".txt")) {
                    continue;
                }

                Path destinoArquivo = destino.resolve(arquivo.getFileName());

                // Decisão: sobrescrever se já existir no destino.
                // Justificativa: o cenário é "mover para processados", então
                // se o arquivo já foi processado antes, a versão mais recente
                // faz mais sentido prevalecer do que travar o processo inteiro.
                Files.move(arquivo, destinoArquivo, StandardCopyOption.REPLACE_EXISTING);
                contador++;
            }
        }

        System.out.println("Arquivos movidos: " + contador);
    }
}
```

**Raciocínio:** dois pontos merecem atenção. Primeiro, `Files.list()` retorna um `Stream<Path>` que — diferente da maioria dos `Stream`s que você vai ver na Trilha de Programação Funcional — **precisa ser fechado**, porque por baixo dos panos ele mantém um recurso do sistema operacional aberto (um handle de diretório) enquanto está sendo consumido; por isso o try-with-resources aqui não é estilo, é necessidade. Segundo, o filtro `Files.isDirectory(arquivo)` evita o erro clássico de tentar "processar" o próprio subdiretório `processados/` que você acabou de criar dentro do diretório que está listando — sem esse filtro, na segunda execução do programa ele tentaria mover a pasta `processados` para dentro dela mesma.

**Sobre a decisão de sobrescrever:** o enunciado pedia pra decidir e justificar — "pular" também seria uma resposta válida (com `Files.exists(destinoArquivo)` como guarda antes do `move`), dependendo da regra de negócio real. O importante é a decisão ser intencional e explícita no código, não acidental.

---

#### Exercício 4 (Desafio)

java

```java
import java.io.IOException;
import java.nio.file.*;
import java.util.stream.Stream;

public class Exercicio4 {

    public static void main(String[] args) throws IOException {
        Path raiz = Paths.get("logs");

        long totalBytes;

        // Files.walk() percorre recursivamente o diretório e todos os subdiretórios,
        // devolvendo um Stream<Path> com cada arquivo/diretório encontrado.
        try (Stream<Path> caminhos = Files.walk(raiz)) {

            totalBytes = caminhos
                    .filter(Files::isRegularFile)                     // ignora diretórios
                    .filter(p -> p.toString().endsWith(".log"))       // só arquivos .log
                    .mapToLong(p -> {
                        try {
                            return Files.size(p);
                        } catch (IOException e) {
                            // Se um arquivo específico falhar (ex: foi deletado
                            // entre o walk() e a leitura), não derruba o total inteiro.
                            System.err.println("Não consegui ler: " + p + " (" + e.getMessage() + ")");
                            return 0L;
                        }
                    })
                    .sum();
        }

        System.out.println("Total em bytes nos arquivos .log: " + totalBytes);
    }
}
```

**Raciocínio:** `Files.walk()` é a ferramenta certa quando você precisa de recursão em profundidade sobre uma árvore de diretórios — ele devolve um único `Stream<Path>` "achatado" (todos os níveis de subdiretório juntos), então não precisa escrever recursão manual. O `filter(Files::isRegularFile)` é essencial: sem ele, diretórios também passariam pelo filtro de nome (embora seja raro um diretório terminar em `.log`, é mais seguro e correto excluir diretórios explicitamente do cálculo de tamanho). O `try/catch` dentro do `mapToLong` existe porque, entre o momento em que `walk()` lista o arquivo e o momento em que `Files.size()` é chamado, o arquivo pode ter sido deletado por outro processo (situação real em sistema de log ativo) — sem esse tratamento, um único arquivo problemático derrubaria a soma inteira com uma exceção não capturada.

**Alternativa mais avançada** (fora do escopo de hoje, só citando pra você saber que existe): `Files.walkFileTree()` com um `SimpleFileVisitor` dá mais controle fino — por exemplo, permite parar de descer em certos subdiretórios de propósito — mas para o caso de "soma tudo", `Files.walk()` com Stream é mais direto e é o que você vai ver mais no dia a dia.