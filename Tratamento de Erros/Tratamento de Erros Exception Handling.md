#### 1. Teoria

**O que é uma exceção**

Uma _exception_ é um objeto que representa uma condição anormal que interrompeu o fluxo normal de execução do programa. Quando algo dá errado (divisão por zero, arquivo que não existe, índice fora do array), a JVM (ou seu próprio código) cria esse objeto e "lança" ele. A partir daí, a execução para de seguir linha a linha e o controle "sobe" pela pilha de chamadas até encontrar alguém preparado para tratar aquele tipo específico de problema.

Pensa assim: é como um alarme que dispara no andar onde o problema aconteceu e sobe pelos andares do prédio (os métodos que chamaram uns aos outros) até alguém apertar o botão de "eu cuido disso" (`catch`). Se ninguém apertar o botão em nenhum andar, o alarme chega ao topo (o `main`, ou a thread) e o programa cai.

**Hierarquia — `Throwable`**

Tudo que pode ser lançado em Java herda de `Throwable`. Ela se divide em dois ramos:

- **`Error`** — problemas graves de ambiente/JVM (ex: `OutOfMemoryError`, `StackOverflowError`). Na prática, você **não trata** `Error` no seu código de negócio; se aconteceu, geralmente o processo está comprometido de um jeito que capturar não resolve.
- **`Exception`** — problemas que a aplicação pode, em tese, prever e reagir. Esse é o ramo que interessa no dia a dia.

Dentro de `Exception`, existe uma divisão importante:

||Checked|Unchecked|
|---|---|---|
|Exemplo|`IOException`, `SQLException`|`NullPointerException`, `IllegalArgumentException`, `ArithmeticException`|
|Herda de|`Exception` diretamente|`RuntimeException` (que herda de `Exception`)|
|Compilador exige tratamento?|Sim — ou `catch`, ou `throws` na assinatura|Não|
|Quando usar|Condição esperada, externa, que o chamador tem chance real de se recuperar (arquivo não existe, rede caiu)|Erro de programação/contrato violado (argumento inválido, estado nulo)|

Isso é diferente de **"quando um erro é grave"** — checked vs unchecked não é sobre gravidade, é sobre se o compilador **força** você a lidar com aquilo explicitamente. É comum confundir os dois eixos: gente assume que checked = mais sério, o que não é verdade.

**`try` / `catch` / `finally`**

- `try`: bloco onde você tenta o código que pode falhar.
- `catch`: bloco que trata um tipo específico de exceção (ou vários, com multi-catch).
- `finally`: bloco que **sempre** executa, dê certo ou errado — inclusive se houver `return` dentro do `try` ou `catch`. Só não executa se a JVM for encerrada abruptamente (`System.exit()`, crash da JVM).

Ordem dos `catch` importa: exceções mais específicas primeiro, mais genéricas depois. Se você colocar `catch (Exception e)` antes de `catch (IOException e)`, o segundo bloco nunca é alcançado — e isso é **erro de compilação** em Java (código inalcançável), não é só um descuido silencioso.

**`try-with-resources` (Java 7+)**

Quando você trabalha com algo que implementa `AutoCloseable` (streams, conexões JDBC, readers), em vez de fechar manualmente no `finally`, você declara o recurso dentro dos parênteses do `try`:

java

```java
try (BufferedReader reader = new BufferedReader(new FileReader(caminho))) {
    // usa o reader
} // fechado automaticamente aqui, mesmo se der exceção
```

Isso elimina uma classe inteira de bugs de vazamento de recurso (esquecer de fechar conexão, stream, etc.) — extremamente relevante quando você chegar em JDBC/Spring Data JPA mais na frente.

**Multi-catch (Java 7+)**

Quando dois ou mais tipos de exceção não relacionados por herança pedem o mesmo tratamento, você combina num catch só com `|`:

java

```java
catch (NumberFormatException | ArithmeticException e) { ... }
```

Atenção: os tipos precisam ser **irmãos**, não pai/filho. Se um for subclasse do outro (ex: `FileNotFoundException` e `IOException`), o compilador rejeita — vou mostrar isso na prática no gabarito.

**Exceções customizadas e `throw` vs `throws`**

- `throw` é o comando que efetivamente lança uma exceção agora.
- `throws` é a declaração na assinatura do método avisando "esse método pode lançar isso, quem me chamar precisa lidar".

Você cria exceções customizadas estendendo `Exception` (checked) ou `RuntimeException` (unchecked), normalmente pra representar erros de negócio de forma mais expressiva do que uma `RuntimeException` genérica (ex: `SaldoInsuficienteException` em vez de deixar estourar um erro sem nome).

**Encadeamento de exceções (exception chaining)**

Quando você captura uma exceção e lança outra no lugar (pra dar mais contexto, por exemplo), é importante **preservar a causa original** passando ela no construtor:

java

```java
throw new MinhaExcecao("mensagem", exceptionOriginal); // preserva o stack trace original
```

Se você não fizer isso, perde a informação de onde o erro realmente começou — isso é uma das armadilhas mais comuns, vou detalhar abaixo.

**Onde isso aparece no dia a dia de backend**

Em Spring Boot, exceções de negócio lançadas em serviços normalmente são capturadas de forma centralizada nos controllers REST via `@ExceptionHandler`/`@ControllerAdvice`, convertendo em respostas HTTP padronizadas (400, 404, 500 etc.) — isso será aprofundado quando chegarmos em Spring. Por enquanto, o que importa é entender bem o mecanismo puro da linguagem, porque é isso que sustenta aquele mecanismo depois.

---

#### 2. Exemplo de código comentado

**Exemplo principal — exceção customizada checked + try-with-resources + encadeamento**

java

```java
import java.io.*;

public class ProcessadorDeArquivo {

    // Exceção customizada checked: obriga quem chama lerPrimeiraLinha()
    // a lidar com ela explicitamente (catch ou throws)
    static class ArquivoInvalidoException extends Exception {
        public ArquivoInvalidoException(String mensagem, Throwable causa) {
            super(mensagem, causa); // encadeamento: preserva a exceção original como causa
        }
    }

    public static String lerPrimeiraLinha(String caminho) throws ArquivoInvalidoException {
        // try-with-resources: BufferedReader implementa AutoCloseable,
        // então é fechado automaticamente ao sair do bloco, com ou sem exceção
        try (BufferedReader reader = new BufferedReader(new FileReader(caminho))) {
            String linha = reader.readLine();
            if (linha == null || linha.isBlank()) {
                throw new ArquivoInvalidoException("Arquivo vazio: " + caminho, null);
            }
            return linha;
        } catch (FileNotFoundException e) {
            // FileNotFoundException é subclasse de IOException, então precisa vir
            // ANTES do catch de IOException — senão o compilador acusa código inalcançável
            throw new ArquivoInvalidoException("Arquivo não encontrado: " + caminho, e);
        } catch (IOException e) {
            // Qualquer outro problema de leitura (permissão, disco, etc.)
            throw new ArquivoInvalidoException("Erro de leitura: " + caminho, e);
        }
    }

    public static void main(String[] args) {
        try {
            String linha = lerPrimeiraLinha("dados.txt");
            System.out.println("Primeira linha: " + linha);
        } catch (ArquivoInvalidoException e) {
            System.err.println("Falha ao processar arquivo: " + e.getMessage());
            if (e.getCause() != null) {
                System.err.println("Causa raiz: " + e.getCause());
            }
        } finally {
            // Executa sempre — dá certo ou errado. Bom lugar pra logs de "processamento encerrado",
            // métricas, ou liberação de recursos que NÃO são AutoCloseable.
            System.out.println("Processamento finalizado.");
        }
    }
}
```

**Exemplo curto — multi-catch correto (tipos irmãos, não pai/filho)**

java

```java
public static int parseEDividir(String numeroStr, int divisor) {
    try {
        int numero = Integer.parseInt(numeroStr);
        return numero / divisor;
    } catch (NumberFormatException | ArithmeticException e) {
        // NumberFormatException (parsing inválido) e ArithmeticException (divisão por zero)
        // não têm relação de herança entre si — por isso podem ser combinadas num catch só
        System.err.println("Entrada inválida: " + e.getMessage());
        return 0;
    }
}
```

---

#### 3. Armadilhas comuns

1. **`catch (Exception e)` genérico demais** — captura qualquer coisa, inclusive bugs que deveriam quebrar o programa e te avisar (como `NullPointerException` por um erro de lógica). Isso esconde problema em vez de resolver. Capture o tipo mais específico possível.
2. **Bloco `catch` vazio ("engolir" a exceção)** — `catch (Exception e) {}` faz o erro desaparecer silenciosamente. O programa continua rodando com estado inconsistente e ninguém nunca vai saber o que aconteceu até o bug explodir em outro lugar, bem mais difícil de rastrear.
3. **Perder a causa original ao relançar** — fazer `throw new MinhaExcecao(e.getMessage())` em vez de `throw new MinhaExcecao(e.getMessage(), e)`. Sem passar `e` como causa, você perde o stack trace original — na hora de debugar em produção, isso é a diferença entre achar o bug em 2 minutos ou em 2 horas.
4. **Confundir divisão inteira com divisão de ponto flutuante** — `int / 0` lança `ArithmeticException`. `double / 0.0` **não lança nada**, retorna `Infinity` (ou `NaN` se for `0.0/0.0`). É um erro comum achar que qualquer divisão por zero vai estourar exceção — depende do tipo.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Escreva um método `dividir(int a, int b)` que trata a divisão por zero com `try/catch`, imprime uma mensagem amigável de erro e retorna `Double.NaN` nesse caso. Critério de pronto: chamar `dividir(10, 0)` não derruba o programa e imprime uma mensagem clara.

**Exercício 2 (Fácil/Médio)**  
Escreva um método `validarIdade(int idade)` que lança `IllegalArgumentException` com mensagem descritiva se a idade for negativa ou maior que 150. Escreva também um pequeno `main` que chama esse método com um valor inválido, captura a exceção e imprime a mensagem. Critério de pronto: idade válida não lança nada; idade inválida lança com mensagem clara indicando o valor recebido.

**Exercício 3 (Médio)**  
Crie uma classe `ContaBancaria` com saldo inicial e um método `sacar(double valor)`. Se o valor solicitado for maior que o saldo, lance uma exceção customizada **checked** chamada `SaldoInsuficienteException`, que deve carregar o saldo atual e o valor solicitado como informação (não só uma mensagem de texto). Critério de pronto: o método `sacar` compila só se o chamador tratar ou propagar a exceção; a mensagem da exceção mostra os dois valores formatados.

**Exercício 4 (Desafio)**  
Implemente um método `lerArquivos(List<String> caminhos)` que tenta ler a primeira linha de cada arquivo da lista, usando `try-with-resources`. Se um arquivo falhar, o processamento **não deve parar** — continue tentando os próximos. Ao final, se houve pelo menos uma falha, lance uma exceção customizada `FalhaAgregadaException` que carregue a lista de todas as exceções que ocorreram. Se não houve falha nenhuma, retorne um `Map<String, String>` com caminho → primeira linha. Critério de pronto: um arquivo inexistente no meio da lista não impede a leitura dos demais, e o erro final reporta todas as falhas, não só a primeira.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Calculadora {
    public static double dividir(int a, int b) {
        try {
            return a / b;
        } catch (ArithmeticException e) {
            System.out.println("Erro: não é possível dividir por zero.");
            return Double.NaN;
        }
    }
}
```

_Raciocínio:_ `ArithmeticException` é unchecked, lançada automaticamente pela JVM na divisão inteira por zero — você não precisa (e não pode) declarar `throws` pra ela. Uma alternativa válida seria validar `b != 0` **antes** de dividir e lançar `IllegalArgumentException` proativamente, em vez de deixar a JVM lançar e capturar depois. Trade-off: validar antes deixa a intenção mais explícita no código; deixar a JVM lançar é mais direto quando o erro realmente é uma situação excepcional (não esperada no fluxo normal). Nesse caso específico, ambas as abordagens são aceitáveis — a diferença fica mais relevante quando a checagem prévia é barata e clara.

**Exercício 2**

java

```java
public class ValidadorIdade {
    public static void validarIdade(int idade) {
        if (idade < 0 || idade > 150) {
            throw new IllegalArgumentException(
                "Idade inválida: " + idade + ". Deve estar entre 0 e 150.");
        }
    }

    public static void main(String[] args) {
        try {
            validarIdade(-5);
        } catch (IllegalArgumentException e) {
            System.out.println("Capturado: " + e.getMessage());
        }
    }
}
```

_Raciocínio:_ `IllegalArgumentException` é a escolha certa aqui porque é **unchecked** — passar um argumento inválido é erro do programador que chamou o método, não uma condição externa recuperável (como um arquivo que sumiu). Se isso fosse checked, todo chamador seria obrigado a `try/catch` só pra validar idade, o que polui a API sem ganho real.

**Exercício 3**

java

```java
public class SaldoInsuficienteException extends Exception {
    private final double saldoAtual;
    private final double valorSolicitado;

    public SaldoInsuficienteException(double saldoAtual, double valorSolicitado) {
        super(String.format(
            "Saldo insuficiente: saldo atual R$%.2f, solicitado R$%.2f",
            saldoAtual, valorSolicitado));
        this.saldoAtual = saldoAtual;
        this.valorSolicitado = valorSolicitado;
    }

    public double getSaldoAtual() { return saldoAtual; }
    public double getValorSolicitado() { return valorSolicitado; }
}

public class ContaBancaria {
    private double saldo;

    public ContaBancaria(double saldoInicial) {
        this.saldo = saldoInicial;
    }

    public void sacar(double valor) throws SaldoInsuficienteException {
        if (valor > saldo) {
            throw new SaldoInsuficienteException(saldo, valor);
        }
        saldo -= valor;
    }

    public double getSaldo() { return saldo; }
}
```

_Raciocínio:_ aqui a escolha por **checked** é defensável porque saldo insuficiente é uma condição de negócio esperada, que o chamador tem chance real de tratar de forma diferente (avisar o usuário, sugerir outro valor) — não é um bug. Guardar `saldoAtual` e `valorSolicitado` como campos (em vez de só texto na mensagem) permite que quem captura a exceção tome decisões programáticas com esses valores, não só exiba uma string.

_Trade-off relevante:_ algumas equipes preferem modelar até exceções de negócio como **unchecked**, pra evitar `throws` em cascata subindo várias camadas (service → controller), tratando tudo de forma centralizada mais adiante. As duas abordagens são usadas no mercado — quando chegarmos em Spring, você vai ver como o `@ControllerAdvice` lida bem com qualquer uma das duas.

**Exercício 4**

java

```java
import java.io.*;
import java.util.*;

public class LeitorMultiploArquivos {

    static class FalhaAgregadaException extends Exception {
        private final List<Exception> falhas;

        public FalhaAgregadaException(List<Exception> falhas) {
            super(falhas.size() + " arquivo(s) falharam ao processar.");
            this.falhas = falhas;
        }

        public List<Exception> getFalhas() {
            return falhas;
        }
    }

    public static Map<String, String> lerArquivos(List<String> caminhos) throws FalhaAgregadaException {
        Map<String, String> resultados = new LinkedHashMap<>();
        List<Exception> falhas = new ArrayList<>();

        for (String caminho : caminhos) {
            try (BufferedReader reader = new BufferedReader(new FileReader(caminho))) {
                resultados.put(caminho, reader.readLine());
            } catch (IOException e) {
                // Só IOException aqui — não FileNotFoundException | IOException.
                // FileNotFoundException é subclasse de IOException, e o Java proíbe
                // multi-catch entre tipos numa relação pai/filho (erro de compilação).
                // Capturar IOException já cobre FileNotFoundException também.
                falhas.add(e);
            }
        }

        if (!falhas.isEmpty()) {
            throw new FalhaAgregadaException(falhas);
        }
        return resultados;
    }
}
```

_Raciocínio:_ o ponto central aqui é continuar o loop mesmo com falha (`falhas.add(e)` em vez de deixar propagar na hora), pra não interromper o processamento dos demais arquivos — é o padrão de "coletar erros, decidir no final" em vez de "falha rápido no primeiro problema", útil em processamento em lote. `FalhaAgregadaException` guarda a lista completa, então quem captura no `main` consegue reportar **todos** os arquivos problemáticos, não só o primeiro.

_Nota de precisão:_ se você tentasse escrever `catch (FileNotFoundException | IOException e)`, o compilador rejeitaria com erro do tipo "alternative FileNotFoundException is a subclass of alternative IOException" — é exatamente o tipo de detalhe que vale confirmar antes de assumir, porque é fácil escrever multi-catch "por hábito" sem checar a hierarquia dos tipos envolvidos.