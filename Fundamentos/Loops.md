### 1. Teoria

**O que são loops?**

Loops são construções que repetem um bloco de código enquanto uma condição for satisfeita, evitando duplicar código manualmente quando você precisa executar a mesma lógica várias vezes (com ou sem variação nos dados a cada repetição). Java oferece quatro formas principais.

#### `for` tradicional

java

```java
for (inicialização; condição; incremento) {
    // corpo do loop
}
```

As três partes rodam nesta ordem lógica: inicialização (uma vez, no início) → checa condição → executa corpo → executa incremento → checa condição de novo → repete até a condição ser `false`. É o loop ideal quando você sabe (ou consegue calcular) quantas vezes precisa repetir, e/ou precisa de um contador/índice.

#### `while`

java

```java
while (condição) {
    // corpo do loop
}
```

Testa a condição **antes** de cada execução do corpo. Se a condição já começar `false`, o corpo nunca executa nem uma vez. Ideal quando você não sabe de antemão quantas repetições vai precisar — depende de algo que só se descobre durante a execução (ex: ler dados até encontrar um valor sentinela).

#### `do-while`

java

```java
do {
    // corpo do loop
} while (condição);
```

Testa a condição **depois** de cada execução — o que garante que o corpo executa **pelo menos uma vez**, mesmo que a condição já comece `false`. É o menos usado dos quatro, mas resolve bem casos como "peça uma entrada ao usuário, e repita enquanto a entrada for inválida" (você precisa pedir pelo menos uma vez antes de checar).

#### `for-each` (enhanced for)

Já visto no tópico de Arrays:

java

```java
for (tipo elemento : colecaoOuArray) {
    // corpo
}
```

Não dá acesso ao índice diretamente, e não permite controlar o "passo" (pular de 2 em 2, por exemplo) — para isso, use o `for` tradicional.

**Controle de fluxo dentro de loops**

- **`break`** — interrompe o loop imediatamente, saindo dele por completo (mesmo comportamento já visto em `switch`, mas aqui interrompe repetição, não fall-through).
- **`continue`** — pula o restante do corpo **da iteração atual** e vai direto pra próxima checagem de condição, sem sair do loop inteiro.
- **Labels** (rótulos) — permitem que `break`/`continue` afetem um loop _externo_ específico, quando há loops aninhados:

java

```java
externo:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 1) continue externo; // pula pra próxima iteração do loop EXTERNO, não do interno
        System.out.println(i + "," + j);
    }
}
```

Label é pouco usado no dia a dia, mas é importante reconhecer a sintaxe — sem ela, `break`/`continue` só afetam o loop mais interno onde estão.

**Loop infinito (proposital ou por bug)**

java

```java
for (;;) { ... }      // for sem nenhuma das 3 partes = loop infinito proposital
while (true) { ... }  // equivalente, mais legível
```

Isso é usado deliberadamente em situações como um servidor esperando conexões continuamente — mas também é a causa mais comum de bug de "programa travou", quando a condição de parada nunca é alcançada por erro de lógica (ex: esquecer de incrementar a variável de controle dentro de um `while`).

**Onde isso aparece na prática (backend real)**

Loops estão em todo lugar: iterar sobre resultados de query (`List<Pedido>` retornado de um repository), processar itens de uma requisição em lote, validar cada campo de um formulário. Na prática de backend moderno com Java 8+, muita dessa iteração é feita com Stream API (`.forEach()`, `.map()`, etc.) em vez de loop explícito — mas Stream API é construído sobre os mesmos conceitos de iteração que você está aprendendo agora, e será aprofundado no bloco de Programação Funcional.

---

### 2. Exemplo de código comentado

java

```java
public class LoopsExemplo {
    public static void main(String[] args) {

        // for tradicional — sei exatamente quantas vezes quero repetir
        for (int i = 1; i <= 5; i++) {
            System.out.println("Contagem: " + i);
        }

        // while — não sei de antemão quantas iterações, depende de condição dinâmica
        int saldo = 100;
        int meses = 0;
        while (saldo > 0) {
            saldo -= 30; // gasto mensal fixo, só de exemplo
            meses++;
        }
        System.out.println("Saldo acabou depois de " + meses + " meses");

        // do-while — corpo executa pelo menos uma vez, mesmo com condição falsa de cara
        int tentativas = 0;
        int senhaCorreta = 1234;
        int senhaDigitada = 9999;
        do {
            tentativas++;
            // aqui normalmente teria leitura de entrada real; simulando com valor fixo
        } while (senhaDigitada != senhaCorreta && tentativas < 3);
        System.out.println("Tentativas gastas: " + tentativas);

        // continue — pula números pares, só processa ímpares
        System.out.print("Ímpares de 1 a 10: ");
        for (int i = 1; i <= 10; i++) {
            if (i % 2 == 0) {
                continue; // volta pro topo do loop, sem executar o println abaixo
            }
            System.out.print(i + " ");
        }
        System.out.println();

        // break — para assim que encontrar o que procura
        int[] numeros = {4, 8, 15, 16, 23, 42};
        int alvo = 16;
        int posicaoEncontrada = -1;
        for (int i = 0; i < numeros.length; i++) {
            if (numeros[i] == alvo) {
                posicaoEncontrada = i;
                break; // não precisa continuar procurando depois de achar
            }
        }
        System.out.println("Encontrado na posição: " + posicaoEncontrada);

        // Loop aninhado com label — break/continue afetando o loop externo
        externo:
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                if (i == 1 && j == 1) {
                    break externo; // sai dos DOIS loops de uma vez, não só do interno
                }
                System.out.println("i=" + i + " j=" + j);
            }
        }
    }
}
```

---

### 3. Armadilhas comuns

1. **Loop infinito por esquecer de atualizar a variável de controle.** `while (contador < 10) { System.out.println(contador); }` sem `contador++` dentro do corpo nunca termina — o programa "trava" (na verdade está rodando pra sempre, só que não avança).
2. **Off-by-one em `for` tradicional.** `for (int i = 0; i <= array.length; i++)` tenta acessar `array[array.length]`, que não existe — o erro clássico já visto no tópico de Arrays, mas que reaparece aqui porque loop e array andam sempre juntos.
3. **Confundir `break` com `continue`.** `break` sai do loop inteiro; `continue` só pula a iteração atual e segue pra próxima. Trocar um pelo outro por engano muda completamente o comportamento do programa, e o compilador não vai avisar — os dois são sintaticamente válidos em qualquer um dos dois contextos.
4. **Modificar uma coleção dentro de um `for-each` que está percorrendo ela.** Isso lança `ConcurrentModificationException` em tempo de execução (será visto com mais detalhe quando chegarmos em Collections/Iterator) — array puro não lança essa exceção especificamente, mas a prática de alterar a estrutura que você está percorrendo, no meio da iteração, é fonte de bug sutil de qualquer forma.

---

### 4. Exercícios práticos

**Exercício 1 (fácil)**  
Escreva um programa que imprima todos os números de 1 a 20 usando um `for` tradicional, mas pulando (sem imprimir) os múltiplos de 3, usando `continue`. Critério de pronto: a saída não contém 3, 6, 9, 12, 15, 18, e contém todos os outros números de 1 a 20.

**Exercício 2 (médio)**  
Escreva um programa que simule um processo de "adivinhação": uma variável `int numeroSecreto = 42;` fixa, e um array `int[] palpites = {10, 25, 60, 42, 90};` simulando tentativas de um usuário. Usando `for` com `break`, percorra os palpites e pare assim que encontrar o número secreto, imprimindo em qual tentativa (índice + 1, não o índice zero-based) o número foi acertado. Se o array acabar sem acertar, imprima uma mensagem informando que não foi encontrado (pense: como saber, depois do loop, se ele terminou por `break` ou porque chegou ao fim naturalmente? — dica: uma variável `boolean` auxiliar resolve isso). Critério de pronto: funciona corretamente tanto com o array que contém o número secreto quanto com um array de teste que não contém.

**Exercício 3 (difícil)**  
Usando `while` (não `for`), calcule a soma dos dígitos de um número inteiro positivo até sobrar um único dígito (esse processo é conhecido como "raiz digital" — ex: `1234` → `1+2+3+4=10` → `1+0=1`, resultado final `1`). Use o número `987654` como entrada fixa. Dicas de operadores que você vai precisar: `%` (resto da divisão, pra extrair o último dígito) e `/` (divisão inteira, pra "remover" o último dígito). Critério de pronto: o programa imprime cada etapa intermediária da soma (ex: "987654 → 39", "39 → 12", "12 → 3") até chegar ao dígito único final.

**Exercício 4 (desafio)**  
Usando loops aninhados **com label**, escreva um programa que procure um par de números dentro de dois arrays diferentes (`int[] arrayA = {2, 4, 6, 8};` e `int[] arrayB = {5, 7, 9, 11};`) cuja soma seja exatamente `15`. Assim que encontrar o **primeiro par** que soma 15, imprima os dois valores e pare **totalmente** a busca (os dois loops), usando `break` com label — não continue procurando outros pares depois de achar o primeiro. Se quiser, escreva também uma segunda versão sem label, usando uma variável `boolean` de controle, e compare qual ficou mais legível na sua opinião (comentário no código, resposta livre, sem gabarito "certo" pra essa parte de opinião). Critério de pronto: a versão com label encontra e imprime corretamente o primeiro par (dica: existe mais de um par possível nesses arrays — o resultado depende da ordem de iteração escolhida).

---

### 5. Gabarito comentado

**Exercício 1**

java

```java
public class Exercicio1 {
    public static void main(String[] args) {
        for (int i = 1; i <= 20; i++) {
            if (i % 3 == 0) {
                continue;
            }
            System.out.println(i);
        }
    }
}
```

Raciocínio: `i % 3 == 0` é o teste padrão pra "é múltiplo de 3" — resto zero na divisão por 3. O `continue` pula só o `println` daquela iteração específica, sem interromper o loop inteiro; as próximas iterações continuam normalmente.

**Exercício 2**

java

```java
public class Exercicio2 {
    public static void main(String[] args) {
        int numeroSecreto = 42;
        int[] palpites = {10, 25, 60, 42, 90};
        boolean encontrado = false;

        for (int i = 0; i < palpites.length; i++) {
            if (palpites[i] == numeroSecreto) {
                System.out.println("Acertou na tentativa " + (i + 1));
                encontrado = true;
                break;
            }
        }

        if (!encontrado) {
            System.out.println("Número secreto não foi encontrado nos palpites");
        }
    }
}
```

Raciocínio: a variável `boolean encontrado` é o que resolve o problema de "como saber por que o loop terminou" — sem ela, não haveria como diferenciar programaticamente "saiu por break" de "saiu porque o array acabou". Usei `(i + 1)` na impressão porque o enunciado pediu a tentativa em contagem humana (1ª, 2ª...), não o índice zero-based interno.

**Exercício 3**

java

```java
public class Exercicio3 {
    public static void main(String[] args) {
        int numero = 987654;

        while (numero >= 10) { // continua enquanto tiver mais de um dígito
            int soma = 0;
            int original = numero;

            while (numero > 0) {
                soma += numero % 10; // extrai o último dígito
                numero /= 10;         // remove o último dígito (divisão inteira)
            }

            System.out.println(original + " → " + soma);
            numero = soma; // prepara para a próxima rodada externa, se precisar
        }

        System.out.println("Raiz digital final: " + numero);
    }
}
```

Saída:

```
987654 → 39
39 → 12
12 → 3
Raiz digital final: 3
```

Raciocínio: há dois `while` porque são dois problemas distintos aninhados: o loop interno soma os dígitos de UM número (até esse número virar zero, dígito por dígito); o loop externo repete esse processo inteiro enquanto o resultado ainda tiver mais de um dígito (`>= 10`). Guardei `original` só para poder imprimir a etapa de forma legível (`987654 → 39`), sem perder o valor de `numero` que estava sendo consumido pelo loop interno.

**Exercício 4**

java

```java
public class Exercicio4 {
    public static void main(String[] args) {
        int[] arrayA = {2, 4, 6, 8};
        int[] arrayB = {5, 7, 9, 11};

        buscaComLabel:
        for (int i = 0; i < arrayA.length; i++) {
            for (int j = 0; j < arrayB.length; j++) {
                if (arrayA[i] + arrayB[j] == 15) {
                    System.out.println("Par encontrado: " + arrayA[i] + " + " + arrayB[j] + " = 15");
                    break buscaComLabel; // interrompe os DOIS loops de uma vez
                }
            }
        }
    }
}
```

Saída (dado a ordem de iteração desses arrays especificamente): `Par encontrado: 4 + 11 = 15`

Raciocínio: sem o label, um `break` simples dentro do loop interno só sairia do loop de `j`, e o loop de `i` continuaria rodando normalmente — o que faria o programa continuar testando outros pares mesmo depois de já ter achado um, potencialmente sobrescrevendo o resultado. O label `buscaComLabel:` amarra o `break` ao loop externo especificamente, garantindo que a busca pare de verdade assim que a primeira soma válida é encontrada.

Versão alternativa sem label, com variável de controle (pra comparação):

java

```java
public class Exercicio4SemLabel {
    public static void main(String[] args) {
        int[] arrayA = {2, 4, 6, 8};
        int[] arrayB = {5, 7, 9, 11};
        boolean encontrado = false;

        for (int i = 0; i < arrayA.length && !encontrado; i++) {
            for (int j = 0; j < arrayB.length && !encontrado; j++) {
                if (arrayA[i] + arrayB[j] == 15) {
                    System.out.println("Par encontrado: " + arrayA[i] + " + " + arrayB[j] + " = 15");
                    encontrado = true;
                }
            }
        }
    }
}
```

Trade-off: a versão com label é mais direta e curta; a versão com `boolean` evita o uso de label (que parte considerável da comunidade Java evita por convenção de estilo, considerando-o menos legível em bases de código maiores) ao custo de adicionar a condição `&& !encontrado` em **ambos** os loops — fácil esquecer de adicionar em um dos dois por descuido, o que reintroduziria o bug de continuar buscando após já ter achado.