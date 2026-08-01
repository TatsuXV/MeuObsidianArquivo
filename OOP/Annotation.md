#### 1. Teoria

**Annotation** (anotação) é uma forma de adicionar **metadados** ao código — informação **sobre** o código, que não altera diretamente a lógica de execução do programa, mas que pode ser lida e processada por: o compilador, ferramentas de build, frameworks em runtime (via _reflection_), ou até por outras anotações/processadores de anotação. Sintaticamente, toda anotação começa com `@`.

Você já usou anotações antes sem necessariamente ter parado pra formalizar o conceito — `@Override`, que você usou desde o tópico de Method Overloading/Overriding, é uma anotação.

**Três formas de uma anotação ser processada:**

1. **Só pelo compilador, em tempo de compilação**, sem gerar nada no `.class` final. Ex: `@Override` — existe só pra o compilador checar "esse método realmente sobrescreve algo da superclasse?" e dar erro se não. Depois de compilado, essa informação não sobrevive no bytecode.
2. **Mantida no `.class` mas lida via reflection, em runtime, por bibliotecas/frameworks.** Ex: `@Autowired`, `@Service`, `@RestController` do Spring — o framework escaneia as classes em runtime, procurando essas anotações, e decide o que fazer baseado nelas (injetar dependência, registrar como bean, mapear rota HTTP).
3. **Processada em tempo de compilação por um _annotation processor_, gerando código novo.** Ex: Lombok (`@Getter`, `@Setter`) gera getters/setters automaticamente durante a compilação — você não os escreve, mas eles existem no `.class` final.

**Anotações built-in mais comuns do `java.lang`:**

- `@Override` — já visto.
- `@Deprecated` — marca um método/classe como obsoleto; o compilador emite um aviso (warning) em qualquer uso.
- `@SuppressWarnings("tipo")` — instrui o compilador a **não** emitir um aviso específico naquele ponto (ex: `@SuppressWarnings("unchecked")` em cast genérico).
- `@FunctionalInterface` — marca uma interface como tendo exatamente um método abstrato, permitindo uso com lambda; o compilador dá erro se a interface tiver mais de um método abstrato. Você vai ver isso com profundidade quando chegar em Functional Interfaces, no Bloco 12.

**Como criar sua própria anotação:**  
Você define uma anotação com a palavra-chave `@interface`. Para controlar como ela se comporta, usa-se **meta-anotações** (anotações que anotam outras anotações):

- `@Retention` — define até quando a anotação "sobrevive": `SOURCE` (só no código-fonte, descartada na compilação), `CLASS` (vai pro `.class`, mas não é lida em runtime — padrão se você não especificar), `RUNTIME` (disponível via reflection em tempo de execução — a mais usada quando você quer que um framework leia sua anotação).
- `@Target` — define **onde** a anotação pode ser aplicada: `TYPE` (classe/interface), `METHOD`, `FIELD`, `PARAMETER`, `CONSTRUCTOR`, etc. Pode aceitar múltiplos.

**Diferença de Annotation vs. Interface (sintaticamente parecidas, mas totalmente diferentes):**  
Embora `@interface` pareça uma variação de `interface`, o propósito é completamente diferente: interface define um **contrato de comportamento** que uma classe implementa. Annotation define **metadados descritivos** que não implementam nada — uma classe não "implementa" uma anotação, ela é **anotada por** ela.

**Onde aparece no dia a dia de backend Java/Spring:**  
Isso é praticamente onipresente no ecossistema Spring moderno — é impossível trabalhar com Spring Boot sem esbarrar em anotações a cada arquivo: `@RestController`, `@GetMapping`, `@Autowired`, `@Entity`, `@Id`, `@Transactional`, `@Valid`, `@Test` (JUnit), `@Mock` (Mockito). Entender que anotação é só metadado, e que **é o framework que decide o que fazer com ela via reflection**, tira o "mistério" de como o Spring parece "adivinhar" o que fazer só de você colocar uma anotação em cima de uma classe.

---

#### 2. Exemplo de código comentado

java

```java
// Exemplo 1: usando anotações built-in
public class Veiculo {

    @Deprecated
    public void ligarMotorAntigo() {
        // Método mantido só por compatibilidade; qualquer uso gera warning
        // do compilador, incentivando quem usa a migrar pro método novo.
        System.out.println("Ligando motor (forma antiga)");
    }

    public void ligarMotor() {
        System.out.println("Ligando motor");
    }

    @SuppressWarnings("unchecked")
    public void exemploComCastGenerico() {
        Object objeto = new java.util.ArrayList<String>();
        // Sem a anotação acima, essa linha geraria um warning do compilador
        // sobre "unchecked cast", já que o compilador não consegue garantir
        // em tempo de compilação que o cast genérico é seguro.
        java.util.List<String> lista = (java.util.List<String>) objeto;
    }
}
```

java

```java
// Exemplo 2: criando uma anotação própria, simples, sem parâmetros
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.las*    // (nota: usar import correto abaixo)
```

Deixa eu corrigir e mostrar isso de forma limpa e compilável:

java

```java
// Arquivo: TestavelManualmente.java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME) // precisa sobreviver até runtime, pra reflection ler depois
@Target(ElementType.METHOD)          // só pode ser usada em métodos
public @interface TestavelManualmente {
    // Anotação sem parâmetros — só "marca" o método, funciona como uma flag
}
```

java

```java
// Exemplo 3: anotação própria COM parâmetros (chamados "elementos")
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Auditoria {
    String responsavel();               // elemento obrigatório (sem valor padrão)
    String descricao() default "N/A";    // elemento opcional (tem valor padrão)
}
```

java

```java
// Uso das anotações customizadas
public class RelatorioService {

    @TestavelManualmente
    public void gerarRelatorioMensal() {
        System.out.println("Gerando relatório mensal...");
    }

    @Auditoria(responsavel = "equipe-financeiro", descricao = "Processa fechamento de caixa")
    public void processarFechamentoCaixa() {
        System.out.println("Processando fechamento de caixa...");
    }
}
```

java

```java
// Exemplo 4: LENDO a anotação via reflection em runtime — isso é, na prática,
// o mecanismo que faz frameworks como Spring "funcionarem por baixo dos panos"
import java.lang.reflect.Method;

public class LeitorDeAnotacoes {
    public static void main(String[] args) throws NoSuchMethodException {
        Method metodo = RelatorioService.class.getMethod("processarFechamentoCaixa");

        if (metodo.isAnnotationPresent(Auditoria.class)) {
            Auditoria auditoria = metodo.getAnnotation(Auditoria.class);
            System.out.println("Responsável: " + auditoria.responsavel());
            System.out.println("Descrição: " + auditoria.descricao());
        }
    }
}
// Saída:
// Responsável: equipe-financeiro
// Descrição: Processa fechamento de caixa
//
// Isso é literalmente o que o Spring faz, em escala muito maior e mais
// sofisticada, quando escaneia suas classes procurando @Service, @Autowired,
// @RequestMapping etc. — reflection lendo anotações e decidindo o que fazer.
```

---

#### 3. Armadilhas comuns

1. **Esquecer `@Retention(RetentionPolicy.RUNTIME)` numa anotação própria que precisa ser lida via reflection.** Se você criar uma anotação sem especificar `@Retention`, o padrão é `CLASS` — ela vai pro `.class`, mas `getAnnotation(...)` em runtime retorna `null`, mesmo a anotação "estando lá" no código-fonte. É um erro silencioso e frustrante de debugar até você lembrar de checar o `@Retention`.
2. **Achar que anotação, por si só, "faz alguma coisa".** Uma anotação sozinha **não executa nenhum código automaticamente**. `@Auditoria(...)` no Exemplo 3 não audita nada sozinha — é só metadado. É **necessário** um código separado (seu, ou de um framework como o Spring) que **leia** essa anotação via reflection e **decida agir** com base nela. Sem esse "leitor", a anotação é só documentação estruturada.
3. **Colocar `@Target` errado ou permissivo demais.** Se você esquece de restringir `@Target`, a anotação pode ser aplicada em lugares sem sentido (ex: uma anotação pensada só pra métodos sendo aplicada acidentalmente numa classe), e isso só vai gerar confusão pra quem usa depois — o compilador não vai te alertar se `@Target` não estiver restringindo corretamente.
4. **Confundir anotação de compilador (`@Override`, `@Deprecated`) com anotação de framework, achando que toda anotação precisa de reflection.** `@Override` nunca é lida em runtime — ela nem sobrevive à compilação (retention `SOURCE`). É puramente uma checagem estática do compilador. Nem toda anotação segue o "padrão Spring" de ser lida dinamicamente.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie uma classe `Calculadora` com um método antigo `somarAntigo(int a, int b)` marcado com `@Deprecated`, e um método novo `somar(int a, int b)` que faz a mesma coisa. No `main`, chame ambos e observe (ou provoque, se sua IDE permitir) o warning de depreciação ao usar o método antigo. Comente no código o motivo prático de se usar `@Deprecated` em vez de simplesmente apagar o método antigo.  
_Critério de pronto:_ o código compila (com warning esperado no método antigo), e o comentário explica corretamente o propósito de `@Deprecated` (compatibilidade retroativa + aviso, sem quebrar quem ainda usa).

**Exercício 2 (Médio)**  
Crie sua própria anotação `@InformacaoDoAutor` com dois elementos: `nome()` (obrigatório) e `data()` (obrigatório), com `@Retention(RetentionPolicy.RUNTIME)` e `@Target(ElementType.TYPE)` (ou seja, só pode ser usada em classes, não em métodos). Aplique essa anotação em uma classe qualquer de sua escolha, preenchendo os dois elementos.  
_Critério de pronto:_ a anotação compila, é aplicada corretamente na classe, e (bônus) você tenta aplicá-la também num método só pra confirmar que o compilador recusa, por causa do `@Target(ElementType.TYPE)`.

**Exercício 3 (Difícil)**  
Usando a anotação `@InformacaoDoAutor` do Exercício 2, escreva um programa que usa reflection (`Class.isAnnotationPresent`, `Class.getAnnotation`) para ler e imprimir o `nome()` e a `data()` da anotação aplicada na classe. Depois, crie uma segunda classe **sem** a anotação, e demonstre que seu código de leitura trata esse caso graciosamente (sem lançar `NullPointerException`), imprimindo algo como `"Sem informação de autor"`.  
_Critério de pronto:_ rodando para a classe anotada, os dados aparecem corretamente; rodando para a classe sem anotação, nenhuma exceção é lançada e uma mensagem apropriada é exibida.

**Exercício 4 (Desafio)**  
Crie uma anotação `@Validar` com um elemento `int minimo()`, com `@Target(ElementType.METHOD)` e `@Retention(RetentionPolicy.RUNTIME)`. Crie uma classe `Produto` com um método `calcularDesconto(int percentual)`, anotado com `@Validar(minimo = 0)`. Escreva um "validador manual" via reflection: um método que recebe um objeto e o nome de um método a chamar (via `String`), usa reflection pra descobrir se esse método tem `@Validar`, e — **antes** de efetivamente invocar o método com `Method.invoke(...)` — verifica se o argumento passado é maior ou igual ao `minimo()` definido na anotação, lançando `IllegalArgumentException` caso não seja.  
_Critério de pronto:_ chamar o método validador com um argumento válido invoca `calcularDesconto` normalmente; chamar com um argumento abaixo do mínimo lança a exceção **antes** de `calcularDesconto` ser executado (você pode provar isso com um `System.out.println` dentro de `calcularDesconto` que não deve aparecer no caso inválido).

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Calculadora {

    @Deprecated
    public int somarAntigo(int a, int b) {
        return a + b;
    }

    public int somar(int a, int b) {
        return a + b;
    }
}

public class Main {
    public static void main(String[] args) {
        Calculadora calc = new Calculadora();
        System.out.println(calc.somarAntigo(2, 3)); // gera warning de depreciação na compilação
        System.out.println(calc.somar(2, 3));        // sem warning
    }
}
// @Deprecated é usado em vez de simplesmente apagar o método antigo porque,
// em bibliotecas/APIs usadas por OUTRAS partes do código (ou por outros times,
// ou por consumidores externos), apagar um método quebra imediatamente
// qualquer código que ainda dependa dele. @Deprecated permite uma transição
// gradual: quem usa é avisado, tem tempo de migrar, e o método antigo só é
// removido de fato numa versão futura, depois que ninguém mais depende dele.
```

_Raciocínio:_ isso conecta diretamente com o conceito de compatibilidade retroativa em bibliotecas/APIs — algo extremamente relevante em backend, onde uma mudança "pequena" pode quebrar sistemas inteiros que dependem daquele código.

**Exercício 2**

java

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE) // só pode anotar classes/interfaces, não métodos
public @interface InformacaoDoAutor {
    String nome();
    String data();
}
```

java

```java
@InformacaoDoAutor(nome = "Ana Silva", data = "2026-07-31")
public class ServicoDeRelatorios {
    public void gerar() {
        System.out.println("Gerando relatório...");
    }

    // @InformacaoDoAutor(nome = "x", data = "y") // se descomentado AQUI, no método:
    // public void outroMetodo() {}
    // ERRO DE COMPILAÇÃO: "annotation type not applicable to this kind of declaration"
    // porque @Target(ElementType.TYPE) só permite uso em classe/interface, nunca em método.
}
```

_Raciocínio:_ esse exercício confirma na prática que `@Target` não é "documentação" — é uma restrição real, verificada pelo compilador, que impede uso indevido da anotação em lugares para os quais ela não foi projetada.

**Exercício 3**

java

```java
public class ServicoAnotado {
    // Reaproveitando a anotação e classe do Exercício 2 — supondo
    // que ServicoDeRelatorios já está anotada como acima.
}

public class ServicoSemAnotacao {
    public void executar() {
        System.out.println("Executando sem anotação de autor...");
    }
}

public class LeitorDeAutor {

    public static void imprimirAutor(Class<?> classe) {
        if (classe.isAnnotationPresent(InformacaoDoAutor.class)) {
            InformacaoDoAutor info = classe.getAnnotation(InformacaoDoAutor.class);
            System.out.println("Autor: " + info.nome() + " | Data: " + info.data());
        } else {
            System.out.println("Sem informação de autor");
        }
    }

    public static void main(String[] args) {
        imprimirAutor(ServicoDeRelatorios.class);
        // Autor: Ana Silva | Data: 2026-07-31

        imprimirAutor(ServicoSemAnotacao.class);
        // Sem informação de autor
    }
}
```

_Raciocínio:_ o método `isAnnotationPresent` **antes** de `getAnnotation` é a peça-chave aqui — é exatamente o tipo de checagem defensiva que evita o erro comum descrito na Armadilha 1 (tentar ler uma anotação ausente e receber `null`, ou pior, esquecer de checar e tomar `NullPointerException` ao tentar chamar `.nome()` num `null`).

**Exercício 4**

java

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Validar {
    int minimo();
}

class Produto {
    @Validar(minimo = 0)
    public void calcularDesconto(int percentual) {
        System.out.println("Aplicando desconto de " + percentual + "%");
    }
}

public class ValidadorReflection {

    public static void chamarComValidacao(Object alvo, String nomeMetodo, int argumento)
            throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {

        Method metodo = alvo.getClass().getMethod(nomeMetodo, int.class);

        if (metodo.isAnnotationPresent(Validar.class)) {
            Validar validar = metodo.getAnnotation(Validar.class);
            if (argumento < validar.minimo()) {
                throw new IllegalArgumentException(
                    "Argumento " + argumento + " é menor que o mínimo permitido (" + validar.minimo() + ")"
                );
            }
        }

        // Só chega aqui se passou pela validação (ou não havia @Validar)
        metodo.invoke(alvo, argumento);
    }

    public static void main(String[] args) throws Exception {
        Produto produto = new Produto();

        System.out.println("--- Chamada válida ---");
        chamarComValidacao(produto, "calcularDesconto", 10);
        // Aplicando desconto de 10%

        System.out.println("--- Chamada inválida ---");
        try {
            chamarComValidacao(produto, "calcularDesconto", -5);
        } catch (IllegalArgumentException e) {
            System.out.println("Bloqueado: " + e.getMessage());
            // "Aplicando desconto..." NÃO aparece, porque a exceção é lançada
            // ANTES de metodo.invoke(...) ser chamado
        }
    }
}
```

_Raciocínio:_ este exercício simula, em miniatura, o mecanismo real por trás de bibliotecas de validação como o Bean Validation (`@Valid`, `@Min`, `@NotNull` do Spring) — a anotação sozinha não valida nada; é o código de reflection (que, no caso do Spring, já vem pronto dentro do framework) que **intercepta** a chamada, **lê** a anotação, **decide** se deixa passar, e só **então** invoca o método real. A ordem importa: a validação precisa acontecer **antes** de `Method.invoke(...)`, senão o "efeito colateral" do método (aqui, o `println`) já teria acontecido mesmo num caso que deveria ter sido bloqueado — isso é exatamente o tipo de bug sutil que aconteceria se a validação fosse malfeita num sistema real (ex: uma operação financeira sendo executada antes da validação rodar).