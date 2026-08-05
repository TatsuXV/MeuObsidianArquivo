#### 1. Teoria

O suporte a regex no Java vive no pacote `java.util.regex`, com duas classes centrais:

- **`Pattern`** — representa um regex **compilado**. Compilar é uma operação relativamente cara, então se você vai reutilizar o mesmo regex várias vezes (dentro de um loop, ou num método chamado repetidamente), compile uma vez só (`static final`) e reutilize — ao contrário dos métodos de conveniência da `String` (`matches()`, `replaceAll()`, `split()`), que compilam um `Pattern` novo a cada chamada.
- **`Matcher`** — aplica um `Pattern` a uma entrada específica. É com ele que você efetivamente busca (`find()`), valida (`matches()`) ou extrai (`group()`) o resultado.

Sintaxe essencial que você vai usar o tempo todo:

- Classes predefinidas: `\d` (dígito), `\w` (letra/dígito/underscore), `\s` (espaço em branco) — e as versões maiúsculas negam (`\D`, `\W`, `\S`).
- Quantificadores: `*` (0+), `+` (1+), `?` (0 ou 1), `{n}`, `{n,}`, `{n,m}` — todos **gulosos** por padrão (pegam o máximo possível); adicionar `?` depois torna **preguiçoso** (`*?`, `+?`, pega o mínimo).
- Grupos: `(...)` captura, `(?:...)` captura _não numerada_ (útil quando você só precisa agrupar pra aplicar quantificador, sem precisar recuperar o valor depois), `(?<nome>...)` grupo **nomeado** — mais legível que `group(1)`, `group(2)`.
- Âncoras: `^` início, `$` fim, `\b` fronteira de palavra.
- Lookahead/lookbehind: `(?=...)` (positivo, olha à frente sem consumir), `(?!...)` (negativo), `(?<=...)`/`(?<!...)` (atrás) — usados pra validar "isso precisa existir por perto" sem incluir no match.

Diferença importante de comportamento: **`matches()` exige casar a string inteira** (ancorado implicitamente em `^...$`); **`find()` procura em qualquer parte da string**, e pode ser chamado repetidamente pra achar todas as ocorrências.

Onde isso aparece no dia a dia: validação de formato (e-mail, CEP, telefone) antes de bater no banco, extração de dado de log/texto não estruturado, e sanitização de input. Pra casos simples — "essa string contém essa substring fixa?" — prefira `String.contains()`/`startsWith()`, que são mais rápidos e mais legíveis; regex é ferramenta pra _padrão_, não pra texto literal.

#### 2. Exemplo de código comentado

java

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class RegexExample {

    // compilado uma vez só, fora do método — Pattern é imutável e thread-safe, reutilizável
    private static final Pattern DATA_PATTERN =
        Pattern.compile("(?<dia>\\d{2})/(?<mes>\\d{2})/(?<ano>\\d{4})");

    public static void main(String[] args) {
        String texto = "Prazo final: 15/03/2024. Segunda tentativa: 22/04/2024.";

        Matcher matcher = DATA_PATTERN.matcher(texto);

        while (matcher.find()) { // find() avança pra próxima ocorrência a cada chamada, não só no início da string
            String dia = matcher.group("dia"); // grupo nomeado — mais legível que group(1)
            String mes = matcher.group("mes");
            String ano = matcher.group("ano");
            System.out.printf("Data: %s/%s/%s (match completo: %s)%n", dia, mes, ano, matcher.group());
        }
    }
}
```

#### 3. Armadilhas comuns

1. **Usar `matches()` esperando achar o padrão em qualquer parte da string.** `matches()` exige que o padrão bata a string **inteira**. Pra buscar em qualquer posição, use `find()`.
2. **Recompilar o mesmo `Pattern.compile()` dentro de um loop ou de um método chamado repetidamente.** `compile()` é caro — se o regex é fixo, declare como `static final` fora do método.
3. **Catastrophic backtracking (ReDoS).** Regex com quantificadores aninhados mal escritos (ex: `(a+)+b` aplicado a uma entrada que não termina em "b") pode fazer o motor de regex testar um número exponencial de combinações e travar a aplicação com uma entrada relativamente curta. Cuidado redobrado ao validar input vindo direto do usuário — evite grupos quantificados dentro de outros grupos quantificados quando possível.
4. **Esquecer de escapar caractere especial do regex que é literal no texto que você quer casar.** `.`, `*`, `+`, `(`, `[`, `{`, `|`, `\`, `^`, `$` têm significado especial — pra casar um ponto literal (ex: num IP), escreva `\\.`, não `.`. Pra tratar uma string inteira como literal, use `Pattern.quote(texto)`.

#### 4. Exercícios práticos

**Fácil** — Escreva `validarFormatoEmail(String email)`, validação de formato simples (não precisa ser RFC completo — algo como `usuario@dominio.com`), retornando `boolean`.

**Médio** — Escreva `extrairTelefones(String texto)`, retornando uma `List<String>` com todos os telefones em formato brasileiro encontrados num texto livre (aceite tanto `"(61) 99999-8888"` quanto `"61999998888"`).

**Difícil** — Escreva `mascararCpf(String texto)`, que encontra qualquer CPF no formato `"000.000.000-00"` dentro de um texto e substitui os 6 dígitos do meio por asteriscos, mantendo os 3 primeiros e os 2 últimos visíveis (ex: `"123.***.***-00"`). Use grupos de captura + referência a grupo na substituição.

**Desafio** — Escreva `isSenhaForte(String senha)`: mínimo 8 caracteres, pelo menos 1 maiúscula, 1 minúscula, 1 dígito e 1 caractere especial — tudo numa única expressão regular usando lookaheads encadeados.

#### 5. Gabarito comentado

**Fácil:**

java

```java
private static final Pattern EMAIL_PATTERN =
    Pattern.compile("^[\\w.+-]+@[\\w-]+\\.[a-zA-Z]{2,}$");

public static boolean validarFormatoEmail(String email) {
    return EMAIL_PATTERN.matcher(email).matches();
}
```

`^` e `$` aqui são redundantes com `matches()` (que já ancora implicitamente), mas deixá-los explícitos documenta a intenção — e passam a fazer diferença se você trocar `matches()` por `find()` no futuro.

**Médio:**

java

```java
private static final Pattern TELEFONE_PATTERN =
    Pattern.compile("\\(?\\d{2}\\)?\\s?9?\\d{4}-?\\d{4}");

public static List<String> extrairTelefones(String texto) {
    List<String> resultado = new ArrayList<>();
    Matcher matcher = TELEFONE_PATTERN.matcher(texto);
    while (matcher.find()) {
        resultado.add(matcher.group());
    }
    return resultado;
}
```

`\\(?` e `\\)?` tornam os parênteses do DDD opcionais; `\\s?` torna o espaço opcional; `9?` cobre celular (9 dígitos) e fixo (8 dígitos) com o mesmo padrão.

**Difícil:**

java

```java
private static final Pattern CPF_PATTERN =
    Pattern.compile("(\\d{3})\\.(\\d{3})\\.(\\d{3})-(\\d{2})");

public static String mascararCpf(String texto) {
    return CPF_PATTERN.matcher(texto).replaceAll("$1.***.***-$4");
}
```

Na string de substituição, `$1` e `$4` referenciam o 1º e o 4º grupo capturado no match — assim os 3 primeiros dígitos e os 2 últimos são preservados, e os grupos 2 e 3 (que não são referenciados) somem, dando lugar aos asteriscos literais.

**Desafio:**

java

```java
private static final Pattern SENHA_FORTE_PATTERN = Pattern.compile(
    "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[^a-zA-Z0-9]).{8,}$"
);

public static boolean isSenhaForte(String senha) {
    return SENHA_FORTE_PATTERN.matcher(senha).matches();
}
```

Raciocínio: cada `(?=...)` é um lookahead — checa se o padrão existe _a partir da posição atual_ sem consumir caractere nenhum. Como nenhum deles avança o cursor, todos os quatro são testados a partir do **mesmo ponto** (início da string), de forma independente. Só o `.{8,}` no final realmente consome os caracteres e garante o tamanho mínimo.

**Alternativa e trade-off:** essa regex única é compacta, mas pouco legível e difícil de dar mensagem de erro específica ("falta caractere especial" vs "falta maiúscula"). Em produção, muitas vezes é melhor validar cada regra separadamente (métodos booleanos ou uma lista de validadores), mesmo sendo mais verboso — você ganha mensagens de erro específicas pro usuário e o código fica mais fácil de manter.