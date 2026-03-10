# Phase 16.3 — Cryptography in Java

## Understanding Cryptography from First Principles

Before touching the Java API, you need a solid mental model of what cryptographic primitives do — because misusing them is worse than not using them at all. A false sense of security from poorly applied cryptography is more dangerous than knowing you have no security.

Cryptography addresses four fundamental security needs. **Confidentiality** ensures that only intended parties can read data. **Integrity** ensures that data has not been modified in transit. **Authentication** ensures that you know who created or sent data. **Non-repudiation** ensures that a sender cannot later deny having sent something. Different primitives address different subsets of these needs, and combining them correctly is the art of applied cryptography.

Java provides cryptographic functionality through the **Java Cryptography Architecture (JCA)** and **Java Cryptography Extension (JCE)**, both living primarily in the `java.security` and `javax.crypto` packages. The architecture is provider-based — the API is standard, but the underlying implementation is pluggable (the default provider is "SunJCE"). This means your code can switch cryptographic providers without changing.

---

## SecureRandom — The Foundation of All Cryptography

Before we discuss any algorithm, you must understand `SecureRandom`, because virtually all cryptographic operations depend on high-quality random numbers for their security properties — key generation, nonce creation, initialization vectors, salts, token generation. `Math.random()` and the standard `java.util.Random` use a **predictable pseudo-random algorithm** — they are completely unsuitable for security because an attacker who observes enough output can predict future values.

`SecureRandom` uses the operating system's entropy pool (hardware events like disk seeks, network timings, mouse movements) to produce genuinely unpredictable output:

```java
// ✅ Correct — use SecureRandom for all security-sensitive random numbers
SecureRandom secureRandom = new SecureRandom();

// Generate a 32-byte (256-bit) random value — suitable as an AES key, a token, a salt
byte[] randomBytes = new byte[32];
secureRandom.nextBytes(randomBytes);

// Generate a secure random token (e.g., for a password reset link)
// This is 256 bits of randomness, Base64-encoded for URL safety
String token = Base64.getUrlEncoder().withoutPadding()
    .encodeToString(randomBytes);  // e.g., "k7QxB2mP9LnR4vS..."

// ❌ NEVER use these for security-sensitive purposes:
// Math.random()  — predictable, seeded from clock
// new Random()   — predictable, seeded from clock
// UUID.randomUUID() is backed by SecureRandom and IS safe for tokens
```

One important performance note: `new SecureRandom()` can be slow on Linux because it may block waiting for the OS entropy pool to fill (reading from `/dev/random`). In modern JVMs (Java 8+), `SecureRandom` defaults to `/dev/urandom` which is non-blocking and cryptographically safe. If you create a `SecureRandom` in a high-throughput context, create it once as a singleton rather than on every call.

---

## MessageDigest — Hash Functions (SHA-256, SHA-512)

A **cryptographic hash function** takes an input of arbitrary length and produces a fixed-length output (the hash or digest). Hash functions have three essential properties. They are **deterministic** — the same input always produces the same output. They are **one-way** — given the output, it is computationally infeasible to reconstruct the input. They are **collision-resistant** — it is computationally infeasible to find two different inputs that produce the same output.

Hash functions are used everywhere in security: verifying file integrity, creating message authentication codes, in digital signature schemes, and (improperly) for password storage.

**SHA-256** produces a 256-bit (32-byte) hash. **SHA-512** produces a 512-bit (64-byte) hash. Both are part of the SHA-2 family and are cryptographically sound for current use. **MD5** and **SHA-1** are broken and must not be used for any security-sensitive purpose — researchers have demonstrated practical collision attacks against both.

```java
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

public class HashingExample {

    // Hash arbitrary data with SHA-256 — returns the raw bytes
    public static byte[] sha256(byte[] data) throws NoSuchAlgorithmException {
        // MessageDigest instances are NOT thread-safe — always create a new instance
        // or use a ThreadLocal pattern for high-performance scenarios
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        return digest.digest(data);  // Single-call convenience method
    }

    // Hash a String and return a hex-encoded string (common for display/storage)
    public static String sha256Hex(String input) throws NoSuchAlgorithmException {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hashBytes = digest.digest(input.getBytes(StandardCharsets.UTF_8));
        // HexFormat (Java 17+) is the cleanest way to convert bytes to hex
        return HexFormat.of().formatHex(hashBytes);
    }

    // Hash a large file incrementally — important for files too big to load into memory
    public static String hashFile(Path filePath) throws Exception {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        // update() can be called multiple times — the digest accumulates all input
        try (InputStream is = Files.newInputStream(filePath)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = is.read(buffer)) != -1) {
                digest.update(buffer, 0, bytesRead);
            }
        }
        return HexFormat.of().formatHex(digest.digest());  // digest() finalizes and resets
    }

    // Verify file integrity by comparing expected and actual hashes
    public static boolean verifyIntegrity(Path file, String expectedSha256Hex) throws Exception {
        String actualHash = hashFile(file);
        // Use MessageDigest.isEqual() for constant-time comparison
        // String.equals() can leak timing information in edge cases
        return MessageDigest.isEqual(
            HexFormat.of().parseHex(expectedSha256Hex),
            HexFormat.of().parseHex(actualHash)
        );
    }
}
```

A critical point about hashing passwords: **never use a plain hash (even SHA-256 or SHA-512) to store passwords**. Password hashes must be slow and memory-hard to defeat brute-force attacks. Use **BCrypt**, **SCrypt**, or **Argon2** instead (covered in Phase 16.1). Plain hashes like SHA-256 are designed to be fast — an attacker with a GPU can compute billions of SHA-256 hashes per second, making a brute-force search of common passwords trivially fast.

---

## HMAC — Message Authentication Codes

A **Message Authentication Code (MAC)** provides both **integrity** and **authentication**. While a plain hash verifies that data has not been corrupted, it provides no authentication — anyone can compute the hash of any data. An HMAC mixes a shared secret key into the hash calculation, so only someone with the key can compute or verify it.

`Mac` in Java with HMAC-SHA256 is the workhorse of API request signing, JWT signatures (when using the HS256 algorithm), webhook verification, and tamper-evident tokens:

```java
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

public class HmacExample {

    // Compute HMAC-SHA256 of a message using a shared secret key
    public static byte[] hmacSha256(byte[] key, byte[] message) throws Exception {
        SecretKeySpec secretKey = new SecretKeySpec(key, "HmacSHA256");
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(secretKey);
        return mac.doFinal(message);
    }

    // Verify a webhook payload — a common real-world use case
    // (GitHub, Stripe, and many other services use HMAC to sign webhooks)
    public static boolean verifyWebhookSignature(
            String payload, String receivedSignature, String webhookSecret)
            throws Exception {

        byte[] key = webhookSecret.getBytes(StandardCharsets.UTF_8);
        byte[] message = payload.getBytes(StandardCharsets.UTF_8);

        byte[] expectedSignature = hmacSha256(key, message);
        byte[] receivedBytes = HexFormat.of().parseHex(
            receivedSignature.replace("sha256=", "")
        );

        // CRITICAL: Use constant-time comparison to prevent timing attacks.
        // A timing attack measures how long the comparison takes — if it
        // exits early on the first mismatch (like String.equals() does),
        // an attacker can infer correct bytes by measuring response time.
        // MessageDigest.isEqual() always runs in time proportional to the
        // length of the arrays, regardless of where the mismatch occurs.
        return MessageDigest.isEqual(expectedSignature, receivedBytes);
    }
}
```

The timing attack prevention is not theoretical — it has been exploited in real systems. Always use `MessageDigest.isEqual()` or `Arrays.equals()` is NOT constant-time; only `MessageDigest.isEqual()` is guaranteed constant-time in the JDK.

---

## Cipher — Symmetric Encryption with AES

**Symmetric encryption** uses the same key to encrypt and decrypt data. **AES (Advanced Encryption Standard)** with 256-bit keys is the gold standard for symmetric encryption and is used everywhere — HTTPS, file encryption, database encryption at rest, and secure storage.

AES itself is a block cipher — it operates on 128-bit blocks. To encrypt data of arbitrary length, you need a **mode of operation**. This is where many developers make critical mistakes.

**ECB (Electronic Codebook) mode is dangerously insecure** — never use it. It encrypts each block independently, so identical plaintext blocks produce identical ciphertext blocks. This leaks structural information about the plaintext (famously demonstrated with the "ECB penguin" — an image of Tux the Linux mascot that remains recognizable after ECB encryption because solid colour regions encrypt to the same ciphertext blocks).

**CBC (Cipher Block Chaining) mode** is better — each block is XORed with the previous ciphertext block before encryption, so identical plaintext blocks produce different ciphertext. However, CBC requires a random Initialization Vector (IV) and is vulnerable to **padding oracle attacks** if error messages are not carefully handled.

**GCM (Galois/Counter Mode)** is the modern, correct choice. It is an **Authenticated Encryption with Associated Data (AEAD)** mode — it simultaneously encrypts the data AND produces an authentication tag that verifies both the ciphertext integrity and any associated unencrypted metadata. This combines confidentiality and integrity in one operation, and it is what you should use in new code.

```java
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;

public class AesGcmEncryption {

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int KEY_SIZE_BITS = 256;
    private static final int GCM_IV_LENGTH_BYTES = 12;   // 96 bits — optimal for GCM
    private static final int GCM_TAG_LENGTH_BITS = 128;  // 128-bit auth tag — maximum strength

    // Generate a new AES-256 key — store this in a Key Management Service, not in code
    public static SecretKey generateKey() throws Exception {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(KEY_SIZE_BITS, new SecureRandom());
        return keyGen.generateKey();
    }

    // Encrypt plaintext with AES-256-GCM
    // Returns the IV prepended to the ciphertext (IV is not secret, just needs to be unique)
    public static byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
        // CRITICAL: A fresh, random IV (nonce) MUST be generated for every single encryption.
        // Reusing the same IV with the same key in GCM mode is catastrophic —
        // it allows an attacker to recover the key and decrypt all messages.
        byte[] iv = new byte[GCM_IV_LENGTH_BYTES];
        new SecureRandom().nextBytes(iv);

        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH_BITS, iv);
        cipher.init(Cipher.ENCRYPT_MODE, key, parameterSpec);

        // Optional: updateAAD() — associate additional non-encrypted data
        // with this ciphertext. The GCM tag will cover both the ciphertext
        // AND the AAD, so if either is tampered with, authentication fails.
        // cipher.updateAAD("user-id:12345".getBytes());

        byte[] ciphertext = cipher.doFinal(plaintext);

        // Prepend the IV to the ciphertext for storage/transmission.
        // The IV is not secret — it just must be unique per encryption.
        // Format: [12 bytes IV][variable-length ciphertext + 16-byte GCM tag]
        byte[] result = new byte[iv.length + ciphertext.length];
        System.arraycopy(iv, 0, result, 0, iv.length);
        System.arraycopy(ciphertext, 0, result, iv.length, ciphertext.length);
        return result;
    }

    // Decrypt ciphertext encrypted by the encrypt() method above
    public static byte[] decrypt(byte[] encryptedData, SecretKey key) throws Exception {
        // Extract the IV from the first 12 bytes
        byte[] iv = Arrays.copyOfRange(encryptedData, 0, GCM_IV_LENGTH_BYTES);
        byte[] ciphertext = Arrays.copyOfRange(encryptedData, GCM_IV_LENGTH_BYTES, encryptedData.length);

        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH_BITS, iv);
        cipher.init(Cipher.DECRYPT_MODE, key, parameterSpec);

        // doFinal() automatically verifies the GCM authentication tag.
        // If the ciphertext or tag has been tampered with, this throws
        // AEADBadTagException — you must catch this and treat it as a security event.
        try {
            return cipher.doFinal(ciphertext);
        } catch (javax.crypto.AEADBadTagException e) {
            throw new SecurityException("Ciphertext authentication failed — data may be tampered", e);
        }
    }

    // Practical example: encrypting sensitive database fields
    public static String encryptField(String plaintext, SecretKey key) throws Exception {
        byte[] encrypted = encrypt(plaintext.getBytes(StandardCharsets.UTF_8), key);
        // Base64-encode for storage in a VARCHAR database column
        return Base64.getEncoder().encodeToString(encrypted);
    }

    public static String decryptField(String encryptedBase64, SecretKey key) throws Exception {
        byte[] encrypted = Base64.getDecoder().decode(encryptedBase64);
        byte[] plaintext = decrypt(encrypted, key);
        return new String(plaintext, StandardCharsets.UTF_8);
    }
}
```

---

## KeyPairGenerator — Asymmetric Encryption with RSA

**Asymmetric (public-key) cryptography** uses a mathematically linked key pair: a **public key** that you share openly, and a **private key** that you keep secret. The fundamental property is that data encrypted with the public key can only be decrypted by the corresponding private key, and vice versa.

RSA is the most widely known asymmetric algorithm. In Java, you use `KeyPairGenerator` to create the key pair. RSA keys should be at least **2048 bits** for current security; **4096 bits** is preferred for long-lived keys. Never use 1024-bit RSA — it is considered broken.

```java
import java.security.*;
import javax.crypto.Cipher;

public class RsaEncryption {

    // Generate a 2048-bit RSA key pair
    public static KeyPair generateKeyPair() throws Exception {
        KeyPairGenerator keyPairGen = KeyPairGenerator.getInstance("RSA");
        keyPairGen.initialize(2048, new SecureRandom());
        return keyPairGen.generateKeyPair();
    }

    // Encrypt a small message with the recipient's PUBLIC key.
    // Use RSA-OAEP padding — NEVER use RSA/ECB/PKCS1Padding (it has known vulnerabilities)
    // IMPORTANT: RSA is only suitable for small data (< ~200 bytes for 2048-bit key).
    // For larger data, use RSA to encrypt a symmetric AES key (hybrid encryption),
    // then encrypt the actual data with AES.
    public static byte[] encryptWithPublicKey(byte[] data, PublicKey publicKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        return cipher.doFinal(data);
    }

    // Decrypt with the PRIVATE key
    public static byte[] decryptWithPrivateKey(byte[] ciphertext, PrivateKey privateKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        return cipher.doFinal(ciphertext);
    }

    // HYBRID ENCRYPTION PATTERN — encrypt large data with RSA
    // Step 1: Generate a random AES key
    // Step 2: Encrypt the data with AES-GCM
    // Step 3: Encrypt the AES key with RSA (only the small key is RSA-encrypted)
    // Step 4: Transmit both the RSA-encrypted AES key and the AES-encrypted data
    public static HybridEncryptedData encryptLargeData(byte[] data, PublicKey recipientPublicKey)
            throws Exception {

        // Generate ephemeral AES key (used only for this message)
        SecretKey aesKey = AesGcmEncryption.generateKey();

        // Encrypt the actual data with AES-GCM
        byte[] encryptedData = AesGcmEncryption.encrypt(data, aesKey);

        // Encrypt the AES key with the recipient's RSA public key
        byte[] encryptedKey = encryptWithPublicKey(aesKey.getEncoded(), recipientPublicKey);

        return new HybridEncryptedData(encryptedKey, encryptedData);
    }
}
```

---

## Digital Signatures — Signature Class

A **digital signature** provides authentication and non-repudiation. The sender signs data with their **private key**; anyone with the sender's **public key** can verify the signature. This proves both that the data came from the claimed sender (authentication) and that it has not been modified (integrity).

Digital signatures are the foundation of TLS certificates, code signing, JWT verification with RS256, and document signing.

```java
import java.security.*;

public class DigitalSignatureExample {

    // Sign a document with the signer's private key
    public static byte[] sign(byte[] data, PrivateKey privateKey) throws Exception {
        // SHA256withRSA is the standard choice for RSA-based signatures
        // For elliptic curve keys, use SHA256withECDSA
        Signature signer = Signature.getInstance("SHA256withRSA");
        signer.initSign(privateKey, new SecureRandom());
        signer.update(data);
        return signer.sign();
    }

    // Verify a signature with the signer's public key
    public static boolean verify(byte[] data, byte[] signature, PublicKey publicKey)
            throws Exception {
        Signature verifier = Signature.getInstance("SHA256withRSA");
        verifier.initVerify(publicKey);
        verifier.update(data);
        // Returns true if the signature is valid, false if it has been tampered with
        return verifier.verify(signature);
    }

    // Practical use case: sign a JSON payload before sending it to a partner
    public static void signedApiExample() throws Exception {
        KeyPair keyPair = RsaEncryption.generateKeyPair();

        String payload = "{\"orderId\":\"12345\",\"amount\":99.99,\"currency\":\"USD\"}";
        byte[] payloadBytes = payload.getBytes(StandardCharsets.UTF_8);

        // Sign the payload
        byte[] signature = sign(payloadBytes, keyPair.getPrivate());
        String signatureBase64 = Base64.getEncoder().encodeToString(signature);

        // The partner verifies with your public key (which you've shared in advance)
        byte[] receivedSignature = Base64.getDecoder().decode(signatureBase64);
        boolean isValid = verify(payloadBytes, receivedSignature, keyPair.getPublic());

        System.out.println("Signature valid: " + isValid);  // true
        // If the payload was modified in transit, isValid would be false
    }
}
```

---

## X.509 Certificates and KeyStore

An **X.509 certificate** binds a public key to an identity (a domain name, organization, or person), and this binding is signed by a trusted **Certificate Authority (CA)**. When your browser connects to `https://example.com`, it receives the server's X.509 certificate, verifies the CA's signature, checks that the certificate's Common Name matches the domain, and then uses the certificate's public key to establish the TLS session.

In Java, certificates and keys are stored in a **KeyStore** — a secure container file (typically `.jks` or the newer `.p12`/`.pfx` PKCS#12 format). The KeyStore holds your private keys (for a server presenting its own certificate) and trusted CA certificates (a TrustStore, for a client verifying server certificates).

```java
import java.security.KeyStore;
import java.security.cert.Certificate;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;

public class CertificateExample {

    // Load a certificate from a PEM file (common format from CAs like Let's Encrypt)
    public static X509Certificate loadCertificate(InputStream pemStream) throws Exception {
        CertificateFactory factory = CertificateFactory.getInstance("X.509");
        return (X509Certificate) factory.generateCertificate(pemStream);
    }

    // Read details from a certificate — useful for validating before use
    public static void inspectCertificate(X509Certificate cert) {
        System.out.println("Subject: " + cert.getSubjectX500Principal().getName());
        System.out.println("Issuer: " + cert.getIssuerX500Principal().getName());
        System.out.println("Valid from: " + cert.getNotBefore());
        System.out.println("Valid until: " + cert.getNotAfter());
        System.out.println("Serial number: " + cert.getSerialNumber().toString(16));

        // Check certificate validity (throws if expired or not yet valid)
        try {
            cert.checkValidity();
            System.out.println("Certificate is currently valid.");
        } catch (Exception e) {
            System.out.println("Certificate is INVALID: " + e.getMessage());
        }
    }

    // Load a KeyStore from a file (your server's certificate + private key)
    public static KeyStore loadKeyStore(String keystorePath, char[] password) throws Exception {
        KeyStore keyStore = KeyStore.getInstance("PKCS12");  // Prefer PKCS12 over the legacy JKS
        try (FileInputStream fis = new FileInputStream(keystorePath)) {
            keyStore.load(fis, password);
        }
        return keyStore;
    }
}
```

---

## TLS/SSL — SSLContext, TrustManager, KeyManager

**TLS (Transport Layer Security)** is the protocol that secures HTTPS, SMTPS, LDAPS, and essentially all encrypted network communication. Understanding how Java configures TLS is important for both building servers that serve TLS and writing clients that connect to TLS endpoints.

The key concepts are the **KeyManager** (manages your local private key and certificate, used when your application presents a certificate — i.e., acting as a server, or in mutual TLS) and the **TrustManager** (decides which remote certificates to trust — i.e., which CAs are trusted when acting as a client).

```java
import javax.net.ssl.*;

public class TlsConfiguration {

    // Configure HTTPS for an embedded Jetty/Tomcat server (Spring Boot does this via application.yml,
    // but understanding the underlying API is valuable)
    public static SSLContext createServerSslContext(
            String keystorePath, char[] keystorePassword) throws Exception {

        // Load the server's certificate and private key from a PKCS12 file
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream(keystorePath)) {
            keyStore.load(fis, keystorePassword);
        }

        // KeyManagerFactory manages the server's own identity (private key + certificate)
        KeyManagerFactory kmf = KeyManagerFactory.getInstance(
            KeyManagerFactory.getDefaultAlgorithm());  // Default: "SunX509"
        kmf.init(keyStore, keystorePassword);

        // Build the SSLContext — use TLSv1.3 or at minimum TLSv1.2
        SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
        sslContext.init(
            kmf.getKeyManagers(),   // Server's key material
            null,                   // null = use the JVM's default trust store (cacerts)
            new SecureRandom()
        );
        return sslContext;
    }

    // For Java 11+ HTTP Client — configure which certificates to trust
    public static HttpClient createSecureHttpClient(String trustStorePath, char[] trustStorePassword)
            throws Exception {

        // Load the custom trust store (used for internal services with self-signed certs
        // or a private CA — NOT for public internet connections where the JVM's built-in
        // cacerts file is sufficient)
        KeyStore trustStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream(trustStorePath)) {
            trustStore.load(fis, trustStorePassword);
        }

        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
            TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(trustStore);

        SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
        sslContext.init(null, tmf.getTrustManagers(), new SecureRandom());

        return HttpClient.newBuilder()
            .sslContext(sslContext)
            .build();
    }

    // ❌ WARNING: NEVER DO THIS IN PRODUCTION — disabling certificate validation
    // This is sometimes done "temporarily" by developers who can't get certs working,
    // and then it makes it to production. It makes TLS completely useless.
    public static SSLContext createInsecureContext_DO_NOT_USE() throws Exception {
        TrustManager[] trustAllCerts = new TrustManager[]{
            new X509TrustManager() {
                public void checkClientTrusted(java.security.cert.X509Certificate[] c, String a) {}
                public void checkServerTrusted(java.security.cert.X509Certificate[] c, String a) {}
                public java.security.cert.X509Certificate[] getAcceptedIssuers() { return new java.security.cert.X509Certificate[]{}; }
            }
        };
        SSLContext ctx = SSLContext.getInstance("TLS");
        ctx.init(null, trustAllCerts, new SecureRandom());
        return ctx;
        // This makes your HTTPS connection completely vulnerable to man-in-the-middle attacks.
        // Use it ONLY in test environments with no real data, and remove it before going to production.
    }
}
```

For Spring Boot, TLS configuration is typically done via `application.yml` rather than directly:

```yaml
# application.yml — enabling HTTPS in Spring Boot
server:
  port: 8443
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: myapp
    enabled: true
    # Enforce TLS 1.2 or higher — disable old, vulnerable protocols
    protocol: TLS
    enabled-protocols: TLSv1.2,TLSv1.3
    # Disable weak cipher suites
    ciphers:
      - TLS_AES_256_GCM_SHA384
      - TLS_AES_128_GCM_SHA256
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

---

## Elliptic Curve Cryptography (ECC)

For new systems, **Elliptic Curve Cryptography (ECC)** using the **P-256** or **P-384** curves provides equivalent security to RSA with much smaller keys. A 256-bit EC key is roughly equivalent in security to a 3072-bit RSA key, which means smaller certificates, faster handshakes, and less computational overhead. Modern TLS 1.3 uses ECC by default.

```java
// Generate an EC key pair for use with ECDSA signatures or ECDH key exchange
KeyPairGenerator keyPairGen = KeyPairGenerator.getInstance("EC");
keyPairGen.initialize(new ECGenParameterSpec("secp256r1"), new SecureRandom());
// "secp256r1" is the standard P-256 curve (same as prime256v1 in OpenSSL)
KeyPair ecKeyPair = keyPairGen.generateKeyPair();

// Sign with ECDSA
Signature ecSigner = Signature.getInstance("SHA256withECDSA");
ecSigner.initSign(ecKeyPair.getPrivate(), new SecureRandom());
ecSigner.update("data to sign".getBytes(StandardCharsets.UTF_8));
byte[] signature = ecSigner.sign();
```

---

## Best Practices Summary

The most important principle in applied cryptography is to use well-established, peer-reviewed algorithms and libraries — never invent your own. "Rolling your own crypto" is an extremely common source of critical vulnerabilities, even among experienced developers, because the subtle interactions between primitives are easy to get wrong.

Use **AES-256-GCM** for symmetric encryption. Use **RSA-2048 with OAEP padding** or **ECDSA with P-256** for asymmetric operations. Use **SHA-256** or **SHA-512** for hashing. Use **BCrypt, SCrypt, or Argon2** for passwords. Use **HMAC-SHA256** for message authentication. These choices are safe, well-supported, and will not embarrass you in a security audit.

Store keys in a **Key Management Service (KMS)** — AWS KMS, GCP Cloud KMS, HashiCorp Vault, or a Hardware Security Module (HSM) for the highest security requirements. A key that is not properly protected provides no real security guarantee for the data it encrypts. Rotate keys periodically — the more data a single key encrypts, the more valuable it becomes to an attacker. Implement key versioning so you can decrypt old data with old keys while encrypting new data with new keys.

Never hardcode secrets, passwords, or cryptographic keys in your source code. This is one of the single most common security mistakes, and it is especially catastrophic when the code is open-source or when the repository history is exposed. Treat a committed secret as permanently compromised — rotate it immediately and audit what was accessible with it.
