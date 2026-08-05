#### 1. Teoria

Antes do Java 8, a única opção era `java.util.Date` e `java.util.Calendar` — classes mutáveis, não thread-safe, com API confusa (mês indexado a partir de 0, por exemplo). O Java 8 trouxe o pacote `java.time` (JSR-310, inspirado no Joda-Time), com classes **imutáveis** e **thread-safe** que separam claramente os conceitos:

- **`LocalDate`** — só data, sem hora, sem fuso. Uso: data de nascimento, vencimento, prazo.
- **`LocalTime`** — só hora, sem data.
- **`LocalDateTime`** — data + hora, **sem fuso horário**. Cuidado: isso não representa um instante real e único no tempo — "2024-03-15T10:00" é ambíguo até você saber em qual fuso.
- **`ZonedDateTime`** — data + hora + fuso. Representa um momento real, e sabe lidar com regras de horário de verão de cada região.
- **`Instant`** — um ponto na linha do tempo, sempre em UTC. É o que você guarda no banco pra "quando isso aconteceu" (timestamp de criação, log de evento).
- **`Duration`** — quantidade de tempo em horas/minutos/segundos/nanos. Funciona entre `Instant`/`LocalTime`.
- **`Period`** — quantidade de tempo em anos/meses/dias. Funciona entre `LocalDate`.
- **`DateTimeFormatter`** — parse e formatação, imutável e thread-safe (ao contrário do antigo `SimpleDateFormat`, que é uma armadilha clássica em código concorrente).
- **`ZoneId`** — identifica um fuso horário (ex: `"America/Sao_Paulo"`), com as regras de DST daquela região.

Prática comum em backend: guardar timestamps como `Instant` (UTC) no banco, e só converter pro fuso do usuário na camada de apresentação (na resposta da API, no front). Entidades JPA costumam usar `LocalDate`/`LocalDateTime`/`Instant` como tipo de campo, dependendo se o dado precisa ou não carregar fuso.

#### 2. Exemplo de código comentado

java

```java
import java.time.*;
import java.time.format.DateTimeFormatter;

public class DateTimeExample {
    public static void main(String[] args) {
        // LocalDate: só data — bom pra vencimento, data de nascimento, etc.
        LocalDate hoje = LocalDate.now();
        LocalDate proximoVencimento = hoje.plusDays(30); // imutável: plusDays retorna um NOVO LocalDate

        // LocalDateTime: data + hora, SEM fuso — não representa um instante real
        LocalDateTime agora = LocalDateTime.now();
        System.out.println("Agora (sem fuso): " + agora);

        // ZonedDateTime: data + hora + fuso — momento real, sabe lidar com horário de verão
        ZonedDateTime agoraSaoPaulo = ZonedDateTime.now(ZoneId.of("America/Sao_Paulo"));
        ZonedDateTime agoraLondres = agoraSaoPaulo.withZoneSameInstant(ZoneId.of("Europe/London")); // mesmo instante, fuso diferente
        System.out.println("São Paulo: " + agoraSaoPaulo);
        System.out.println("Londres: " + agoraLondres);

        // Instant: ponto na linha do tempo em UTC — o que você guarda no banco
        Instant timestamp = Instant.now();
        System.out.println("Instant (UTC): " + timestamp);

        // Duration: baseado em horas/minutos/segundos — entre Instants ou LocalTimes
        Instant inicio = Instant.now();
        // ... alguma operação aqui ...
        Instant fim = Instant.now();
        Duration tempoDecorrido = Duration.between(inicio, fim);
        System.out.println("Levou " + tempoDecorrido.toMillis() + "ms");

        // Period: baseado em anos/meses/dias — entre LocalDates
        LocalDate nascimento = LocalDate.of(1998, Month.MARCH, 15); // mês NÃO é zero-indexed aqui (diferente do java.util.Date antigo)
        Period idade = Period.between(nascimento, hoje);
        System.out.println("Idade: " + idade.getYears() + " anos");

        // DateTimeFormatter: imutável e thread-safe, diferente do antigo SimpleDateFormat
        DateTimeFormatter formatoBr = DateTimeFormatter.ofPattern("dd/MM/yyyy");
        System.out.println("Vencimento: " + proximoVencimento.format(formatoBr));
    }
}
```

#### 3. Armadilhas comuns

1. **Usar `LocalDateTime` pra representar "quando algo aconteceu".** Sem fuso, é ambíguo. Pra timestamp de evento real (criação de registro, log), use `Instant` — ou `ZonedDateTime` se você realmente precisa saber em qual fuso o evento ocorreu.
2. **Salvar em UTC mas esquecer de converter pro fuso do usuário na exibição** — gera hora errada na tela, mesmo com o dado certo no banco.
3. **Confundir `Duration` com `Period`.** `Duration` é baseado em segundos/nanos e funciona com `Instant`/`LocalTime`; `Period` é baseado em anos/meses/dias e funciona com `LocalDate`. Usar o tipo errado no contexto errado nem compila, ou dá resultado sem sentido.
4. **Ainda usar `java.util.Date`/`Calendar`/`SimpleDateFormat` em código novo.** São mutáveis e não thread-safe — `SimpleDateFormat` compartilhado entre threads é um bug clássico de produção (corrompe resultado sob concorrência). Desde Java 8 não há motivo pra começar algo novo com essas classes.

#### 4. Exercícios práticos

**Fácil** — Escreva `diasAteVencimento(LocalDate vencimento)`, que retorna quantos dias faltam a partir de hoje (pode ser negativo se já venceu). Critério de pronto: testar com uma data no passado e uma no futuro.

**Médio** — Escreva `converterParaFusoDoUsuario(Instant timestampUtc, String fusoDoUsuario)`, que retorna uma `String` formatada (`dd/MM/yyyy HH:mm`) do timestamp convertido pro fuso informado. Critério de pronto: funciona corretamente considerando horário de verão — teste com um fuso que tem DST (ex: `"America/New_York"`) em duas datas do ano diferentes.

**Difícil** — Escreva `calcularIdadeCompleta(LocalDate nascimento, LocalDate dataReferencia)`, retornando a idade em anos completos. Critério de pronto: nascido em 15/03/2000, calculado em 14/03/2024, deve retornar **23** (não 24) — aniversário ainda não chegou naquele ano.

**Desafio** — Implemente `temConflito(LocalTime novoInicio, Duration novaDuracao, List<Agendamento> existentes)`, onde `Agendamento` tem início (`LocalTime`) e duração (`Duration`), verificando se o novo horário sobrepõe algum já existente no mesmo dia. Critério de pronto: retorna `true` se houver qualquer sobreposição; caso de borda — se o novo agendamento começa exatamente quando outro termina, **não é conflito**.

#### 5. Gabarito comentado

**Fácil:**

java

```java
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

public static long diasAteVencimento(LocalDate vencimento) {
    return ChronoUnit.DAYS.between(LocalDate.now(), vencimento);
}
```

`ChronoUnit.DAYS.between(início, fim)` retorna `fim - início` em dias — negativo se `vencimento` já passou.

**Médio:**

java

```java
import java.time.Instant;
import java.time.ZoneId;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;

public static String converterParaFusoDoUsuario(Instant timestampUtc, String fusoDoUsuario) {
    ZonedDateTime zoned = timestampUtc.atZone(ZoneId.of(fusoDoUsuario)); // converte o instante pro fuso informado
    return zoned.format(DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm"));
}
```

`atZone()` já aplica as regras de horário de verão daquele `ZoneId` automaticamente — você não precisa (e não deve) tratar DST manualmente.

**Difícil:**

java

```java
import java.time.LocalDate;
import java.time.Period;

public static int calcularIdadeCompleta(LocalDate nascimento, LocalDate dataReferencia) {
    return Period.between(nascimento, dataReferencia).getYears();
}
```

`Period.between` já resolve internamente "ainda não fez aniversário este ano" — é exatamente pra evitar esse cálculo manual (subtrair anos e depois checar mês/dia "na unha") que a classe existe.

**Desafio:**

java

```java
import java.time.LocalTime;
import java.time.Duration;
import java.util.List;

public record Agendamento(LocalTime inicio, Duration duracao) {
    LocalTime fim() {
        return inicio.plus(duracao);
    }
}

public static boolean temConflito(LocalTime novoInicio, Duration novaDuracao, List<Agendamento> existentes) {
    LocalTime novoFim = novoInicio.plus(novaDuracao);

    for (Agendamento existente : existentes) {
        boolean semSobreposicao = novoFim.compareTo(existente.inicio()) <= 0
                                || novoInicio.compareTo(existente.fim()) >= 0;
        if (!semSobreposicao) {
            return true; // sobreposição encontrada
        }
    }
    return false;
}
```

Raciocínio: dois intervalos `[aInicio, aFim)` e `[bInicio, bFim)` **não** se sobrepõem se `aFim <= bInicio` OU `aInicio >= bFim`. Negando essa condição, sobra exatamente a sobreposição. É por isso que "começar quando o outro termina" não conta como conflito: `novoFim.compareTo(existente.inicio()) <= 0` é verdadeiro nesse caso (intervalo meio-aberto).