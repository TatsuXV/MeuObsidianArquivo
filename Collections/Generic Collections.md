### 1. Teoria

**Generic Collections** não é uma estrutura de dados nova — é o mecanismo de **Generics** (tipos parametrizados) aplicado ao Collections Framework, que você já vem usando desde o início (todo `List<String>`, `Map<String, Integer>` que escrevemos até aqui é uma coleção genérica). Agora vamos entender o **porquê** e o **como** por trás disso, o suficiente pra você escrever suas próprias classes/métodos genéricos, não só consumir os prontos.

**O problema que Generics resolve:** antes do Java 5 (quando Generics foi introduzido), coleções guardavam `Object`. Isso significava que `ArrayList` podia guardar qualquer coisa misturada, e você precisava fazer **cast manual** toda vez que tirava um elemento de volta — e esse cast só falhava em **tempo de execução**, não de compilação, se o tipo estivesse errado.

java

```java
// Como era antes de Generics (Java pré-5, código legado que você pode encontrar)
List listaAntiga = new ArrayList(); // sem tipo parametrizado
listaAntiga.add("texto");
listaAntiga.add(42); // nada impede misturar tipos incompatíveis

String valor = (String) listaAntiga.get(1); // ClassCastException SÓ EM RUNTIME - compila normalmente!
```

Com Generics, `List<String>` significa "essa lista só aceita `String`, e o compilador garante isso" — o erro de tipo incompatível vira erro de **compilação**, muito mais barato de corrigir do que um `ClassCastException` estourando em produção.

**Sintaxe de classes/métodos genéricos:** você pode criar suas próprias classes e métodos parametrizados, não só usar os do Java. A convenção de nomenclatura de letra única é: `T` (Type), `E` (Element, comum em coleções), `K`/`V` (Key/Value, comum em Map), `R` (Return).

java

```java
// Classe genérica própria
public class Caixa<T> {
    private T conteudo;
    public void guardar(T item) { this.conteudo = item; }
    public T pegar() { return conteudo; }
}

// Uso: o compilador sabe que Caixa<String> só aceita/retorna String
Caixa<String> caixaDeTexto = new Caixa<>();
caixaDeTexto.guardar("olá");
```

**Wildcards (`?`)** aparecem quando você quer aceitar coleções de tipos relacionados, mas não sabe (ou não quer travar) o tipo exato:

|Wildcard|Significado|Uso típico|
|---|---|---|
|`List<?>`|Lista de algum tipo desconhecido|Quando você só vai **ler** de forma genérica (ex: imprimir), sem se importar com o tipo|
|`List<? extends Number>`|Lista de `Number` ou qualquer subtipo (`Integer`, `Double`...)|**Produtor** — você vai _ler_ elementos como `Number`|
|`List<? super Integer>`|Lista de `Integer` ou qualquer supertipo (`Number`, `Object`...)|**Consumidor** — você vai _inserir_ elementos `Integer`|

Essa regra tem um nome mnemônico famoso: **PECS** (_Producer Extends, Consumer Super_) — se a coleção só vai **produzir** dados pra você (você lê dela), use `extends`; se ela só vai **consumir** dados que você insere, use `super`.

**Erasure (apagamento de tipo):** informação valiosa pra não se confundir depois — em tempo de execução, o Java **apaga** a informação genérica (por compatibilidade retroativa com código pré-Java 5). Isso significa que `List<String>` e `List<Integer>` são, em runtime, literalmente o mesmo `List` por baixo — é por isso que você não consegue fazer `if (lista instanceof List<String>)` (não compila) nem criar um array genérico diretamente (`new T[10]` não é permitido). Isso é um detalhe avançado — mencionado aqui porque explica comportamentos estranhos que você pode encontrar, mas será aprofundado se/quando for relevante num tópico futuro.

**Onde aparece no dia a dia de backend:** você escreve métodos e classes genéricas o tempo todo sem perceber que é isso — repositórios genéricos do Spring Data JPA (`JpaRepository<T, ID>`), classes de resposta padronizada de API (`ApiResponse<T>` envolvendo qualquer tipo de dado retornado), utilitários que operam sobre qualquer tipo de lista. Entender Generics de verdade é o que separa "copiar `<T>` sem saber por quê" de escrever suas próprias abstrações reutilizáveis.

---

### 2. Exemplo de código comentado

java

```java
import java.util.*;

public class GenericCollectionsExample {

    // Classe genérica própria: uma "caixa" que guarda um item de qualquer tipo T
    static class Caixa<T> {
        private T conteudo;

        void guardar(T item) {
            this.conteudo = item;
        }

        T pegar() {
            return conteudo;
        }
    }

    // Método genérico: <T> antes do tipo de retorno declara o parâmetro de tipo do método
    static <T> T primeiroElemento(List<T> lista) {
        if (lista.isEmpty()) {
            throw new NoSuchElementException("Lista vazia");
        }
        return lista.get(0);
    }

    // Wildcard "extends": método que só LÊ números de qualquer subtipo de Number
    static double somarTodos(List<? extends Number> numeros) {
        double soma = 0;
        for (Number n : numeros) {
            soma += n.doubleValue(); // podemos LER como Number com segurança
        }
        return soma;
        // numeros.add(10) NÃO compilaria aqui - o compilador não sabe se é List<Integer>,
        // List<Double> etc., então não permite inserir nada, só ler
    }

    // Wildcard "super": método que só ESCREVE Integer numa lista de Integer ou qualquer supertipo
    static void preencherComInteiros(List<? super Integer> lista, int quantidade) {
        for (int i = 1; i <= quantidade; i++) {
            lista.add(i); // podemos INSERIR Integer com segurança
        }
        // Number valor = lista.get(0) NÃO compilaria com segurança de tipo aqui -
        // o compilador só garante que dá pra ler como Object
    }

    public static void main(String[] args) {
        // Caixa<T> genérica em uso
        Caixa<String> caixaTexto = new Caixa<>();
        caixaTexto.guardar("Olá, Generics!");
        System.out.println(caixaTexto.pegar());

        Caixa<Integer> caixaNumero = new Caixa<>();
        caixaNumero.guardar(42);
        System.out.println(caixaNumero.pegar());

        // Método genérico funciona com qualquer List<T>
        System.out.println(primeiroElemento(List.of("a", "b", "c"))); // a
        System.out.println(primeiroElemento(List.of(10, 20, 30)));    // 10

        // Wildcard extends: aceita List<Integer> OU List<Double>, ambas são "? extends Number"
        List<Integer> inteiros = List.of(1, 2, 3);
        List<Double> decimais = List.of(1.5, 2.5, 3.5);
        System.out.println("Soma inteiros: " + somarTodos(inteiros));   // 6.0
        System.out.println("Soma decimais: " + somarTodos(decimais));  // 7.5

        // Wildcard super: aceita List<Integer>, List<Number> ou List<Object>
        List<Number> listaDeNumbers = new ArrayList<>();
        preencherComInteiros(listaDeNumbers, 3);
        System.out.println(listaDeNumbers); // [1, 2, 3]
    }
}
```

---

### 3. Armadilhas comuns

1. **Usar tipos raw (sem parametrização), tipo `List listaAntiga = new ArrayList();`** — compila (por compatibilidade retroativa), mas perde toda a segurança de tipo que Generics oferece, e o compilador te dá warnings ("unchecked call"). Em código novo, isso deve ser evitado — sempre parametrize.
2. **Tentar inserir elementos numa lista com wildcard `? extends`** — `List<? extends Number> lista = ...; lista.add(10);` não compila. O compilador não consegue garantir que `10` (um `Integer`) é compatível com o tipo real e desconhecido daquela lista (poderia ser `List<Double>`, por exemplo), então bloqueia qualquer inserção por segurança. Isso não é um bug do seu código — é a garantia de tipo funcionando como deveria.
3. **Confundir quando usar `extends` vs `super`** — a forma mais fácil de lembrar é o mnemônico PECS: se o método vai **ler/produzir** dados da coleção pra você, use `extends`; se o método vai **escrever/consumir** dados que você fornece, use `super`. Errar essa escolha normalmente se manifesta como "o compilador não deixa eu fazer X", que é exatamente o Generics protegendo você de um erro de tipo em potencial.
4. **Achar que `List<Integer>` é um subtipo de `List<Number>`**, só porque `Integer` é subtipo de `Number` — **não é**. Generics em Java não são covariantes por padrão (diferente de array, que vimos lá na armadilha #4 de Array vs ArrayList, e que é justamente por isso que arrays têm o problema do `ArrayStoreException` em runtime). `List<Integer> lista = new ArrayList<Number>();` não compila. É exatamente pra contornar essa limitação de forma segura que existem os wildcards (`? extends`/`? super`).

---

### 4. Exercícios práticos

**1. Fácil**  
Crie uma classe genérica `Par<A, B>` que guarda dois valores de tipos possivelmente diferentes (`primeiro` do tipo `A`, `segundo` do tipo `B`), com um construtor que recebe os dois valores e métodos `getPrimeiro()`/`getSegundo()`. Sobrescreva `toString()` para retornar algo como `"(valor1, valor2)"`. Critério de pronto: `new Par<>("idade", 25)` deve imprimir `(idade, 25)` ao usar `System.out.println`.

**2. Fácil/Médio**  
Escreva um método genérico estático `<T> boolean todosIguais(List<T> lista)` que retorna `true` se todos os elementos da lista forem iguais entre si (usando `.equals()`), e `false` caso contrário (lista vazia ou com um único elemento deve retornar `true`). Critério de pronto: `todosIguais(List.of(5, 5, 5))` → `true`; `todosIguais(List.of("a", "b", "a"))` → `false`; `todosIguais(List.of(1))` → `true`.

**3. Médio**  
Escreva um método `static double somarNumeros(List<? extends Number> lista)` que soma todos os elementos de qualquer lista de números (funcionando tanto para `List<Integer>` quanto `List<Double>` quanto `List<Long>`, sem precisar de um método sobrecarregado pra cada tipo). Teste com pelo menos duas listas de tipos diferentes. Critério de pronto: deve funcionar corretamente tanto para `List.of(1, 2, 3)` (Integer) quanto para `List.of(1.5, 2.5)` (Double), sem duplicar código.

**4. Difícil/Desafio**  
Implemente uma classe genérica `Pilha<T>` do zero (sem usar `Deque`/`Stack` do Java internamente — implemente usando um `ArrayList<T>` como armazenamento interno), com os métodos `push(T item)`, `T pop()` (lança uma exceção customizada `PilhaVaziaException` — crie essa exceção também — se a pilha estiver vazia), `T peek()` (mesma regra de exceção) e `boolean isEmpty()`. Depois, escreva um método genérico separado `static <T> Pilha<T> inverterPilha(Pilha<T> original)` que recebe uma pilha e retorna uma **nova** pilha com os elementos na ordem invertida, sem modificar a pilha original (dica: você vai precisar de uma pilha auxiliar). Critério de pronto: empilhando `1, 2, 3` na pilha original (nessa ordem) e invertendo, a nova pilha deve desempilhar na ordem `1, 2, 3` (o oposto de LIFO da original, que desempilharia `3, 2, 1`).

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Par<A, B> {
    private A primeiro;
    private B segundo;

    public Par(A primeiro, B segundo) {
        this.primeiro = primeiro;
        this.segundo = segundo;
    }

    public A getPrimeiro() { return primeiro; }
    public B getSegundo() { return segundo; }

    @Override
    public String toString() {
        return "(" + primeiro + ", " + segundo + ")";
    }

    public static void main(String[] args) {
        Par<String, Integer> par = new Par<>("idade", 25);
        System.out.println(par); // (idade, 25)
    }
}
```

_Raciocínio:_ `Par<A, B>` usa **dois** parâmetros de tipo independentes, um pra cada campo — isso é exatamente o padrão usado, por exemplo, em `Map.Entry<K, V>` do próprio Java. Cada instância de `Par` fixa seus próprios tipos concretos (`String` e `Integer` nesse exemplo) no momento da criação, e o compilador garante que `getPrimeiro()` sempre retorna `String` e `getSegundo()` sempre retorna `Integer`, sem necessidade de cast.

**Exercício 2**

java

```java
public class Exercicio2 {
    static <T> boolean todosIguais(List<T> lista) {
        if (lista.size() <= 1) {
            return true;
        }
        T primeiro = lista.get(0);
        for (T item : lista) {
            if (!item.equals(primeiro)) {
                return false;
            }
        }
        return true;
    }

    public static void main(String[] args) {
        System.out.println(todosIguais(List.of(5, 5, 5)));       // true
        System.out.println(todosIguais(List.of("a", "b", "a"))); // false
        System.out.println(todosIguais(List.of(1)));              // true
    }
}
```

_Raciocínio:_ o método é genérico em `T` porque não nos importa o tipo concreto dos elementos — só precisamos comparar com `.equals()`, que todo objeto Java tem (herdado de `Object`). O caso `size() <= 1` cobre tanto lista vazia quanto lista de um elemento, ambos considerados "todos iguais" por vacuidade/trivialidade — evita comparar contra algo que não existe.

**Exercício 3**

java

```java
public class Exercicio3 {
    static double somarNumeros(List<? extends Number> lista) {
        double soma = 0;
        for (Number n : lista) {
            soma += n.doubleValue();
        }
        return soma;
    }

    public static void main(String[] args) {
        List<Integer> inteiros = List.of(1, 2, 3);
        List<Double> decimais = List.of(1.5, 2.5);

        System.out.println(somarNumeros(inteiros)); // 6.0
        System.out.println(somarNumeros(decimais)); // 4.0
    }
}
```

_Raciocínio:_ `List<? extends Number>` é exatamente o cenário "produtor" do PECS — o método só **lê** valores da lista (nunca insere nada nela), então usar `extends` é seguro e permite aceitar `List<Integer>`, `List<Double>`, `List<Long>` etc., todas com o mesmo método, sem sobrecarga (overload) repetida pra cada tipo. `doubleValue()` é um método que toda subclasse de `Number` implementa, garantindo que conseguimos extrair um valor numérico independente do tipo concreto.

**Exercício 4**

java

```java
public class PilhaVaziaException extends RuntimeException {
    public PilhaVaziaException(String mensagem) {
        super(mensagem);
    }
}

public class Pilha<T> {
    private List<T> elementos = new ArrayList<>();

    void push(T item) {
        elementos.add(item); // adiciona no FIM da lista - vamos tratar o fim como "topo"
    }

    T pop() {
        if (isEmpty()) {
            throw new PilhaVaziaException("Não é possível desempilhar: pilha vazia");
        }
        return elementos.remove(elementos.size() - 1); // remove e retorna o último (topo)
    }

    T peek() {
        if (isEmpty()) {
            throw new PilhaVaziaException("Não é possível espiar: pilha vazia");
        }
        return elementos.get(elementos.size() - 1);
    }

    boolean isEmpty() {
        return elementos.isEmpty();
    }

    static <T> Pilha<T> inverterPilha(Pilha<T> original) {
        Pilha<T> auxiliar = new Pilha<>();
        Pilha<T> invertida = new Pilha<>();

        // Passo 1: esvazia a original numa auxiliar, SEM modificar a original de verdade -
        // então empilhamos de volta na original ao final, restaurando seu estado
        List<T> backup = new ArrayList<>();
        while (!original.isEmpty()) {
            T item = original.pop();
            backup.add(item);       // guarda pra restaurar a original depois
            auxiliar.push(item);
        }
        // restaura a pilha original ao estado inicial
        for (int i = backup.size() - 1; i >= 0; i--) {
            original.push(backup.get(i));
        }

        // Passo 2: desempilhar a auxiliar (que está na ordem inversa da original)
        // empilhando na "invertida" produz a ordem invertida de fato
        while (!auxiliar.isEmpty()) {
            invertida.push(auxiliar.pop());
        }

        return invertida;
    }

    public static void main(String[] args) {
        Pilha<Integer> original = new Pilha<>();
        original.push(1);
        original.push(2);
        original.push(3);

        Pilha<Integer> invertida = inverterPilha(original);

        System.out.println("Original desempilha: " + original.pop() + ", " + original.pop() + ", " + original.pop());
        // 3, 2, 1 (LIFO normal - confirma que a original não foi alterada)

        System.out.println("Invertida desempilha: " + invertida.pop() + ", " + invertida.pop() + ", " + invertida.pop());
        // 1, 2, 3 (ordem invertida)
    }
}
```

_Raciocínio:_ a parte mais delicada do exercício é o requisito **"sem modificar a pilha original"** — como só temos `push`/`pop`/`peek` (não temos como "olhar" a pilha sem alterá-la), a única forma de inspecionar todos os elementos é esvaziá-la temporariamente. Por isso, guardamos uma cópia em `backup` (uma `List` comum, só usada como memória temporária) enquanto esvaziamos a original na `auxiliar`, e depois **restauramos** a original percorrendo o backup de trás pra frente (`for` decrescente), reconstruindo exatamente a ordem original de push. Já a pilha `auxiliar` ficou com os elementos em ordem invertida em relação à original (porque desempilhar é sempre o oposto de empilhar); desempilhar a `auxiliar` e empilhar cada item na `invertida` aplica essa inversão **mais uma vez**, o que dá o resultado final: a `invertida` tem os elementos na ordem exatamente oposta ao LIFO da pilha original — ou seja, ela desempilha na mesma ordem em que os itens foram originalmente empilhados.