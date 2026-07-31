#### 1. Teoria

_Access specifiers_ (ou modificadores de acesso) controlam **quem pode enxergar e usar** um atributo, método, construtor ou classe. Em Java, existem quatro níveis:

|Modificador|Mesma classe|Mesmo pacote|Subclasse (outro pacote)|Qualquer lugar|
|---|---|---|---|---|
|`private`|✅|❌|❌|❌|
|_(default/package-private)_ — sem modificador|✅|✅|❌|❌|
|`protected`|✅|✅|✅|❌|
|`public`|✅|✅|✅|✅|

Não confunda isso com:

- **Static keyword**: `static` define se o membro pertence à classe ou à instância — é um eixo completamente diferente e ortogonal ao _access specifier_. Você pode ter um atributo `private static`, `public static`, etc. (Static é o próximo item do checklist, vamos aprofundar lá.)
- **Final**: `final` controla se o valor pode ser reatribuído — também é um eixo diferente. Nada a ver com quem pode _ver_ o membro, e sim com se ele pode _mudar_.
- **Encapsulamento**: _access specifiers_ são a **ferramenta** usada para implementar encapsulamento, mas não são a mesma coisa. Encapsulamento é o princípio de OOP (Bloco 3) de esconder o estado interno e expor só o necessário; `private`/`public` é o mecanismo da linguagem que viabiliza isso.

**Onde isso aparece no dia a dia de backend**: em qualquer entidade `@Entity` do Spring Data JPA, o padrão é atributos `private` com getters/setters `public` — isso protege o estado do objeto de ser alterado sem controle a partir de outra classe, mesmo dentro do mesmo projeto.

#### 2. Exemplo de código comentado

java

```java
package com.exemplo.banco;

public class ContaBancaria {

    private double saldo;              // só a própria classe acessa direto
    protected String numeroConta;      // subclasses (mesmo fora do pacote) podem acessar
    String agencia;                    // default: só quem está no mesmo pacote acessa
    public String titular;             // qualquer lugar pode acessar

    public double getSaldo() {         // getter público: forma controlada de expor o saldo
        return saldo;
    }

    private void logOperacao(String msg) {
        // método auxiliar interno — não faz sentido nenhuma outra classe chamar isso
        System.out.println("[LOG] " + msg);
    }

    public void depositar(double valor) {
        this.saldo += valor;
        logOperacao("Depósito de " + valor); // chamada permitida: mesma classe
    }
}
```

java

```java
package com.exemplo.banco.subclasse; // pacote DIFERENTE

import com.exemplo.banco.ContaBancaria;

public class ContaPoupanca extends ContaBancaria {
    public void exemplo() {
        // this.numeroConta acessível (protected + é subclasse)
        // this.agencia NÃO acessível aqui (default, pacote diferente)
        // this.titular acessível (public)
        // this.saldo NÃO acessível aqui (private)
    }
}
```

#### 3. Armadilhas comuns

1. **Deixar atributos `public` "porque é mais rápido de escrever"** — funciona no curto prazo, mas qualquer classe pode alterar o estado sem validação nenhuma; em projeto real isso vira bug difícil de rastrear.
2. **Achar que `protected` é só "um pouco mais aberto que `private`"** — na prática, `protected` libera acesso pra qualquer subclasse, mesmo em pacotes completamente diferentes, o que é bem mais permissivo do que parece à primeira vista.
3. **Esquecer que não colocar modificador nenhum NÃO é a mesma coisa que `public`** — ausência de modificador é o nível _default_ (restrito ao pacote), um erro comum de quem vem de outras linguagens onde "sem modificador" costuma significar "público".
4. **Marcar métodos auxiliares internos como `public` por padrão** — se um método só existe para ser usado por outro método da mesma classe, ele deveria ser `private`; deixar tudo `public` "pra não ter que pensar" aumenta a superfície de uso indevido da classe.

#### 4. Exercícios práticos

**1. Fácil** — Crie uma classe `Funcionario` com um atributo `salario` (double) marcado como `private`, e um atributo `nome` (String) marcado como `public`. Adicione um método `getSalario()` público que retorna o salário. Critério de pronto: em uma classe separada (pode ser a `main`), mostrar que `nome` pode ser acessado direto (`funcionario.nome`), mas `salario` só através de `getSalario()` — tentar acessar `funcionario.salario` direto deve dar erro de compilação (comente essa linha no código pra provar que você testou).

**2. Médio** — Crie duas classes no **mesmo pacote**: `Veiculo` (com um atributo `int velocidadeAtual` sem modificador — default) e `Garagem` (que tenta ler `veiculo.velocidadeAtual` diretamente). Depois, mova a classe `Garagem` para um pacote diferente e observe o que acontece ao tentar compilar. Critério de pronto: documentar em um comentário o erro exato que aparece quando `Garagem` está em outro pacote.

**3. Difícil** — Crie uma classe base `Animal` com um método `protected void emitirSom()` (implementação vazia ou com um `System.out.println` genérico). Crie uma subclasse `Cachorro` (em outro pacote) que sobrescreve `emitirSom()` chamando `super.emitirSom()` antes de imprimir "Au au!". Critério de pronto: instanciar `Cachorro` e chamar `emitirSom()`, mostrando que a subclasse consegue acessar o método `protected` da superclasse mesmo estando em pacote diferente.

**4. Desafio** — Modele uma classe `ContaBancaria` "à prova de bala": todos os atributos `private`, um construtor público que valida os dados de entrada (ex: saldo inicial não pode ser negativo, lançando exceção se for), métodos públicos `depositar`/`sacar` com validação, e pelo menos um método `private` auxiliar (ex: `validarValor(double valor)`) reutilizado internamente por `depositar` e `sacar` para evitar duplicação de código de validação. Critério de pronto: nenhum atributo pode ser alterado de fora da classe sem passar por um método que valida a operação.

#### 5. Gabarito comentado

**1. Funcionario**

java

```java
public class Funcionario {
    private double salario;
    public String nome;

    public Funcionario(String nome, double salario) {
        this.nome = nome;
        this.salario = salario;
    }

    public double getSalario() {
        return salario;
    }
}

// Em outra classe:
Funcionario f = new Funcionario("Carla", 5000.0);
System.out.println(f.nome);          // funciona: nome é public
// System.out.println(f.salario);    // ERRO DE COMPILAÇÃO: salario é private
System.out.println(f.getSalario());  // funciona: acesso controlado via getter
```

Raciocínio: o exercício existe pra você sentir na prática a diferença entre "dado exposto sem controle" (`nome`) e "dado protegido, exposto de forma controlada" (`salario` via `getSalario()`). Em projeto real, o ideal seria os dois serem `private` com getters — deixamos `nome` público aqui só para efeito didático de comparação.

**2. Veiculo/Garagem**

java

```java
// pacote com.exemplo.transporte
class Veiculo {
    int velocidadeAtual = 80; // default: visível só no mesmo pacote
}

class Garagem {
    void checar() {
        Veiculo v = new Veiculo();
        System.out.println(v.velocidadeAtual); // OK: mesmo pacote
    }
}
```

Se `Garagem` for movida para `com.exemplo.outraarea`:

java

```java
// v.velocidadeAtual não pode ser acessado
// erro típico: "velocidadeAtual has default access in Veiculo; cannot be accessed from outside package"
```

Raciocínio: esse exercício mostra visualmente o que a tabela de teoria descreve — o nível _default_ depende inteiramente de pacote, não de herança nem de nada mais.

**3. Animal/Cachorro**

java

```java
// pacote com.exemplo.animais
public class Animal {
    protected void emitirSom() {
        System.out.println("Som genérico de animal");
    }
}
```

java

```java
// pacote com.exemplo.animais.subclasses
package com.exemplo.animais.subclasses;
import com.exemplo.animais.Animal;

public class Cachorro extends Animal {
    @Override
    protected void emitirSom() {
        super.emitirSom();      // chama a implementação da superclasse primeiro
        System.out.println("Au au!");
    }
}

// Uso:
Cachorro c = new Cachorro();
c.emitirSom();
// Saída:
// Som genérico de animal
// Au au!
```

Raciocínio: `super.emitirSom()` é o que permite "estender" o comportamento da superclasse em vez de simplesmente substituí-lo — o cachorro ainda emite o som genérico, e adiciona o próprio em cima. Isso só é possível por causa do `protected`: se fosse `private` na superclasse, a subclasse nem enxergaria o método pra poder chamar via `super`.

**4. ContaBancaria à prova de bala**

java

```java
public class ContaBancaria {
    private double saldo;

    public ContaBancaria(double saldoInicial) {
        validarValor(saldoInicial); // reaproveitado aqui também
        this.saldo = saldoInicial;
    }

    public void depositar(double valor) {
        validarValor(valor);
        this.saldo += valor;
    }

    public void sacar(double valor) {
        validarValor(valor);
        if (valor > this.saldo) {
            throw new IllegalStateException("Saldo insuficiente");
        }
        this.saldo -= valor;
    }

    private void validarValor(double valor) {
        if (valor < 0) {
            throw new IllegalArgumentException("Valor não pode ser negativo");
        }
    }

    public double getSaldo() {
        return saldo;
    }
}
```

Raciocínio: `validarValor` é `private` de propósito — ela não faz sentido nenhum sendo chamada de fora da classe, é um detalhe de implementação interno. Ao extrair essa validação para um método único, evitamos duplicar a mesma checagem em `depositar`, `sacar` e no construtor — se a regra de validação mudar amanhã (ex: adicionar um teto máximo), você só altera em um lugar. Isso é um pequeno prenúncio de um princípio maior de design (DRY — Don't Repeat Yourself) que você vai ver formalizado mais pra frente.