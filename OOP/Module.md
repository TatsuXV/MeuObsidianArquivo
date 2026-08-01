#### 1. Teoria

**Module** é a unidade de organização introduzida no **Java 9** através do **JPMS** (Java Platform Module System, também conhecido pelo codinome do projeto que o criou, "Project Jigsaw). Você já viu, no tópico de Packages, a diferença conceitual entre os dois: package organiza **classes**; módulo organiza **packages**, controlando explicitamente o que é exposto para fora e do que ele depende.

**O problema que módulos resolvem:**

Antes do Java 9, existia um problema conhecido como **"JAR Hell"** (ou "classpath hell"): toda a JDK e todas as bibliotecas de terceiros ficavam disponíveis num único **classpath** gigante e "achatado". Isso trazia consequências reais:

1. **Nenhum encapsulamento real entre bibliotecas.** Mesmo uma classe sendo `public`, ela era acessível por **qualquer** código no classpath, mesmo que o autor da biblioteca nunca tivesse pretendido que fosse uma API pública de uso externo — só existia "público pra tudo" ou "privado pra tudo" (dentro da própria classe/package).
2. **Sem verificação de dependências em tempo de inicialização.** Se faltasse uma classe necessária, você só descobria em **runtime**, quando aquele código específico fosse executado (um `NoClassDefFoundError` no meio da aplicação rodando), não ao iniciar a aplicação.
3. **JDK monolítica.** Antes do Java 9, você baixava a JDK inteira, com todos os seus módulos internos, mesmo que sua aplicação usasse uma fração pequena disso — isso pesava em cenários como containers Docker, onde o tamanho da imagem importa.

**Como um módulo é declarado:**

Um módulo é definido por um arquivo especial chamado `module-info.java`, na raiz do código-fonte do módulo:

java

```java
module com.empresa.meuapp {
    requires java.sql;              // este módulo DEPENDE do módulo java.sql
    requires transitive java.logging; // dependência que também é repassada a quem usar este módulo

    exports com.empresa.meuapp.api;  // este package é PÚBLICO pra outros módulos
    // com.empresa.meuapp.internal (não listado em exports) fica OCULTO —
    // mesmo suas classes sendo 'public', módulos externos não conseguem acessá-las

    opens com.empresa.meuapp.model; // permite REFLECTION nesse package
                                      // (necessário para frameworks como Spring/Hibernate,
                                      // que inspecionam classes via reflection em runtime)
}
```

**Diretivas principais do `module-info.java`:**

- `requires nome.do.modulo` — declara uma dependência.
- `requires transitive nome.do.modulo` — declara uma dependência que é automaticamente repassada para quem depender **deste** módulo (evita que todo mundo na cadeia precise redeclarar a mesma dependência).
- `exports nome.do.package` — torna um package visível/acessível para outros módulos.
- `opens nome.do.package` — permite que outros módulos acessem esse package via **reflection**, mesmo sem `exports` explícito (muito relevante, porque frameworks como Spring e Hibernate dependem pesadamente de reflection para funcionar).

**Diferença de `exports` vs. `opens`:**  
`exports` permite acesso normal em tempo de compilação **e** runtime (chamar métodos públicos diretamente no código). `opens` permite **apenas** acesso via reflection em runtime — é mais restrito, e existe especificamente porque frameworks precisam inspecionar/instanciar classes dinamicamente (ex: o Spring precisa conseguir criar uma instância de uma classe anotada com `@Entity` via reflection, mesmo que essa classe não devesse ser "chamada diretamente" por código externo).

**Módulo "unnamed" (não-nomeado):**  
Se seu projeto não tem um `module-info.java`, ele roda no que se chama **classpath tradicional**, dentro de um módulo especial chamado "unnamed module" — é assim que a esmagadora maioria dos projetos Java, incluindo praticamente todo projeto Spring Boot, funciona até hoje. O sistema de módulos é **opcional**: você pode usar toda a linguagem Java sem nunca escrever um `module-info.java`.

**Onde isso aparece no dia a dia de backend Java/Spring — e por que este tópico é diferente dos anteriores:**  
Esta é uma observação importante para gerenciar expectativa: **na prática do mercado júnior brasileiro, você provavelmente não vai escrever `module-info.java` no seu dia a dia.** A esmagadora maioria dos projetos Spring Boot em produção **não** adota JPMS explicitamente — continuam usando o classpath tradicional com Maven/Gradle gerenciando dependências (o que resolve o problema de "gerenciar bibliotecas" de uma forma diferente, sem exigir módulos). O motivo de este tópico existir no roadmap é: (1) entender **por que** ele existe ajuda a entender decisões de design da própria JDK moderna (a própria JDK é internamente modularizada desde o Java 9 — é por isso que `java.base`, `java.sql`, etc. existem como conceito); (2) você pode encontrar `module-info.java` em projetos legados/específicos ou bibliotecas de terceiros mais modernas; (3) é conhecimento que aparece esporadicamente em entrevista técnica conceitual, como "você sabe o que mudou no sistema de módulos a partir do Java 9?".

---

#### 2. Exemplo de código comentado

Estrutura de exemplo com dois módulos:

```
projeto/
├── modulo.core/
│   ├── module-info.java
│   └── com/empresa/core/
│       ├── api/
│       │   └── Calculadora.java
│       └── internal/
│           └── LogicaInterna.java
└── modulo.app/
    ├── module-info.java
    └── com/empresa/app/
        └── Main.java
```

java

```java
// Arquivo: modulo.core/module-info.java
module com.empresa.core {
    exports com.empresa.core.api;
    // com.empresa.core.internal NÃO é exportado — fica invisível
    // para qualquer módulo que dependa de com.empresa.core
}
```

java

```java
// Arquivo: modulo.core/com/empresa/core/api/Calculadora.java
package com.empresa.core.api;

import com.empresa.core.internal.LogicaInterna;

public class Calculadora {
    public int somar(int a, int b) {
        return LogicaInterna.somarInterno(a, b);
    }
}
```

java

```java
// Arquivo: modulo.core/com/empresa/core/internal/LogicaInterna.java
package com.empresa.core.internal;

// Esta classe é 'public', mas como o package 'internal' NÃO está em 'exports'
// no module-info.java, nenhum módulo externo consegue acessá-la — isso é
// encapsulamento real, aplicado a nível de MÓDULO, algo que package-private
// sozinho nunca conseguiria (porque package-private só protege dentro do
// mesmo package, e aqui estamos falando de proteger entre módulos diferentes)
public class LogicaInterna {
    public static int somarInterno(int a, int b) {
        return a + b;
    }
}
```

java

```java
// Arquivo: modulo.app/module-info.java
module com.empresa.app {
    requires com.empresa.core; // declara dependência explícita do módulo core
}
```

java

```java
// Arquivo: modulo.app/com/empresa/app/Main.java
package com.empresa.app;

import com.empresa.core.api.Calculadora; // OK: 'api' foi exportado
// import com.empresa.core.internal.LogicaInterna;
// ERRO DE COMPILAÇÃO se descomentado: "package com.empresa.core.internal
// is not visible" — porque 'internal' nunca foi exportado pelo módulo core,
// mesmo LogicaInterna sendo uma classe 'public'

public class Main {
    public static void main(String[] args) {
        Calculadora calc = new Calculadora();
        System.out.println(calc.somar(2, 3)); // 5
    }
}
```

java

```java
// Exemplo de 'opens' para reflection — cenário típico de um projeto
// que usa alguma biblioteca de injeção de dependência ou ORM
module com.empresa.dados {
    exports com.empresa.dados.api;
    opens com.empresa.dados.entidades; // permite que frameworks como Hibernate
                                         // instanciem/inspecionem essas classes
                                         // via reflection, mesmo sem exports
}
```

---

#### 3. Armadilhas comuns

1. **Achar que módulo é obrigatório em todo projeto Java moderno.** Como mencionado na Teoria, ele é **opcional**. A grande maioria dos projetos Spring Boot roda no "unnamed module" (classpath tradicional) até hoje, e isso não é "código desatualizado" nem prática ruim — é simplesmente a abordagem dominante do mercado, especialmente para aplicações (diferente de bibliotecas).
2. **Confundir `module-info.java` com `package-info.java`.** São arquivos diferentes, com propósitos diferentes: `module-info.java` declara um módulo inteiro (na raiz do código-fonte). `package-info.java` é opcional e serve pra documentar/anotar um package específico (ex: adicionar uma anotação `@NonNullApi` em nível de package) — não tem relação direta com JPMS.
3. **Esquecer `opens` e tomar erro de reflection em runtime com framework.** Se você (raramente, mas pode acontecer) estiver num projeto modularizado e usar uma biblioteca que precisa de reflection (Hibernate, Jackson para serialização JSON, etc.) num package que só tem `exports` mas não `opens`, você toma uma exceção em runtime tipo `InaccessibleObjectException`, porque `exports` sozinho **não** libera acesso via reflection profundo (reflection que acessa campos/construtores privados, por exemplo).
4. **Tentar exportar um package que não existe fisicamente ou está com nome errado.** Assim como packages precisam corresponder à estrutura de diretórios (visto no tópico anterior), `exports com.empresa.algo;` precisa apontar para um package que realmente existe dentro daquele módulo — erro de digitação aqui é um erro de compilação direto no `module-info.java`.

---

#### 4. Exercícios práticos

> **Nota sobre este bloco de exercícios:** diferente dos tópicos anteriores, aqui o objetivo não é "praticar module system no dia a dia" (você provavelmente não vai usar isso rotineiramente, como explicado na Teoria) — é **consolidar o entendimento conceitual** o suficiente pra reconhecer o padrão se aparecer em entrevista ou em projeto de terceiros. Os exercícios pedem para você **descrever/planejar** a estrutura, mais do que necessariamente compilar um projeto multi-módulo completo (que exige configuração de build mais elaborada do que vale a pena neste ponto do seu aprendizado).

**Exercício 1 (Fácil)**  
Escreva, em texto (comentário ou resposta direta, sem precisar compilar), o conteúdo de um `module-info.java` para um módulo chamado `com.empresa.utilidades`, que expõe um package `com.empresa.utilidades.texto` e mantém oculto um package `com.empresa.utilidades.interno`.  
_Critério de pronto:_ a sintaxe do `module-info.java` escrita está correta (`module`, `exports`), e a explicação de por que o segundo package não aparece está correta.

**Exercício 2 (Médio)**  
Monte fisicamente (compilando de verdade, se sua configuração de ambiente permitir — ou, alternativamente, escreva os arquivos completos e explique o que aconteceria ao tentar compilar) o cenário do Exemplo 2 da Teoria (módulo `core` com `api`/`internal`, módulo `app` dependendo dele). Modifique o exemplo para que `Main` tente acessar `LogicaInterna` diretamente, e documente/reproduza o erro exato de compilação que isso gera.  
_Critério de pronto:_ você consegue reproduzir (ou, no mínimo, descrever com precisão, citando a mensagem de erro típica) a falha de acesso a um package não-exportado.

**Exercício 3 (Difícil)**  
Explique, em suas próprias palavras (pode ser um texto corrido, não precisa ser código), por que uma biblioteca como o Hibernate — que precisa criar instâncias de classes de entidade (`@Entity`) e ler/escrever seus campos `private` via reflection — teria problemas se um módulo só usasse `exports` no package dessas entidades, em vez de `opens`. Relacione sua explicação com o conceito de _reflection profundo_ (deep reflection) mencionado na Armadilha 3.  
_Critério de pronto:_ a explicação diferencia corretamente o que `exports` permite (acesso normal a métodos/classes públicas) do que `opens` permite adicionalmente (reflection acessando inclusive membros não-públicos).

**Exercício 4 (Desafio)**  
Pesquise (usando busca na web, já que isso é o tipo de detalhe que vale confirmar em vez de arriscar) e responda: qual é o nome do módulo que contém a classe `java.lang.String`, e o que aconteceria (em termos gerais) se você tentasse, hoje, escrever uma classe sua chamada `java.lang.String` dentro de um projeto modularizado. Relacione sua resposta ao conceito de encapsulamento forte trazido pelo JPMS, comparando com o comportamento do classpath tradicional (unnamed module), onde esse tipo de tentativa tem outro comportamento (também vale confirmar via pesquisa).  
_Critério de pronto:_ a resposta identifica corretamente o módulo (`java.base`), e descreve a diferença de comportamento entre tentar isso num contexto modularizado vs. no classpath tradicional, citando a fonte usada na pesquisa.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
module com.empresa.utilidades {
    exports com.empresa.utilidades.texto;
    // com.empresa.utilidades.interno NÃO aparece em nenhuma linha 'exports',
    // então permanece invisível para qualquer módulo externo que declare
    // 'requires com.empresa.utilidades' — mesmo que suas classes sejam 'public'.
}
```

_Raciocínio:_ a ausência de uma diretiva `exports` para `interno` é, por si só, suficiente para ocultá-lo — não existe uma palavra-chave "esconde este package" explícita, porque o comportamento padrão do JPMS já é "oculto, a menos que explicitamente exportado". Isso é uma decisão de design deliberada: **opt-in** para visibilidade, não opt-out.

**Exercício 2**

java

```java
// modulo.app/com/empresa/app/Main.java (versão modificada)
package com.empresa.app;

import com.empresa.core.internal.LogicaInterna; // tentativa de acesso direto

public class Main {
    public static void main(String[] args) {
        System.out.println(LogicaInterna.somarInterno(2, 3));
    }
}
```

Ao tentar compilar isso (com `module-info.java` de `com.empresa.core` exportando só `com.empresa.core.api`, como no Exemplo da Teoria), o erro típico do compilador (`javac`) é:

```
error: package com.empresa.core.internal is not visible
  (package com.empresa.core.internal is declared in module com.empresa.core, 
   which does not export it)
```

_Raciocínio:_ essa mensagem de erro é bem mais informativa do que o clássico `NoClassDefFoundError` em runtime que aconteceria no mundo pré-módulos — o compilador já pega o problema **em tempo de compilação**, dizendo exatamente qual módulo tem o package e por que ele não está acessível. Isso ilustra concretamente o segundo problema mencionado na Teoria (verificação de dependências antecipada, em vez de só descobrir em runtime).

**Exercício 3**

_Resposta esperada (paráfrase de uma resposta correta):_ `exports` só libera acesso ao que já seria naturalmente acessível pelas regras normais de visibilidade Java — ou seja, classes `public` e seus membros `public`. Mas o Hibernate não acessa entidades apenas chamando métodos públicos: ele precisa, via reflection, **instanciar objetos usando construtores que podem não ser públicos**, e **ler/escrever diretamente em campos privados** (`private Long id;`, por exemplo), contornando os getters/setters normais em certos cenários de otimização interna. Esse tipo de acesso — reflection que alcança membros não-públicos — é chamado de **reflection profundo** (_deep reflection_), e é exatamente o que `opens` permite e `exports` sozinho não permite. Se o módulo das entidades só tivesse `exports` (sem `opens`), o Hibernate conseguiria ver e chamar os métodos públicos da entidade normalmente, mas ao tentar usar reflection para setar um campo `private` diretamente (algo que ORMs fazem com frequência), receberia uma exceção de acesso negado em runtime — o encapsulamento do módulo estaria, nesse caso, ativamente **atrapalhando** o funcionamento do framework, e por isso `opens` existe como uma forma de conceder essa permissão extra especificamente para reflection, sem abrir mão do encapsulamento normal (`exports`) para o resto do mundo.

**Exercício 4**

[Nota: este exercício pede pesquisa. Uma resposta correta, ao pesquisar, confirmaria que:] `java.lang.String` pertence ao módulo `java.base` — o módulo fundamental da própria JDK, implicitamente requerido por todo módulo (você nunca precisa escrever `requires java.base;`, é automático). Se você tentar criar sua própria classe `java.lang.String` **dentro de um módulo nomeado**, o JPMS bloqueia isso explicitamente através de uma regra chamada **encapsulamento forte de packages da plataforma** — o compilador rejeita, porque nenhum módulo além de `java.base` tem permissão de declarar tipos dentro do package `java.lang` (essa restrição existe precisamente para impedir que código de aplicação "sequestre" ou substitua classes centrais da própria linguagem). Já no classpath tradicional (unnamed module, sem `module-info.java`), historicamente essa proteção era mais fraca — era tecnicamente possível, em cenários bem específicos, colocar uma classe própria dentro do package `java.lang` (o que gerava comportamento estranho e imprevisível, e nunca foi uma prática recomendada). Essa diferença de comportamento entre os dois mundos é, na prática, um dos exemplos mais concretos de por que o JPMS foi criado: fechar brechas de encapsulamento que o classpath tradicional nunca conseguiu impedir de verdade. Ao pesquisar, é importante confirmar a terminologia exata e o comportamento atual, já que esse é precisamente o tipo de detalhe técnico específico que a regra de precisão deste tutor pede para verificar em vez de afirmar de memória.