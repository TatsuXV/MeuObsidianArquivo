#### 1. Teoria

O Java não tem "uma API de criptografia" — ele tem a **JCA (Java Cryptography Architecture)**, um framework baseado em _providers_: você pede um algoritmo pelo nome (`MessageDigest.getInstance("SHA-256")`, `Cipher.getInstance("AES/GCM/NoPadding")`) e a JVM entrega a implementação registrada. O provider padrão (SunJCE) cobre o básico; se precisar de algo mais específico (Argon2, por exemplo), você adiciona uma lib como Bouncy Castle.

As peças principais que você vai usar em backend real:

- **`MessageDigest`** — hash. Transforma qualquer entrada em uma saída de tamanho fixo, de forma **irreversível**. Serve pra verificar integridade (o conteúdo mudou?), não pra "proteger" dado que você precisa recuperar depois.
- **`Cipher` + `KeyGenerator`/`KeyPairGenerator`** — criptografia simétrica (AES) e assimétrica (RSA). Isso é **reversível**: você cifra com uma chave e decifra com a mesma chave (simétrica) ou com o par correspondente (assimétrica). Usa quando precisa recuperar o dado original — ex: CPF criptografado no banco que a aplicação precisa ler de volta.
- **`SecretKeyFactory` + `PBEKeySpec`** — derivação de chave a partir de senha (PBKDF2). É o que você usa pra fazer hash de senha _de verdade_ — a diferença pro `MessageDigest` puro está nas armadilhas, abaixo.
- **`Mac`** — hash com chave (HMAC), garante integridade **e** autenticidade (só quem tem a chave gera o mesmo MAC). Base de como JWT assina tokens com HS256.
- **`SecureRandom`** — gerador de números aleatórios criptograficamente seguro. Diferente de `java.util.Random`, que é previsível.

Onde isso aparece no dia a dia: hash de senha antes de salvar no banco (nunca texto puro), geração de token de sessão/API key, criptografar campo sensível em repouso, e por trás da geração/validação de JWT no Spring Security.

Uma linha fora do escopo, só pra contexto: a partir do Java 21 existe também a **KEM API (JEP 452)**, um jeito padronizado de estabelecer chave compartilhada — relevante pra criptografia pós-quântica, mas isso é assunto avançado, não é o que cai em vaga júnior.

#### 2. Exemplo de código comentado — hash de integridade com SHA-256

java

```java
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.nio.charset.StandardCharsets;

public class IntegrityHashExample {

    public static String sha256Hex(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256"); // pega a implementação registrada no provider padrão
            byte[] hashBytes = digest.digest(input.getBytes(StandardCharsets.UTF_8)); // calcula o hash de uma vez

            StringBuilder hex = new StringBuilder();
            for (byte b : hashBytes) {
                hex.append(String.format("%02x", b)); // cada byte -> 2 caracteres hexadecimais
            }
            return hex.toString();
        } catch (NoSuchAlgorithmException e) {
            // só ocorreria se o algoritmo não existisse no provider; SHA-256 é padrão em qualquer JVM
            throw new IllegalStateException("SHA-256 não disponível", e);
        }
    }

    public static void main(String[] args) {
        String conteudo = "conteúdo do arquivo que quero verificar depois";
        System.out.println(sha256Hex(conteudo));
        // se 1 único bit do conteúdo mudar, o hash inteiro muda (efeito avalanche)
    }
}
```

Repare: isso é pra **verificar integridade** de algo não-secreto (o conteúdo mudou desde a última vez?). Não é assim que se guarda senha — motivo na próxima seção.

#### 3. Armadilhas comuns

1. **Usar `MessageDigest` puro (SHA-256/MD5) pra hash de senha.** Sem salt e sem custo computacional ajustável, um atacante com GPU testa bilhões de senhas por segundo contra o hash vazado. Senha precisa de algoritmo _lento de propósito_ (PBKDF2, BCrypt, Argon2), não um hash rápido feito pra checar integridade de arquivo.
2. **Reutilizar o mesmo IV/nonce com a mesma chave** em modos como GCM ou CTR. Isso quebra a segurança do modo inteiro (permite recuperar informação do texto cifrado). IV tem que ser único a cada operação — por isso ele é gerado com `SecureRandom` a cada chamada.
3. **Usar `java.util.Random` em vez de `SecureRandom`** pra gerar chave, salt, IV ou token. `Random` é um PRNG previsível — dado a semente (ou parte da sequência), dá pra prever o resto.
4. **Chave de criptografia hardcoded no código-fonte** (ou commitada no repositório). Deveria vir de variável de ambiente ou secret manager (Vault, AWS KMS, etc.) — nunca do código versionado.

#### 4. Exercícios práticos

**Fácil** — Escreva `verificarIntegridade(String conteudoOriginal, String hashEsperadoHex)`, que recalcula o SHA-256 do conteúdo (reaproveite `sha256Hex`) e retorna `true`/`false` se bate com o hash esperado. Critério de pronto: alterar 1 caractere do conteúdo original deve fazer o método retornar `false`.

**Médio** — Implemente hash de senha de verdade com PBKDF2: `hash(char[] senha)` retorna uma `String` contendo salt (gerado com `SecureRandom`) + hash derivado (`SecretKeyFactory` com `PBEKeySpec`, no mínimo 10.000 iterações); `verify(char[] senha, String hashArmazenado)` confere se uma senha bate com o hash guardado. Critério de pronto: duas chamadas de `hash()` com a mesma senha devem gerar strings _diferentes_ (por causa do salt aleatório); `verify()` retorna `true` pra senha correta e `false` pra errada.

**Difícil** — Implemente `encrypt(String texto, SecretKey chave)` e `decrypt(String textoCifrado, SecretKey chave)` usando AES-256 em modo GCM, com IV aleatório gerado a cada chamada e embutido junto do resultado (pra poder decifrar depois sem precisar guardar o IV em outro lugar). Critério de pronto: `decrypt(encrypt(x)) == x`; duas chamadas de `encrypt()` com o mesmo texto e a mesma chave devem gerar saídas _diferentes_.

**Desafio** — Combine os conceitos: um serviço de "token de recuperação de senha". `gerarToken(String username)` cria um token aleatório de alta entropia (mínimo 32 bytes, `SecureRandom`), codificado em Base64 URL-safe, mas **só guarda o hash SHA-256 do token** num `Map` (simulando o banco) — nunca o token em texto puro. `validarToken(String username, String tokenRecebido)` recalcula o hash do token recebido e compara com o armazenado, usando comparação em tempo constante. Critério de pronto: token gerado corretamente valida uma vez; token errado/adulterado retorna `false`.

#### 5. Gabarito comentado

**Fácil:**

java

```java
public static boolean verificarIntegridade(String conteudoOriginal, String hashEsperadoHex) {
    String hashCalculado = sha256Hex(conteudoOriginal);
    return hashCalculado.equalsIgnoreCase(hashEsperadoHex);
}
```

Aqui `String.equals` (via `equalsIgnoreCase`) é suficiente — o conteúdo não é secreto, então não existe timing attack relevante. Isso muda nos próximos exercícios, onde estamos comparando segredo (senha/token).

**Médio:**

java

```java
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.PBEKeySpec;
import java.security.SecureRandom;
import java.security.MessageDigest;
import java.util.Base64;
import java.util.Arrays;

public class PasswordHasher {

    private static final int ITERATIONS = 65536; // custo computacional: mais alto = mais lento pra atacante (e pra você)
    private static final int KEY_LENGTH = 256;
    private static final String ALGORITHM = "PBKDF2WithHmacSHA256";

    public static String hash(char[] senha) {
        try {
            byte[] salt = gerarSalt(); // salt único por senha impede ataque de rainbow table
            PBEKeySpec spec = new PBEKeySpec(senha, salt, ITERATIONS, KEY_LENGTH);
            SecretKeyFactory factory = SecretKeyFactory.getInstance(ALGORITHM);
            byte[] hash = factory.generateSecret(spec).getEncoded();

            // salt precisa ir junto do hash — sem ele não dá pra verificar depois
            return Base64.getEncoder().encodeToString(salt) + ":" + Base64.getEncoder().encodeToString(hash);
        } catch (Exception e) {
            throw new RuntimeException("Erro ao gerar hash da senha", e);
        } finally {
            Arrays.fill(senha, '0'); // zera o array em memória — boa prática pra dado sensível
        }
    }

    public static boolean verify(char[] senha, String hashArmazenado) {
        String[] partes = hashArmazenado.split(":");
        byte[] salt = Base64.getDecoder().decode(partes[0]);
        byte[] hashEsperado = Base64.getDecoder().decode(partes[1]);

        try {
            PBEKeySpec spec = new PBEKeySpec(senha, salt, ITERATIONS, KEY_LENGTH);
            SecretKeyFactory factory = SecretKeyFactory.getInstance(ALGORITHM);
            byte[] hashCalculado = factory.generateSecret(spec).getEncoded();
            return MessageDigest.isEqual(hashCalculado, hashEsperado); // comparação em tempo constante
        } catch (Exception e) {
            throw new RuntimeException("Erro ao verificar senha", e);
        }
    }

    private static byte[] gerarSalt() {
        byte[] salt = new byte[16];
        new SecureRandom().nextBytes(salt);
        return salt;
    }
}
```

**Alternativa e trade-off:** em projeto Spring real, é mais comum usar `BCryptPasswordEncoder` do Spring Security em vez de implementar PBKDF2 na mão. BCrypt (e Argon2, ainda melhor) são _memory-hard_ — mais resistentes a ataque com GPU/ASIC do que PBKDF2, que é só CPU-bound. PBKDF2 vale o exercício porque usa só a JDK padrão (sem dependência extra) e ensina o mecanismo por trás; em produção, prefira BCrypt/Argon2 prontos.

**Difícil:**

java

```java
import javax.crypto.Cipher;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import java.security.SecureRandom;
import java.nio.charset.StandardCharsets;
import java.util.Base64;

public class SymmetricCryptoExample {

    private static final String TRANSFORMATION = "AES/GCM/NoPadding";
    private static final int TAG_LENGTH_BITS = 128;
    private static final int IV_LENGTH_BYTES = 12; // 96 bits é o tamanho recomendado de IV pro GCM

    public static String encrypt(String texto, SecretKey chave) throws Exception {
        byte[] iv = new byte[IV_LENGTH_BYTES];
        new SecureRandom().nextBytes(iv); // IV único a cada chamada, nunca reutilizado com a mesma chave

        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        cipher.init(Cipher.ENCRYPT_MODE, chave, new GCMParameterSpec(TAG_LENGTH_BITS, iv));
        byte[] textoCifrado = cipher.doFinal(texto.getBytes(StandardCharsets.UTF_8));

        // concatena IV + texto cifrado — IV não é segredo, só precisa ser único, então pode ir junto
        byte[] combinado = new byte[iv.length + textoCifrado.length];
        System.arraycopy(iv, 0, combinado, 0, iv.length);
        System.arraycopy(textoCifrado, 0, combinado, iv.length, textoCifrado.length);

        return Base64.getEncoder().encodeToString(combinado);
    }

    public static String decrypt(String textoCifradoBase64, SecretKey chave) throws Exception {
        byte[] combinado = Base64.getDecoder().decode(textoCifradoBase64);

        byte[] iv = new byte[IV_LENGTH_BYTES];
        byte[] textoCifrado = new byte[combinado.length - IV_LENGTH_BYTES];
        System.arraycopy(combinado, 0, iv, 0, IV_LENGTH_BYTES);
        System.arraycopy(combinado, IV_LENGTH_BYTES, textoCifrado, 0, textoCifrado.length);

        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        cipher.init(Cipher.DECRYPT_MODE, chave, new GCMParameterSpec(TAG_LENGTH_BITS, iv));
        return new String(cipher.doFinal(textoCifrado), StandardCharsets.UTF_8);
    }
}
```

**Alternativa e trade-off:** dava pra usar `AES/CBC/PKCS5Padding`, mas CBC sozinho só garante confidencialidade, não integridade — precisaria de um HMAC separado por cima pra evitar _padding oracle attack_. GCM já entrega os dois (confidencialidade + autenticação) numa operação só, por isso é a escolha padrão hoje pra criptografia simétrica nova.

**Desafio:**

java

```java
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.nio.charset.StandardCharsets;
import java.util.Base64;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class PasswordResetTokenService {

    private final Map<String, String> hashesAtivos = new ConcurrentHashMap<>(); // username -> hash do token

    public String gerarToken(String username) {
        byte[] tokenBytes = new byte[32]; // 256 bits de entropia
        new SecureRandom().nextBytes(tokenBytes);
        String token = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes); // URL-safe, vai num link de email

        hashesAtivos.put(username, sha256Hex(token)); // só o hash fica guardado — se o "banco" vazar, o token não é reconstruível
        return token; // texto puro só existe aqui, pra ser enviado; nunca é persistido
    }

    public boolean validarToken(String username, String tokenRecebido) {
        String hashArmazenado = hashesAtivos.get(username);
        if (hashArmazenado == null) return false;

        String hashRecebido = sha256Hex(tokenRecebido);
        return MessageDigest.isEqual(
            hashArmazenado.getBytes(StandardCharsets.UTF_8),
            hashRecebido.getBytes(StandardCharsets.UTF_8)
        ); // comparação em tempo constante — aqui SIM estamos comparando dado sensível
    }

    private String sha256Hex(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder hex = new StringBuilder();
            for (byte b : hash) hex.append(String.format("%02x", b));
            return hex.toString();
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }
}
```

Em produção você adicionaria expiração (guardar timestamp junto do hash) e invalidar o token após o primeiro uso (`remove()` do map depois de validar com sucesso) — fora do escopo do exercício, mas é o próximo passo natural.