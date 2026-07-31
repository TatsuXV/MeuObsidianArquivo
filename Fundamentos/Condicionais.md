### 1. Teoria

**O que são condicionais?**

Condicionais são as construções que permitem ao programa **desviar o fluxo de execução** com base em uma expressão booleana (`true`/`false`). Sem elas, um programa executaria sempre a mesma sequência linear de instruções, do início ao fim, sem nenhuma capacidade de "decidir" nada.

Java oferece três formas principais de expressar lógica condicional:

#### `if / else if / else`

java

```java
if (condição) {
    // executa se condição for true
} else if (outraCondição) {
    // executa se a primeira for false e essa for true
} else {
    // executa se nenhuma das anteriores for true
}
```

Pontos importantes:

- A condição **precisa** ser do tipo `boolean`. Diferente de C, onde `if (1)` é válido (qualquer valor não-zero é "verdadeiro"), em Java `if (1)` **não compila** — erro de tipo incompatível.
- As chaves `{ }` são opcionais quando o bloco tem uma única instrução, mas isso é considerado má prática em código profissional (veremos por quê nas armadilhas).
- `else if` não é uma palavra-chave especial — é simplesmente um `else` cujo corpo é outro `if`. A indentação que usamos é só convenção visual.

#### Operador ternário

java

```java
tipo variavel = condição ? valorSeVerdadeiro : valorSeFalso;
```

É uma expressão (retorna um valor), não um comando — por isso pode ser usado direto numa atribuição ou dentro de uma chamada de método. Útil para casos simples; abusar dele encadeando vários ternários aninhados prejudica legibilidade.

#### `switch`

Existem duas sintaxes válidas hoje em dia:

**Switch clássico (statement, com fall-through):**

java

```java
switch (variavel) {
    case VALOR1:
        // código
        break; // sem break, o fluxo "cai" pro próximo case (fall-through)
    case VALOR2:
        // código
        break;
    default:
        // código padrão
}
```

**Switch expression moderno (Java 14+, sintaxe com seta):**

java

```java
tipo resultado = switch (variavel) {
    case VALOR1 -> valorA;
    case VALOR2 -> valorB;
    default -> valorPadrao;
};
```

No formato com `->`, não existe fall-through — cada `case` executa isoladamente, sem precisar de `break`. Essa forma também pode ser usada como _expressão_ (atribuindo direto a uma variável), o que o switch clássico não permite nativamente.

**Operadores lógicos usados em condições**

|Operador|Significado|Curto-circuito?|
|---|---|---|
|`&&`|E lógico|Sim — se o lado esquerdo já for `false`, o direito nem é avaliado|
|`\|`|OU lógico|Sim — se o lado esquerdo já for `true`, o direito nem é avaliado|
|`!`|Negação|—|
|`&` / `\|`|E / OU sem curto-circuito|Não — avalia os dois lados sempre|

O curto-circuito de `&&` e `||` não é só uma otimização de performance — é usado deliberadamente para evitar erros, como em `if (objeto != null && objeto.getValor() > 0)`: se `objeto` for `null`, a segunda parte nunca é avaliada, evitando um `NullPointerException`.

**Onde isso aparece na prática (backend real)**

Validação de entrada (`if (usuario == null) throw new ...`), controle de fluxo de negócio (`switch` em cima de um enum de status de pedido), e roteamento de lógica em `@Service` são exemplos do dia a dia. `switch expression` com enums é especialmente comum em backend Spring moderno, porque o compilador consegue avisar se você esqueceu de tratar algum valor do enum (exhaustiveness checking), reduzindo bug de "esqueci de tratar esse status".

---

### 2. Exemplo de código comentado

java

```java
public class Condicionais {
    public static void main(String[] args) {

        int idade = 20;
        double saldoConta = -50.0;

        // if / else if / else clássico
        if (idade < 12) {
            System.out.println("Criança");
        } else if (idade < 18) {
            System.out.println("Adolescente");
        } else {
            System.out.println("Adulto");
        }

        // Operador ternário — expressão, não comando
        String status = saldoConta >= 0 ? "Conta regular" : "Conta negativada";
        System.out.println(status);

        // Curto-circuito evitando NullPointerException
        String texto = null;
        if (texto != null && texto.length() > 0) {
            System.out.println("Texto não vazio");
        } else {
            System.out.println("Texto nulo ou vazio"); // este é o caminho executado
        }
        // Se aqui fosse usado o operador & (sem curto-circuito) em vez de &&,
        // texto.length() seria avaliado mesmo com texto null, lançando exceção.

        // switch clássico com fall-through intencional
        int diaDaSemana = 6;
        switch (diaDaSemana) {
            case 6:
            case 7:
                System.out.println("Fim de semana"); // 6 "cai" pro mesmo bloco de 7, de propósito
                break;
            default:
                System.out.println("Dia útil");
        }

        // switch expression moderno — sem break, sem fall-through, retorna valor
        String tipoDia = switch (diaDaSemana) {
            case 6, 7 -> "Fim de semana"; // múltiplos valores no mesmo case, separados por vírgula
            default -> "Dia útil";
        };
        System.out.println(tipoDia);
    }
}
```

---

### 3. Armadilhas comuns

1. **Esquecer o `break` no switch clássico.** Sem `break`, o fluxo "cai" (fall-through) pro próximo `case` mesmo que a condição dele não bata — isso já causou bug real em produção incontáveis vezes. É justamente por isso que o switch expression moderno (`->`) foi introduzido: elimina essa classe inteira de bug.
2. **Omitir chaves `{ }` em blocos de uma linha só.** `if (condição) instrucaoA(); instrucaoB();` — só `instrucaoA()` pertence ao `if`; `instrucaoB()` roda sempre, independente da condição, mesmo que a indentação visual sugira o contrário. Por isso, mesmo quando o compilador permite omitir, a convenção profissional é sempre usar chaves.
3. **Usar `=` em vez de `==` na condição.** `if (ativo = true)` é um erro de digitação clássico. Em Java isso **não compila** quando a variável não é `boolean` (diferente de C, onde compila silenciosamente e é uma fonte de bug grave) — mas se a variável já for `boolean`, `if (ativo = true)` compila e sempre atribui `true` a `ativo`, mascarando a lógica pretendida.
4. **Comparar `String` com `==` em vez de `.equals()`.** Como `String` é tipo de referência, `==` compara se é o _mesmo objeto_ na memória, não se o conteúdo é igual. `"abc" == new String("abc")` pode dar `false` mesmo com conteúdo idêntico. Isso será aprofundado quando chegarmos em Strings, mas já vale o alerta agora porque aparece direto em condicionais.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Escreva um programa que declare uma variável `int nota` com um valor de sua escolha (0 a 10) e imprima o conceito correspondente usando `if/else if/else`: `nota >= 9` → "Excelente", `nota >= 7` → "Bom", `nota >= 5` → "Regular", qualquer outro caso → "Insuficiente". Critério de pronto: teste mentalmente com pelo menos 3 valores diferentes de nota e confirme que cada um cai no bloco certo.

**Exercício 2 (médio)**  
Reescreva a lógica do Exercício 1 usando **switch expression moderno**, mas adaptando pra uma variável `int faixaNota` que já vem pré-calculada como `nota / 1` truncado em faixas fixas de 0 a 10 (ou seja, use o switch pra tratar `case 10, 9 -> ...`, `case 8, 7 -> ...` etc., agrupando valores como no exemplo de dia da semana). Critério de pronto: o switch expression atribui o resultado direto a uma variável `String conceito`, sem usar `break` em nenhum lugar.

**Exercício 3 (difícil)**  
Escreva um programa que simule uma validação de cadastro: uma variável `String email` (pode ser `null` ou uma String qualquer) e uma variável `int idade`. Usando `&&` com curto-circuito, verifique — numa única condição `if` — se `email` não é `null` **e** contém o caractere `@` (dica: `String` tem um método `.contains(String)`) **e** `idade >= 18`. Teste seu programa com `email = null` e confirme que não lança `NullPointerException` (é exatamente o curto-circuito evitando isso). Critério de pronto: roda sem exceção mesmo com `email = null`, e imprime mensagens diferentes pra cada combinação de sucesso/falha que você testar.

**Exercício 4 (desafio)**  
O código abaixo tem um bug de fall-through intencionalmente inserido. Sem rodar ainda, escreva por escrito qual vai ser a saída pra `mes = 4` e explique **por que** — depois rode pra confirmar, e corrija o bug pra que cada `case` funcione de forma isolada, sem fall-through indesejado:

java

```java
public class Exercicio4 {
    public static void main(String[] args) {
        int mes = 4;
        switch (mes) {
            case 3:
            case 4:
            case 5:
                System.out.println("Outono");
            case 6:
            case 7:
            case 8:
                System.out.println("Inverno");
                break;
            default:
                System.out.println("Outra estação");
        }
    }
}
```

Critério de pronto: você explica corretamente o fall-through antes de corrigir, e a versão corrigida imprime **apenas** "Outono" para `mes = 4`.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        int nota = 8;

        if (nota >= 9) {
            System.out.println("Excelente");
        } else if (nota >= 7) {
            System.out.println("Bom");
        } else if (nota >= 5) {
            System.out.println("Regular");
        } else {
            System.out.println("Insuficiente");
        }
    }
}
```

Raciocínio: a ordem das comparações importa — como é `else if` em cadeia, uma vez que `nota >= 9` for `false`, o programa já sabe que `nota < 9`, então `nota >= 7` sozinho já delimita corretamente a faixa "entre 7 e 8". Se a ordem fosse invertida (checar `>= 5` primeiro), a lógica quebraria, porque uma nota 9 também satisfaz `>= 5` e cairia no bloco errado.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        int faixaNota = 8;

        String conceito = switch (faixaNota) {
            case 10, 9 -> "Excelente";
            case 8, 7 -> "Bom";
            case 6, 5 -> "Regular";
            default -> "Insuficiente";
        };

        System.out.println(conceito);
    }
}
```

Raciocínio: diferente do `if/else if`, aqui não existe ordem de prioridade entre os `case` — cada valor possível de `faixaNota` bate em exatamente um `case` (ou no `default`), então não há risco de "cair no bloco errado" por causa de ordem, como acontecia no Exercício 1. É uma vantagem do switch quando a lógica é baseada em valores discretos e não em faixas contínuas de comparação.

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        String email = null;
        int idade = 20;

        if (email != null && email.contains("@") && idade >= 18) {
            System.out.println("Cadastro válido");
        } else {
            System.out.println("Cadastro inválido");
        }
    }
}
```

Raciocínio: a ordem das condições dentro do `&&` **importa aqui, e muito**. `email != null` precisa vir _antes_ de `email.contains("@")`, porque `&&` avalia da esquerda pra direita com curto-circuito: se `email != null` já for `false`, o Java nem tenta avaliar `email.contains("@")` — que lançaria `NullPointerException` se tentasse chamar um método num valor `null`. Se a ordem fosse invertida (`email.contains("@") && email != null`), o programa quebraria com `email = null`.

**Exercício 4**

Saída para `mes = 4` (com o bug):

```
Outono
Inverno
```

Explicação do fall-through: `case 4` não tem `break` — ele imprime "Outono" e o fluxo **continua caindo** pros próximos `case` (5, depois 6, 7, 8) até encontrar um `break`. Como o primeiro `break` só aparece depois do `System.out.println("Inverno")`, o programa imprime as duas linhas, mesmo `mes` sendo claramente "outono" e não "inverno".

Versão corrigida:

java

```java
public class Exercicio4Corrigido {
    public static void main(String[] args) {
        int mes = 4;
        switch (mes) {
            case 3:
            case 4:
            case 5:
                System.out.println("Outono");
                break; // break adicionado aqui, isolando o grupo
            case 6:
            case 7:
            case 8:
                System.out.println("Inverno");
                break;
            default:
                System.out.println("Outra estação");
        }
    }
}
```

Raciocínio: bastou adicionar o `break` faltante depois do grupo de "Outono" pra isolar os dois blocos. Esse é exatamente o tipo de bug que motivou a criação do switch expression moderno (`->`) — nele, esse erro seria estruturalmente impossível de acontecer, porque cada `case` já é isolado por padrão.