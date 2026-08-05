#### 1. Teoria

O pacote `java.net` (e o `java.net.http`, mais novo) cobre dois níveis de abstração:

**Nível baixo — TCP/UDP direto:**

- **`Socket`** — extremidade de uma conexão TCP no cliente. TCP é orientado a conexão: garante entrega, ordem e detecção de erro, com o custo de handshake e overhead.
- **`ServerSocket`** — escuta uma porta e aceita conexões TCP de entrada (`accept()` bloqueia até um cliente conectar).
- **`DatagramSocket`/`DatagramPacket`** — UDP. Sem conexão, sem garantia de entrega ou ordem, menor overhead. Usado onde velocidade importa mais que garantia (streaming, jogos, DNS) — não é o que você usa pra API REST.
- **`InetAddress`** — resolve hostname para IP (e vice-versa).

**Nível alto — HTTP:**

- **`HttpURLConnection`** — API legada (desde Java 1.1), verbosa, sem suporte nativo a HTTP/2.
- **`java.net.http.HttpClient`** — padronizado no **Java 11 (JEP 321)**, é a API moderna: suporta HTTP/1.1 e HTTP/2, requisições síncronas e assíncronas, builder pattern. É o que você usa hoje quando quer chamar uma API externa sem depender de uma lib externa (OkHttp) ou de um framework inteiro (Spring WebClient).

Na prática de backend: você raramente escreve `Socket`/`ServerSocket` na mão — o servlet container (Tomcat, embutido no Spring Boot) já implementa isso por você, atendendo cada requisição HTTP recebida numa thread. Onde você _escreve_ código de rede é ao consumir uma API externa — aí entra o `HttpClient`, ou o `WebClient`/`RestClient` do Spring (que são wrappers de mais alto nível sobre a mesma ideia).

Uma linha fora do escopo: existe também `java.nio` (NIO), um modelo não-bloqueante de I/O usado por servidores de alta concorrência (Netty, por exemplo, usado pelo WebClient reativo por baixo) — assunto de concorrência avançada, fica pra depois.

#### 2. Exemplo de código comentado — servidor e cliente TCP (echo)

java

```java
// Servidor
import java.io.*;
import java.net.*;

public class EchoServer {
    public static void main(String[] args) throws IOException {
        int porta = 8080;
        try (ServerSocket serverSocket = new ServerSocket(porta)) { // escuta conexões nessa porta
            System.out.println("Servidor ouvindo na porta " + porta);

            while (true) {
                try (Socket clientSocket = serverSocket.accept(); // bloqueia até um cliente conectar
                     BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                     PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true)) { // true = autoFlush

                    String linha;
                    while ((linha = in.readLine()) != null) { // bloqueia até chegar uma linha ou a conexão fechar
                        System.out.println("Recebido: " + linha);
                        out.println("Echo: " + linha);
                        if (linha.equalsIgnoreCase("sair")) break;
                    }
                }
            }
        }
    }
}
```

java

```java
// Cliente
import java.io.*;
import java.net.*;

public class EchoClient {
    public static void main(String[] args) throws IOException {
        try (Socket socket = new Socket("localhost", 8080); // abre conexão TCP com o servidor
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
             BufferedReader teclado = new BufferedReader(new InputStreamReader(System.in))) {

            String mensagem;
            while ((mensagem = teclado.readLine()) != null) {
                out.println(mensagem);
                System.out.println(in.readLine());
                if (mensagem.equalsIgnoreCase("sair")) break;
            }
        }
    }
}
```

Rode o `EchoServer` primeiro, depois o `EchoClient` — isso é literalmente a base de qualquer coisa que troca dado pela rede, incluindo HTTP (que é só um protocolo de texto rodando em cima de TCP).

#### 3. Armadilhas comuns

1. **Não fechar sockets/streams.** Vaza conexão/porta. Sempre use try-with-resources (como no exemplo) — todas essas classes implementam `Closeable`.
2. **Achar que UDP garante entrega ou ordem, igual TCP.** Se o protocolo pede confiabilidade (que é o caso de praticamente toda API REST), a resposta é TCP/`Socket`, não `DatagramSocket`.
3. **`ServerSocket.accept()` bloqueando o loop principal, atendendo um cliente por vez.** É assim que o `EchoServer` acima funciona — didático, mas não escala. Em produção isso é resolvido com thread por conexão (ou I/O não-bloqueante) — é literalmente o que um servlet container já faz por você.
4. **Não configurar timeout em chamada HTTP.** Sem timeout, uma chamada pra uma API externa que não responde trava a thread indefinidamente — em produção isso derruba o serviço inteiro sob carga (esgota o pool de threads).

#### 4. Exercícios práticos

**Fácil** — Escreva `resolverIp(String hostname)` usando `InetAddress`, que retorna o IP de um hostname (ex: `"www.google.com"`) como `String`. Trate o caso de hostname inválido.

**Médio** — Usando `HttpClient` (`java.net.http`), escreva `buscarConteudo(String url)` que faz um GET síncrono e retorna o corpo da resposta como `String`, lançando exceção se o status não for 200.

**Difícil** — Modifique o `EchoServer` do exemplo pra atender múltiplos clientes ao mesmo tempo, usando um `ExecutorService` (pool de threads) em vez de processar um cliente por vez no loop principal.

**Desafio** — Escreva `postJson(String url, String jsonBody)` usando `HttpClient`, fazendo um POST com corpo JSON, configurando timeout de conexão **e** de resposta (3 segundos cada), retornando `Optional<String>` — vazio se der timeout ou erro, com o corpo se der certo.

#### 5. Gabarito comentado

**Fácil:**

java

```java
import java.net.InetAddress;
import java.net.UnknownHostException;

public static String resolverIp(String hostname) {
    try {
        InetAddress endereco = InetAddress.getByName(hostname);
        return endereco.getHostAddress();
    } catch (UnknownHostException e) {
        throw new RuntimeException("Não foi possível resolver: " + hostname, e);
    }
}
```

**Médio:**

java

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class HttpFetcher {
    // HttpClient é thread-safe e caro de criar — reutilize uma instância só,
    // igual você faria com um @Bean no Spring
    private static final HttpClient client = HttpClient.newHttpClient();

    public static String buscarConteudo(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .GET()
                .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        if (response.statusCode() != 200) {
            throw new RuntimeException("Requisição falhou, status: " + response.statusCode());
        }
        return response.body();
    }
}
```

**Difícil:**

java

```java
import java.io.*;
import java.net.*;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class EchoServerMultithread {
    public static void main(String[] args) throws IOException {
        int porta = 8080;
        ExecutorService pool = Executors.newFixedThreadPool(10); // limita quantas threads simultâneas, evita esgotar recursos

        try (ServerSocket serverSocket = new ServerSocket(porta)) {
            System.out.println("Servidor ouvindo na porta " + porta);
            while (true) {
                Socket clientSocket = serverSocket.accept(); // só bloqueia esperando a PRÓXIMA conexão
                pool.submit(() -> atenderCliente(clientSocket)); // processamento roda em outra thread
            }
        }
    }

    private static void atenderCliente(Socket clientSocket) {
        try (clientSocket;
             BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
             PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true)) {

            String linha;
            while ((linha = in.readLine()) != null) {
                out.println("Echo: " + linha);
                if (linha.equalsIgnoreCase("sair")) break;
            }
        } catch (IOException e) {
            System.err.println("Erro ao atender cliente: " + e.getMessage());
        }
    }
}
```

Raciocínio: `accept()` continua bloqueando o loop principal, mas só até a _próxima_ conexão chegar — o cliente atual é processado numa thread do pool, então múltiplos clientes são atendidos "ao mesmo tempo". Esse é exatamente o modelo (thread por request) que o Tomcat implementa por baixo do Spring Boot.

**Desafio:**

java

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.http.HttpTimeoutException;
import java.time.Duration;
import java.util.Optional;

public class HttpPoster {
    private static final HttpClient client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(3)) // timeout pra ESTABELECER a conexão (handshake TCP)
            .build();

    public static Optional<String> postJson(String url, String jsonBody) {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .header("Content-Type", "application/json")
                .timeout(Duration.ofSeconds(3)) // timeout pra RESPOSTA completa, depois de conectado
                .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
                .build();

        try {
            HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() >= 200 && response.statusCode() < 300) {
                return Optional.of(response.body());
            }
            return Optional.empty();
        } catch (HttpTimeoutException e) {
            System.err.println("Timeout: servidor não respondeu a tempo");
            return Optional.empty();
        } catch (Exception e) {
            System.err.println("Erro na requisição: " + e.getMessage());
            return Optional.empty();
        }
    }
}
```

Repare que existem dois timeouts diferentes: `connectTimeout` (no `HttpClient.Builder`) controla o tempo pra fechar a conexão TCP; `.timeout()` (no `HttpRequest.Builder`) controla o tempo total esperando a resposta depois de já conectado. Em produção você configuraria os dois — só um não cobre todos os cenários de trava.

**Alternativa e trade-off:** em projeto Spring, o normal é usar `WebClient` (reativo) ou `RestClient` (síncrono, desde Spring 6.1) em vez de `HttpClient` cru — eles já vêm integrados com serialização JSON automática, interceptors, retry, etc. Vale conhecer `HttpClient` puro porque é isso que roda por baixo (e é útil pra scripts/ferramentas que não têm Spring no classpath).