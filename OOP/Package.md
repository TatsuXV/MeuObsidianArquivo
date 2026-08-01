#### 1. Teoria

**Package** é o mecanismo do Java para **organizar e agrupar classes relacionadas**, e também para **controlar visibilidade** entre grupos de classes. É o equivalente conceitual a pastas num sistema de arquivos — e, na prática, packages **são** literalmente refletidos na estrutura de diretórios do projeto: o package `com.empresa.projeto.model` corresponde ao caminho de diretório `com/empresa/projeto/model/`.

**Para que packages servem, concretamente:**

1. **Organização.** Em qualquer projeto backend real, você não tem 10 classes — tem centenas. Sem packages, tudo ficaria numa pilha só, sem estrutura navegável. Convenção comum em projetos Spring: `controller`, `service`, `repository`, `model`/`entity`, `dto`, `exception`, `config`.
2. **Evitar colisão de nomes.** Duas classes podem se chamar `Usuario` sem conflito, desde que estejam em packages diferentes (`com.empresa.auth.Usuario` e `com.empresa.relatorio.Usuario` são tipos completamente distintos para o compilador).
3. **Controle de acesso.** O modificador de acesso **padrão** (quando você não escreve `public`, `private` nem `protected` — chamado de _package-private_) faz com que a classe/método/campo só seja visível **dentro do mesmo package**. Isso é uma ferramenta real de encapsulamento em nível de módulo, não só de classe — algo que você já viu parcialmente quando estudou Access Specifiers, mas que só faz sentido completo agora que você entende o que é um package.

**Convenção de nomenclatura (é convenção, não regra do compilador, mas seguida universalmente na indústria):**

- Tudo em minúsculo.
- Domínio invertido como prefixo: `com.nomeempresa.nomeprojeto...` (ex: `com.google.gson`, `org.springframework`).
- Isso existe historicamente para garantir que packages de empresas diferentes nunca colidam entre si, já que domínios de internet são únicos.

**`import` vs. package — não confunda:**

- `package com.empresa.model;` na primeira linha de um arquivo **declara** a qual package aquela classe pertence.
- `import com.empresa.model.Usuario;` em outro arquivo **traz** uma classe de outro package para poder ser referenciada pelo nome simples (`Usuario`) em vez do nome totalmente qualificado (`com.empresa.model.Usuario`) toda vez que for usada.
- Classes do mesmo package **não precisam de `import`** entre si — só quando você referencia uma classe de um package **diferente**.
- O package `java.lang` (contém `String`, `Object`, `Integer`, etc.) é importado **automaticamente e implicitamente** em todo arquivo Java — é por isso que você nunca escreveu `import java.lang.String;`.

**Diferença de Package vs. Módulo (Module, que está no seu roadmap mais à frente):**  
Package é uma organização **lógica** de classes dentro do código. Módulo (introduzido no Java 9, sistema `module-info.java`) é uma unidade **maior**, que agrupa vários packages e controla explicitamente quais deles são expostos para fora do módulo (`exports`) e quais dependências ele exige (`requires`). Você vai aprofundar isso quando chegar no tópico "Modules" do Bloco 3 — por ora, o que importa é: package organiza classes; módulo organiza packages.

**Onde aparece no dia a dia de backend Java/Spring:**

- Toda estrutura de projeto Maven/Gradle segue `src/main/java/com/empresa/projeto/...` — os packages **são** a arquitetura visível do seu projeto.
- Anotações do Spring como `@ComponentScan` literalmente escaneiam packages procurando classes anotadas (`@Service`, `@Repository`, `@Controller`) para registrar como beans — se sua classe estiver num package fora do escaneado, o Spring simplesmente não a encontra (erro clássico de iniciante em Spring).
- Separar `dto` (o que trafega na API) de `entity`/`model` (o que é persistido no banco) em packages diferentes é prática padrão de mercado, evitando expor a estrutura interna do banco diretamente na API.

---

#### 2. Exemplo de código comentado

Estrutura de diretórios de exemplo:

```
src/main/java/
└── com/
    └── loja/
        └── sistema/
            ├── model/
            │   └── Produto.java
            ├── service/
            │   └── EstoqueService.java
            └── Main.java
```

java

```java
// Arquivo: src/main/java/com/loja/sistema/model/Produto.java

package com.loja.sistema.model; // declara a qual package esta classe pertence

public class Produto {
    private String nome;
    private double preco;

    // Construtor, getters, setters omitidos por brevidade

    public Produto(String nome, double preco) {
        this.nome = nome;
        this.preco = preco;
    }

    public String getNome() {
        return nome;
    }

    public double getPreco() {
        return preco;
    }
}
```

java

```java
// Arquivo: src/main/java/com/loja/sistema/service/EstoqueService.java

package com.loja.sistema.service;

// Produto está em outro package (model), então precisa de import explícito
import com.loja.sistema.model.Produto;

import java.util.ArrayList;
import java.util.List;

public class EstoqueService {
    // 'itens' é 'package-private' aqui de propósito (sem modificador):
    // só código dentro de com.loja.sistema.service pode acessar diretamente,
    // forçando quem estiver fora a usar os métodos públicos abaixo
    List<Produto> itens = new ArrayList<>();

    public void adicionar(Produto produto) {
        itens.add(produto);
    }

    public double valorTotalEmEstoque() {
        double total = 0;
        for (Produto p : itens) {
            total += p.getPreco();
        }
        return total;
    }
}
```

java

```java
// Arquivo: src/main/java/com/loja/sistema/Main.java

package com.loja.sistema;

import com.loja.sistema.model.Produto;
import com.loja.sistema.service.EstoqueService;

public class Main {
    public static void main(String[] args) {
        EstoqueService estoque = new EstoqueService();

        estoque.adicionar(new Produto("Teclado", 250.0));
        estoque.adicionar(new Produto("Mouse", 80.0));

        System.out.println("Total em estoque: " + estoque.valorTotalEmEstoque());

        // Alternativa SEM import: usar o nome totalmente qualificado direto no código
        // (funciona, mas é raro fazer isso — só útil quando há colisão de nomes
        // entre duas classes de mesmo nome vindas de packages diferentes)
        com.loja.sistema.model.Produto outro = new com.loja.sistema.model.Produto("Monitor", 900.0);
        estoque.adicionar(outro);
    }
}
```

---

#### 3. Armadilhas comuns

1. **Estrutura de pastas não corresponder ao package declarado.** Se a classe declara `package com.loja.sistema.model;`, ela **precisa** estar fisicamente em `.../com/loja/sistema/model/Produto.java`. Se a pasta não bater com o package declarado, o projeto não compila (ou, dependendo da IDE/ferramenta de build, gera erros confusos de "classe não encontrada" mesmo o arquivo existindo).
2. **Usar `import pacote.*;` (wildcard) e achar que isso importa "tudo, incluindo subpacotes".** O `*` importa apenas as classes **diretamente** dentro daquele package, não de subpacotes. `import com.loja.sistema.*;` NÃO traz automaticamente as classes de `com.loja.sistema.model`. Além disso, a maioria dos guias de estilo profissionais (e o próprio Google Java Style Guide) recomenda **evitar wildcard import**, preferindo import explícito por classe — deixa claro exatamente quais dependências aquele arquivo tem, e evita colisão silenciosa de nomes.
3. **Esquecer que "package-private" (sem modificador) não é a mesma coisa que `private`.** Iniciante às vezes usa "esqueci de colocar `public`" e `private` como sinônimos de "não visível fora". Mas package-private **é** visível para qualquer classe do mesmo package — só não é visível fora dele. É um nível intermediário, frequentemente esquecido porque é o único modificador de acesso em Java que não tem uma palavra-chave própria (você simplesmente omite o modificador).
4. **Duas classes de mesmo nome, pacotes diferentes, ambas importadas no mesmo arquivo.** Se você tenta `import com.loja.model.Produto;` e `import com.financeiro.model.Produto;` no mesmo arquivo, dá erro de compilação por ambiguidade — o Java não sabe qual `Produto` você quer dizer quando usar o nome simples. Solução: importar só um dos dois e referenciar o outro pelo nome totalmente qualificado onde for necessário.

---

#### 4. Exercícios práticos

**Exercício 1 (Fácil)**  
Crie dois arquivos em packages diferentes: `com.escola.model.Aluno` (com campos `nome` e `nota`) e `com.escola.Main` (classe com `main`). No `Main`, importe e instancie um `Aluno`, imprimindo seus dados. Garanta que a estrutura de pastas do projeto corresponda exatamente aos packages declarados.  
_Critério de pronto:_ o projeto compila e roda sem erro de "package does not match expected directory" (ou equivalente da sua IDE/build tool).

**Exercício 2 (Médio)**  
Crie três classes no mesmo package `com.escola.model`: `Pessoa`, `Aluno` (estende `Pessoa`) e `Professor` (estende `Pessoa`). Adicione um campo `protected String instituicao` em `Pessoa`, sem modificador de acesso explícito num outro campo chamado `codigoInterno` (ou seja, package-private). Crie uma quarta classe, `com.escola.relatorio.GeradorRelatorio` (package **diferente**), que tenta acessar `codigoInterno` de um `Aluno` e demonstra, via comentário, por que isso não compila — mesmo `GeradorRelatorio` estando na mesma "família" de domínio do projeto.  
_Critério de pronto:_ o código de `GeradorRelatorio` tentando acessar `codigoInterno` diretamente está comentado (não compilaria se descomentado), com uma explicação de por que package-private bloqueia esse acesso.

**Exercício 3 (Difícil)**  
Recrie o cenário de "colisão de nomes": crie duas classes chamadas `Relatorio`, uma em `com.escola.academico.Relatorio` (com um método `gerar()` que retorna `"Relatório acadêmico"`) e outra em `com.escola.financeiro.Relatorio` (com um método `gerar()` que retorna `"Relatório financeiro"`). Na classe `Main`, importe apenas uma delas normalmente, e instancie a outra usando o nome totalmente qualificado, sem importá-la. Imprima o resultado de `.gerar()` de ambas.  
_Critério de pronto:_ o código compila e imprime corretamente as duas mensagens diferentes, demonstrando a técnica de nome totalmente qualificado como solução prática pra colisão.

**Exercício 4 (Desafio)**  
Organize um mini-projeto simulando a estrutura real de um backend Spring (sem usar Spring de fato, só a organização de packages): crie os packages `model`, `repository`, `service` e `controller` (todos sob um pacote raiz de sua escolha, ex: `com.loja.api`). Em `model`, uma classe `Produto`. Em `repository`, uma classe `ProdutoRepository` com uma `List<Produto>` estática simulando um "banco" em memória e métodos `salvar(Produto p)` e `listarTodos()`. Em `service`, uma classe `ProdutoService` que **depende de** `ProdutoRepository` (recebido via construtor) e expõe um método `cadastrarProduto(String nome, double preco)`. Em `controller`, uma classe `ProdutoController` que depende de `ProdutoService` (também via construtor) e tem um método `simularRequisicaoPost(String nome, double preco)` que chama o service e imprime uma mensagem de sucesso. No `Main`, monte a cadeia manualmente (`new ProdutoRepository()`, passar pro `new ProdutoService(repo)`, passar pro `new ProdutoController(service)`) e chame `simularRequisicaoPost`.  
_Critério de pronto:_ a estrutura de packages reflete a separação de camadas (`controller` não conhece `repository` diretamente, só fala com `service`); rodando o `Main`, o produto é "cadastrado" e uma mensagem de sucesso aparece.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
// Arquivo: com/escola/model/Aluno.java
package com.escola.model;

public class Aluno {
    private String nome;
    private double nota;

    public Aluno(String nome, double nota) {
        this.nome = nome;
        this.nota = nota;
    }

    public String getNome() { return nome; }
    public double getNota() { return nota; }
}
```

java

```java
// Arquivo: com/escola/Main.java
package com.escola;

import com.escola.model.Aluno;

public class Main {
    public static void main(String[] args) {
        Aluno aluno = new Aluno("Carlos", 8.5);
        System.out.println(aluno.getNome() + ": " + aluno.getNota());
    }
}
```

_Raciocínio:_ o ponto central deste exercício não é o código em si (trivial), mas confirmar na prática que a pasta `com/escola/model/` precisa existir fisicamente com `Aluno.java` dentro, e `com/escola/` com `Main.java` — o compilador Java usa essa correspondência para resolver os `import`s.

**Exercício 2**

java

```java
// com/escola/model/Pessoa.java
package com.escola.model;

public class Pessoa {
    protected String instituicao;
    String codigoInterno; // sem modificador = package-private

    public Pessoa(String instituicao, String codigoInterno) {
        this.instituicao = instituicao;
        this.codigoInterno = codigoInterno;
    }
}
```

java

```java
// com/escola/model/Aluno.java
package com.escola.model;

public class Aluno extends Pessoa {
    public Aluno(String instituicao, String codigoInterno) {
        super(instituicao, codigoInterno);
    }
}
```

java

```java
// com/escola/relatorio/GeradorRelatorio.java
package com.escola.relatorio; // PACKAGE DIFERENTE de com.escola.model

import com.escola.model.Aluno;

public class GeradorRelatorio {
    public void gerar(Aluno aluno) {
        // System.out.println(aluno.codigoInterno);
        // ERRO DE COMPILAÇÃO se descomentado:
        // "codigoInterno has package-private access in Pessoa"
        //
        // Porque codigoInterno não tem NENHUM modificador (package-private),
        // e GeradorRelatorio está em com.escola.relatorio, um package DIFERENTE
        // de com.escola.model, onde Pessoa/codigoInterno foram declarados.
        // Mesmo Aluno herdando de Pessoa, a visibilidade package-private
        // é avaliada pelo package de quem TENTA ACESSAR, não pela herança.

        System.out.println("Instituição: " + aluno.instituicao);
        // Isso funciona, pois 'instituicao' é 'protected', e protected
        // permite acesso também por SUBCLASSES em outros packages —
        // mas cuidado: aqui funciona porque estamos acessando através
        // de um objeto do tipo Aluno dentro do próprio contexto de herança
        // sendo relevante; em outros contextos, 'protected' tem regras
        // mais específicas que valem a pena revisar no tópico de Access Specifiers.
    }
}
```

_Raciocínio:_ esse exercício evidencia a diferença prática entre os quatro níveis de acesso (`private`, package-private, `protected`, `public`) quando **múltiplos packages** entram em jogo — algo que não dá pra perceber plenamente até existir mais de um package no projeto.

**Exercício 3**

java

```java
// com/escola/academico/Relatorio.java
package com.escola.academico;

public class Relatorio {
    public String gerar() {
        return "Relatório acadêmico";
    }
}
```

java

```java
// com/escola/financeiro/Relatorio.java
package com.escola.financeiro;

public class Relatorio {
    public String gerar() {
        return "Relatório financeiro";
    }
}
```

java

```java
// com/escola/Main.java
package com.escola;

import com.escola.academico.Relatorio; // só este é importado

public class Main {
    public static void main(String[] args) {
        Relatorio academico = new Relatorio(); // usa o nome simples, resolvido pelo import

        // O financeiro é referenciado pelo nome TOTALMENTE QUALIFICADO,
        // já que "Relatorio" sozinho já está "ocupado" pelo import acima
        com.escola.financeiro.Relatorio financeiro = new com.escola.financeiro.Relatorio();

        System.out.println(academico.gerar());   // Relatório acadêmico
        System.out.println(financeiro.gerar());  // Relatório financeiro
    }
}
```

_Raciocínio:_ isso mostra que nome totalmente qualificado não é só curiosidade acadêmica — é a solução real e única (fora de renomear uma das classes) para o cenário de colisão. Uma alternativa comum na indústria, aliás, é justamente **evitar nomes genéricos demais** (`Relatorio`) e preferir nomes que já embutem o contexto (`RelatorioAcademico`, `RelatorioFinanceiro`), evitando a colisão na origem — mas é importante saber resolver o problema quando ele já existe (ex: ao integrar duas bibliotecas de terceiros que por acaso têm classes de mesmo nome).

**Exercício 4**

java

```java
// com/loja/api/model/Produto.java
package com.loja.api.model;

public class Produto {
    private final String nome;
    private final double preco;

    public Produto(String nome, double preco) {
        this.nome = nome;
        this.preco = preco;
    }

    public String getNome() { return nome; }
    public double getPreco() { return preco; }
}
```

java

```java
// com/loja/api/repository/ProdutoRepository.java
package com.loja.api.repository;

import com.loja.api.model.Produto;

import java.util.ArrayList;
import java.util.List;

public class ProdutoRepository {
    private final List<Produto> produtos = new ArrayList<>();

    public void salvar(Produto produto) {
        produtos.add(produto);
    }

    public List<Produto> listarTodos() {
        return produtos;
    }
}
```

java

```java
// com/loja/api/service/ProdutoService.java
package com.loja.api.service;

import com.loja.api.model.Produto;
import com.loja.api.repository.ProdutoRepository;

public class ProdutoService {
    private final ProdutoRepository repository;

    // Dependência recebida via construtor — isso é literalmente Dependency
    // Injection "manual", o mesmo princípio que o Spring automatiza depois
    public ProdutoService(ProdutoRepository repository) {
        this.repository = repository;
    }

    public void cadastrarProduto(String nome, double preco) {
        Produto produto = new Produto(nome, preco);
        repository.salvar(produto);
    }
}
```

java

```java
// com/loja/api/controller/ProdutoController.java
package com.loja.api.controller;

import com.loja.api.service.ProdutoService;

public class ProdutoController {
    private final ProdutoService service;

    public ProdutoController(ProdutoService service) {
        this.service = service;
    }

    public void simularRequisicaoPost(String nome, double preco) {
        service.cadastrarProduto(nome, preco);
        System.out.println("Produto '" + nome + "' cadastrado com sucesso!");
    }
}
```

java

```java
// com/loja/api/Main.java
package com.loja.api;

import com.loja.api.controller.ProdutoController;
import com.loja.api.repository.ProdutoRepository;
import com.loja.api.service.ProdutoService;

public class Main {
    public static void main(String[] args) {
        ProdutoRepository repository = new ProdutoRepository();
        ProdutoService service = new ProdutoService(repository);
        ProdutoController controller = new ProdutoController(service);

        controller.simularRequisicaoPost("Cadeira Gamer", 1200.0);
        // Saída: Produto 'Cadeira Gamer' cadastrado com sucesso!
    }
}
```

_Raciocínio:_ o detalhe mais importante aqui não é o código rodar — é a **direção das dependências**: `controller` importa `service`, `service` importa `repository`, mas `repository` não sabe que `service` existe, e `service` não sabe que `controller` existe. Essa é exatamente a arquitetura em camadas que você vai ver formalizada quando chegar em Spring: cada camada só conhece a camada imediatamente abaixo dela, nunca a de cima, e essa separação por **packages** é o que torna essa regra visível e organizável no projeto — não é acaso que todo tutorial de Spring Boot usa essa mesma estrutura de pastas.