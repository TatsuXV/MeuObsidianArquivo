#### 1. Teoria

**JDBC (Java Database Connectivity)** é a API padrão do Java para se comunicar com bancos de dados relacionais. Ela define um conjunto de interfaces (`Connection`, `Statement`, `ResultSet`, etc.) que qualquer banco pode implementar através de um **driver** — por isso o mesmo código Java funciona com PostgreSQL, MySQL, H2, Oracle, etc., trocando só a dependência do driver e a connection string.

**Onde ele se encaixa no que você já vai estudar depois:** Spring Data JPA e Hibernate (próximos itens deste mesmo bloco) são camadas de abstração _sobre_ o JDBC. Quando você chama `repository.findById(1L)` no Spring Data JPA, por baixo dos panos alguém está montando um `PreparedStatement`, executando, e mapeando o `ResultSet` pra objeto — exatamente o que você vai fazer manualmente agora. Entender isso primeiro é o que separa quem só "usa Spring" de quem entende o que o Spring está fazendo por você (e consegue debugar quando algo foge do script).

**Peças principais:**

|Peça|Papel|
|---|---|
|`DriverManager`|Ponto de entrada mais simples pra obter uma `Connection` (em produção, isso normalmente é feito por um `DataSource` com pool de conexões — ex: HikariCP, que é o padrão do Spring Boot; isso será aprofundado quando você chegar em Spring Data JPA)|
|`Connection`|Representa a sessão aberta com o banco|
|`Statement`|Executa SQL estático, sem parâmetros|
|`PreparedStatement`|Executa SQL com parâmetros (`?`), pré-compilado pelo banco — **é o que você deve usar quase sempre**, `Statement` puro é raro em código de produção|
|`ResultSet`|O cursor sobre as linhas retornadas por um `SELECT`|

**Diferença importante — `Statement` vs `PreparedStatement`:** `Statement` monta a query concatenando string (perigoso — abre brecha pra SQL Injection). `PreparedStatement` separa o SQL dos valores: o banco recebe o SQL com `?` no lugar dos parâmetros, e os valores são enviados separadamente, tipados. Isso não é só sobre segurança — também é sobre performance (o banco pode reutilizar o plano de execução).

**Fluxo padrão de uma operação JDBC:**

1. Obter `Connection`
2. Criar `Statement`/`PreparedStatement`
3. Executar (`executeQuery` para `SELECT`, `executeUpdate` para `INSERT`/`UPDATE`/`DELETE`)
4. Processar o `ResultSet` (se houver)
5. Fechar os recursos — hoje isso é feito automaticamente com `try-with-resources`, mas você precisa entender por que isso importa: cada `Connection` não fechada é uma conexão que fica presa no banco, e bancos têm limite de conexões simultâneas.

---

#### 2. Exemplo de código comentado

Usando **H2** (banco relacional em memória, ótimo pra estudar/testar sem instalar nada — é só adicionar a dependência `com.h2database:h2` no Maven/Gradle).

java

```java
import java.sql.*;

public class JdbcExemplo {

    private static final String URL = "jdbc:h2:mem:testdb"; // banco em memória, apaga ao encerrar a JVM
    private static final String USER = "sa";
    private static final String PASSWORD = "";

    public static void main(String[] args) throws SQLException {

        // try-with-resources: Connection implementa AutoCloseable desde o Java 7 (JDBC 4.1)
        // então ela é fechada automaticamente ao sair do bloco, mesmo se der exceção
        try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD)) {

            criarTabela(conn);
            inserirProduto(conn, "Teclado mecânico", 350.00);
            inserirProduto(conn, "Mouse gamer", 180.00);

            listarProdutos(conn);
        }
    }

    private static void criarTabela(Connection conn) throws SQLException {
        // Statement puro é aceitável aqui porque não há parâmetro externo/dinâmico —
        // DDL (CREATE TABLE) normalmente não vem de input do usuário
        try (Statement stmt = conn.createStatement()) {
            stmt.execute("""
                CREATE TABLE produtos (
                    id BIGINT AUTO_INCREMENT PRIMARY KEY,
                    nome VARCHAR(100) NOT NULL,
                    preco DECIMAL(10,2) NOT NULL
                )
            """);
        }
    }

    private static void inserirProduto(Connection conn, String nome, double preco) throws SQLException {
        String sql = "INSERT INTO produtos (nome, preco) VALUES (?, ?)";

        // PreparedStatement: o "?" é o parâmetro. O índice começa em 1, não em 0.
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, nome);   // primeiro "?"
            ps.setDouble(2, preco);  // segundo "?"
            ps.executeUpdate();      // retorna int = número de linhas afetadas
        }
    }

    private static void listarProdutos(Connection conn) throws SQLException {
        String sql = "SELECT id, nome, preco FROM produtos";

        try (PreparedStatement ps = conn.prepareStatement(sql);
             ResultSet rs = ps.executeQuery()) { // executeQuery -> retorna ResultSet (usado só pra SELECT)

            while (rs.next()) { // next() move o cursor pra próxima linha; retorna false quando acaba
                long id = rs.getLong("id");         // pode buscar por nome da coluna...
                String nome = rs.getString(2);      // ...ou por índice (também começa em 1)
                double preco = rs.getDouble("preco");

                System.out.printf("ID: %d | Nome: %s | Preço: %.2f%n", id, nome, preco);
            }
        }
    }
}
```

---

#### 3. Armadilhas comuns

1. **Usar `Statement` com concatenação de string em vez de `PreparedStatement`** — abre SQL Injection na hora (`"SELECT * FROM users WHERE nome = '" + input + "'"` é uma porta aberta se `input` vier de fora). Trate isso como regra, não como exceção: se o valor vem de fora do código, é `PreparedStatement`.
2. **Esquecer que o índice de coluna/parâmetro começa em 1, não em 0** — `rs.getString(0)` lança `SQLException`, não retorna a primeira coluna. Erro clássico de quem vem de arrays/listas.
3. **Não fechar recursos (antes do `try-with-resources` virar hábito)** — cada `Connection` esquecida aberta é uma conexão presa no pool do banco. Em produção isso derruba aplicação por esgotamento de conexões, e é um bug difícil de reproduzir porque só aparece sob carga.
4. **Fazer INSERT/UPDATE em loop, um por vez, em vez de usar batch** — cada `executeUpdate()` dentro de um loop é uma ida e volta separada ao banco. Pra inserir 1000 linhas, isso é 1000 round-trips de rede. O padrão certo é `addBatch()` + `executeBatch()` (você vai praticar isso no exercício 4).

---

#### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Crie uma tabela `usuarios` com colunas `id` (auto increment, PK), `nome` (VARCHAR) e `email` (VARCHAR). Escreva um método `inserirUsuario(Connection conn, String nome, String email)` usando `PreparedStatement`. Critério de pronto: rodar o método duas vezes com dados diferentes e confirmar via `SELECT` que ambos os registros existem.

**Exercício 2 (médio)**  
Escreva um método `buscarUsuarioPorEmail(Connection conn, String email)` que retorna o nome do usuário correspondente, ou `null` se não encontrar ninguém. Use `ResultSet` corretamente (cuidado: o que acontece se você chamar `rs.getString()` sem antes chamar `rs.next()`?). Critério de pronto: funciona tanto pra email existente quanto pra email inexistente, sem lançar exceção no segundo caso.

**Exercício 3 (difícil)**  
Escreva um método `atualizarEmailUsuario(Connection conn, long id, String novoEmail)` que retorna `boolean` — `true` se algum registro foi de fato atualizado, `false` se o `id` não existia. Dica: `executeUpdate()` retorna quantas linhas foram afetadas — use isso, não faça um `SELECT` antes só pra checar existência (isso seria uma query desnecessária). Critério de pronto: `true` para id existente, `false` para id inexistente.

**Exercício 4 (desafio)**  
Escreva um método `inserirUsuariosEmLote(Connection conn, List<String[]> usuarios)` onde cada `String[]` é `{nome, email}`. Use `addBatch()` e `executeBatch()` em vez de chamar `executeUpdate()` dentro de um loop. Critério de pronto: inserir uma lista com pelo menos 5 usuários numa única chamada de batch, e confirmar que todos foram persistidos.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
private static void criarTabelaUsuarios(Connection conn) throws SQLException {
    try (Statement stmt = conn.createStatement()) {
        stmt.execute("""
            CREATE TABLE usuarios (
                id BIGINT AUTO_INCREMENT PRIMARY KEY,
                nome VARCHAR(100) NOT NULL,
                email VARCHAR(100) NOT NULL
            )
        """);
    }
}

private static void inserirUsuario(Connection conn, String nome, String email) throws SQLException {
    String sql = "INSERT INTO usuarios (nome, email) VALUES (?, ?)";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, nome);
        ps.setString(2, email);
        ps.executeUpdate();
    }
}
```

Raciocínio: nada além do que já foi mostrado no exemplo — o objetivo aqui é fixar o padrão `prepareStatement` → `set*` → `executeUpdate`.

**Exercício 2**

java

```java
private static String buscarUsuarioPorEmail(Connection conn, String email) throws SQLException {
    String sql = "SELECT nome FROM usuarios WHERE email = ?";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, email);
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {          // next() retorna true SE existe uma linha pra ler
                return rs.getString("nome");
            }
            return null;               // se next() retornou false, não existe usuário com esse email
        }
    }
}
```

Raciocínio: a armadilha aqui é justamente o item 2 da seção anterior — `rs.next()` precisa ser chamado _antes_ de qualquer `get*()`, senão o cursor não está posicionado em linha nenhuma e você recebe `SQLException`. Como só esperamos 0 ou 1 resultado (email deveria ser único), usamos `if`, não `while`.

**Exercício 3**

java

```java
private static boolean atualizarEmailUsuario(Connection conn, long id, String novoEmail) throws SQLException {
    String sql = "UPDATE usuarios SET email = ? WHERE id = ?";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, novoEmail);
        ps.setLong(2, id);
        int linhasAfetadas = ps.executeUpdate(); // UPDATE/DELETE também usam executeUpdate, não executeQuery
        return linhasAfetadas > 0;
    }
}
```

Raciocínio: `executeUpdate()` já te diz quantas linhas mudaram — usar isso evita uma query extra de `SELECT ... WHERE id = ?` só pra checar existência antes de atualizar. É uma única ida ao banco em vez de duas. Esse tipo de raciocínio ("preciso mesmo dessa query extra, ou o retorno da operação já me dá a resposta?") é o tipo de coisa que separa código júnior de código que passa em review sênior.

**Exercício 4**

java

```java
private static void inserirUsuariosEmLote(Connection conn, List<String[]> usuarios) throws SQLException {
    String sql = "INSERT INTO usuarios (nome, email) VALUES (?, ?)";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        for (String[] usuario : usuarios) {
            ps.setString(1, usuario[0]);
            ps.setString(2, usuario[1]);
            ps.addBatch();      // acumula o comando, não executa ainda
        }
        ps.executeBatch();      // envia todos os comandos acumulados numa única ida ao banco
    }
}
```

Raciocínio: a diferença central pro erro do item 4 da seção de armadilhas é que `addBatch()` acumula os parâmetros no driver, e `executeBatch()` manda tudo de uma vez — o banco processa em lote, e a rede só é atravessada uma vez em vez de N vezes. Existe uma alternativa (usar `Statement.addBatch(String sql)` com SQL completo em vez de `PreparedStatement`), mas ela reintroduz o problema de concatenação de string do item 1 das armadilhas — prefira sempre a versão com `PreparedStatement` batch, mesmo custando uma linha a mais de setup.