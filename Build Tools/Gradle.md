#### 1. Teoria

**O que muda em relação ao Maven**

Gradle resolve o mesmo problema que o Maven (build, dependências, empacotamento), mas com diferenças de fundo:

||Maven|Gradle|
|---|---|---|
|Arquivo de config|`pom.xml` (XML declarativo)|`build.gradle` (Groovy) ou `build.gradle.kts` (Kotlin DSL) — script de verdade, não só declaração|
|Modelo de execução|Ciclo de vida fixo de fases|**Tasks** — grafo de tarefas que você pode customizar e criar novas|
|Performance|Reexecuta o que a fase manda|_Incremental build_ + _build cache_: só refaz o que realmente mudou desde o último build|
|Curva de aprendizado|Mais rígido, mais verboso, mais previsível|Mais flexível, mas exige entender que é um script (mais poder = mais formas de errar)|

Isso explica a percepção de mercado: Maven domina projetos legados porque é mais padronizado e "chato de errar"; Gradle vem crescendo em projetos novos por causa de performance de build e flexibilidade, especialmente em builds grandes/multi-módulo.

**`build.gradle.kts` — o equivalente ao `pom.xml`**

A ideia central é a mesma do Maven (declarar plugins, dependências, propriedades), só que sintaticamente como código Kotlin (ou Groovy), não XML.

**Tasks em vez de fases fixas**

No Maven você tem fases fixas (`compile`, `test`, `package`...). No Gradle, tudo é uma **task**, e as tasks padrão (`compileJava`, `test`, `build`, `bootRun`) já vêm de plugins aplicados (`java`, `org.springframework.boot`), mas você pode criar tasks customizadas quando precisar. Ainda existe uma noção de dependência entre tasks (`build` depende de `test`, que depende de `compileJava`), só que o modelo é um **grafo de tasks**, mais flexível que a sequência fixa do Maven.

**Configurations em vez de scope**

O que o Maven chama de `scope` (`compile`, `test`, `provided`, `runtime`), o Gradle chama de **configuration**, com nomes um pouco diferentes:

- `implementation` — equivalente a `scope compile`, mas com uma diferença importante: dependências declaradas como `implementation` **não vazam** para quem depende do seu módulo (encapsulamento melhor entre módulos). É a opção padrão recomendada hoje.
- `api` — igual ao antigo `compile` do Maven no sentido de "vaza pra frente" — usado quando você quer expor essa dependência para quem consome seu módulo.
- `testImplementation` — equivalente a `scope test`.
- `runtimeOnly` — equivalente a `scope runtime`.

**Repositórios e Maven Central**

Mesmo sendo "Gradle", ele normalmente busca dependências no mesmo lugar que o Maven — o Maven Central — declarado com `repositories { mavenCentral() }`. As coordenadas de dependência (`groupId:artifactId:version`) são as mesmas, só a sintaxe de declaração muda.

**O plugin do Spring Boot pro Gradle**

Funciona de forma análoga ao `spring-boot-maven-plugin`: adiciona a task `bootJar` (equivalente ao goal `repackage` do Maven — gera o jar executável com dependências embutidas) e a task `bootRun` (equivalente a `mvn spring-boot:run`). Também traz o plugin `io.spring.dependency-management`, que cumpre o mesmo papel do `parent`/BOM do Maven: você declara dependências do ecossistema Spring **sem versão**, e a versão certa é resolvida centralmente.

---

#### 2. Exemplo de código comentado

kotlin

```kotlin
// build.gradle.kts

plugins {
    java
    id("org.springframework.boot") version "4.1.0"
    id("io.spring.dependency-management") version "1.1.7"
    // equivalente ao <parent> do Maven: resolve versões compatíveis entre si
}

group = "com.exemplo"
version = "0.0.1-SNAPSHOT"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

repositories {
    mavenCentral() // mesmo repositório usado pelo Maven por padrão
}

dependencies {
    // "implementation" ≈ scope compile do Maven, mas sem vazar para módulos que dependem deste
    implementation("org.springframework.boot:spring-boot-starter-web")
    // sem <version> aqui — resolvido pelo dependency-management, igual ao BOM do Maven

    // ≈ scope test do Maven: só disponível para compilar/rodar testes
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

Comandos de terminal equivalentes ao que você já viu em Maven:

bash

```bash
./gradlew build       # ≈ mvn package (compila, testa, empacota)
./gradlew test        # ≈ mvn test
./gradlew bootRun      # ≈ mvn spring-boot:run
./gradlew clean build  # limpa e reconstrói — mas aqui 'clean' é uma task normal,
                        # não um ciclo de vida separado como no Maven
```

> `./gradlew` é o **Gradle Wrapper** — um script incluído no projeto que baixa e usa a versão exata do Gradle configurada, sem exigir instalação global. É o motivo de você quase nunca rodar `gradle` diretamente, e sim `./gradlew` (ou `gradlew.bat` no Windows).

---

#### 3. Armadilhas comuns

1. **Achar que `build.gradle.kts` é só um "pom.xml com sintaxe diferente"** — na prática, é um script executável. Isso dá poder (lógica condicional, funções, etc.), mas também significa que build lento ou build "mágico demais" geralmente vem de gente colocando lógica complexa demais no próprio arquivo de build, dificultando manutenção. Simplicidade aqui vale tanto quanto no seu código Java.
2. **Confundir `implementation` com `api` sem necessidade** — usar `api` por padrão (só porque parece "mais permissivo") faz vazar dependências transitivas desnecessariamente para quem consome seu módulo, aumentando acoplamento. Em projetos de aplicação única (não biblioteca), isso raramente importa — mas em projetos multi-módulo é uma decisão real de design.
3. **Não versionar o Gradle Wrapper (`gradlew`, `gradlew.bat`, pasta `gradle/wrapper/`)** — sem esses arquivos no repositório Git, outra pessoa que clonar o projeto não tem garantia de rodar a mesma versão do Gradle que você, podendo gerar builds diferentes ou até quebrar.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Sem gerar projeto: escreva a tabela de equivalência entre os termos que você já usa em Maven e o nome correspondente em Gradle, para os seguintes quatro conceitos: (a) arquivo de configuração principal; (b) escopo "só para testes"; (c) comando que gera o jar final empacotado; (d) comando que roda a aplicação sem empacotar.

**Exercício 2 (Fácil/Médio)**  
Gere um projeto Spring Boot mínimo em start.spring.io escolhendo **Gradle - Kotlin** como build tool (em vez de Maven, como nos exercícios anteriores), com dependência `Spring Web`. Rode `./gradlew build` no terminal. Critério de pronto: o build completa sem erro e você consegue localizar o jar executável gerado (procure dentro de `build/libs/`, que é o equivalente ao `target/` do Maven).

**Exercício 3 (Médio)**  
No `build.gradle.kts` do projeto do exercício 2, adicione a mesma dependência `org.apache.commons:commons-lang3` que você usou no exercício de Maven, só que agora como `implementation`. Escreva a mesma classe de teste usando `StringUtils.isBlank(...)` e rode `./gradlew build` para confirmar que compila. Depois, mude a dependência para `testImplementation` e tente compilar de novo com a classe ainda em `src/main/kotlin` (ou `src/main/java`, dependendo de como você gerou o projeto) — descreva o resultado e compare com o que aconteceu no exercício equivalente de Maven.

---

#### 5. Gabarito comentado

**Exercício 1**

|Conceito|Maven|Gradle|
|---|---|---|
|Arquivo de configuração principal|`pom.xml`|`build.gradle.kts` (ou `build.gradle`)|
|Escopo "só para testes"|`scope test`|`testImplementation`|
|Gera o jar final empacotado|`mvn package` (ou `mvn clean package`)|`./gradlew build` (task `bootJar` especificamente gera o executável)|
|Roda a aplicação sem empacotar|`mvn spring-boot:run`|`./gradlew bootRun`|

_Raciocínio:_ o objetivo desse exercício é consolidar que as duas ferramentas resolvem exatamente os mesmos problemas — a barreira real de aprender a segunda depois da primeira é vocabulário/sintaxe, não conceito novo.

**Exercício 2**

Estrutura esperada em `build/libs/`:

```
build/
└── libs/
    ├── demo-0.0.1-SNAPSHOT.jar          ← jar executável final (com dependências embutidas)
    └── demo-0.0.1-SNAPSHOT-plain.jar    ← jar "fino", só as classes do projeto
```

_Raciocínio:_ repare que a lógica é a mesma do Maven (`target/demo-...jar` + `.jar.original`), só o nome do arquivo "fino" muda de convenção (`-plain.jar` em vez de `.jar.original`) e a pasta muda de `target/` para `build/libs/`. O mecanismo por trás — o plugin do Spring Boot pegando o jar simples e reempacotando com as dependências dentro — é conceitualmente idêntico ao que você já viu.

**Exercício 3**

Com `implementation`, a classe em `src/main` compila normalmente — mesmo resultado do exercício equivalente em Maven com `scope compile`.

Ao mudar para `testImplementation`, `./gradlew build` falha ao compilar `src/main`, com erro de que o pacote `org.apache.commons.lang3` não existe — pelo mesmo motivo do Maven com `scope test`: a configuration `testImplementation` só disponibiliza a dependência no classpath de compilação/execução de **testes** (`src/test`), nunca no de `src/main`.

_Raciocínio:_ essa é exatamente a mesma regra que você já validou em Maven, só que com nome de configuration diferente — reforça que "escopo/configuration" não é um detalhe de sintaxe, é uma regra real do build que existe nas duas ferramentas porque resolve o mesmo problema real (impedir que código de produção dependa acidentalmente de biblioteca de teste).