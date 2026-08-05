#### 1. Teoria

**Por que não usar `System.out.println`?**  
Porque log não é sobre "aparecer no console" — é sobre criar um histórico consultável, com nível de severidade, timestamp, thread, e (em produção) formato estruturado que outra ferramenta vai ler (ex: ELK, Datadog, Grafana Loki). `println` não tem nível, não pode ser desligado seletivamente, não vai pra arquivo sem você reescrever tudo na mão, e não sabe qual thread ou request gerou aquela linha.

**SLF4J vs Logback — a distinção que mais confunde iniciante**

- **SLF4J** (_Simple Logging Facade for Java_) é uma **API/interface**, não faz logging sozinho. Você programa contra `org.slf4j.Logger` e `org.slf4j.LoggerFactory`.
- **Logback** é uma **implementação real** que faz o trabalho de fato (formata, escreve em console/arquivo, gira arquivo por tamanho/data, etc).

Isso existe pra desacoplar seu código da biblioteca concreta: se amanhã o time decidir trocar Logback por outra implementação, seu código de negócio (que só conhece `Logger` do SLF4J) não muda uma linha. É o mesmo princípio de "programar contra a interface, não a implementação" que você já viu em OOP (Bloco 3).

Quando você cria um projeto com `spring-boot-starter-web` (ou qualquer starter), o Spring Boot já traz `spring-boot-starter-logging`, que resolve SLF4J + Logback prontos, sem você precisar declarar nada — é por isso que essa dupla é a **[ESSENCIAL]** do checklist.

**Níveis de log (ordem de severidade crescente)**

```
TRACE < DEBUG < INFO < WARN < ERROR
```

Quando você configura um nível (ex: `INFO`), o logger emite aquele nível **e tudo acima dele** (`INFO`, `WARN`, `ERROR`), mas ignora o que está abaixo (`TRACE`, `DEBUG`). Isso é o que te permite deixar `logger.debug(...)` espalhado pelo código o tempo todo e só "ligar" quando precisar investigar um bug em produção, mudando a configuração — sem tocar no código.

**Logging parametrizado (o motivo de usar `{}` em vez de `+`)**

java

```java
logger.info("Usuário {} fez login às {}", usuarioId, horario);
```

Isso não é só estética. Se o nível `INFO` estiver desabilitado, o SLF4J **nem monta a String** — o custo de formatação só acontece se a mensagem realmente vai ser emitida. Com concatenação manual (`"Usuário " + usuarioId + ...`), a concatenação acontece **sempre**, mesmo que o log nunca seja escrito.

**MDC (Mapped Diagnostic Context)**

É um mapa chave-valor **por thread** que o Logback pode injetar automaticamente em toda linha de log daquela thread — o uso mais comum em backend é colocar um `requestId` no início de uma requisição HTTP, pra depois conseguir filtrar/agrupar todos os logs daquela requisição específica no meio de milhares de linhas de outras requisições concorrentes.

**Arquitetura do Logback (os 3 conceitos que aparecem em toda config)**

- **Logger**: quem recebe a chamada `logger.info(...)` no seu código. Tem nome (geralmente o nome da classe) e existe numa hierarquia (logger de pacote herda do root, salvo configuração contrária).
- **Appender**: pra onde o log vai (console, arquivo, etc). Um logger pode ter vários appenders.
- **Encoder/Layout**: como a linha é formatada (timestamp, nível, thread, mensagem).

**`logback.xml` vs `logback-spring.xml` — qual usar no Spring Boot**

Ambos funcionam, mas a documentação oficial do Spring Boot recomenda `logback-spring.xml`: com esse nome, o Spring Boot processa o arquivo antes de repassar pro Logback, o que habilita extensões próprias do Boot — a mais usada na prática é a tag `<springProfile name="dev">`, que permite ter configuração de log diferente por ambiente (dev/prod) dentro do mesmo arquivo. Você pode adicionar um arquivo logback.xml na raiz do classpath para o logback encontrar, ou usar logback-spring.xml se quiser usar as extensões do Spring Boot para Logback, que fornece várias configurações prontas para serem incluídas na sua própria configuração. [Spring](https://docs.spring.io/spring-boot/docs/2.1.8.RELEASE/reference/html/howto-logging.html)

Um detalhe que costuma confundir: **sem nenhum arquivo de configuração**, o Logback "puro" cai num modo mínimo automático (`BasicConfigurator`), que anexa um ConsoleAppender ao logger root com nível DEBUG por padrão. Só que dentro de um projeto Spring Boot isso raramente acontece "cru" — o próprio Spring Boot já aplica sua configuração padrão (nível **INFO** no root) antes disso entrar em jogo. Ou seja: Logback sozinho → DEBUG por padrão; Spring Boot → INFO por padrão. [Logback](https://logback.qos.ch/manual/configuration.html)

---

#### 2. Exemplo de código comentado

java

```java
package com.example.pedidos.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

public class PedidoService {

    // 1. Um Logger por classe, como campo estático e final.
    //    LoggerFactory.getLogger(Class) usa o nome totalmente qualificado da classe
    //    como "nome do logger" -- é isso que aparece no output e permite configurar
    //    nível por pacote/classe depois, no logback-spring.xml.
    private static final Logger logger = LoggerFactory.getLogger(PedidoService.class);

    public void processarPedido(String pedidoId, double valor) {
        // 2. MDC: guarda um valor num mapa thread-local que o Logback pode
        //    imprimir automaticamente em toda linha de log dessa thread,
        //    sem repetir "pedidoId=" manualmente em cada chamada.
        MDC.put("pedidoId", pedidoId);

        try {
            logger.info("Iniciando processamento do pedido");

            // 3. Logging parametrizado: {} é substituído pelo argumento.
            //    A montagem da String só acontece se o nível INFO estiver ativo.
            logger.info("Valor do pedido: {}", valor);

            if (valor <= 0) {
                // 4. WARN: situação estranha, mas que não impede o fluxo.
                logger.warn("Pedido com valor não positivo: {}", valor);
            }

            validar(pedidoId, valor);
            logger.info("Pedido processado com sucesso");

        } catch (IllegalArgumentException e) {
            // 5. Ao logar uma exception, passe o Throwable como último argumento
            //    (não faça e.getMessage() concatenado). Isso faz o Logback
            //    imprimir a stack trace inteira -- essencial pra debugar produção.
            logger.error("Falha ao validar pedido {}: {}", pedidoId, e.getMessage(), e);
            throw e;

        } finally {
            // 6. MDC é thread-local, e em servidores web a thread é reaproveitada
            //    (thread pool). Se não limpar, a próxima requisição na mesma
            //    thread pode "herdar" esse pedidoId por engano. SEMPRE limpe.
            MDC.remove("pedidoId");
        }
    }

    private void validar(String pedidoId, double valor) {
        logger.debug("Validando regras de negócio para pedido {}", pedidoId);
        if (pedidoId == null || pedidoId.isBlank()) {
            throw new IllegalArgumentException("pedidoId não pode ser vazio");
        }
    }
}
```

`logback-spring.xml` correspondente (fica em `src/main/resources`):

xml

```xml
<configuration>

    <!-- Reaproveita o padrão de console que o próprio Spring Boot já define
         (cores, timestamp, nome da thread) em vez de reescrever do zero -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- springProfile só funciona porque o arquivo se chama logback-spring.xml -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

</configuration>
```

---

#### 3. Armadilhas comuns

1. **Usar `System.out.println` "só pra debugar rápido" e deixar no código.** Sem nível, sem controle de ambiente, sem possibilidade de desligar em produção sem recompilar. Se precisa investigar algo, use `logger.debug(...)` — já nasce desligável.
2. **Concatenar String manualmente em vez de usar `{}`.** Além de menos legível, desperdiça performance mesmo quando o nível está desabilitado, porque a concatenação roda independentemente do log ser emitido ou não.
3. **Vazamento de contexto no MDC.** Colocar valor no MDC (ex: `requestId`) e esquecer o `MDC.remove(...)` no `finally`. Em uma aplicação web com pool de threads, a próxima requisição atendida pela mesma thread herda o valor antigo por engano — gerando logs com `requestId` errado, que é péssimo pra debugar em produção justamente quando você mais precisa confiar no log.
4. **Logar dado sensível sem mascarar** (senha, token, número de cartão, CPF completo). É erro comum de iniciante logar o objeto request inteiro "pra debugar" e isso acabar em arquivo de log ou ferramenta de terceiros — em ambiente real isso é falha de segurança, não só descuido.

---

#### 4. Exercícios práticos

**1. Fácil**  
Crie uma classe `NiveisDemo` com um `Logger` (SLF4J) e um método `main` que:

- Loga uma mensagem em cada um dos 5 níveis (`trace`, `debug`, `info`, `warn`, `error`).
- Usa logging parametrizado (`{}`) em pelo menos 2 dessas chamadas, passando uma variável.

Critério de pronto: compila e roda; você consegue explicar, sem rodar o código, quais níveis apareceriam no console com a configuração padrão do Logback puro (sem nenhum arquivo de config) e quais apareceriam com a configuração padrão do Spring Boot.

**2. Médio**  
Crie uma classe `ValidadorIdade` com um método `validar(int idade)` que:

- Lança `IllegalArgumentException` se `idade < 18`.
- Loga `INFO` quando a idade é válida, `WARN` quando é negativa (caso inválido "estranho", tipo -5), e `ERROR` com stack trace completa quando lança a exception por ser menor de idade (ex: idade = 15).

Critério de pronto: os 3 cenários (válido, negativo, menor de idade) geram log no nível correto, e o log de erro mostra a stack trace, não só a mensagem.

**3. Difícil**  
Simule um "processamento de requisições" com um método `processarRequisicao()` chamado 3 vezes em sequência (num loop), onde cada chamada:

- Gera um `UUID` como `requestId` e coloca no MDC no início.
- Loga pelo menos 2 mensagens durante o processamento.
- Remove o `requestId` do MDC no final (mesmo se der exception no meio).

Critério de pronto: ao rodar as 3 chamadas, cada bloco de log mostra um `requestId` diferente e nenhum log de uma chamada aparece com o `requestId` de outra — prove isso mostrando o output.

**4. Desafio**  
Configure um `logback-spring.xml` com:

- Um `ConsoleAppender` mostrando apenas `INFO` pra cima.
- Um `RollingFileAppender` escrevendo em arquivo tudo a partir de `DEBUG`, com rotação diária (um arquivo novo por dia).
- Um logger específico para o pacote `com.example.pedidos` configurado em `DEBUG`, mesmo que o root esteja em `INFO`.

Critério de pronto: rodando a aplicação, o console mostra só `INFO+`, o arquivo mostra `DEBUG+`, e uma classe fora do pacote `com.example.pedidos` (ex: uma lib de terceiro) não polui o log em `DEBUG` — só o seu pacote fica verboso.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
package com.example.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class NiveisDemo {
    private static final Logger logger = LoggerFactory.getLogger(NiveisDemo.class);

    public static void main(String[] args) {
        int usuarioId = 42;

        logger.trace("Entrando no método main");
        logger.debug("Usuário carregado: id={}", usuarioId);
        logger.info("Aplicação iniciada com sucesso");
        logger.warn("Cache não configurado, usando valor padrão");
        logger.error("Falha simulada para teste de log");
    }
}
```

Raciocínio: com **Logback puro, sem nenhum arquivo de configuração**, o comportamento padrão (`BasicConfigurator`) põe o root logger em `DEBUG` — então apareceriam `debug`, `info`, `warn`, `error` (tudo, exceto `trace`, que fica abaixo de `DEBUG`). Já dentro de um **projeto Spring Boot**, o próprio Boot aplica sua configuração padrão de `INFO` no root antes disso — então só apareceriam `info`, `warn`, `error`. Esse é exatamente o tipo de detalhe de "comportamento por baixo do capô" que vale mais entender o porquê do que decorar.

**Exercício 2**

java

```java
package com.example.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ValidadorIdade {
    private static final Logger logger = LoggerFactory.getLogger(ValidadorIdade.class);

    public void validar(int idade) {
        if (idade < 0) {
            logger.warn("Idade negativa recebida: {}", idade);
        }

        if (idade < 18) {
            IllegalArgumentException ex =
                new IllegalArgumentException("Idade mínima é 18, recebido: " + idade);
            logger.error("Validação de idade falhou para valor {}", idade, ex);
            throw ex;
        }

        logger.info("Idade válida: {}", idade);
    }
}
```

Raciocínio: o `WARN` de idade negativa e o `ERROR` de menor de idade **não são mutuamente exclusivos** — se alguém passar `-5`, ambos disparam (primeiro o warn de "número estranho", depois o error de "não passou na regra"), o que é realista: no mundo real um dado pode ser simultaneamente "esquisito" e "inválido" por motivos diferentes. Repare que o `ex` é passado como último argumento do `logger.error`, não `ex.getMessage()` — é isso que garante a stack trace completa no log, não só o texto da mensagem.

**Exercício 3**

java

```java
package com.example.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import java.util.UUID;

public class RequisicaoDemo {
    private static final Logger logger = LoggerFactory.getLogger(RequisicaoDemo.class);

    public static void main(String[] args) {
        for (int i = 0; i < 3; i++) {
            processarRequisicao();
        }
    }

    private static void processarRequisicao() {
        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);

        try {
            logger.info("Requisição recebida");
            logger.info("Requisição processada com sucesso");
        } finally {
            MDC.remove("requestId");
        }
    }
}
```

Raciocínio: o padrão de log precisa incluir `%X{requestId}` (sintaxe do Logback pra ler valor do MDC) no `logback-spring.xml` pra esse valor aparecer no console — sem isso no pattern, o MDC existe no mapa mas não é impresso. O ponto central do exercício é o `finally`: como o loop roda na **mesma thread** (`main`), sem o `MDC.remove`, o segundo e terceiro `requestId` iriam se acumular incorretamente ou o valor antigo vazaria — é a simulação direta da armadilha #3 da seção anterior, só que provocada de propósito pra você ver o efeito.

**Exercício 4**

xml

```xml
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/aplicacao.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/aplicacao.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <logger name="com.example.pedidos" level="DEBUG"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

Raciocínio: repare que **root** está em `INFO`, mas o `<logger name="com.example.pedidos" level="DEBUG"/>` sobrescreve isso especificamente pra esse pacote — essa é a hierarquia de loggers em ação (um logger mais específico vence o mais genérico). O `RollingFileAppender` recebe tudo porque ele está anexado ao mesmo root em `INFO`... então, tecnicamente, pra realmente capturar `DEBUG` completo no arquivo _e_ manter o console só em `INFO`, você precisaria de dois roots efetivos ou usar um filtro de nível (`ThresholdFilter`) no `CONSOLE` em vez de dividir por root — vale a pena você tentar essa variação como exercício extra, já que é exatamente esse tipo de ajuste fino que aparece em projeto real.