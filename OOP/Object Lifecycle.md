#### 1. Teoria

**Object Lifecycle** (ciclo de vida do objeto) é o conjunto de fases pelas quais todo objeto Java passa: **criação → uso → elegibilidade para coleta de lixo → destruição (coleta)**. Entender isso é fundamental pra escrever código que não vaza memória e pra saber exatamente em que ordem código de inicialização executa numa classe complexa (com herança, blocos estáticos, blocos de instância, etc.).

**Fase 1 — Carregamento da classe (Class Loading)**  
Antes de qualquer objeto existir, a **classe** precisa ser carregada pela JVM (uma única vez, na primeira vez que ela é referenciada). Nesse momento rodam os **blocos estáticos** e são inicializados os **campos estáticos**, na ordem em que aparecem no código.

**Fase 2 — Criação do objeto (`new`)**  
Quando você chama `new NomeDaClasse(...)`, acontece nesta ordem, sempre:

1. Memória é alocada para o objeto.
2. Campos de instância recebem seus valores-padrão (0, `false`, `null` — antes de qualquer código seu rodar).
3. **Construtor da superclasse** é chamado primeiro (implícito `super()` se você não escrever, ou explícito se você chamar `super(args)`).
4. **Blocos de inicialização de instância** e **inicializadores de campo** rodam, na ordem em que aparecem no código-fonte.
5. O **corpo do construtor** da própria classe roda por último.

Isso vale mesmo em uma cadeia de herança: a JVM sempre monta o objeto de "cima pra baixo" — primeiro garante que a superclasse esteja completamente construída, para só então construir a subclasse em cima dela.

**Fase 3 — Uso do objeto**  
O objeto existe na _heap_ (área de memória para objetos) enquanto houver pelo menos uma referência viva apontando para ele a partir de algum lugar alcançável (variável local em uso, campo de um objeto vivo, etc.).

**Fase 4 — Elegibilidade para Garbage Collection (GC)**  
Quando **nenhuma referência viva** aponta mais para o objeto (ex: a variável saiu de escopo, ou foi reatribuída para outra coisa), o objeto se torna **elegível** para coleta de lixo. Isso não significa que ele é destruído imediatamente — o Garbage Collector da JVM decide **quando** rodar, de forma automática e não determinística. Você não controla o momento exato.

**Fase 5 — Finalização e coleta**  
Antes do Java 9, existia o método `finalize()`, chamado pelo GC antes de destruir o objeto — mas ele foi **deprecated** (formalmente descontinuado) porque seu comportamento é imprevisível (pode nem rodar, roda em thread separada, pode causar vazamento se lançar exceção). A prática moderna recomendada, quando você precisa liberar recursos (arquivos, conexões, sockets), é usar `try-with-resources` com a interface `AutoCloseable`, não depender do ciclo de vida do GC.

**Diferença de Object Lifecycle vs. Garbage Collection:**  
São coisas relacionadas mas não sinônimas. _Object Lifecycle_ é o conceito amplo (todo o percurso do objeto). _Garbage Collection_ é especificamente o mecanismo/algoritmo que a JVM usa na fase de "destruição" desse ciclo — é uma etapa do lifecycle, não o lifecycle inteiro.

**Onde aparece no dia a dia de backend Java/Spring:**

- Entender a ordem construtor da superclasse → campos → construtor da subclasse é essencial pra debugar `NullPointerException` estranho em hierarquias de classe (ex: chamar um método sobrescrito dentro do construtor da superclasse, antes da subclasse terminar de inicializar seus próprios campos — armadilha clássica, ver abaixo).
- Recursos como conexão de banco, arquivo aberto, cliente HTTP: **nunca** confie no GC pra fechá-los. Use `try-with-resources` ou, no Spring, deixe o framework gerenciar o ciclo de vida do bean (`@PreDestroy`, `DisposableBean`).
- Beans do Spring, na verdade, têm seu **próprio** ciclo de vida gerenciado pelo container (instanciação → injeção de dependência → `@PostConstruct` → uso → `@PreDestroy`) — isso é uma camada em cima do Object Lifecycle da JVM, e você vai ver isso com profundidade quando chegar em Spring/Dependency Injection.

---

#### 2. Exemplo de código comentado

java

```java
public class Animal {
    // Bloco estático: roda UMA ÚNICA VEZ, quando a classe é carregada pela primeira vez
    static {
        System.out.println("1. Bloco estático de Animal");
    }

    // Bloco de inicialização de instância: roda TODA VEZ que um objeto é criado,
    // depois do super() e antes do corpo do construtor
    {
        System.out.println("3. Bloco de instância de Animal");
    }

    public Animal() {
        System.out.println("4. Construtor de Animal");
    }
}

public class Cachorro extends Animal {
    static {
        System.out.println("2. Bloco estático de Cachorro");
    }

    {
        System.out.println("5. Bloco de instância de Cachorro");
    }

    public Cachorro() {
        // super() é chamado IMPLICITAMENTE aqui, mesmo sem escrever
        System.out.println("6. Construtor de Cachorro");
    }
}

public class Main {
    public static void main(String[] args) {
        System.out.println("--- Criando o primeiro Cachorro ---");
        new Cachorro();

        System.out.println("--- Criando o segundo Cachorro ---");
        new Cachorro();
    }
}
```

```
Saída:
--- Criando o primeiro Cachorro ---
1. Bloco estático de Animal
2. Bloco estático de Cachorro
3. Bloco de instância de Animal
4. Construtor de Animal
5. Bloco de instância de Cachorro
6. Construtor de Cachorro
--- Criando o segundo Cachorro ---
7. Bloco de instância de Animal
8. Construtor de Animal
9. Bloco de instância de Cachorro
10. Construtor de Cachorro
```

Note que os blocos estáticos (linhas 1 e 2) **só aparecem uma vez**, mesmo criando dois objetos — porque pertencem à classe, carregada uma única vez. Já os blocos de instância e construtores (3 a 6) rodam **em toda criação de objeto**, sempre respeitando: superclasse inteira primeiro, depois subclasse.

java

```java
// Exemplo de recurso liberado corretamente com try-with-resources,
// SEM depender do GC ou de finalize()
public void lerArquivo(String caminho) {
    try (java.io.BufferedReader reader = new java.io.BufferedReader(new java.io.FileReader(caminho))) {
        String linha = reader.readLine();
        System.out.println(linha);
    } catch (java.io.IOException e) {
        System.out.println("Erro ao ler arquivo: " + e.getMessage());
    }
    // O reader é fechado automaticamente aqui, garantido, mesmo se der exceção.
    // Isso NÃO tem relação com Garbage Collection — é determinístico, acontece
    // no momento exato em que o bloco try termina, não "quando o GC quiser".
}
```

java

```java
// Exemplo de referência tornando-se elegível para GC
public void exemploEscopo() {
    Object objeto = new Object(); // objeto criado, referência viva em 'objeto'
    // ... uso de 'objeto' aqui ...
} // 'objeto' sai de escopo aqui -> nenhuma referência viva -> elegível para GC
  // MAS: não sabemos QUANDO a JVM vai efetivamente coletar. Pode ser já,
  // pode ser daqui a segundos, depende do estado da heap.
```

---

#### 3. Armadilhas comuns

1. **Chamar um método sobrescritível dentro do construtor da superclasse.** Se `Animal` chamar `this.metodoQueCachorroSobrescreve()` dentro do seu próprio construtor, e `Cachorro` sobrescrever esse método usando um campo próprio, esse campo **ainda não foi inicializado** (porque o construtor de `Cachorro` nem começou a rodar — ver ordem na Teoria). Resultado clássico: `NullPointerException` ou valor `0`/`null` inesperado, difícil de debugar porque "o código parece certo". Regra geral: evite chamar métodos sobrescritíveis (`não-final`, não-`private`) dentro de construtores.
2. **Achar que `System.gc()` força a coleta de lixo imediatamente.** Esse método é apenas uma **sugestão/pedido** para a JVM — ela pode ignorar completamente. Nunca escreva código que depende de `System.gc()` ter efeito garantido.
3. **Confiar em `finalize()` para fechar recursos.** Além de deprecated, `finalize()` pode simplesmente nunca ser chamado se a aplicação encerrar antes do GC agir, causando vazamento de arquivo/conexão/socket aberto. Sempre prefira `try-with-resources`.
4. **Esquecer que um objeto pode "vazar" mesmo saindo de escopo local, se ainda houver referência em outro lugar vivo** (ex: guardado numa coleção estática, num listener registrado, num cache). Esse é o cenário clássico de _memory leak_ em Java — o objeto nunca fica elegível para GC porque uma referência esquecida (geralmente estática, ou um listener/callback nunca removido) continua viva por toda a duração da aplicação.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie duas classes, `Veiculo` e `Carro` (que estende `Veiculo`). Em cada uma, adicione um construtor que imprime seu próprio nome (ex: `"Construtor de Veiculo"`). Crie um objeto `Carro` no `main` e explique, em um comentário, a ordem exata de execução que você observa e por quê.  
_Critério de pronto:_ o código roda, imprime a ordem correta (superclasse antes de subclasse), e o comentário justifica isso com base na Teoria (chamada implícita de `super()`).

**Exercício 2 (Médio)**  
Reproduza um exemplo próprio (diferente do usado na Teoria) demonstrando a ordem completa: bloco estático, bloco de instância, e construtor, numa hierarquia de duas classes. Adicione um contador estático (`static int contadorDeObjetos`) que é incrementado em cada construção, e imprima esse contador ao final de cada criação, para provar visualmente que o bloco estático realmente roda uma única vez enquanto o contador segue subindo a cada novo objeto.  
_Critério de pronto:_ a saída mostra claramente o bloco estático aparecendo só na primeira criação, e o contador aumentando (1, 2, 3...) a cada novo objeto criado depois.

**Exercício 3 (Difícil)**  
Escreva uma classe `Pai` cujo construtor chama um método `mostrarValor()` (não-`final`, não-`private`, portanto sobrescritível). Escreva uma classe `Filho` que estende `Pai`, tem um campo `private final String valor = "inicializado"`, e sobrescreve `mostrarValor()` para imprimir esse campo. Crie um objeto `Filho` no `main` e explique, com base no que você observar rodando, por que o valor impresso não é o que "intuitivamente" se esperaria.  
_Critério de pronto:_ o código roda e demonstra o problema descrito na Armadilha 1 (o campo aparece como `null` na primeira chamada, mesmo tendo um valor "óbvio" atribuído na declaração), com uma explicação escrita do porquê, ligando à ordem de inicialização vista na Teoria.

**Exercício 4 (Desafio)**  
Escreva um método que demonstra, na prática, uma situação de referência "presa" (memory leak conceitual) usando uma lista `static` de um tipo `Cache` que nunca é limpa. Crie objetos dentro de um loop, adicione-os à lista estática, e depois, mesmo removendo todas as referências locais, explique (em comentário) por que esses objetos **não** são elegíveis para GC — e reescreva o método de forma corrigida, limpando a lista estática ao final, mostrando a diferença.  
_Critério de pronto:_ duas versões do código (a "com vazamento" e a "corrigida"), cada uma com um comentário explicando exatamente qual referência mantém (ou deixa de manter) o objeto vivo.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Veiculo {
    public Veiculo() {
        System.out.println("Construtor de Veiculo");
    }
}

public class Carro extends Veiculo {
    public Carro() {
        // super() é chamado implicitamente ANTES da primeira linha deste construtor,
        // mesmo eu não escrevendo — por isso "Construtor de Veiculo" imprime primeiro.
        System.out.println("Construtor de Carro");
    }
}

public class Main {
    public static void main(String[] args) {
        new Carro();
        // Saída:
        // Construtor de Veiculo
        // Construtor de Carro
        //
        // A JVM SEMPRE garante que a superclasse esteja 100% construída
        // antes de começar a construir a subclasse por cima dela.
    }
}
```

_Raciocínio:_ isso é consequência direta da Fase 2 descrita na Teoria — passo 3 (construtor da superclasse) sempre acontece antes do passo 5 (corpo do construtor da própria classe).

**Exercício 2**

java

```java
public class Sensor {
    static {
        System.out.println("[Estático] Classe Sensor carregada");
    }
    {
        System.out.println("[Instância] Bloco de Sensor");
    }
    public Sensor() {
        System.out.println("[Construtor] Sensor");
    }
}

public class SensorTemperatura extends Sensor {
    static int contadorDeObjetos = 0;

    static {
        System.out.println("[Estático] Classe SensorTemperatura carregada");
    }
    {
        System.out.println("[Instância] Bloco de SensorTemperatura");
    }
    public SensorTemperatura() {
        contadorDeObjetos++;
        System.out.println("[Construtor] SensorTemperatura #" + contadorDeObjetos);
    }
}

public class Main {
    public static void main(String[] args) {
        new SensorTemperatura();
        System.out.println("---");
        new SensorTemperatura();
        System.out.println("---");
        new SensorTemperatura();
    }
}
```

```
Saída:
[Estático] Classe Sensor carregada
[Estático] Classe SensorTemperatura carregada
[Instância] Bloco de Sensor
[Construtor] Sensor
[Instância] Bloco de SensorTemperatura
[Construtor] SensorTemperatura #1
---
[Instância] Bloco de Sensor
[Construtor] Sensor
[Instância] Bloco de SensorTemperatura
[Construtor] SensorTemperatura #2
---
[Instância] Bloco de Sensor
[Construtor] Sensor
[Instância] Bloco de SensorTemperatura
[Construtor] SensorTemperatura #3
```

_Raciocínio:_ os blocos `[Estático]` só aparecem **antes do primeiro `new`**, nunca mais depois — comprovando que carregamento de classe acontece uma única vez. Já `contadorDeObjetos`, embora também seja `static` (uma cópia só, compartilhada), tem seu **valor** mutado a cada construtor — isso mostra a diferença entre "bloco estático roda uma vez" e "campo estático existe uma vez mas pode mudar de valor várias vezes".

**Exercício 3**

java

```java
public class Pai {
    public Pai() {
        System.out.println("Construtor de Pai chamando mostrarValor():");
        mostrarValor(); // PERIGO: chamando método sobrescritível dentro do construtor
    }

    public void mostrarValor() {
        System.out.println("Pai: sem valor específico");
    }
}

public class Filho extends Pai {
    private final String valor = "inicializado";

    public Filho() {
        super(); // implícito, mas explicitado aqui pra clareza
        System.out.println("Construtor de Filho terminou, valor = " + valor);
    }

    @Override
    public void mostrarValor() {
        System.out.println("Filho: valor = " + valor);
    }
}

public class Main {
    public static void main(String[] args) {
        new Filho();
    }
}
```

```
Saída:
Construtor de Pai chamando mostrarValor():
Filho: valor = null       <- aqui está o problema
Construtor de Filho terminou, valor = inicializado
```

_Raciocínio:_ isso acontece porque, mesmo `mostrarValor()` sendo sobrescrito em `Filho`, o **objeto é o mesmo objeto sendo construído** — polimorfismo em runtime chama a versão de `Filho`, não a de `Pai` (isso é _dynamic binding_, que você já viu em tópico anterior). Só que, no momento em que `Pai()` está rodando, o construtor de `Filho` **ainda nem começou** (estamos no passo 3 da ordem descrita na Teoria: construtor da superclasse). O campo `valor` de `Filho` só recebe `"inicializado"` no passo 4/5, que acontece **depois**. Resultado: o método sobrescrito roda, mas lê um campo que ainda está no seu valor-padrão (`null`, pois é `String`), mesmo sendo `final` — `final` garante que só pode ser atribuído uma vez, não que a atribuição acontece antes de qualquer outro código rodar.

**Exercício 4**

java

```java
import java.util.ArrayList;
import java.util.List;

public class Cache {
    private final byte[] dadosGrandes = new byte[1_000_000]; // simula objeto "pesado"
    private final int id;

    public Cache(int id) {
        this.id = id;
    }
}

public class ExemploVazamento {

    // Lista estática: vive enquanto a APLICAÇÃO viver, não enquanto um método vive
    private static final List<Cache> cacheComVazamento = new ArrayList<>();

    public void criarObjetosComVazamento() {
        for (int i = 0; i < 1000; i++) {
            Cache c = new Cache(i);
            cacheComVazamento.add(c); // referência extra, guardada FORA do escopo do método
        }
        // 'c' (a variável local) sai de escopo ao fim do loop/método,
        // MAS cada objeto Cache ainda tem uma referência viva dentro de
        // cacheComVazamento, que é 'static' — vive até a lista ser limpa
        // ou a aplicação encerrar. Nenhum desses 1000 objetos é elegível para GC.
    }

    // Versão corrigida
    public void criarObjetosSemVazamento() {
        List<Cache> cacheLocal = new ArrayList<>(); // lista LOCAL, não estática

        for (int i = 0; i < 1000; i++) {
            cacheLocal.add(new Cache(i));
        }

        // ... uso de cacheLocal aqui, se necessário ...

    } // cacheLocal sai de escopo aqui -> nenhuma referência viva a ela nem aos
      // Cache que ela continha -> TODOS os 1000 objetos ficam elegíveis para GC
      // assim que o método termina.
}
```

_Raciocínio:_ a causa raiz do vazamento não é "esquecer de dar `null`" em algo — é que uma coleção **`static`**, por definição, vive pelo tempo de vida da própria classe (praticamente a aplicação inteira). Qualquer objeto adicionado a ela permanece alcançável (e portanto não-elegível para GC) até alguém explicitamente removê-lo da coleção (`cacheComVazamento.clear()` ou `.remove(...)`). Isso é exatamente o padrão mais comum de memory leak real em aplicações Java de produção — não é ponteiro solto como em C, é referência **esquecida e ainda alcançável**.