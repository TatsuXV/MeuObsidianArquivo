### 1. Teoria

**O que é o "ciclo de vida" de um programa Java?**

É a sequência de etapas que o seu código percorre desde o arquivo-texto `.java` que você escreve até a execução real na máquina. Entender isso não é só curiosidade acadêmica — explica _por que_ certos erros acontecem em momentos diferentes (erro de compilação vs. erro de execução), e o que a JVM está fazendo por baixo dos panos.

#### As etapas, em ordem

**1. Escrita do código-fonte (`.java`)**

Você escreve texto seguindo as regras de sintaxe que já vimos desde o primeiro tópico. Nesse ponto, é só um arquivo de texto — a máquina ainda não entende nada disso diretamente.

**2. Compilação (`javac`)**

O compilador Java (`javac`) transforma o arquivo `.java` em **bytecode**, gerando um arquivo `.class`. Bytecode **não é** código de máquina nativo (como seria em C, por exemplo) — é uma representação intermediária, independente de sistema operacional e arquitetura de processador.

bash

```bash
javac Programa.java   # gera Programa.class
```

Erros pegos **nesta etapa** são erros de **sintaxe/compilação**: `;` faltando, tipo incompatível, variável não declarada, chave desbalanceada — tudo que já vimos como "não compila" nos tópicos anteriores acontece exatamente aqui. Se a compilação falhar, o `.class` não é gerado, e não existe execução alguma.

**3. Carregamento (Class Loading)**

Quando você manda executar (`java Programa`), a JVM aciona o **Class Loader**, responsável por localizar o(s) arquivo(s) `.class` necessários e carregá-los na memória. Isso acontece de forma dinâmica e sob demanda — nem toda classe do seu projeto é carregada de uma vez; a JVM carrega conforme precisa (isso é relevante pra entender, mais adiante, exceções como `ClassNotFoundException` e `NoClassDefFoundError`).

**4. Verificação (Bytecode Verification)**

Antes de executar, a JVM verifica se o bytecode é válido e seguro — checa se não há violação de acesso de memória, se os tipos batem nas operações, etc. Essa etapa é uma camada de segurança que impede bytecode malicioso ou corrompido de rodar.

**5. Execução (JVM + JIT)**

A JVM interpreta o bytecode e o executa. Aqui, a JVM não roda linha a linha de forma "burra" o tempo todo — ela usa um mecanismo chamado **JIT (Just-In-Time) Compiler**, que identifica trechos de código executados repetidamente (_hot spots_) e os compila para código de máquina nativo em tempo de execução, otimizando a performance ao longo da vida do programa. É por isso que a JVM é apelidada, em parte, de "HotSpot" (o nome oficial da implementação de referência da Oracle é literalmente "HotSpot JVM").

Dentro da execução, especificamente o `main`:

```
JVM localiza a classe indicada → procura o método main(String[] args) exato →
inicializa atributos estáticos da classe (uma única vez, na primeira vez que a classe é usada) →
executa o corpo do main, linha a linha, seguindo o fluxo do programa (condicionais, loops, chamadas de método)
```

Erros pegos **nesta etapa** são erros de **runtime** (tempo de execução): `ArrayIndexOutOfBoundsException`, `NullPointerException`, `ClassCastException`, `NumberFormatException` — todos esses já vistos como possibilidades nos tópicos anteriores acontecem aqui, não na compilação. O programa compila perfeitamente e só quebra quando de fato tenta executar aquele caminho específico do código.

**6. Garbage Collection (durante a execução)**

Enquanto o programa roda, objetos são criados na memória (heap). Quando um objeto não tem mais nenhuma referência apontando pra ele (nada no código consegue mais acessá-lo), ele se torna elegível para **Garbage Collection** — a JVM libera essa memória automaticamente, sem você precisar gerenciar isso manualmente (diferente de linguagens como C, onde `free()` é responsabilidade do programador). Isso será aprofundado quando chegarmos em Java Memory Model, mas o ponto central aqui é: liberar memória não é uma etapa que você chama explicitamente — é um processo contínuo, automático, rodando em paralelo à execução do seu programa.

**7. Encerramento**

O programa termina quando o `main` termina naturalmente (chega na última linha), quando alguém chama `System.exit(codigo)` explicitamente, ou quando uma exceção não tratada "sobe" até o topo sem ser capturada, encerrando o programa com stack trace impresso no console.

#### JDK, JRE e JVM — a distinção que costuma confundir

|Sigla|Nome completo|O que é|
|---|---|---|
|**JVM**|Java Virtual Machine|A "máquina virtual" que executa o bytecode. É a peça que garante "escreva uma vez, rode em qualquer lugar" — cada sistema operacional tem sua própria implementação de JVM, mas o bytecode que ela executa é sempre o mesmo.|
|**JRE**|Java Runtime Environment|JVM + bibliotecas padrão necessárias pra **rodar** um programa Java já compilado. Não inclui o compilador.|
|**JDK**|Java Development Kit|JRE + ferramentas de **desenvolvimento** (`javac`, debugger, etc.). É o que você instala pra desenvolver, não só executar.|

Hoje em dia, a distinção JRE/JDK separada praticamente não existe mais em distribuições modernas — desde o Java 11, a Oracle passou a distribuir só o JDK completo, mas vale entender a diferença conceitual porque ela ainda aparece em documentação, discussões técnicas e, ocasionalmente, em pergunta de entrevista.

**Onde isso aparece na prática (backend real)**

Entender esse ciclo explica comportamentos que parecem "mágicos" no dia a dia com Spring Boot: por que um erro de configuração aparece só quando a aplicação sobe (não na compilação), por que `@PostConstruct` roda depois da injeção de dependência mas antes do sistema aceitar requisições, e por que otimizações de performance da JVM (JIT) fazem uma aplicação Spring Boot ficar mais rápida depois de "aquecida" (rodando há alguns minutos), comparado ao primeiro request logo após o start.

---

### 2. Exemplo de código comentado

java

```java
public class CicloDeVida {

    // Atributo estático — inicializado UMA VEZ, quando a classe é carregada pela JVM,
    // antes mesmo do main começar a executar
    static int contadorEstatico = inicializarContador();

    static int inicializarContador() {
        System.out.println("1. Classe sendo carregada — inicializando atributo estático");
        return 100;
    }

    public static void main(String[] args) {
        System.out.println("2. Método main começou a executar");
        System.out.println("Valor do contador estático: " + contadorEstatico);

        // Criação de objeto — alocado na heap, referência guardada na variável local "objeto"
        Object objeto = new Object();
        System.out.println("3. Objeto criado na memória");

        // "Perdendo" a referência ao objeto — ele se torna elegível pra Garbage Collection
        // (a JVM decide QUANDO de fato coletar, não é imediato nem controlável diretamente)
        objeto = null;
        System.out.println("4. Referência removida — objeto agora é candidato a Garbage Collection");

        // Simulando um erro de RUNTIME (não de compilação) — o programa compilou perfeitamente,
        // mas vai quebrar exatamente nesta linha durante a EXECUÇÃO
        int[] numeros = {1, 2, 3};
        try {
            System.out.println(numeros[10]); // ArrayIndexOutOfBoundsException aqui
        } catch (ArrayIndexOutOfBoundsException e) {
            // Exception Handling será aprofundado em bloco próprio —
            // aqui só pra mostrar que o erro acontece em RUNTIME, não trava a compilação
            System.out.println("5. Erro de runtime capturado: " + e.getMessage());
        }

        System.out.println("6. Método main terminando — programa vai encerrar");
    } // JVM encerra o processo quando main termina, se não houver mais nada pendente
}
```

Saída esperada (na ordem exata):

```
1. Classe sendo carregada — inicializando atributo estático
2. Método main começou a executar
Valor do contador estático: 100
3. Objeto criado na memória
4. Referência removida — objeto agora é candidato a Garbage Collection
5. Erro de runtime capturado: Index 10 out of bounds for length 3
6. Método main terminando — programa vai encerrar
```

---

### 3. Armadilhas comuns

1. **Confundir erro de compilação com erro de runtime.** "Meu código não funciona" pode significar duas coisas bem diferentes: não compilou (o `.class` nem foi gerado, erro de sintaxe/tipo) ou compilou e quebrou durante a execução (erro de lógica/dado, tipo `NullPointerException`). Saber distinguir isso acelera muito o diagnóstico de qualquer bug.
2. **Achar que atributos estáticos são inicializados "quando o programa começa" de forma genérica.** Na verdade, a inicialização acontece na **primeira vez que a classe é efetivamente usada/carregada** — em programas com várias classes, isso pode não ser logo no início, dependendo de qual classe é referenciada primeiro.
3. **Achar que `= null` libera memória imediatamente.** Remover a referência só torna o objeto **elegível** pra Garbage Collection — quando exatamente a JVM vai de fato coletar aquela memória não é determinístico nem controlável diretamente pelo programador (existe `System.gc()`, mas é apenas uma "sugestão" à JVM, não uma garantia).
4. **Esperar que bytecode (`.class`) rode em qualquer JVM de qualquer versão, sem restrição.** Bytecode compilado com uma versão mais nova do JDK pode não rodar numa JVM mais antiga (erro comum: `UnsupportedClassVersionError`) — bytecode é "roda em qualquer lugar", mas _versão_ ainda importa, é preciso a JVM ser igual ou mais nova que a versão usada na compilação.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Escreva um programa com uma classe que tenha um atributo estático inicializado com uma chamada de método (como no exemplo acima), que imprime uma mensagem de "classe carregada" dentro desse método. No `main`, imprima uma mensagem antes e depois de acessar esse atributo estático pela primeira vez. Critério de pronto: você consegue prever corretamente, por escrito, a ordem exata das mensagens impressas, antes de rodar — e confirma rodando depois.

**Exercício 2 (médio)**  
Escreva um programa que **intencionalmente** cause um erro de **compilação** (ex: esquecer um `;`), rode e cole (em comentário) a mensagem de erro exata que o compilador deu. Depois, corrija esse erro, mas insira um erro de **runtime** proposital (ex: acessar índice inválido de array), rode e cole (em comentário) a mensagem de erro/exceção que aparece dessa vez. Critério de pronto: você tem os dois tipos de erro documentados em comentário no mesmo arquivo, com uma frase explicando a diferença entre quando cada um foi detectado (antes de rodar vs. durante a execução).

**Exercício 3 (difícil)**  
Escreva um programa com duas classes no mesmo arquivo: `Principal` (com `main`) e `Auxiliar` (com um atributo estático inicializado via método, que imprime uma mensagem quando é carregado). No `main`, **não referencie `Auxiliar` de forma alguma nas primeiras linhas** — imprima algumas mensagens primeiro, e só depois, mais adiante no código, use `Auxiliar` pela primeira vez (acessando o atributo estático dela, por exemplo). Critério de pronto: a mensagem de "carregamento" da classe `Auxiliar` só aparece no console depois das mensagens iniciais do `main`, provando que o carregamento de classe é sob demanda (lazy), não tudo de uma vez no início do programa.

**Exercício 4 (desafio)**  
Pesquise (usando busca na web, já que isso não é um comportamento que dá pra "adivinhar" com segurança) a diferença entre as exceções `NoClassDefFoundError` e `ClassNotFoundException` — ambas relacionadas a problemas de carregamento de classe, mas em momentos e circunstâncias diferentes. Escreva, em português, um resumo de 3-5 linhas explicando quando cada uma tipicamente acontece e por que são coisas diferentes (uma delas nem é uma `Exception`, é um `Error` — isso já é uma pista importante da diferença de gravidade/tratamento entre elas). Não precisa reproduzir os erros na prática — o objetivo aqui é pesquisa e compreensão conceitual, já que provocar esses erros de propósito exige configuração de classpath mais elaborada do que faz sentido nesta fase do aprendizado.

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    static int valorInicial = inicializar();

    static int inicializar() {
        System.out.println("Classe carregada — atributo estático inicializado");
        return 42;
    }

    public static void main(String[] args) {
        System.out.println("Main começou");
        System.out.println("Acessando atributo estático: " + valorInicial);
        System.out.println("Main terminando");
    }
}
```

Saída:

```
Classe carregada — atributo estático inicializado
Main começou
Acessando atributo estático: 42
Main terminando
```

Raciocínio: mesmo o atributo estático estando declarado **antes** do `main` no código-fonte, a mensagem "Main começou" só apareceria _depois_ da inicialização do atributo se a JVM já tivesse carregado a classe antes de rodar `main` — e é exatamente isso que acontece: a classe inteira (incluindo seus atributos estáticos) é carregada e inicializada **antes** de `main` começar a executar, porque `main` está dentro dessa mesma classe, então carregá-la é pré-requisito pra sequer encontrar o `main`.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        // ETAPA 1 (comentada pra não quebrar a compilação final do arquivo):
        // int x = 10
        // System.out.println(x);
        // Erro de COMPILAÇÃO gerado: "';' expected"
        // Detectado ANTES de qualquer execução — o .class nem chega a ser gerado.

        // ETAPA 2 — erro de RUNTIME real, deixado ativo:
        int[] numeros = {1, 2, 3};
        System.out.println(numeros[5]);
        // Erro de RUNTIME gerado: "Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException:
        // Index 5 out of bounds for length 3"
        // Detectado DURANTE a execução — o programa compilou normalmente,
        // e só quebrou quando essa linha específica realmente rodou.
    }
}
```

Raciocínio: a diferença fundamental é _quando_ o problema é detectado. O erro de compilação impede o programa de sequer existir como `.class` executável — é um problema estrutural do código-fonte. O erro de runtime só se manifesta quando o fluxo de execução realmente passa por aquela linha problemática — o resto do programa, até ali, roda normalmente.

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        System.out.println("Main começou, Auxiliar ainda não foi tocada");
        System.out.println("Fazendo outras coisas...");

        // Só AQUI, pela primeira vez, a classe Auxiliar é referenciada
        System.out.println("Agora vou acessar Auxiliar: " + Auxiliar.valor);
    }
}

class Auxiliar {
    static int valor = inicializar();

    static int inicializar() {
        System.out.println("Auxiliar sendo carregada agora, sob demanda");
        return 7;
    }
}
```

Saída:

```
Main começou, Auxiliar ainda não foi tocada
Fazendo outras coisas...
Auxiliar sendo carregada agora, sob demanda
Agora vou acessar Auxiliar: 7
```

Raciocínio: a mensagem "Auxiliar sendo carregada agora" só aparece depois das duas primeiras mensagens do `main`, mesmo `Auxiliar` sendo uma classe totalmente separada e "disponível" desde o início do programa — isso confirma que o Class Loading é _lazy_ (sob demanda): a JVM só carrega e inicializa uma classe no momento exato em que ela é efetivamente referenciada pela primeira vez, não antecipadamente.

**Exercício 4**

Resumo (baseado em pesquisa):

`ClassNotFoundException` é uma exceção checked, lançada quando o código tenta carregar uma classe dinamicamente em tempo de execução — tipicamente via `Class.forName()`, `ClassLoader.loadClass()` ou `findSystemClass()` — e a classe solicitada não é encontrada no classpath naquele momento, sendo um cenário comum ao tentar carregar drivers de banco de dados (JDBC) cujo JAR não está disponível. Já `NoClassDefFoundError` é um **Error** (não uma Exception), e ocorre numa situação diferente: a classe estava disponível durante a compilação, permitindo que o programa compilasse e "linkasse" com sucesso, mas não pode ser encontrada ou carregada em tempo de execução. Uma causa adicional interessante: esse erro também pode surgir de uma exceção lançada durante a inicialização estática de uma classe, quando essa classe (que falhou ao carregar) é referenciada novamente depois pelo runtime. Outra diferença relevante de tratamento: ClassNotFoundException é checked (exige try-catch obrigatório), enquanto NoClassDefFoundError é um Error unchecked, que não exige tratamento explícito. [Difference between ClassNotFoundException vs NoClassDefFoundError in Java +5](https://javarevisited.blogspot.com/2011/07/classnotfoundexception-vs.html)

Raciocínio da distinção de gravidade: por ser um `Error` (não uma `Exception`), o Java está sinalizando algo mais sério do que um problema de lógica de negócio comum — geralmente indica um problema estrutural do ambiente (JAR faltando no classpath em produção, versão errada de dependência, etc.), não algo que o código deveria simplesmente "capturar e seguir em frente" como faria com uma exceção de validação normal.