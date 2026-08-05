#### 1. Teoria

**Spring Data JPA** é uma camada que fica _acima_ do JPA e do JDBC que você acabou de estudar. A cadeia é: **JDBC** (baixo nível, você escreve SQL na mão) → **JPA** (especificação — um conjunto de interfaces que definem "como mapear objeto Java pra tabela") → **Hibernate** (a implementação mais usada dessa especificação, é quem de fato gera o SQL) → **Spring Data JPA** (gera automaticamente as implementações de repositório, você só declara uma _interface_).

**Diferenciação que costuma confundir:**

- **JPA** não é uma biblioteca, é uma especificação (interfaces como `EntityManager`, anotações como `@Entity`). Sozinha, não faz nada.
- **Hibernate** implementa essa especificação — é o motor que de fato conversa com o banco.
- **Spring Data JPA** não substitui o Hibernate, ele _usa_ o Hibernate por baixo e elimina o código repetitivo de repositório (você não escreve mais `EntityManager em; em.persist(objeto);` manualmente).

**Detalhe de versão que muda o import que você vai usar:** desde o **Spring Boot 3** (que já é o padrão atual), as anotações JPA vêm do pacote `jakarta.persistence.*`, não mais `javax.persistence.*` — houve uma migração de namespace da Oracle pra Eclipse Foundation. Se você ver tutorial antigo com `import javax.persistence.Entity`, é código pra Spring Boot 2.x; hoje o import correto é `jakarta.persistence.Entity`. Confirmei isso antes de escrever o exemplo abaixo justamente pra não te ensinar o import errado.

**Peças principais:**

|Peça|Papel|
|---|---|
|`@Entity`|Marca uma classe Java como tabela do banco|
|`@Id` / `@GeneratedValue`|Define a chave primária e como ela é gerada|
|`JpaRepository<Entidade, TipoDoId>`|Interface que você **declara** — o Spring gera a implementação em tempo de execução via proxy. Você não escreve `ProdutoRepositoryImpl`|
|Query methods|Métodos como `findByNome(String nome)` — o Spring interpreta o _nome do método_ e monta o SQL/JPQL sozinho, sem você escrever nada|
|`@Query`|Pra quando o nome do método ficaria absurdo de longo ou a query é complexa demais pra derivar do nome|

**Conexão direta com o que você já viu (Bloco 4 — Coleções):** os métodos de busca por ID do Spring Data JPA retornam `Optional<T>`, não o objeto direto nem `null`. É exatamente o "Optionals aparecem toda hora em retornos de repository" — agora você está vendo isso na prática, não só na teoria.

---

#### 2. Exemplo de código comentado

**Entidade:**

java

```java
package com.exemplo.model;

import jakarta.persistence.*;

@Entity
@Table(name = "produtos")
public class Produto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // deixa o banco gerar o ID (AUTO_INCREMENT)
    private Long id;

    @Column(nullable = false)
    private String nome;

    @Column(nullable = false)
    private Double preco;

    protected Produto() {
        // construtor vazio: o JPA exige, ele usa reflection pra reconstruir o objeto ao ler do banco
    }

    public Produto(String nome, Double preco) {
        this.nome = nome;
        this.preco = preco;
    }

    public Long getId() { return id; }
    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
    public Double getPreco() { return preco; }
    public void setPreco(Double preco) { this.preco = preco; }
}
```

**Repositório — é só uma interface, sem implementação:**

java

```java
package com.exemplo.repository;

import com.exemplo.model.Produto;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;

public interface ProdutoRepository extends JpaRepository<Produto, Long> {

    // Query method: o Spring lê "findByNome" e monta
    // "SELECT p FROM Produto p WHERE p.nome = ?1" sozinho, sem você escrever SQL
    Optional<Produto> findByNome(String nome);

    // "Containing" = LIKE %valor%, "IgnoreCase" = case-insensitive
    List<Produto> findByNomeContainingIgnoreCase(String trecho);

    List<Produto> findByPrecoGreaterThan(Double preco);
}
```

**Uso — nenhuma linha de SQL escrita à mão:**

java

```java
@Service
public class ProdutoService {

    private final ProdutoRepository produtoRepository;

    public ProdutoService(ProdutoRepository produtoRepository) { // injeção via construtor
        this.produtoRepository = produtoRepository;
    }

    public Produto cadastrar(String nome, Double preco) {
        Produto produto = new Produto(nome, preco);
        return produtoRepository.save(produto); // id == null -> Spring/Hibernate faz INSERT
    }

    public Optional<Produto> buscarPorId(Long id) {
        return produtoRepository.findById(id); // método herdado de JpaRepository, retorna Optional
    }

    public List<Produto> buscarPorTrecho(String trecho) {
        return produtoRepository.findByNomeContainingIgnoreCase(trecho);
    }
}
```

Repare: você não escreveu `CREATE TABLE`, não escreveu `INSERT`, não fez `try-with-resources` de `Connection`. Tudo isso que você fez na mão no bloco de JDBC está acontecendo aqui — só que gerado automaticamente.

---

#### 3. Armadilhas comuns

1. **Problema N+1** — buscar uma lista de entidades (1 query) e, dentro de um loop, acessar uma associação que ainda não foi carregada (ex: `pedido.getUsuario().getNome()` numa lista de 50 pedidos) dispara **uma query adicional por item** — 50 queries a mais só pra popular uma associação. É o erro de performance mais clássico de quem começa com JPA.
2. **`LazyInitializationException`** — tentar acessar uma associação `LAZY` fora do contexto transacional (ex: no controller, depois que o service já "fechou" a transação, tentando serializar a entidade pra JSON). A associação só é carregada quando acessada, e nesse ponto a sessão do Hibernate que faria essa busca já não existe mais.
3. **Confundir o comportamento de `save()`** — `save()` faz _insert ou update_ dependendo se a entidade já tem `@Id` preenchido. Se você setar um ID manualmente achando que está criando um registro novo, pode acabar sobrescrevendo um registro existente com aquele ID.
4. **Erro de digitação em query method não dá erro de compilação** — `findByNoem(String x)` (typo em vez de `findByNome`) compila normalmente e só quebra em **runtime**, quando o Spring tenta interpretar o nome e não encontra o campo `noem` na entidade. Isso é diferente de escrever SQL errado num `PreparedStatement`, que também só falha em runtime — mas aqui a causa raiz é o nome do método, não a sintaxe SQL.

_Nota de escopo: o `@ManyToOne` tem `FetchType.EAGER` como padrão da especificação JPA (diferente de `@OneToMany`/`@ManyToMany`, que são `LAZY` por padrão) — isso importa direto no exercício 4 abaixo, então não vou aprofundar agora, você vai decidir isso na prática._

---

#### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Crie a entidade `Usuario` (`id`, `nome`, `email`) com `jakarta.persistence`, e a interface `UsuarioRepository extends JpaRepository<Usuario, Long>`. Escreva um método que persiste 2 usuários usando `save()` e depois lista todos usando `findAll()`. Critério de pronto: os 2 usuários aparecem na listagem, com `id` preenchido automaticamente.

**Exercício 2 (médio)**  
Adicione ao `UsuarioRepository` um query method `findByEmail(String email)` retornando `Optional<Usuario>`. Escreva um método de serviço que retorna o nome do usuário, ou a string `"não encontrado"` caso não exista — **sem usar `isPresent()` + `get()`** (use os métodos funcionais do `Optional`). Critério de pronto: funciona pros dois casos (email existente e inexistente) sem lançar exceção.

**Exercício 3 (difícil)**  
Adicione um campo `idade` (Integer) na entidade `Usuario`. Crie um query method que retorna todos os usuários com idade maior ou igual a um valor, **ordenados por nome** — resolva isso só com o nome do método (sem `@Query`). Critério de pronto: o método funciona e você consegue explicar por que usar `@Query` aqui seria desnecessário.

**Exercício 4 (desafio)**  
Crie a entidade `Pedido` com um campo `@ManyToOne` apontando pra `Usuario` (um pedido pertence a um usuário). Decida se você mantém o `FetchType.EAGER` (padrão do `@ManyToOne`) ou sobrescreve pra `LAZY`, e escreva um comentário no código justificando a escolha considerando o cenário: uma tela que lista 100 pedidos, mas só mostra o nome do usuário quando o operador clica em "detalhes" de um pedido específico.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
@Entity
@Table(name = "usuarios")
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nome;
    private String email;

    protected Usuario() {}

    public Usuario(String nome, String email) {
        this.nome = nome;
        this.email = email;
    }

    public Long getId() { return id; }
    public String getNome() { return nome; }
    public String getEmail() { return email; }
}
```

java

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {}
```

java

```java
usuarioRepository.save(new Usuario("Ana Silva", "ana@email.com"));
usuarioRepository.save(new Usuario("Bruno Costa", "bruno@email.com"));

List<Usuario> todos = usuarioRepository.findAll();
```

Raciocínio: `JpaRepository<Usuario, Long>` já vem com `save`, `findAll`, `findById`, `deleteById` prontos — você não precisa declarar nada além da interface vazia pra ter isso funcionando.

**Exercício 2**

java

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    Optional<Usuario> findByEmail(String email);
}
```

java

```java
public String buscarNomePorEmail(String email) {
    return usuarioRepository.findByEmail(email)
            .map(Usuario::getNome)
            .orElse("não encontrado");
}
```

Raciocínio: `.map()` só executa se o `Optional` tiver valor; `.orElse()` cobre o caso vazio. Isso evita o padrão frágil `if (opt.isPresent()) { opt.get()... }`, que é fácil de esquecer de tratar o `else` e tomar `NoSuchElementException`.

**Exercício 3**

java

```java
List<Usuario> findByIdadeGreaterThanEqualOrderByNomeAsc(Integer idade);
```

Raciocínio: o Spring Data JPA interpreta `GreaterThanEqual` como `>=` e `OrderByNomeAsc` como `ORDER BY nome ASC`, tudo a partir do nome do método. `@Query` só compensa quando a lógica não é expressável de forma legível só com nome de método (junções complexas, agregações, subqueries) — aqui o nome já descreve a intenção inteira sem ficar ilegível, então adicionar `@Query` seria complexidade desnecessária pro mesmo resultado.

**Exercício 4**

java

```java
@Entity
@Table(name = "pedidos")
public class Pedido {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // Sobrescrevendo o padrão EAGER do @ManyToOne para LAZY:
    // na tela de listagem (100 pedidos), carregar o Usuario de cada um eagerly
    // significaria buscar 100 usuários mesmo quando ninguém vai olhar o nome deles.
    // Com LAZY, o Usuario só é buscado no banco quando getUsuario() é
    // efetivamente chamado (na tela de detalhes de UM pedido específico).
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "usuario_id")
    private Usuario usuario;

    private String descricao;

    protected Pedido() {}

    public Pedido(Usuario usuario, String descricao) {
        this.usuario = usuario;
        this.descricao = descricao;
    }
}
```

Raciocínio: o padrão `EAGER` do `@ManyToOne` existe por motivo histórico de implementação, não porque seja a escolha certa na maioria dos casos — no cenário descrito (listagem grande, detalhe raro), `LAZY` evita buscar dado que na maior parte das vezes não vai ser usado. O trade-off é que, se você **de fato** precisar do nome do usuário em toda linha da listagem, `LAZY` sozinho reintroduz o problema N+1 (uma query por pedido) — nesse caso a solução não é voltar pra `EAGER`, e sim usar `JOIN FETCH` numa query específica pra aquela tela. Isso é avançado demais pra esse tópico, fica pra quando você estudar otimização de queries.