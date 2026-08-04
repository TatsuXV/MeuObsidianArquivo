#### 1. Teoria

**Por que existe uma ferramenta de build**

Uma aplicação Java real não é só "compilar um `.java`". Você precisa: baixar bibliotecas de terceiros (e as bibliotecas que _essas_ bibliotecas dependem, em cascata), compilar tudo na ordem certa, rodar testes automatizados, empacotar num `.jar`/`.war`, e às vezes publicar esse artefato em algum lugar. Maven automatiza e padroniza esse processo inteiro através de um arquivo de configuração declarativo: o `pom.xml`.

**POM — Project Object Model**

É o `pom.xml`, a "receita" do projeto. Nele você declara: identidade do projeto (`groupId`, `artifactId`, `version`), dependências, plugins, e configurações de build. A ideia central do Maven é **convenção sobre configuração**: se você seguir a estrutura de pastas padrão, quase nada precisa ser configurado manualmente.

**Coordenadas Maven (GAV)**

Toda dependência (e todo projeto Maven) é identificada por três coordenadas:

- `groupId` — geralmente o domínio invertido da organização (ex: `org.springframework.boot`)
- `artifactId` — nome do módulo/biblioteca (ex: `spring-boot-starter-web`)
- `version` — versão específica (ex: `4.1.0`)

Essas três coordenadas juntas apontam pra um artefato único num repositório (Maven Central, por padrão).

**Estrutura de diretórios padrão**

```
meu-projeto/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/        ← seu código-fonte
│   │   └── resources/   ← application.properties, arquivos estáticos, etc.
│   └── test/
│       ├── java/        ← código de teste
│       └── resources/   ← recursos usados só nos testes
└── target/               ← gerado pelo Maven (compilados, jar final) — não versiona no Git
```

Essa convenção é o motivo de você quase nunca precisar dizer ao Maven "onde está meu código" — ele já sabe, porque você seguiu o padrão.

**Ciclo de vida de build (build lifecycle)**

Maven tem três ciclos de vida independentes, mas o que você usa 95% do tempo é o **default**, que é uma sequência de fases. Cada fase executa todas as fases anteriores automaticamente:

|Fase|O que faz|
|---|---|
|`validate`|Confere se o projeto está correto e todas as informações necessárias estão disponíveis|
|`compile`|Compila o código-fonte principal (`src/main/java`)|
|`test`|Roda os testes unitários (`src/test/java`), usando o código já compilado|
|`package`|Empacota o compilado no formato definido (`.jar` ou `.war`)|
|`verify`|Roda verificações adicionais sobre resultados de testes de integração (se houver)|
|`install`|Instala o pacote no repositório Maven **local** (`~/.m2`), disponível para outros projetos locais|
|`deploy`|Copia o pacote final para um repositório **remoto**, para compartilhar com outros devs/CI|

Ou seja: rodar `mvn install` executa `validate → compile → test → package → verify → install`, nessa ordem, sempre. É por isso que `mvn package` já roda os testes antes de empacotar — não é comportamento separado, é consequência direta do ciclo de vida.

**Escopo de dependência (`scope`)**

Controla _quando_ uma dependência está disponível:

- `compile` (padrão) — disponível em todas as fases, e vai junto no artefato final.
- `test` — disponível só para compilar/rodar testes (ex: JUnit, Mockito). Não vai pro `.jar` final.
- `provided` — disponível em compilação, mas espera-se que o ambiente de execução já forneça isso (ex: certas dependências de servlet container).
- `runtime` — não necessário para compilar, só para rodar (ex: driver JDBC, dependendo de como o código acessa ele).

**Dependências transitivas**

Se o seu projeto depende da biblioteca A, e A depende da biblioteca B, o Maven baixa B automaticamente para você — isso é uma dependência **transitiva**. É extremamente conveniente, mas também é fonte de um problema comum: **conflito de versão**, quando duas dependências diferentes do seu projeto trazem versões diferentes da mesma biblioteca transitiva. O Maven resolve isso com uma estratégia de "mais próximo no grafo de dependência vence" — mas às vezes você precisa intervir manualmente com `<exclusions>`.

**`parent` POM e `dependencyManagement`**

Em projetos Spring Boot, você normalmente herda de um `parent` (`spring-boot-starter-parent`) ou importa um BOM (_Bill of Materials_) via `dependencyManagement`. Isso centraliza as **versões** de um conjunto grande de dependências compatíveis entre si — é por isso que, em projetos Spring Boot, você normalmente não escreve `<version>` em cada dependência individual: a versão já foi decidida pelo parent/BOM, você só declara _quais_ dependências quer.

**Plugins**

Maven, por padrão, só sabe compilar e empacotar Java puro. Comportamentos extras vêm de plugins — o mais relevante pra você agora é o `spring-boot-maven-plugin`, que adiciona o goal `repackage`, responsável por transformar o `.jar` gerado num **jar executável** (com todas as dependências embutidas dentro dele — o chamado _fat jar_ / _über-jar_), permitindo rodar com `java -jar`.

---

#### 2. Exemplo de código comentado

xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                              http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- Herda configurações e versões padrão do Spring Boot,
         incluindo plugins e dependencyManagement -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.1.0</version>
    </parent>

    <!-- Coordenadas do SEU projeto -->
    <groupId>com.exemplo</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <!-- SNAPSHOT = versão em desenvolvimento, ainda não é um release final -->

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Starter web: traz Spring MVC + Tomcat embutido + Jackson.
             Repare que NÃO tem <version> aqui — quem decide a versão
             é o parent (spring-boot-starter-parent) via dependencyManagement -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Dependência de teste: só disponível durante mvn test,
             não entra no jar final de produção -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Sem esse plugin, "mvn package" gera um jar comum,
                 sem as dependências embutidas — não roda sozinho com java -jar -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

Comandos de terminal correspondentes (o que cada um dispara no ciclo de vida):

bash

```bash
mvn compile     # roda validate + compile
mvn test        # roda validate + compile + test
mvn package     # roda tudo até package (gera o .jar em target/)
mvn clean package  # 'clean' é de um ciclo de vida SEPARADO — apaga target/ antes de reconstruir
mvn spring-boot:run  # goal específico do plugin: roda a aplicação direto, sem empacotar
```

---

#### 3. Armadilhas comuns

1. **Confundir `mvn clean` com fase do ciclo `default`** — `clean` pertence a um ciclo de vida totalmente separado (o ciclo _clean_, que só tem `pre-clean`, `clean`, `post-clean`). É por isso que você escreve `mvn clean package` (dois comandos "encadeados" na mesma chamada) e não existe uma fase `clean` dentro da sequência `validate → compile → ...`. Sem `clean`, arquivos compilados antigos podem permanecer em `target/` e mascarar problemas.
2. **Não entender por que uma dependência "some" ao rodar `mvn package`** — geralmente é confusão de `scope`. Uma dependência com `scope test` (como JUnit) nunca deveria estar disponível no código de `src/main` — se você importar Mockito lá por engano, o build vai falhar dizendo que a classe não foi encontrada em tempo de compilação principal, mesmo que `mvn test` funcione (porque na fase `test` esse escopo _está_ disponível).
3. **Deixar `-SNAPSHOT` em produção sem perceber** — versões `SNAPSHOT` são mutáveis (podem ser sobrescritas no repositório a qualquer redeploy), o que é ótimo durante desenvolvimento, mas perigoso pra um artefato que vai pra produção — você perde garantia de reprodutibilidade (duas pessoas rodando `mvn install` em momentos diferentes podem obter jars ligeiramente diferentes da "mesma" versão SNAPSHOT).
4. **Ignorar conflito de versão transitiva até o app quebrar em runtime** — dois `NoSuchMethodError` ou `ClassNotFoundException` inexplicáveis em produção, quando o código compila normalmente, é sinal clássico de conflito de versão entre dependências transitivas. O comando `mvn dependency:tree` mostra a árvore completa de dependências e ajuda a identificar duplicatas/conflitos — vale conhecer esse comando antes de precisar dele em pânico.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Sem gerar projeto nenhum ainda: escreva, de memória (depois confira), a ordem exata das fases do ciclo de vida `default` do Maven, do início até `install`. Em seguida, explique em uma frase por que rodar `mvn install` também executa os testes do projeto.

**Exercício 2 (Fácil/Médio)**  
Gere um projeto Spring Boot mínimo (via start.spring.io, com dependência `Spring Web`), baixe/descompacte, e rode `mvn clean package` no terminal, dentro da pasta do projeto. Cole a estrutura de pastas gerada dentro de `target/` após o comando (pode usar `ls target/` ou equivalente) e identifique qual arquivo é o jar executável final.

**Exercício 3 (Médio)**  
No `pom.xml` do projeto gerado no exercício 2, adicione a dependência `org.apache.commons:commons-lang3` (escolha uma versão recente, confira no Maven Central) com `scope` padrão (`compile`). Escreva uma classe simples que use algum método utilitário dessa biblioteca (ex: `StringUtils.isBlank(...)`) e rode `mvn compile` para confirmar que compila sem erro. Depois, mude o `scope` dessa dependência para `test` e tente rodar `mvn compile` de novo usando a mesma classe em `src/main/java` — descreva o que acontece e por quê.

**Exercício 4 (Desafio)**  
Rode `mvn dependency:tree` no projeto do exercício 2 e cole o resultado. Identifique: (a) pelo menos duas dependências transitivas que vieram de `spring-boot-starter-web` sem você ter declarado diretamente; (b) o `scope` de cada uma delas na árvore. Em seguida, adicione a dependência `spring-boot-starter-test` (se ainda não tiver) e explique, olhando a árvore, por que ela sozinha já traz JUnit, Mockito e AssertJ juntos — que mecanismo do Maven possibilita isso.

---

#### 5. Gabarito comentado

**Exercício 1**

Ordem: `validate → compile → test → package → verify → install`.

_Raciocínio:_ cada fase do ciclo `default` executa **todas as fases anteriores a ela** antes de rodar a sua própria lógica — isso é uma característica central do mecanismo, não uma coincidência de configuração. `install` depende de `package`, que depende de `test`, que depende de `compile`, que depende de `validate`. Não existe como "pular" a fase `test` chamando `install` diretamente (a única forma seria desabilitar testes explicitamente com uma flag como `-DskipTests`, decisão consciente e não o comportamento padrão).

**Exercício 2**

Resposta esperada (pode variar levemente por versão do Spring Boot/plugin):

```
target/
├── classes/                    ← .class compilados de src/main/java
├── test-classes/               ← .class compilados de src/test/java
├── demo-0.0.1-SNAPSHOT.jar          ← jar "fino", só o código do projeto
└── demo-0.0.1-SNAPSHOT.jar.original ← backup do jar antes do repackage
```

_Raciocínio:_ o arquivo executável final é `demo-0.0.1-SNAPSHOT.jar` (sem o sufixo `.original`). O `.jar.original` existe porque o `spring-boot-maven-plugin` primeiro deixa o Maven gerar o jar "normal" (só as classes do seu projeto), e depois o goal `repackage` pega esse jar, renomeia pra `.original` como backup, e gera um novo `.jar` com o mesmo nome — agora contendo todas as dependências embutidas. É esse jar final "gordo" que roda sozinho com `java -jar demo-0.0.1-SNAPSHOT.jar`.

**Exercício 3**

Com `scope compile` (padrão), a classe em `src/main/java` compila normalmente — a dependência está disponível nessa fase.

Ao mudar para `scope test`, `mvn compile` **falha**, com erro de "package org.apache.commons.lang3 does not exist" (ou equivalente, dependendo da versão do compilador/Maven).

_Raciocínio:_ isso ilustra exatamente a Armadilha 2 da seção anterior. `scope test` só disponibiliza a dependência no classpath usado para compilar e rodar `src/test/java` — a fase `compile` (que processa só `src/main/java`) nunca enxerga essa dependência. É o Maven fazendo exatamente o que foi configurado pra fazer: impedir que código de produção dependa acidentalmente de algo pensado só pra testes.

**Exercício 4**

Exemplo de trecho esperado de saída (a versão exata varia conforme a versão do Spring Boot usada):

```
[INFO] com.exemplo:demo:jar:0.0.1-SNAPSHOT
[INFO] \- org.springframework.boot:spring-boot-starter-web:jar:4.1.0:compile
[INFO]    +- org.springframework.boot:spring-boot-starter:jar:4.1.0:compile
[INFO]    +- org.springframework.boot:spring-boot-starter-json:jar:4.1.0:compile
[INFO]    |  +- com.fasterxml.jackson.core:jackson-databind:jar:...:compile
[INFO]    +- org.springframework.boot:spring-boot-starter-tomcat:jar:4.1.0:compile
[INFO]    |  \- org.apache.tomcat.embed:tomcat-embed-core:jar:...:compile
[INFO]    \- org.springframework:spring-web:jar:...:compile
```

_Raciocínio:_ `jackson-databind` (serialização JSON) e `tomcat-embed-core` (servidor embutido) são exemplos de dependências transitivas — você nunca escreveu `<dependency>` pra elas, mas `spring-boot-starter-web` as trouxe automaticamente, todas com `scope compile`, porque fazem parte do funcionamento básico de uma aplicação web Spring Boot.

Sobre `spring-boot-starter-test`: ela é, ela mesma, um "starter agregador" — uma dependência que não contém código próprio relevante, só existe pra declarar, via suas _próprias_ dependências transitivas, um conjunto coeso de bibliotecas de teste (JUnit 5, Mockito, AssertJ, e outras) com versões já compatíveis entre si, decididas centralmente pelo parent/BOM do Spring Boot. É o mesmo mecanismo de dependência transitiva do parágrafo anterior — só que usado deliberadamente como um "pacote de conveniência", em vez de ser um efeito colateral incidental.