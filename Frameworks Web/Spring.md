#### 1. Teoria

**Spring vs. Spring Boot — não são a mesma coisa**

Spring (o _Spring Framework_) é um ecossistema enorme de módulos para construir aplicações Java: injeção de dependência, acesso a dados, segurança, mensageria, etc. Ele existe desde os anos 2000 e, historicamente, exigia muita configuração manual em XML.

**Spring Boot** é uma camada em cima do Spring Framework que resolve exatamente esse problema de configuração. Ele entrega três coisas principais:

1. **Auto-configuração** — o Spring Boot olha o que você tem no classpath (as dependências do seu `pom.xml`/`build.gradle`) e configura automaticamente o que faz sentido. Se você adicionou a dependência de banco H2, ele já prepara um `DataSource` sem você escrever XML nenhum.
2. **Servidor embutido** — sua aplicação roda com `java -jar`, porque um servidor HTTP (Tomcat, por padrão) já vem embutido dentro do próprio artefato. Você não precisa instalar um Tomcat separado e fazer deploy de um `.war` nele.
3. **Starters** — dependências "pacote" no Maven/Gradle (ex: `spring-boot-starter-web`) que já trazem tudo que você precisa pra um determinado tipo de aplicação, com versões compatíveis entre si.

> Nota de atualização: no momento em que escrevo isso, a versão estável do Spring Boot é a linha **4.x** (branch atual sobre Spring Framework 7), que exige Java 17 como mínimo. Boa parte do mercado brasileiro, especialmente em projetos legados, ainda roda em **Spring Boot 2.x/3.x** — isso não muda os conceitos que você vai aprender aqui, mas é bom saber que existe essa variação entre projetos. Ao criar seu primeiro projeto, prefira sempre a versão estável mais recente indicada em start.spring.io.

**Inversão de Controle (IoC) e Injeção de Dependência (DI)**

Isso é o coração do Spring, não é feature exclusiva do Boot. Em vez de uma classe criar (`new`) as dependências que ela precisa, você declara o que ela precisa e o **Spring Container** entrega isso pra ela — geralmente pelo construtor. As classes que o Spring gerencia desse jeito são chamadas de **beans**.

Você marca uma classe como candidata a virar bean com anotações de **estereótipo**:

- `@Component` — genérico, qualquer classe gerenciada pelo Spring.
- `@Service` — semanticamente indica lógica de negócio (tecnicamente é um `@Component` especializado).
- `@Repository` — indica acesso a dados; além de marcar, também ativa tradução automática de exceções de persistência específicas de banco para exceções do Spring (`DataAccessException`) — isso será aprofundado quando chegarmos em JDBC/JPA.
- `@Controller` / `@RestController` — camada web. `@RestController` é `@Controller` + `@ResponseBody` combinados, ou seja, todo retorno de método já vira corpo de resposta HTTP (normalmente JSON) automaticamente, sem anotar método por método.

**A anotação `@SpringBootApplication`**

Todo projeto Spring Boot tem uma classe principal marcada com `@SpringBootApplication`. Essa única anotação combina três outras: `@Configuration` (essa classe pode declarar beans), `@EnableAutoConfiguration` (liga o mecanismo de auto-configuração) e `@ComponentScan` (varre o pacote atual e subpacotes procurando por classes anotadas com `@Component` e afins, pra registrar como beans automaticamente).

**Camadas de uma aplicação Spring Boot (arquitetura em camadas)**

O padrão de mercado é dividir em três camadas:

- **Controller** — recebe requisição HTTP, valida entrada básica, delega pro service, devolve resposta. Não tem lógica de negócio.
- **Service** — lógica de negócio de verdade.
- **Repository** — acesso a dados (banco). Vamos usar uma versão simplificada (em memória) nos exercícios, porque JDBC/Spring Data JPA é bloco futuro do roadmap — mas a _forma_ de estruturar a camada já vale a pena praticar agora.

**Anotações de mapeamento REST básicas**

- `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` — mapeiam método HTTP + caminho para um método Java.
- `@PathVariable` — extrai valor de dentro da URL (ex: `/usuarios/{id}`).
- `@RequestParam` — extrai valor de query string (ex: `/usuarios?nome=Ana`).
- `@RequestBody` — desserializa o corpo da requisição (normalmente JSON) num objeto Java.
- `ResponseEntity<T>` — permite controlar explicitamente o status HTTP da resposta (200, 201, 404, etc.), além do corpo.

**`application.properties` / `application.yml`**

Arquivo de configuração externa da aplicação (porta do servidor, dados de conexão, nível de log, etc.), lido automaticamente pelo Spring Boot na inicialização. Evita hardcode de configuração no código Java.

**Onde isso conecta com o que você já viu**

O Bloco 5 (Tratamento de Erros) volta a aparecer aqui: em vez de deixar uma exceção estourar sem controle, o padrão Spring é capturar de forma centralizada com `@ExceptionHandler`/`@ControllerAdvice` e converter em uma resposta HTTP com status apropriado (400, 404, etc.) — vamos praticar uma versão simples disso no exercício desafio.

---

#### 2. Exemplo de código comentado

java

```java
// Classe principal — ponto de entrada da aplicação
package com.exemplo.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
        // Sobe o contexto do Spring (cria e conecta os beans) e inicia o servidor embutido
    }
}
```

java

```java
// Camada de domínio — objeto simples, sem lógica
package com.exemplo.demo;

public class Produto {
    private Long id;
    private String nome;
    private double preco;

    public Produto(Long id, String nome, double preco) {
        this.id = id;
        this.nome = nome;
        this.preco = preco;
    }

    // getters — necessários para o Spring conseguir serializar o objeto em JSON
    public Long getId() { return id; }
    public String getNome() { return nome; }
    public double getPreco() { return preco; }
}
```

java

```java
// Camada de serviço — lógica de negócio, sem nenhuma anotação web
package com.exemplo.demo;

import org.springframework.stereotype.Service;
import java.util.*;

@Service // registra essa classe como bean gerenciado pelo Spring
public class ProdutoService {

    // "Banco" em memória só para fins didáticos — JDBC/JPA vêm em bloco futuro
    private final Map<Long, Produto> produtos = new HashMap<>();
    private long proximoId = 1;

    public Produto criar(String nome, double preco) {
        Produto novo = new Produto(proximoId++, nome, preco);
        produtos.put(novo.getId(), novo);
        return novo;
    }

    public Optional<Produto> buscarPorId(Long id) {
        return Optional.ofNullable(produtos.get(id));
        // Optional aqui não é coincidência — é o mesmo conceito do Bloco 4,
        // usado exatamente da forma como aparece em repositórios reais do Spring Data JPA
    }

    public List<Produto> listarTodos() {
        return new ArrayList<>(produtos.values());
    }
}
```

java

```java
// Camada de controller — só orquestra HTTP, delega a lógica pro service
package com.exemplo.demo;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController // = @Controller + @ResponseBody: todo retorno vira corpo da resposta HTTP
@RequestMapping("/produtos")
public class ProdutoController {

    private final ProdutoService produtoService;

    // Injeção via CONSTRUTOR — o Spring detecta automaticamente que esse construtor
    // precisa de um ProdutoService e injeta o bean correspondente.
    // Não é necessário @Autowired aqui quando há um único construtor (desde o Spring 4.3+).
    public ProdutoController(ProdutoService produtoService) {
        this.produtoService = produtoService;
    }

    @PostMapping
    public ResponseEntity<Produto> criar(@RequestBody Produto entrada) {
        Produto criado = produtoService.criar(entrada.getNome(), entrada.getPreco());
        return ResponseEntity.status(201).body(criado); // 201 Created
    }

    @GetMapping("/{id}")
    public ResponseEntity<Produto> buscar(@PathVariable Long id) {
        return produtoService.buscarPorId(id)
                .map(ResponseEntity::ok)          // se achou: 200 com o produto
                .orElse(ResponseEntity.notFound().build()); // se não achou: 404 sem corpo
    }

    @GetMapping
    public ResponseEntity<java.util.List<Produto>> listar() {
        return ResponseEntity.ok(produtoService.listarTodos());
    }
}
```

---

#### 3. Armadilhas comuns

1. **Colocar lógica de negócio dentro do Controller** — validações complexas, cálculos, regras condicionais direto no método do controller. Isso mistura a responsabilidade de "lidar com HTTP" com "regra de negócio", dificulta teste (você teria que simular requisição HTTP só pra testar uma regra) e reaproveitamento.
2. **Injeção por campo (`@Autowired` direto no atributo) em vez de por construtor** — funciona, mas esconde dependências obrigatórias (a classe parece que não precisa de nada, olhando só o construtor), dificulta testes unitários (não dá pra passar um mock facilmente sem reflection) e impede que o campo seja `final`. Injeção por construtor é o padrão recomendado hoje.
3. **Confundir `@Component`, `@Service` e `@Repository` como "tanto faz"** — funcionalmente, todas registram um bean e o comportamento de scanning é o mesmo. Mas `@Repository` ativa tradução de exceção de persistência, e usar o estereótipo certo comunica intenção pra quem lê o código depois. Usar `@Component` pra tudo funciona, mas perde clareza.
4. **Depender cegamente da auto-configuração sem entender o que está rodando** — quando algo não funciona como esperado (porta errada, bean não encontrado, endpoint 404 inesperado), quem não entende que existe uma auto-configuração acontecendo por trás fica perdido pra debugar. Vale a pena, cedo ou tarde, rodar sua aplicação com `--debug` e olhar o relatório de auto-configuração pelo menos uma vez, só para entender o mecanismo.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie um projeto Spring Boot mínimo (pode usar start.spring.io, com dependência `Spring Web`) com um `@RestController` que responde `GET /hello` retornando a string `"Olá, mundo!"`. Critério de pronto: rodar a aplicação e acessar `http://localhost:8080/hello` no navegador (ou via `curl`) retorna o texto esperado com status 200.

**Exercício 2 (Fácil/Médio)**  
Crie um endpoint `GET /saudacao` que aceita um `@RequestParam` opcional chamado `nome` (padrão `"visitante"` se não for informado) e retorna `"Olá, {nome}!"`. Adicione também um endpoint `GET /saudacao/{nome}` usando `@PathVariable`, com o mesmo comportamento. Critério de pronto: `/saudacao?nome=Ana` retorna "Olá, Ana!"; `/saudacao` sem parâmetro retorna "Olá, visitante!"; `/saudacao/Carlos` retorna "Olá, Carlos!".

**Exercício 3 (Médio)**  
Separando em camadas (Controller → Service), crie um cadastro de `Tarefa` (campos: `id`, `descricao`, `concluida`) mantido em memória (uma `List` ou `Map` dentro do Service, como no exemplo). Implemente: `POST /tarefas` (cria, recebendo `@RequestBody`), `GET /tarefas` (lista todas), `GET /tarefas/{id}` (busca uma, retornando 404 se não existir). Critério de pronto: os três endpoints funcionam e a lógica de armazenamento está inteiramente no Service, não no Controller.

**Exercício 4 (Desafio)**  
Estenda o exercício 3 com: (a) `PUT /tarefas/{id}` para marcar como concluída, retornando 404 se o id não existir; (b) uma exceção customizada `TarefaNaoEncontradaException` (unchecked, lançada pelo Service quando o id não existe); (c) um `@RestControllerAdvice` com um `@ExceptionHandler` que captura essa exceção e devolve `ResponseEntity` com status 404 e um corpo JSON simples tipo `{"erro": "mensagem aqui"}`, centralizando esse tratamento em vez de fazer `if/else` de existência em cada método do controller. Critério de pronto: acessar `GET /tarefas/999` (id inexistente) retorna 404 com o corpo JSON de erro, vindo do `@ControllerAdvice`, não de um `if` dentro do controller.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() {
        return "Olá, mundo!";
    }
}
```

_Raciocínio:_ `@RestController` já faz o retorno virar corpo da resposta automaticamente — não precisa de `@ResponseBody` manual em cada método. Uma `String` simples é serializada como texto puro; se o retorno fosse um objeto, o Spring usaria Jackson (biblioteca padrão) para converter em JSON.

**Exercício 2**

java

```java
@RestController
public class SaudacaoController {

    @GetMapping("/saudacao")
    public String saudacaoPorQueryParam(
            @RequestParam(defaultValue = "visitante") String nome) {
        return "Olá, " + nome + "!";
    }

    @GetMapping("/saudacao/{nome}")
    public String saudacaoPorPath(@PathVariable String nome) {
        return "Olá, " + nome + "!";
    }
}
```

_Raciocínio:_ `defaultValue` no `@RequestParam` resolve o caso de parâmetro ausente sem precisar de `if (nome == null)` manual. `@PathVariable`, diferente de `@RequestParam`, é sempre obrigatório por natureza — se a rota é `/saudacao/{nome}`, não existe "sem nome" nesse formato de URL. Trade-off entre as duas abordagens: query param é melhor pra parâmetros opcionais/filtros; path variable é melhor quando o valor identifica um recurso específico na URL (mais "RESTful").

**Exercício 3**

java

```java
public class Tarefa {
    private Long id;
    private String descricao;
    private boolean concluida;

    public Tarefa() {} // construtor vazio necessário para o Jackson desserializar do JSON

    public Tarefa(Long id, String descricao, boolean concluida) {
        this.id = id;
        this.descricao = descricao;
        this.concluida = concluida;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getDescricao() { return descricao; }
    public void setDescricao(String descricao) { this.descricao = descricao; }
    public boolean isConcluida() { return concluida; }
    public void setConcluida(boolean concluida) { this.concluida = concluida; }
}

@Service
public class TarefaService {
    private final Map<Long, Tarefa> tarefas = new HashMap<>();
    private long proximoId = 1;

    public Tarefa criar(Tarefa entrada) {
        Tarefa nova = new Tarefa(proximoId++, entrada.getDescricao(), false);
        tarefas.put(nova.getId(), nova);
        return nova;
    }

    public List<Tarefa> listarTodas() {
        return new ArrayList<>(tarefas.values());
    }

    public Optional<Tarefa> buscarPorId(Long id) {
        return Optional.ofNullable(tarefas.get(id));
    }
}

@RestController
@RequestMapping("/tarefas")
public class TarefaController {
    private final TarefaService tarefaService;

    public TarefaController(TarefaService tarefaService) {
        this.tarefaService = tarefaService;
    }

    @PostMapping
    public ResponseEntity<Tarefa> criar(@RequestBody Tarefa entrada) {
        return ResponseEntity.status(201).body(tarefaService.criar(entrada));
    }

    @GetMapping
    public List<Tarefa> listar() {
        return tarefaService.listarTodas();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Tarefa> buscar(@PathVariable Long id) {
        return tarefaService.buscarPorId(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
}
```

_Raciocínio:_ note que `Tarefa` agora precisa de construtor vazio + setters — diferente do `Produto` do exemplo (que era só leitura). Isso porque o Jackson, ao desserializar o `@RequestBody` do `POST`, precisa instanciar o objeto e depois preencher campo por campo. Toda a responsabilidade de armazenamento (o `Map`, a geração de id) fica isolada no Service — o Controller não sabe _como_ a tarefa é guardada, só sabe _pedir_ pro Service fazer isso. Essa separação é o que permite, no futuro, trocar o `HashMap` por um banco de verdade via Spring Data JPA sem precisar tocar no Controller.

**Exercício 4**

java

```java
// Exceção de domínio, unchecked — decisão consistente com o Bloco 5:
// "id inexistente" é uma condição de negócio esperada, mas não queremos
// obrigar throws em cascata por várias camadas (Service -> Controller)
public class TarefaNaoEncontradaException extends RuntimeException {
    public TarefaNaoEncontradaException(Long id) {
        super("Tarefa não encontrada: id " + id);
    }
}
```

java

```java
@Service
public class TarefaService {
    // ... campos e método criar() e listarTodas() iguais ao exercício anterior

    public Tarefa buscarPorIdOuFalhar(Long id) {
        Tarefa tarefa = tarefas.get(id);
        if (tarefa == null) {
            throw new TarefaNaoEncontradaException(id);
        }
        return tarefa;
    }

    public Tarefa concluir(Long id) {
        Tarefa tarefa = buscarPorIdOuFalhar(id); // reaproveita a validação acima
        tarefa.setConcluida(true);
        return tarefa;
    }
}
```

java

```java
@RestController
@RequestMapping("/tarefas")
public class TarefaController {
    private final TarefaService tarefaService;

    public TarefaController(TarefaService tarefaService) {
        this.tarefaService = tarefaService;
    }

    // ... criar() e listar() iguais ao exercício anterior

    @GetMapping("/{id}")
    public Tarefa buscar(@PathVariable Long id) {
        // Sem if/else de existência aqui — se não achar, a exceção sobe
        // e quem trata é o ControllerAdvice abaixo
        return tarefaService.buscarPorIdOuFalhar(id);
    }

    @PutMapping("/{id}")
    public Tarefa concluir(@PathVariable Long id) {
        return tarefaService.concluir(id);
    }
}
```

java

```java
@RestControllerAdvice // aplica esse tratamento a TODOS os controllers da aplicação
public class TratadorDeExcecoes {

    @ExceptionHandler(TarefaNaoEncontradaException.class)
    public ResponseEntity<Map<String, String>> tratarNaoEncontrada(TarefaNaoEncontradaException e) {
        Map<String, String> corpo = Map.of("erro", e.getMessage());
        return ResponseEntity.status(404).body(corpo);
    }
}
```

_Raciocínio:_ o ponto central do exercício é a inversão de responsabilidade — em vez de cada método do Controller decidir "se não achou, retorna 404" (repetindo essa lógica em `buscar`, `concluir`, e qualquer endpoint futuro que precise de uma tarefa por id), o Service simplesmente **lança** a exceção quando a regra é violada, e um único lugar (`@RestControllerAdvice`) decide _como isso vira uma resposta HTTP_. Isso escala melhor: se amanhã você tiver 15 endpoints diferentes que podem não achar uma tarefa, o tratamento de "não encontrado → 404 com esse formato de erro" já está centralizado, não precisa duplicar em lugar nenhum. É exatamente o padrão que citei na Teoria como uso real de `@ExceptionHandler`/`@ControllerAdvice`, agora praticado.

#### Exercícios práticos (rodada 2)

**Exercício 1 (Fácil)**  
Adicione `DELETE /tarefas/{id}`. Se a tarefa existir, remova e retorne `204 No Content` (sem corpo). Se não existir, reaproveite `TarefaNaoEncontradaException` — o `@RestControllerAdvice` já existente deve tratar isso automaticamente, sem `if` novo no controller. Critério de pronto: `DELETE` de um id existente retorna 204; `DELETE` de um id inexistente retorna 404 com o corpo de erro já padronizado.

**Exercício 2 (Fácil/Médio)**  
Adicione dois endpoints: `GET /tarefas/pendentes` e `GET /tarefas/concluidas`, cada um retornando só o subconjunto correspondente da lista. Não duplique a lógica de filtro — pense em como reaproveitar um único método privado ou usar Stream API (Bloco 12 — pode usar mesmo sem termos formalizado ainda, você provavelmente já reconhece `.filter()`/`.stream()` de uso prático; se não, pode resolver com loop comum). Critério de pronto: cada endpoint retorna só as tarefas do estado correspondente, e criar uma tarefa nova (que começa não concluída) aparece em `/pendentes`, não em `/concluidas`.

**Exercício 3 (Médio)**  
Extraia uma interface `TarefaRepositorio` com os métodos de armazenamento puro (`salvar`, `buscarPorId`, `listarTodas`, `remover`). Crie `TarefaRepositorioEmMemoria implements TarefaRepositorio` (anotada como `@Repository`) contendo o `Map` que hoje está dentro do `TarefaService`. O `TarefaService` deve passar a receber `TarefaRepositorio` via construtor (injeção por **interface**, não pela implementação concreta) e delegar a ele toda persistência, mantendo só a lógica de negócio. Critério de pronto: `TarefaService` não tem mais nenhum `Map` dentro dele; todos os endpoints continuam funcionando exatamente como antes.

**Exercício 4 (Desafio)**  
Duas partes:  
(a) Adicione validação simples no `criar`: se `descricao` for nula ou em branco, o Service deve lançar uma nova exceção `DescricaoInvalidaException` (unchecked). O `@RestControllerAdvice` deve ganhar um **segundo** `@ExceptionHandler`, tratando esse novo tipo com status `400 Bad Request` (mantendo o handler de `TarefaNaoEncontradaException` já existente, com 404).  
(b) Adicione `GET /tarefas/estatisticas` retornando um objeto (crie uma classe `TarefaEstatisticas`) com `total`, `concluidas` e `pendentes`.  
Critério de pronto: criar uma tarefa com descrição em branco retorna 400 com mensagem de erro clara; buscar id inexistente continua retornando 404 (os dois handlers coexistem sem conflito); `/estatisticas` reflete corretamente os números após criar e concluir tarefas.

---

#### Gabarito comentado

**Exercício 1**

java

```java
// TarefaService
public void remover(Long id) {
    if (!tarefas.containsKey(id)) {
        throw new TarefaNaoEncontradaException(id);
    }
    tarefas.remove(id);
}
```

java

```java
// TarefaController
@DeleteMapping("/{id}")
public ResponseEntity<Void> remover(@PathVariable Long id) {
    tarefaService.remover(id);
    return ResponseEntity.noContent().build(); // 204, sem corpo
}
```

_Raciocínio:_ `ResponseEntity<Void>` comunica explicitamente "não há corpo de resposta nesse caso de sucesso" — `204 No Content` é o status semanticamente correto pra uma remoção bem-sucedida (diferente de `200 OK`, que sugeriria um corpo que não existe). O tratamento de "não encontrado" continua vindo de graça do `@RestControllerAdvice`: o método do controller nem precisa saber que esse caminho de erro existe.

**Exercício 2**

java

```java
// TarefaService
public List<Tarefa> listarPendentes() {
    return filtrarPorStatus(false);
}

public List<Tarefa> listarConcluidas() {
    return filtrarPorStatus(true);
}

private List<Tarefa> filtrarPorStatus(boolean concluida) {
    return tarefas.values().stream()
            .filter(t -> t.isConcluida() == concluida)
            .collect(Collectors.toList());
}
```

java

```java
// TarefaController
@GetMapping("/pendentes")
public List<Tarefa> listarPendentes() {
    return tarefaService.listarPendentes();
}

@GetMapping("/concluidas")
public List<Tarefa> listarConcluidas() {
    return tarefaService.listarConcluidas();
}
```

_Raciocínio:_ o método privado `filtrarPorStatus` evita duplicar a lógica de filtro — os dois métodos públicos só dizem _o quê_ querem (concluída ou não), não _como_ filtrar. Isso é pouco código agora, mas é o hábito que evita duplicação quando a regra de filtro crescer (ex: adicionar "vencidas", "atrasadas" etc. no futuro). Vale registrar: `filter`/`stream` aqui é uma prévia da Trilha de Programação Funcional (Bloco 12) — não se preocupe em dominar a fundo agora, só reconhecer o padrão; ele será formalizado mais pra frente.

**Exercício 3**

java

```java
public interface TarefaRepositorio {
    Tarefa salvar(Tarefa tarefa);
    Optional<Tarefa> buscarPorId(Long id);
    List<Tarefa> listarTodas();
    void remover(Long id);
}
```

java

```java
@Repository
public class TarefaRepositorioEmMemoria implements TarefaRepositorio {
    private final Map<Long, Tarefa> tarefas = new HashMap<>();
    private long proximoId = 1;

    @Override
    public Tarefa salvar(Tarefa tarefa) {
        if (tarefa.getId() == null) {
            tarefa.setId(proximoId++);
        }
        tarefas.put(tarefa.getId(), tarefa);
        return tarefa;
    }

    @Override
    public Optional<Tarefa> buscarPorId(Long id) {
        return Optional.ofNullable(tarefas.get(id));
    }

    @Override
    public List<Tarefa> listarTodas() {
        return new ArrayList<>(tarefas.values());
    }

    @Override
    public void remover(Long id) {
        tarefas.remove(id);
    }
}
```

java

```java
@Service
public class TarefaService {
    private final TarefaRepositorio tarefaRepositorio; // depende da INTERFACE, não da implementação

    public TarefaService(TarefaRepositorio tarefaRepositorio) {
        this.tarefaRepositorio = tarefaRepositorio;
    }

    public Tarefa criar(Tarefa entrada) {
        Tarefa nova = new Tarefa(null, entrada.getDescricao(), false);
        return tarefaRepositorio.salvar(nova);
    }

    public Tarefa buscarPorIdOuFalhar(Long id) {
        return tarefaRepositorio.buscarPorId(id)
                .orElseThrow(() -> new TarefaNaoEncontradaException(id));
    }

    public void remover(Long id) {
        buscarPorIdOuFalhar(id); // valida existência antes de remover
        tarefaRepositorio.remover(id);
    }

    // listarTodas, listarPendentes etc. agora delegam para tarefaRepositorio.listarTodas()
}
```

_Raciocínio:_ essa é a parte mais importante conceitualmente do exercício. O `TarefaService` passou a depender de uma **interface** (`TarefaRepositorio`), não de `TarefaRepositorioEmMemoria` diretamente. O Spring resolve isso em tempo de execução: como só existe um bean implementando a interface, ele injeta `TarefaRepositorioEmMemoria` automaticamente. A vantagem prática: no dia em que você trocar o armazenamento em memória por um banco de verdade via Spring Data JPA, você troca **só a implementação** (`TarefaRepositorioEmMemoria` por algo que fale com banco) — o `TarefaService` não muda uma linha, porque ele nunca soube que estava falando com um `HashMap`. Isso conecta diretamente com Interfaces do Bloco 3 e é literalmente o mesmo mecanismo que sustenta `JpaRepository` mais pra frente — só que aqui você está vendo o "por baixo do capô" antes de o Spring Data esconder isso pra você.

Note também `orElseThrow(() -> ...)` como alternativa mais concisa ao `if (tarefa == null) throw ...` que usamos antes — mesmo resultado, forma mais idiomática quando você já tem um `Optional` em mãos.

**Exercício 4**

java

```java
public class DescricaoInvalidaException extends RuntimeException {
    public DescricaoInvalidaException() {
        super("Descrição da tarefa não pode ser vazia.");
    }
}
```

java

```java
// TarefaService
public Tarefa criar(Tarefa entrada) {
    if (entrada.getDescricao() == null || entrada.getDescricao().isBlank()) {
        throw new DescricaoInvalidaException();
    }
    Tarefa nova = new Tarefa(null, entrada.getDescricao(), false);
    return tarefaRepositorio.salvar(nova);
}
```

java

```java
public class TarefaEstatisticas {
    private final long total;
    private final long concluidas;
    private final long pendentes;

    public TarefaEstatisticas(long total, long concluidas, long pendentes) {
        this.total = total;
        this.concluidas = concluidas;
        this.pendentes = pendentes;
    }

    public long getTotal() { return total; }
    public long getConcluidas() { return concluidas; }
    public long getPendentes() { return pendentes; }
}
```

java

```java
// TarefaService
public TarefaEstatisticas gerarEstatisticas() {
    List<Tarefa> todas = tarefaRepositorio.listarTodas();
    long concluidas = todas.stream().filter(Tarefa::isConcluida).count();
    long total = todas.size();
    return new TarefaEstatisticas(total, concluidas, total - concluidas);
}
```

java

```java
// TarefaController
@GetMapping("/estatisticas")
public TarefaEstatisticas estatisticas() {
    return tarefaService.gerarEstatisticas();
}
```

java

```java
@RestControllerAdvice
public class TratadorDeExcecoes {

    @ExceptionHandler(TarefaNaoEncontradaException.class)
    public ResponseEntity<Map<String, String>> tratarNaoEncontrada(TarefaNaoEncontradaException e) {
        return ResponseEntity.status(404).body(Map.of("erro", e.getMessage()));
    }

    @ExceptionHandler(DescricaoInvalidaException.class)
    public ResponseEntity<Map<String, String>> tratarDescricaoInvalida(DescricaoInvalidaException e) {
        return ResponseEntity.status(400).body(Map.of("erro", e.getMessage()));
    }
}
```

_Raciocínio:_ diferente do `try/catch` do Bloco 5, aqui **não existe ordem que importe** entre os dois métodos `@ExceptionHandler` — o Spring escolhe qual handler chamar com base no tipo exato da exceção lançada (usa a especificidade do tipo, similar ao mecanismo de resolução de `catch`, mas resolvido pelo _dispatcher_ do Spring, não pela ordem física do código). Isso é uma diferença estrutural relevante em relação ao `try/catch`: lá, colocar o handler genérico antes do específico é erro de compilação; aqui, a ordem dos métodos na classe é irrelevante.

Sobre `gerarEstatisticas`: o cálculo de `pendentes` como `total - concluidas` evita rodar duas vezes a lista com dois `.filter()` diferentes — pequeno detalhe de eficiência, mas também deixa explícito que "pendente" e "concluída" são mutuamente exclusivos, o que é uma invariante real do domínio.