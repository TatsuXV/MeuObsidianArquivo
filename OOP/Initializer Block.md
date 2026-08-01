#### 1. Teoria

**Initializer Block** (bloco de inicialização) é um bloco de código, delimitado por `{ }`, escrito **diretamente dentro de uma classe**, mas **fora** de qualquer método ou construtor. Existem dois tipos, diferenciados apenas pela presença ou não da palavra-chave `static`:

- **Bloco de inicialização de instância** (_instance initializer block_): roda toda vez que um objeto é criado.
- **Bloco de inicialização estático** (_static initializer block_): roda uma única vez, quando a classe é carregada.

Você já viu esses dois tipos "em ação" no tópico de Object Lifecycle, mas ali o foco era a **ordem** deles dentro do ciclo de vida do objeto. Aqui o foco é entender o bloco em si: sintaxe, regras, casos de uso reais e por que ele existe quando você já tem construtor.

**Por que isso existe, se já existe construtor?**

Um construtor sempre está associado a uma assinatura específica (recebe certos parâmetros). Se uma classe tem **múltiplos construtores sobrecarregados**, e existe uma lógica de inicialização que precisa rodar **em todos eles, sempre**, copiar essa lógica em cada construtor é repetição desnecessária. O bloco de inicialização de instância roda automaticamente antes de **qualquer** construtor, então é o lugar certo para lógica de inicialização compartilhada entre todos os construtores.

Já o bloco estático resolve um problema diferente: lógica de inicialização de **campos estáticos** que é complexa demais para caber numa única expressão de atribuição (ex: preencher um `Map` com vários pares, ler um arquivo de configuração, tratar uma exceção durante a inicialização).

**Sintaxe:**

java

```java
public class Exemplo {
    static {
        // bloco de inicialização estática
    }

    {
        // bloco de inicialização de instância
    }
}
```

**Regras importantes:**

1. **Pode haver múltiplos blocos do mesmo tipo na mesma classe.** Nesse caso, eles executam **na ordem em que aparecem no código-fonte**, de cima para baixo — junto com os inicializadores de campo (`private int x = 10;` também conta como parte dessa ordem sequencial).
2. **Blocos estáticos rodam antes de qualquer bloco de instância**, e antes de qualquer construtor — já que a classe precisa estar carregada antes de qualquer objeto poder ser criado dela.
3. **Bloco estático pode lançar apenas exceções não-checked (unchecked).** Se um bloco estático lançar uma exceção checked sem tratar, o código não compila. Se uma exceção (checked ou unchecked) escapar de um bloco estático em runtime, a JVM encapsula isso num `ExceptionInInitializerError` — um erro sério, porque significa que a classe **falhou ao carregar**, e isso geralmente derruba a aplicação inteira ou torna a classe inutilizável dali em diante.
4. **Bloco de instância pode lançar qualquer exceção**, mas ela precisa ser tratada ali dentro ou declarada de alguma forma compatível com os construtores da classe (na prática, é raro e geralmente sinal de design que vale reconsiderar).

**Diferença de Initializer Block vs. Inicializador de campo (`int x = 10;`):**  
São, na prática, tratados pela JVM da mesma forma — ambos rodam na ordem em que aparecem no arquivo, junto com os blocos. A diferença é só sintática/estilística: inicializador de campo atribui **um** campo específico numa expressão simples; bloco de inicialização é usado quando a lógica é **mais complexa** do que uma atribuição direta permite (loop, try/catch, múltiplas linhas).

**Onde aparece no dia a dia de backend Java/Spring:**

- É relativamente raro ver bloco de inicialização de **instância** em código de produção moderno — na maioria dos casos, é mais claro colocar a lógica direto no construtor. Vale saber que existe e reconhecer quando aparecer em código legado ou de terceiros.
- Bloco **estático**, por outro lado, aparece com alguma frequência para inicializar constantes complexas, como um `Map` imutável de configuração, ou para carregar um driver JDBC legado (`Class.forName(...)` dentro de bloco estático era padrão comum antes do JDBC 4).
- Um antipadrão que você pode encontrar por aí (mas **não deve usar**) é o chamado _double brace initialization_ — usar uma classe anônima com bloco de instância para "simular" a inicialização de uma coleção numa linha só. É considerado má prática hoje (cria uma subclasse anônima desnecessária, gera vazamento de referência sutil, e existe alternativa melhor com `List.of(...)`/`Map.of(...)`) — mas é bom reconhecer o padrão se aparecer em código antigo.

---

#### 2. Exemplo de código comentado

java

```java
public class ConfiguracaoApp {

    // Inicializador de campo simples — não precisa de bloco, expressão direta resolve
    private static final String NOME_APP = "SistemaLoja";

    // Campo estático que exige lógica mais complexa pra ser montado —
    // aqui SIM faz sentido um bloco estático
    private static final java.util.Map<String, String> CODIGOS_ERRO;

    static {
        // Map.of() só permite poucos pares de forma direta e é imutável;
        // se a lógica for mais elaborada (ex: vier de múltiplas fontes,
        // tiver validação), um bloco estático com HashMap mutável,
        // depois "travado", é a abordagem clássica
        java.util.Map<String, String> mapaTemp = new java.util.HashMap<>();
        mapaTemp.put("404", "Recurso não encontrado");
        mapaTemp.put("500", "Erro interno do servidor");
        mapaTemp.put("403", "Acesso negado");

        CODIGOS_ERRO = java.util.Collections.unmodifiableMap(mapaTemp);
        // Collections.unmodifiableMap: converte o mapa mutável usado durante
        // a montagem em uma "vista" imutável, impedindo alteração depois
        // que a classe termina de carregar.

        System.out.println("[Estático] Configuração de códigos de erro carregada");
    }

    // --- Agora, blocos de INSTÂNCIA ---

    private final String id;
    private final long criadoEm;

    // Bloco de instância: roda antes de QUALQUER construtor abaixo,
    // útil pra lógica que TODOS os construtores precisam compartilhar
    {
        this.criadoEm = System.currentTimeMillis();
        System.out.println("[Instância] Registrando timestamp de criação");
    }

    public ConfiguracaoApp() {
        this.id = "config-padrao";
        System.out.println("[Construtor] Sem parâmetros");
    }

    public ConfiguracaoApp(String idCustomizado) {
        this.id = idCustomizado;
        System.out.println("[Construtor] Com id customizado: " + idCustomizado);
    }

    public String getId() { return id; }
    public long getCriadoEm() { return criadoEm; }
}
```

java

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("--- Criando primeira instância ---");
        ConfiguracaoApp c1 = new ConfiguracaoApp();

        System.out.println("--- Criando segunda instância (outro construtor) ---");
        ConfiguracaoApp c2 = new ConfiguracaoApp("config-especial");
    }
}
```

```
Saída:
--- Criando primeira instância ---
[Estático] Configuração de códigos de erro carregada
[Instância] Registrando timestamp de criação
[Construtor] Sem parâmetros
--- Criando segunda instância (outro construtor) ---
[Instância] Registrando timestamp de criação
[Construtor] Com id customizado: config-especial
```

Note: o bloco estático só aparece **uma vez** (na primeira criação, quando a classe é carregada). Já o bloco de instância aparece **em ambas** as criações, **independente de qual construtor** foi chamado — essa é exatamente a vantagem prática descrita na Teoria: `criadoEm` é preenchido de forma garantida, sem precisar duplicar `this.criadoEm = System.currentTimeMillis();` nos dois construtores.

java

```java
// Exemplo do risco de exceção em bloco estático
public class ConfigCritica {
    static {
        int resultado = 10 / 0; // ArithmeticException em runtime
    }
}
// Ao tentar usar ConfigCritica pela primeira vez em qualquer lugar do código:
// ExceptionInInitializerError, causado por: java.lang.ArithmeticException: / by zero
// A classe fica numa condição de "falha permanente de carregamento" —
// qualquer tentativa futura de usá-la na mesma execução da aplicação
// lança NoClassDefFoundError.
```

---

#### 3. Armadilhas comuns

1. **Não perceber que múltiplos blocos rodam em ordem sequencial de declaração.** Se você espalha blocos de instância e inicializadores de campo misturados na classe, achando que "todos os `static` rodam primeiro, depois todos os de instância, na ordem que eu quiser", o resultado real depende estritamente da posição no arquivo, de cima para baixo. Um campo declarado **depois** de um bloco que tenta usá-lo ainda vai estar com valor padrão (0/null) nesse ponto — é um erro de compilação em alguns casos ("illegal forward reference"), mas nem sempre; vale testar e não assumir.
2. **Usar bloco de instância quando o lugar certo seria simplesmente o construtor.** Na grande maioria dos casos modernos, com um único construtor, não há motivo real pra usar bloco de instância — só adiciona uma camada extra de indireção que quem lê o código precisa "descobrir" que existe (muita gente nem sabe que instance initializer block é sintaxe válida, e pode nem perceber que ele existe/roda ao ler a classe rapidamente). Reserve-o genuinamente para quando há **múltiplos construtores** com lógica compartilhada.
3. **Deixar uma exceção escapar de um bloco estático e não entender o `ExceptionInInitializerError`/`NoClassDefFoundError` resultante.** Isso é particularmente confuso porque o erro aparece "longe" da causa raiz — geralmente na primeira linha de código que tenta usar a classe, não na linha do bloco estático em si, o que dificulta o diagnóstico se você não souber que esse mecanismo existe.
4. **Confundir bloco de inicialização com bloco de instância anônimo usado pra "double brace initialization".** Um padrão como:

java

```java
   List<String> lista = new ArrayList<>() {{
       add("a");
       add("b");
   }};
```

Isso parece um "bloco de inicialização" mas na verdade é uma **classe anônima** (que estende `ArrayList`) com um bloco de instância dentro dela. Funciona, mas cria uma subclasse anônima desnecessária a cada uso (com overhead de classe extra e uma referência implícita à classe externa, que pode causar vazamento de memória sutil se a lista escapar do escopo). A alternativa moderna e preferida é `List.of("a", "b")` ou, se precisar de mutabilidade, `new ArrayList<>(List.of("a", "b"))`.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie uma classe `Log` com um bloco estático que imprime `"Sistema de log inicializado"`, e um campo estático `contador` inicializado em `0` via inicializador de campo simples (não bloco). Crie um construtor que incrementa `contador` e imprime o valor atual. No `main`, crie 3 objetos `Log` e observe/comente a saída.  
_Critério de pronto:_ a mensagem do bloco estático aparece uma única vez, e o contador imprime 1, 2, 3 nas três criações.

**Exercício 2 (Médio)**  
Crie uma classe `Pedido` com **dois construtores**: um que recebe só `String descricao`, e outro que recebe `String descricao` e `int prioridade`. Use um bloco de inicialização de instância para atribuir um campo `final String protocolo`, gerado a partir de `"PED-" + System.nanoTime()`, garantindo que **ambos** os construtores tenham esse protocolo preenchido sem duplicar a lógica em cada um.  
_Critério de pronto:_ criar um `Pedido` por qualquer um dos dois construtores resulta em um `protocolo` preenchido e único; a lógica de geração do protocolo aparece escrita uma única vez no código-fonte.

**Exercício 3 (Difícil)**  
Crie uma classe `TabelaDeConversao` com um campo estático final `Map<String, Double> fatoresParaMetros`, populado dentro de um bloco estático com pelo menos 4 unidades (ex: `"km"` → 1000.0, `"cm"` → 0.01, `"milha"` → 1609.34, `"pe"` → 0.3048), tornado imutável ao final via `Collections.unmodifiableMap`. Adicione um método estático `converterParaMetros(String unidade, double valor)` que usa esse mapa. Demonstre, com um `try/catch` no `main`, o que acontece ao chamar esse método com uma unidade que não existe no mapa (deve ser tratado de forma clara, sem deixar uma exceção genérica não-tratada estourar).  
_Critério de pronto:_ conversões válidas retornam o valor correto; uma unidade inexistente é tratada com uma mensagem de erro clara, não um stack trace cru.

**Exercício 4 (Desafio)**  
Demonstre na prática o cenário de `ExceptionInInitializerError`: crie uma classe `ConfiguracaoPerigosa` cujo bloco estático tenta ler um valor de um `Map` vazio (`mapaVazio.get("chave-inexistente")`) e usá-lo diretamente sem checar `null` (ex: chamando `.length()` num resultado `null`, causando `NullPointerException`). No `main`, dentro de um `try/catch` para `Throwable` (não `Exception` — pesquise/raciocine por que precisa ser `Throwable` aqui), tente instanciar `ConfiguracaoPerigosa` duas vezes seguidas, e imprima o tipo da exceção capturada em cada tentativa, comentando a diferença entre a primeira e a segunda captura.  
_Critério de pronto:_ a primeira tentativa captura `ExceptionInInitializerError` (ou o `Throwable` correspondente); a segunda tentativa captura algo diferente (`NoClassDefFoundError`), e o código comenta por que isso muda entre a primeira e a segunda tentativa.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Log {
    static {
        System.out.println("Sistema de log inicializado");
    }

    private static int contador = 0;

    public Log() {
        contador++;
        System.out.println("Log criado. Contador: " + contador);
    }
}

public class Main {
    public static void main(String[] args) {
        new Log();
        new Log();
        new Log();
    }
}
```

```
Saída:
Sistema de log inicializado
Log criado. Contador: 1
Log criado. Contador: 2
Log criado. Contador: 3
```

_Raciocínio:_ confirma na prática o que a Teoria descreve — o bloco estático é ligado ao **carregamento da classe** (evento único), enquanto o construtor roda a cada `new` (evento repetido), e ambos compartilham o mesmo campo estático `contador`, que persiste seu valor entre as chamadas porque pertence à classe, não a cada instância.

**Exercício 2**

java

```java
public class Pedido {
    private final String descricao;
    private final int prioridade;
    private final String protocolo;

    // Bloco de instância: roda ANTES de qualquer um dos dois construtores abaixo,
    // então o protocolo é gerado uma única vez, num único lugar do código,
    // não importa qual construtor o chamador use
    {
        this.protocolo = "PED-" + System.nanoTime();
    }

    public Pedido(String descricao) {
        this.descricao = descricao;
        this.prioridade = 0; // prioridade padrão
    }

    public Pedido(String descricao, int prioridade) {
        this.descricao = descricao;
        this.prioridade = prioridade;
    }

    @Override
    public String toString() {
        return protocolo + " | " + descricao + " (prioridade " + prioridade + ")";
    }
}

// Uso:
Pedido p1 = new Pedido("Entrega urgente");
Pedido p2 = new Pedido("Entrega normal", 2);
System.out.println(p1);
System.out.println(p2);
// Cada um com seu próprio protocolo único, gerado sem duplicar código
```

_Raciocínio:_ esse é o caso de uso "livro-texto" de bloco de instância: **múltiplos construtores** que precisam de um pedaço de lógica de inicialização comum. Se essa lógica fosse copiada em cada construtor, qualquer mudança futura na forma de gerar o protocolo exigiria lembrar de atualizar em todos os lugares — risco real de inconsistência. Colocando no bloco, existe uma fonte única de verdade.

**Exercício 3**

java

```java
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

public class TabelaDeConversao {

    private static final Map<String, Double> fatoresParaMetros;

    static {
        Map<String, Double> temp = new HashMap<>();
        temp.put("km", 1000.0);
        temp.put("cm", 0.01);
        temp.put("milha", 1609.34);
        temp.put("pe", 0.3048);

        fatoresParaMetros = Collections.unmodifiableMap(temp);
    }

    public static double converterParaMetros(String unidade, double valor) {
        Double fator = fatoresParaMetros.get(unidade);
        if (fator == null) {
            throw new IllegalArgumentException("Unidade não suportada: " + unidade);
        }
        return valor * fator;
    }
}

public class Main {
    public static void main(String[] args) {
        System.out.println(TabelaDeConversao.converterParaMetros("km", 5));
        // 5000.0

        try {
            TabelaDeConversao.converterParaMetros("jarda", 3);
        } catch (IllegalArgumentException e) {
            System.out.println("Erro tratado: " + e.getMessage());
            // Erro tratado: Unidade não suportada: jarda
        }
    }
}
```

_Raciocínio:_ o bloco estático aqui resolve exatamente o problema que `Map.of(...)` sozinho não resolveria tão bem se a lógica de montagem fosse mais elaborada (aqui está simples de propósito, mas em cenário real poderia envolver leitura de arquivo, validação cruzada entre unidades, etc.). A checagem de `null` explícita antes de lançar `IllegalArgumentException` (uma exceção **unchecked**, escolhida de propósito) evita que o método simplesmente devolva um `NullPointerException` "cru" e pouco informativo pro chamador — erro de API mal desenhada muito comum.

**Exercício 4**

java

```java
import java.util.HashMap;
import java.util.Map;

public class ConfiguracaoPerigosa {
    private static final Map<String, String> mapaVazio = new HashMap<>();

    static {
        String valor = mapaVazio.get("chave-inexistente"); // retorna null, não lança nada ainda
        int tamanho = valor.length();
        // NullPointerException AQUI, dentro do bloco estático
    }
}

public class Main {
    public static void main(String[] args) {
        System.out.println("--- Primeira tentativa ---");
        try {
            new ConfiguracaoPerigosa();
        } catch (Throwable t) {
            // Precisa ser Throwable, não Exception, porque ExceptionInInitializerError
            // e NoClassDefFoundError são subclasses de Error, não de Exception —
            // Error representa problemas mais sérios/estruturais da JVM, então
            // o catch(Exception e) simplesmente NÃO capturaria esses casos.
            System.out.println("Capturado: " + t.getClass().getSimpleName());
            // Capturado: ExceptionInInitializerError
        }

        System.out.println("--- Segunda tentativa ---");
        try {
            new ConfiguracaoPerigosa();
        } catch (Throwable t) {
            System.out.println("Capturado: " + t.getClass().getSimpleName());
            // Capturado: NoClassDefFoundError
        }
    }
}
```

_Raciocínio da diferença entre a 1ª e a 2ª captura:_ na **primeira** tentativa, a JVM realmente tenta carregar `ConfiguracaoPerigosa` pela primeira vez, executa o bloco estático, e a `NullPointerException` que escapa dali é encapsulada em `ExceptionInInitializerError` — essa é a exceção "original" do problema. Como o carregamento **falhou**, a JVM marca a classe como **irremediavelmente não-inicializável** para o resto da execução do programa. Por isso, na **segunda** tentativa, a JVM nem tenta rodar o bloco estático de novo (sabe que já falhou) — ela lança diretamente `NoClassDefFoundError`, um erro diferente, que significa essencialmente "essa classe já provou que não pode ser carregada, não vou nem tentar de novo". Esse é o motivo pelo qual código dentro de blocos estáticos deve ser tratado com extremo cuidado contra exceções — o efeito de uma falha ali é muito mais destrutivo e permanente (para aquela execução da aplicação) do que uma exceção comum em um método qualquer.