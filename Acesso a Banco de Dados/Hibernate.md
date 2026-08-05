#### 1. Teoria

O **Hibernate** é a implementação da especificação JPA que o Spring Data JPA usa por baixo dos panos (quando você adiciona `spring-boot-starter-data-jpa`, o Hibernate vem junto como dependência transitiva). Você não vai escrever código Hibernate diretamente no dia a dia — o Spring Data JPA já te isola disso — mas existem dois conceitos do Hibernate que **explicam comportamentos que você vai ver no Spring Data JPA e que, sem entender a causa, parecem mágica ou bug**:

**Persistence Context (contexto de persistência):** é a "memória de trabalho" do Hibernate durante uma transação. Quando você busca uma entidade (`findById`, por exemplo), o Hibernate guarda uma referência a ela e um retrato do estado original — a entidade fica **managed** (gerenciada).

**Dirty Checking (verificação de sujeira):** enquanto a entidade está managed, se você alterar um campo dela (`usuario.setEmail("novo@email.com")`) **dentro de uma transação**, o Hibernate detecta a diferença entre o estado atual e o retrato original, e gera um `UPDATE` sozinho, no fechamento da transação — **sem você chamar `save()`**. É esse mecanismo que explica por que às vezes você vê código Spring que altera um campo e nunca chama `repository.save()`, e o dado é atualizado mesmo assim.

Isso só funciona dentro de uma transação ativa (`@Transactional`) — é justamente o gancho pra entender a próxima armadilha.

---

#### 2. Exemplo de código comentado

java

```java
@Service
public class UsuarioService {

    private final UsuarioRepository usuarioRepository;

    public UsuarioService(UsuarioRepository usuarioRepository) {
        this.usuarioRepository = usuarioRepository;
    }

    @Transactional // ESSENCIAL pro dirty checking funcionar
    public void atualizarEmail(Long id, String novoEmail) {
        Usuario usuario = usuarioRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Usuário não encontrado"));

        usuario.setEmail(novoEmail); // entidade managed, alteração fica "suja" (dirty)

        // Repare: NENHUMA chamada a usuarioRepository.save(usuario) aqui.
        // Ao fechar a transação (fim do método), o Hibernate compara o estado
        // atual com o retrato original e gera o UPDATE sozinho.
    }
}
```

---

#### 3. Armadilhas comuns

1. **Esperar dirty checking funcionar sem `@Transactional`** — sem transação ativa, não existe persistence context de verdade acompanhando a entidade, e a alteração simplesmente não é persistida. É a causa mais comum de "eu alterei o campo e não salvou nada, sem erro nenhum".
2. **Achar que o Persistence Context é compartilhado entre requisições** — ele existe só durante o escopo da transação/sessão. Cada requisição HTTP normalmente tem seu próprio persistence context, criado e descartado. Não é um cache global da aplicação.

---

#### 4. Exercícios práticos

**Exercício 1 (fácil/médio)**  
No `ProdutoService` que você criou na sessão anterior, escreva um método `@Transactional` que busca um produto por ID e altera o `preco`, **sem chamar `save()`**. Confirme (via `findById` numa chamada separada, depois da transação terminar) que o preço foi atualizado no banco.

**Exercício 2 (médio)**  
Remova o `@Transactional` do método do exercício 1 e rode de novo. Descreva o que acontece e por quê — conecte a explicação com o conceito de persistence context desta seção.

---

#### 5. Gabarito comentado

**Exercício 1**

java

```java
@Transactional
public void atualizarPreco(Long id, Double novoPreco) {
    Produto produto = produtoRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("Produto não encontrado"));
    produto.setPreco(novoPreco); // sem save() — dirty checking cuida do UPDATE
}
```

Raciocínio: o `produto` retornado por `findById` dentro do método `@Transactional` está managed. A alteração de `preco` fica marcada como dirty, e o Hibernate sincroniza com o banco automaticamente ao final da transação (flush + commit).

**Exercício 2**  
Sem `@Transactional`, cada chamada ao repositório abre e fecha sua própria transação/sessão internamente (comportamento padrão do Spring Data JPA pra métodos únicos como `findById`). A entidade retornada já sai **detached** (desconectada do persistence context) assim que o método do repositório termina. Alterar um campo dela depois disso não tem efeito nenhum no banco — não existe mais nenhum mecanismo observando aquele objeto pra detectar a mudança. Isso reforça por que `@Transactional` não é só "boa prática", é o que garante que o dirty checking tenha uma janela pra agir.