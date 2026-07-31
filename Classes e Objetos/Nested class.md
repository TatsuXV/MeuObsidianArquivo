#### 1. Teoria

_Nested class_ é uma classe declarada **dentro** de outra classe. Existem quatro tipos, e a diferença entre eles é justamente o que costuma confundir:

1. **Static Nested Class** — declarada com `static` dentro da externa. Não tem acesso aos atributos de instância da classe externa (porque, sendo `static`, ela não está "amarrada" a um objeto específico da externa). Funciona basicamente como uma classe independente, só organizada dentro de outra por conveniência lógica.
2. **Inner Class (não-static)** — declarada sem `static`. Cada instância dela está **amarrada a uma instância específica** da classe externa, e por isso **tem acesso direto** aos atributos/métodos de instância da externa (mesmo os `private`).
3. **Local Class** — declarada **dentro de um método**, só existe/é visível dentro daquele método.
4. **Anonymous Class** — uma classe sem nome, declarada e instanciada no mesmo lugar, geralmente para implementar uma interface ou estender uma classe rapidamente, sem criar um arquivo separado.

Não confunda com:

- **Inheritance (herança)**: nested class não é "filha" da classe externa — não há relação de `extends`/`is-a` automática. É só uma questão de organização/escopo (a classe está definida dentro da outra), não de hierarquia.
- **Package**: nested class não é a mesma coisa que "outra classe no mesmo arquivo/pacote". Ela literalmente vive dentro do corpo da classe externa, com acesso especial aos membros dela (no caso da inner class não-static).
- **Lambda Expressions**: lambdas (que veremos mais à frente, Bloco 3/12) resolvem parte do mesmo problema que anonymous class resolvia antes (implementar uma interface funcional rapidamente), com sintaxe mais enxuta — mas lambda só funciona quando a interface tem exatamente um método abstrato. Isso será aprofundado quando chegarmos lá.

**Onde isso aparece no dia a dia de backend**: static nested class aparece bastante em **classes de configuração/DTO auxiliares** que só fazem sentido dentro do contexto de outra classe (ex: um `Builder` estático dentro da própria classe que ele constrói — padrão Builder). Inner class não-static é mais rara em código de backend do dia a dia; anonymous class ainda aparece em callbacks mais antigos, embora lambda tenha tomado boa parte desse espaço em código moderno.

#### 2. Exemplo de código comentado

java

```java
public class Pedido {

    private String numero;
    private double valorTotal;

    public Pedido(String numero, double valorTotal) {
        this.numero = numero;
        this.valorTotal = valorTotal;
    }

    // 1. STATIC NESTED CLASS: não acessa "numero"/"valorTotal" da instância externa,
    // não precisa de um objeto Pedido para existir.
    public static class Builder {
        private String numero;
        private double valorTotal;

        public Builder numero(String numero) {
            this.numero = numero;
            return this; // method chaining (Bloco 3, item futuro)
        }

        public Builder valorTotal(double valorTotal) {
            this.valorTotal = valorTotal;
            return this;
        }

        public Pedido build() {
            return new Pedido(this.numero, this.valorTotal);
        }
    }

    // 2. INNER CLASS (não-static): acessa "numero" e "valorTotal" da instância
    // externa DIRETAMENTE, porque está amarrada a um objeto Pedido específico.
    public class Resumo {
        public String gerarTexto() {
            // acessa os atributos do Pedido "dono" desta instância de Resumo
            return "Pedido " + numero + " - Total: R$ " + valorTotal;
        }
    }
}
```

java

```java
// Usando a static nested class (Builder) — NÃO precisa de um Pedido existente antes:
Pedido pedido = new Pedido.Builder()
    .numero("PED-001")
    .valorTotal(250.0)
    .build();

// Usando a inner class (Resumo) — PRECISA de um Pedido existente antes:
Pedido.Resumo resumo = pedido.new Resumo(); // sintaxe: instância.new InnerClass()
System.out.println(resumo.gerarTexto()); // Pedido PED-001 - Total: R$ 250.0
```

#### 3. Armadilhas comuns

1. **Tentar instanciar uma inner class não-static sem uma instância da externa** — `new Pedido.Resumo()` direto dá erro de compilação; a sintaxe correta é `pedido.new Resumo()`, porque a inner class precisa "saber" a qual objeto externo ela pertence.
2. **Achar que static nested class tem acesso automático aos atributos de instância da externa** — não tem; se precisar desses dados, tem que receber via construtor/parâmetro, igual qualquer classe independente receberia.
3. **Abusar de nested class quando uma classe separada (arquivo próprio) seria mais clara** — nested class faz sentido quando a classe interna só existe _em função_ da externa (ex: um `Builder` de uma classe específica); se a classe faz sentido sozinha e é reutilizada em vários lugares, ela deveria ser um arquivo próprio.
4. **Confundir local class com anonymous class** — local class tem nome e é declarada dentro de um método; anonymous class não tem nome e é declarada e instanciada no mesmo ponto. São ferramentas parecidas, mas não são sinônimos.

#### 4. Exercícios práticos

**1. Fácil** — Crie uma classe externa `Livro` com atributos `titulo` e `autor`. Dentro dela, crie uma **static nested class** chamada `Genero` com atributos `nome` e `classificacaoEtaria`. Critério de pronto: instanciar um `Livro.Genero` **sem** precisar criar um objeto `Livro` antes, provando que a nested class é independente da externa.

**2. Médio** — Crie uma classe externa `Turma` com atributos `nomeTurma` e um atributo `int totalAlunos`. Dentro dela, crie uma **inner class (não-static)** chamada `Boletim` com um método `gerarResumo()` que acessa diretamente `nomeTurma` e `totalAlunos` da instância externa (sem receber esses valores por parâmetro — o objetivo é usar o acesso automático da inner class). Critério de pronto: criar um objeto `Turma`, depois criar um `Boletim` a partir dele usando a sintaxe `turma.new Boletim()`, e chamar `gerarResumo()`.

**3. Difícil** — Implemente o padrão **Builder** completo (igual ao exemplo de `Pedido` acima, mas para uma classe nova) para uma classe `Endereco` com pelo menos 4 atributos (`rua`, `numero`, `cidade`, `cep`). O `Builder` deve ser uma static nested class com method chaining (cada método `set` retorna `this`) e um método `build()` que retorna o `Endereco` finalizado. Critério de pronto: construir um `Endereco` usando o builder encadeado (`new Endereco.Builder().rua(...).numero(...)...build()`) e imprimir os valores pra provar que foram todos setados corretamente.

**4. Desafio** — Crie uma classe `Calculadora` com um método `criarValidador()` que retorna uma **anonymous class** implementando uma interface funcional simples que você mesmo vai declarar antes, chamada `Validador`, com um único método `boolean validar(double valor)`. A anonymous class retornada deve validar se o valor é positivo. Critério de pronto: chamar `criarValidador().validar(...)` com um valor positivo e um negativo, mostrando que retorna `true`/`false` corretamente — sem criar um arquivo `.java` separado para a implementação de `Validador`.

#### 5. Gabarito comentado

**1. Livro/Genero**

java

```java
public class Livro {
    private String titulo;
    private String autor;

    public Livro(String titulo, String autor) {
        this.titulo = titulo;
        this.autor = autor;
    }

    public static class Genero {
        private String nome;
        private String classificacaoEtaria;

        public Genero(String nome, String classificacaoEtaria) {
            this.nome = nome;
            this.classificacaoEtaria = classificacaoEtaria;
        }
    }
}

// Uso: sem NENHUM objeto Livro criado antes
Livro.Genero genero = new Livro.Genero("Ficção Científica", "12 anos");
```

Raciocínio: como `Genero` é `static`, ela é essencialmente independente — só está "guardada dentro" de `Livro` por organização (faz sentido que gênero literário seja um conceito relacionado a livro), mas não precisa de nenhum `Livro` existente para funcionar.

**2. Turma/Boletim**

java

```java
public class Turma {
    private String nomeTurma;
    private int totalAlunos;

    public Turma(String nomeTurma, int totalAlunos) {
        this.nomeTurma = nomeTurma;
        this.totalAlunos = totalAlunos;
    }

    public class Boletim {
        public String gerarResumo() {
            // acesso direto aos atributos da Turma "dona" desta instância
            return "Turma: " + nomeTurma + " | Total de alunos: " + totalAlunos;
        }
    }
}

// Uso:
Turma turma = new Turma("3A", 28);
Turma.Boletim boletim = turma.new Boletim();
System.out.println(boletim.gerarResumo()); // Turma: 3A | Total de alunos: 28
```

Raciocínio: o ponto do exercício é sentir na prática que `Boletim` não precisou receber `nomeTurma`/`totalAlunos` como parâmetro — ela "enxerga" esses dados porque está amarrada à instância `turma` específica que a criou. Se você criasse outra `Turma` e outro `Boletim` a partir dela, o resumo seria diferente automaticamente.

**3. Endereco com Builder**

java

```java
public class Endereco {
    private String rua;
    private String numero;
    private String cidade;
    private String cep;

    private Endereco(Builder builder) {
        this.rua = builder.rua;
        this.numero = builder.numero;
        this.cidade = builder.cidade;
        this.cep = builder.cep;
    }

    public static class Builder {
        private String rua;
        private String numero;
        private String cidade;
        private String cep;

        public Builder rua(String rua) {
            this.rua = rua;
            return this;
        }

        public Builder numero(String numero) {
            this.numero = numero;
            return this;
        }

        public Builder cidade(String cidade) {
            this.cidade = cidade;
            return this;
        }

        public Builder cep(String cep) {
            this.cep = cep;
            return this;
        }

        public Endereco build() {
            return new Endereco(this);
        }
    }

    @Override
    public String toString() {
        return rua + ", " + numero + " - " + cidade + " - CEP: " + cep;
    }
}

// Uso:
Endereco endereco = new Endereco.Builder()
    .rua("Av. Paulista")
    .numero("1000")
    .cidade("São Paulo")
    .cep("01310-100")
    .build();

System.out.println(endereco);
// Av. Paulista, 1000 - São Paulo - CEP: 01310-100
```

Raciocínio: repare que o construtor de `Endereco` é `private` e recebe o próprio `Builder` como parâmetro — assim, a única forma de construir um `Endereco` "de fora" é passando pelo builder encadeado, o que fica bem legível quando a classe tem muitos atributos (evita um construtor com 4+ parâmetros posicionais, onde é fácil trocar a ordem por engano). Esse é o padrão Builder de verdade, muito comum em bibliotecas Java reais.

**4. Calculadora com anonymous class**

java

```java
public class Calculadora {

    interface Validador {
        boolean validar(double valor);
    }

    public Validador criarValidador() {
        return new Validador() { // anonymous class implementando Validador
            @Override
            public boolean validar(double valor) {
                return valor > 0;
            }
        };
    }
}

// Uso:
Calculadora calc = new Calculadora();
Validador validadorPositivo = calc.criarValidador();
System.out.println(validadorPositivo.validar(10.0));  // true
System.out.println(validadorPositivo.validar(-5.0));  // false
```

Raciocínio: a anonymous class permite criar uma implementação de `Validador` "descartável", só pra esse uso específico, sem precisar criar um arquivo `ValidadorPositivo.java` separado. Vale adiantar uma linha: hoje em dia, pra interfaces com um único método abstrato como essa, a maioria dos devs usaria uma **lambda expression** (`valor -> valor > 0`) em vez de anonymous class, porque é bem mais enxuto — mas isso será aprofundado quando chegarmos no tópico de Lambda Expressions (Bloco 3), e é importante você entender a anonymous class primeiro, porque a lambda é basicamente um "açúcar sintático" em cima desse mesmo mecanismo.