---
title: "[인증과 보안] 6편 — 데이터 보안: 저장과 전송의 암호화"
date: 2026-03-17T22:02:00+09:00
draft: false
tags: ["암호화", "bcrypt", "AES", "KMS", "PII", "보안", "서버"]
series: ["인증과 보안"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 4
summary: "비밀번호 해싱(bcrypt/Argon2)과 레인보우 테이블 방어, 데이터 암호화 at rest(AES-256)·at transit(TLS), 필드 레벨 암호화와 KMS 키 관리, PII 처리 원칙과 마스킹/토크나이제이션까지"
---

데이터베이스가 털렸다. 공격자는 수백만 건의 사용자 레코드를 손에 넣었다.

이 순간, 그 데이터가 얼마나 오래 버티느냐는 전적으로 암호화 설계의 품질에 달려 있다. `password123`을 MD5로 해싱한 `482c811da5d5b4bc6d497ffa98491e38`는 레인보우 테이블 한 번 조회로 끝난다. 반면 적절한 salt와 충분한 cost factor로 bcrypt 처리된 해시는 GPU 클러스터를 동원해도 수십 년이 걸린다.

데이터 보안은 "침해가 일어나지 않도록" 하는 것만이 아니라, "침해가 일어났을 때 피해를 최소화"하는 심층 방어(defense-in-depth)다.

이 글에서는 비밀번호 해싱부터 전송 구간 암호화, 필드 레벨 암호화, KMS 기반 키 관리, PII 처리 원칙까지 서버 엔지니어가 실전에서 직면하는 데이터 보안 전반을 다룬다.

---

## 1. 비밀번호 해싱: bcrypt와 Argon2

비밀번호는 절대 평문으로 저장하지 않는다. 이것은 상식이지만, "어떤 해싱 알고리즘을 쓰느냐"에서 엄청난 차이가 난다.

### 왜 일반 해시 함수(SHA-256, MD5)는 안 되는가

SHA-256은 초당 수십억 번 연산이 가능하도록 설계된 **빠른** 해시 함수다. 이 성질이 비밀번호 저장에는 치명적이다. 현대 GPU는 SHA-256을 초당 수백억 번 계산할 수 있어, 8자리 영숫자 조합 전체를 몇 초 만에 브루트포스할 수 있다.

**레인보우 테이블(Rainbow Table)** 공격은 미리 계산된 해시-평문 쌍을 대규모로 구축해 역조회하는 기법이다. Salt 없이 저장된 `MD5("password") = 5f4dcc3b5aa765d61d8327deb882cf99` 같은 값은 공개 레인보우 테이블에서 즉시 찾을 수 있다.

**Salt**는 각 비밀번호에 무작위 값을 덧붙여 동일한 평문도 다른 해시값을 갖게 만든다. 레인보우 테이블을 무력화하는 핵심 방어 수단이다. 하지만 SHA-256 + salt 조합도 여전히 너무 빠르다. 필요한 것은 **의도적으로 느린** 해시 함수다.

### bcrypt

bcrypt는 1999년 Niels Provos와 David Mazières가 설계한 비밀번호 해싱 알고리즘이다. **cost factor(work factor)**를 통해 연산 비용을 조절할 수 있으며, 하드웨어가 빨라질수록 cost를 올려 동일한 보안 수준을 유지할 수 있다.

```java
// Spring Security BCryptPasswordEncoder 사용
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

@Component
public class PasswordService {

    // strength(cost factor): 기본값 10, 권장 12~14
    private final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);

    public String hashPassword(String rawPassword) {
        return encoder.encode(rawPassword);
        // 결과 예시: $2a$12$K9L3eXXXXXXXXXXXXXXXXueVQ7K8Z8...
        // 구조: $2a$ (알고리즘) $12$ (cost) $salt(22chars) hash(31chars)
    }

    public boolean verifyPassword(String rawPassword, String hashedPassword) {
        return encoder.matches(rawPassword, hashedPassword);
    }
}
```

bcrypt 해시 문자열 `$2a$12$...`에는 알고리즘 버전, cost factor, salt가 모두 포함된다. 별도로 salt를 저장할 필요가 없다.

**cost factor 선택 기준**: 로그인 요청 처리에 약 100~300ms가 걸리는 값을 선택한다. 서버 스펙에 따라 다르지만, 2024년 기준 일반 서버에서는 12~13이 적절하다. 너무 높으면 정상 사용자 경험이 나빠지고 DoS 공격 표면이 넓어진다.

```java
// cost factor 벤치마크
@Component
public class BcryptBenchmark {

    public void benchmark() {
        for (int cost = 10; cost <= 15; cost++) {
            BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(cost);
            long start = System.currentTimeMillis();
            encoder.encode("testpassword");
            long elapsed = System.currentTimeMillis() - start;
            System.out.printf("cost=%d: %dms%n", cost, elapsed);
        }
    }
}
// 출력 예시 (서버 성능마다 다름):
// cost=10: 87ms
// cost=11: 174ms
// cost=12: 348ms  <-- 적절한 범위
// cost=13: 697ms
// cost=14: 1394ms
```

### Argon2: 현대적 대안

Argon2는 2015년 Password Hashing Competition(PHC) 우승 알고리즘이다. bcrypt의 한계(64바이트 비밀번호 제한, GPU 병렬화에 상대적으로 취약)를 개선했다. 세 가지 변형이 있다.

- **Argon2d**: GPU 공격에 강함, 사이드채널에 취약. 암호화폐 등에 적합.
- **Argon2i**: 사이드채널 공격에 강함. 비밀번호 해싱의 초기 권장안.
- **Argon2id**: 두 방식의 하이브리드. **현재 가장 권장되는 변형.**

```java
// Spring Security 6.x + Argon2PasswordEncoder
import org.springframework.security.crypto.argon2.Argon2PasswordEncoder;

@Component
public class PasswordServiceV2 {

    // saltLength=16, hashLength=32, parallelism=1, memory=65536(64KB), iterations=3
    private final Argon2PasswordEncoder encoder =
        new Argon2PasswordEncoder(16, 32, 1, 65536, 3);

    public String hashPassword(String rawPassword) {
        return encoder.encode(rawPassword);
    }

    public boolean verifyPassword(String rawPassword, String stored) {
        return encoder.matches(rawPassword, stored);
    }
}
```

**파라미터 선택 가이드 (OWASP 2023 기준)**:
- memory: 최소 19MB (19456 KB), 권장 64MB
- iterations: 2~3
- parallelism: 서버 CPU 코어 수에 맞게

### 비밀번호 해싱 안티패턴

```java
// 안티패턴 1: MD5/SHA 단순 해싱
String hash = DigestUtils.md5Hex(password); // 절대 금지

// 안티패턴 2: SHA + 고정 salt (레인보우 테이블 부분 무력화 가능)
String hash = DigestUtils.sha256Hex("fixedsalt" + password); // 금지

// 안티패턴 3: 평문 비교 (timing attack 취약)
if (storedHash.equals(computedHash)) { ... } // matches() 사용할 것

// 안티패턴 4: cost factor 1~4 (너무 낮음)
new BCryptPasswordEncoder(4); // 개발환경 전용으로만

// 안티패턴 5: 비밀번호를 암호화(복호화 가능)해서 저장
// 비밀번호는 검증만 필요하지 원본 복구가 필요 없음
// 단방향 해시를 사용해야 함
```

---

## 2. 데이터 암호화 at rest와 at transit

### Encryption at Rest: AES-256

저장된 데이터의 암호화는 여러 계층에서 적용할 수 있다.

**계층 1: 디스크/스토리지 레벨**

AWS RDS, S3, EBS는 기본적으로 AES-256 암호화를 제공한다. 데이터베이스 인스턴스를 생성할 때 암호화를 활성화하면 스토리지 계층 전체가 암호화된다. 이 방식은 물리적 디스크 탈취 시나리오를 방어하지만, DB 접근 권한이 있는 공격자는 여전히 평문 데이터를 읽을 수 있다.

```bash
# AWS RDS 암호화 활성화 (Terraform 예시)
resource "aws_db_instance" "main" {
  engine         = "mysql"
  instance_class = "db.t3.medium"
  storage_encrypted = true          # AES-256 활성화
  kms_key_id    = aws_kms_key.rds.arn  # CMK 지정
  # ...
}
```

**계층 2: 애플리케이션 레벨 (AES-256-GCM)**

AES-GCM(Galois/Counter Mode)은 기밀성과 무결성을 동시에 제공하는 인증 암호화(Authenticated Encryption) 방식이다. AES-CBC와 달리 패딩 오라클 공격에 안전하고 병렬 처리가 가능하다.

```java
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.SecureRandom;
import java.util.Base64;

@Component
public class AesGcmEncryptionService {

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;  // 96 bits (권장)
    private static final int GCM_TAG_LENGTH = 128; // 128 bits (최대)

    /**
     * AES-256-GCM 암호화
     * @param plaintext 평문 데이터
     * @param key 256-bit (32 bytes) AES 키
     * @return Base64(IV + CipherText + AuthTag)
     */
    public String encrypt(String plaintext, SecretKey key) throws Exception {
        // IV는 매 암호화마다 새로 생성해야 함 (절대 재사용 금지)
        byte[] iv = new byte[GCM_IV_LENGTH];
        new SecureRandom().nextBytes(iv);

        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.ENCRYPT_MODE, key, parameterSpec);

        byte[] cipherText = cipher.doFinal(plaintext.getBytes("UTF-8"));

        // IV를 암호문 앞에 붙여서 함께 저장 (IV는 비밀이 아님)
        byte[] combined = new byte[iv.length + cipherText.length];
        System.arraycopy(iv, 0, combined, 0, iv.length);
        System.arraycopy(cipherText, 0, combined, iv.length, cipherText.length);

        return Base64.getEncoder().encodeToString(combined);
    }

    /**
     * AES-256-GCM 복호화
     */
    public String decrypt(String encryptedData, SecretKey key) throws Exception {
        byte[] combined = Base64.getDecoder().decode(encryptedData);

        byte[] iv = new byte[GCM_IV_LENGTH];
        byte[] cipherText = new byte[combined.length - GCM_IV_LENGTH];

        System.arraycopy(combined, 0, iv, 0, iv.length);
        System.arraycopy(combined, iv.length, cipherText, 0, cipherText.length);

        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.DECRYPT_MODE, key, parameterSpec);

        byte[] plainText = cipher.doFinal(cipherText);
        return new String(plainText, "UTF-8");
    }

    /**
     * AES-256 키 생성
     */
    public SecretKey generateKey() throws Exception {
        KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
        keyGenerator.init(256, new SecureRandom());
        return keyGenerator.generateKey();
    }
}
```

### Encryption at Transit: TLS

네트워크를 통한 데이터 전송은 TLS로 보호해야 한다. 현재 권장 버전은 **TLS 1.3**이다. TLS 1.0, 1.1은 2020년 이후 IETF에 의해 deprecated됐고, TLS 1.2도 설정에 따라 취약한 암호 스위트를 사용할 수 있다.

```yaml
# Spring Boot application.yml - TLS 설정
server:
  ssl:
    enabled: true
    protocol: TLS
    enabled-protocols: TLSv1.3,TLSv1.2  # 1.0/1.1 비활성화
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    ciphers:
      # TLS 1.3 cipher suites (자동 선택)
      # TLS 1.2 안전한 cipher suites만 허용
      - TLS_AES_128_GCM_SHA256
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

**내부 서비스 간 통신도 TLS를 적용해야 한다.** 서비스 메시(Istio, Linkerd)를 사용하는 Kubernetes 환경에서는 mTLS(상호 TLS)로 서비스 간 인증까지 처리할 수 있다.

```java
// RestTemplate TLS 설정 예시 (내부 서비스 호출)
@Configuration
public class HttpClientConfig {

    @Bean
    public RestTemplate secureRestTemplate() throws Exception {
        SSLContext sslContext = SSLContextBuilder.create()
            .loadTrustMaterial(trustStore, trustStorePassword)
            .loadKeyMaterial(keyStore, keyStorePassword, keyPassword)
            .build();

        HttpClient httpClient = HttpClients.custom()
            .setSSLContext(sslContext)
            .setSSLHostnameVerifier(SSLConnectionSocketFactory.getDefaultHostnameVerifier())
            .build();

        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(httpClient);

        return new RestTemplate(factory);
    }
}
```

**전송 계층 암호화 안티패턴**:

```java
// 안티패턴 1: SSL 검증 비활성화 (절대 금지 - 프로덕션에서)
TrustManager[] trustAllCerts = new TrustManager[]{
    new X509TrustManager() {
        public void checkClientTrusted(X509Certificate[] c, String a) {}
        public void checkServerTrusted(X509Certificate[] c, String a) {}
        public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
    }
};
// 이 코드는 MITM 공격에 완전히 노출됨

// 안티패턴 2: HTTP로 민감 정보 전송
// RestTemplate.getForObject("http://...", ...) // 반드시 https://

// 안티패턴 3: TLS 1.0/1.1 허용
// enabled-protocols: TLSv1,TLSv1.1,TLSv1.2 // 1.0, 1.1 제거할 것
```

---

## 3. 필드 레벨 암호화와 암호화 키 관리

### 필드 레벨 암호화가 필요한 이유

DB 레벨 암호화(TDE, Transparent Data Encryption)는 물리적 탈취를 막지만, DB 접근 권한을 가진 내부자나 애플리케이션을 통한 침해에는 무력하다. 주민등록번호, 카드번호, 의료기록처럼 특별히 민감한 데이터는 **애플리케이션 레벨에서 필드별로 암호화**해 저장해야 한다.

```java
// JPA Converter를 이용한 투명한 필드 레벨 암호화
@Converter
@Component
public class SensitiveDataConverter implements AttributeConverter<String, String> {

    @Autowired
    private EncryptionService encryptionService;

    @Override
    public String convertToDatabaseColumn(String attribute) {
        if (attribute == null) return null;
        try {
            return encryptionService.encrypt(attribute);
        } catch (Exception e) {
            throw new RuntimeException("암호화 실패", e);
        }
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        try {
            return encryptionService.decrypt(dbData);
        } catch (Exception e) {
            throw new RuntimeException("복호화 실패", e);
        }
    }
}

@Entity
public class UserProfile {

    @Id
    private Long id;

    private String email; // 암호화 안 함 (검색 필요)

    @Convert(converter = SensitiveDataConverter.class)
    @Column(name = "resident_number")
    private String residentNumber; // 주민등록번호: 암호화

    @Convert(converter = SensitiveDataConverter.class)
    @Column(name = "phone_number")
    private String phoneNumber; // 전화번호: 암호화
}
```

### 검색 가능한 암호화 (Searchable Encryption)

필드 레벨 암호화의 함정은 **검색이 불가능해진다**는 점이다. AES-GCM은 동일 평문도 암호화할 때마다 다른 결과를 내므로 `WHERE email = ?` 같은 쿼리가 작동하지 않는다.

해결책:
1. **결정론적 암호화(Deterministic Encryption)**: 동일 입력 → 동일 출력. AES-SIV 또는 HMAC 기반 토큰을 별도 컬럼에 저장. 동등 비교(exact match)는 가능하지만 범위 검색 불가. 패턴 분석 취약.
2. **검색 전용 HMAC 인덱스**: 평문의 HMAC 값을 별도 컬럼에 저장하고 검색에 활용.
3. **암호화 전 정규화**: 검색 전에 평문을 정규화(소문자 변환, 공백 제거)해 일관성 확보.

```java
@Entity
public class UserProfile {

    @Convert(converter = SensitiveDataConverter.class)
    @Column(name = "email_encrypted")
    private String email; // 복호화용 암호문

    @Column(name = "email_hmac")
    private String emailHmac; // 검색용 HMAC (별도 컬럼)

    public void setEmail(String email, HmacService hmacService) {
        this.email = email.toLowerCase().trim(); // 정규화 후 암호화
        this.emailHmac = hmacService.compute(email.toLowerCase().trim()); // HMAC 저장
    }
}

// 이메일로 검색
@Repository
public interface UserRepository extends JpaRepository<UserProfile, Long> {
    Optional<UserProfile> findByEmailHmac(String emailHmac);
}

// 서비스 계층
public Optional<UserProfile> findByEmail(String email) {
    String hmac = hmacService.compute(email.toLowerCase().trim());
    return userRepository.findByEmailHmac(hmac);
}
```

### KMS (Key Management Service)와 키 관리

암호화 키는 어디에 저장해야 하는가? 이것이 키 관리의 핵심 질문이다. 암호화 키를 데이터와 같은 서버에 저장하거나 코드에 하드코딩하는 것은 잠금장치 열쇠를 문 앞에 두는 것과 같다.

**Envelope Encryption (봉투 암호화)** 패턴이 표준이다.

- **DEK(Data Encryption Key)**: 실제 데이터를 암호화하는 키. 매 레코드 또는 배치마다 새로 생성.
- **KEK(Key Encryption Key) / Master Key**: DEK를 암호화하는 키. KMS에만 존재하며 외부로 나오지 않음.

```
[평문 데이터] → AES-256-GCM(DEK) → [암호문]
[DEK]         → KMS.Encrypt(KEK) → [암호화된 DEK]
DB 저장: [암호문] + [암호화된 DEK]

복호화:
[암호화된 DEK] → KMS.Decrypt(KEK) → [DEK]
[암호문]        → AES-256-GCM(DEK) → [평문 데이터]
```

```java
// AWS KMS 기반 Envelope Encryption
@Service
public class KmsEnvelopeEncryptionService {

    private final KmsClient kmsClient;
    private final String masterKeyId; // ARN of KMS CMK

    public EncryptedData encrypt(String plaintext) throws Exception {
        // 1. KMS로부터 DEK 생성 (평문 DEK + 암호화된 DEK 동시 수신)
        GenerateDataKeyRequest dataKeyRequest = GenerateDataKeyRequest.builder()
            .keyId(masterKeyId)
            .keySpec(DataKeySpec.AES_256)
            .build();

        GenerateDataKeyResponse dataKeyResponse = kmsClient.generateDataKey(dataKeyRequest);

        SecretKey dek = new SecretKeySpec(
            dataKeyResponse.plaintext().asByteArray(), "AES");

        // 2. DEK로 데이터 암호화 (로컬에서 수행, 빠름)
        String encryptedData = aesGcmService.encrypt(plaintext, dek);

        // 3. 평문 DEK는 즉시 메모리에서 제거
        // dataKeyResponse.plaintext() 는 SdkBytes, GC 대기
        Arrays.fill(dataKeyResponse.plaintext().asByteArray(), (byte) 0);

        // 4. 암호화된 DEK를 데이터와 함께 저장
        String encryptedDek = Base64.getEncoder().encodeToString(
            dataKeyResponse.ciphertextBlob().asByteArray());

        return new EncryptedData(encryptedData, encryptedDek);
    }

    public String decrypt(EncryptedData encryptedData) throws Exception {
        // 1. KMS로 암호화된 DEK 복호화
        byte[] encryptedDekBytes = Base64.getDecoder().decode(encryptedData.getEncryptedDek());

        DecryptRequest decryptRequest = DecryptRequest.builder()
            .ciphertextBlob(SdkBytes.fromByteArray(encryptedDekBytes))
            .keyId(masterKeyId) // 선택적이지만 명시 권장
            .build();

        DecryptResponse decryptResponse = kmsClient.decrypt(decryptRequest);

        SecretKey dek = new SecretKeySpec(
            decryptResponse.plaintext().asByteArray(), "AES");

        // 2. DEK로 데이터 복호화
        return aesGcmService.decrypt(encryptedData.getCiphertext(), dek);
    }
}
```

### 키 로테이션 (Key Rotation)

암호화 키는 주기적으로 교체해야 한다. 키가 장기간 동일하면 노출 리스크가 누적된다.

**자동 키 로테이션 (AWS KMS)**:

```bash
# CMK 자동 로테이션 활성화 (연간 자동 교체)
aws kms enable-key-rotation --key-id arn:aws:kms:region:account:key/key-id
```

AWS KMS는 자동 로테이션 시 이전 키 버전을 유지해 기존 데이터 복호화를 지원한다. 새 데이터는 최신 키 버전으로 암호화된다. 하지만 이것은 KEK(마스터 키) 로테이션이다.

**DEK 로테이션**: 기존 레코드를 새 DEK로 재암호화하려면 애플리케이션 레벨의 마이그레이션이 필요하다.

```java
// DEK 로테이션 배치 처리
@Component
public class DekRotationJob {

    @Scheduled(cron = "0 0 2 * * SUN") // 매주 일요일 새벽 2시
    public void rotateDeks() {
        // 오래된 DEK로 암호화된 레코드 조회
        List<SensitiveRecord> oldRecords = repository.findByDekVersionBefore(
            LocalDate.now().minusMonths(6));

        for (SensitiveRecord record : oldRecords) {
            try {
                // 현재 DEK로 복호화 후 새 DEK로 재암호화
                String plaintext = decrypt(record);
                EncryptedData newEncrypted = encrypt(plaintext);
                record.updateEncryptedData(newEncrypted);
                repository.save(record);
            } catch (Exception e) {
                log.error("DEK 로테이션 실패: recordId={}", record.getId(), e);
                // 실패해도 계속 진행 (이미 성공한 레코드 롤백 방지)
            }
        }
    }
}
```

**키 관리 안티패턴**:

```java
// 안티패턴 1: 코드에 하드코딩
private static final String SECRET_KEY = "mySecretKey12345"; // 절대 금지

// 안티패턴 2: 환경변수에 평문 키 (OS 레벨 노출 위험)
// AES_KEY=base64encodedkey123... // Secrets Manager 사용 권장

// 안티패턴 3: 키와 암호화 데이터를 같은 DB 테이블에 평문 저장
// CREATE TABLE secrets (data BLOB, encryption_key VARCHAR(256));

// 안티패턴 4: 단일 키로 모든 데이터 암호화 (키 노출 시 전체 노출)
// 데이터 분류에 따라 별도 키 사용 권장

// 안티패턴 5: IV 재사용 (AES-GCM에서 치명적)
private static final byte[] FIXED_IV = new byte[12]; // 절대 금지
// GCM에서 같은 키+IV 조합 재사용 시 인증 완전 붕괴
```

---

## 4. PII 데이터 처리 원칙과 마스킹/토크나이제이션

### PII(개인식별정보)란

PII(Personally Identifiable Information)는 특정 개인을 식별할 수 있는 모든 정보다. 한국의 개인정보보호법, GDPR(유럽), CCPA(캘리포니아) 등 각국 규제가 처리 방식을 강제한다.

**PII 분류 예시**:

| 구분 | 예시 | 처리 기준 |
|------|------|-----------|
| 직접 식별자 | 이름, 주민등록번호, 여권번호 | 최고 수준 보호, 최소화 |
| 간접 식별자 | 이메일, 전화번호, IP 주소 | 높은 수준 보호 |
| 민감 정보 | 의료기록, 금융정보, 생체정보 | 특별 보호 |
| 준식별자 | 생년월일, 우편번호, 직업 | 결합 시 위험, 가명처리 |

**데이터 최소화 원칙**: 서비스에 필요하지 않은 PII는 수집하지 않는다. 수집된 PII는 목적 달성 후 즉시 삭제하거나 익명화한다.

### 데이터 마스킹

마스킹은 원본 데이터의 일부를 숨겨 식별 불가능하게 만드는 기법이다. 로그 출력, 응답 데이터, 내부 도구 화면 등에 활용한다.

```java
@Component
public class PiiMaskingService {

    // 주민등록번호: 901231-1****** → 901231-1******
    public String maskResidentNumber(String residentNumber) {
        if (residentNumber == null || residentNumber.length() < 7) return "***";
        return residentNumber.substring(0, 8) + "******";
    }

    // 전화번호: 010-1234-5678 → 010-****-5678
    public String maskPhoneNumber(String phoneNumber) {
        if (phoneNumber == null) return "***";
        return phoneNumber.replaceAll("(\\d{3})-?(\\d{3,4})-?(\\d{4})",
            "$1-****-$3");
    }

    // 이메일: user@example.com → u***@example.com
    public String maskEmail(String email) {
        if (email == null || !email.contains("@")) return "***";
        String[] parts = email.split("@");
        String localPart = parts[0];
        String domain = parts[1];
        if (localPart.length() <= 1) return email;
        return localPart.charAt(0) +
               "*".repeat(localPart.length() - 1) + "@" + domain;
    }

    // 카드번호: 1234-5678-9012-3456 → 1234-****-****-3456
    public String maskCardNumber(String cardNumber) {
        if (cardNumber == null) return "***";
        String digits = cardNumber.replaceAll("[^0-9]", "");
        if (digits.length() < 8) return "***";
        return digits.substring(0, 4) + "-****-****-" +
               digits.substring(digits.length() - 4);
    }
}
```

**로그에서 PII 자동 마스킹** (Logback 필터 예시):

```java
// Logback Converter로 로그 메시지 내 PII 마스킹
public class PiiMaskingConverter extends ClassicConverter {

    private static final Pattern PHONE_PATTERN =
        Pattern.compile("(01[0-9])-?(\\d{3,4})-?(\\d{4})");
    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}");
    private static final Pattern RESIDENT_PATTERN =
        Pattern.compile("(\\d{6})-?(\\d{7})");

    @Override
    public String convert(ILoggingEvent event) {
        String message = event.getFormattedMessage();
        message = PHONE_PATTERN.matcher(message).replaceAll("$1-****-$3");
        message = EMAIL_PATTERN.matcher(message).replaceAll("[MASKED_EMAIL]");
        message = RESIDENT_PATTERN.matcher(message).replaceAll("$1-*******");
        return message;
    }
}
```

```xml
<!-- logback-spring.xml -->
<configuration>
    <conversionRule conversionWord="piiMasked"
        converterClass="com.example.logging.PiiMaskingConverter"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss} [%thread] %-5level %logger - %piiMasked%n</pattern>
        </encoder>
    </appender>
</configuration>
```

### 토크나이제이션 (Tokenization)

토크나이제이션은 민감 데이터를 **의미 없는 토큰(token)**으로 대체하고, 원본 데이터는 별도의 안전한 토큰 저장소(Token Vault)에 보관하는 기법이다. 마스킹과 달리 토큰으로 원본 데이터를 역조회할 수 있다.

**마스킹 vs 토크나이제이션 비교**:

| 특성 | 마스킹 | 토크나이제이션 |
|------|--------|----------------|
| 원본 복원 | 불가 | 가능 (Vault 통해서) |
| 포맷 보존 | 가능 | 가능 (FPE 사용 시) |
| 검색 | 불가 | 가능 (토큰으로) |
| PCI-DSS 적합 | 부분적 | 적합 |
| 주요 용도 | 로그, 화면 표시 | 결제 정보, DB 저장 |

```java
// 간단한 토크나이제이션 구현 (프로덕션은 전용 Vault 서비스 권장)
@Service
public class TokenizationService {

    private final TokenVaultRepository tokenVault; // 토큰-원본 매핑 저장소
    private final SecureRandom secureRandom = new SecureRandom();

    /**
     * 민감 데이터를 토큰으로 교체
     */
    public String tokenize(String sensitiveData, TokenType type) {
        // 이미 토크나이즈된 데이터는 동일 토큰 반환 (멱등성)
        return tokenVault.findTokenBySensitiveData(sensitiveData, type)
            .orElseGet(() -> {
                String token = generateToken(type);
                tokenVault.save(new TokenMapping(token, sensitiveData, type));
                return token;
            });
    }

    /**
     * 토큰으로 원본 데이터 조회
     */
    public Optional<String> detokenize(String token) {
        // 이 API는 극히 제한된 권한에서만 호출 가능해야 함
        return tokenVault.findSensitiveDataByToken(token);
    }

    private String generateToken(TokenType type) {
        return switch (type) {
            case CARD_NUMBER -> generateFpeToken(16); // 포맷 보존 (숫자 16자리)
            case PHONE_NUMBER -> generateFpeToken(11); // 포맷 보존 (숫자 11자리)
            default -> UUID.randomUUID().toString(); // 범용 토큰
        };
    }

    // Format-Preserving Encryption 토큰 (숫자 형식 유지)
    private String generateFpeToken(int length) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < length; i++) {
            sb.append(secureRandom.nextInt(10));
        }
        return sb.toString();
    }
}

// 결제 서비스에서 토크나이제이션 활용
@Service
public class PaymentService {

    public PaymentResult processPayment(PaymentRequest request) {
        // 카드번호를 즉시 토큰화
        String cardToken = tokenizationService.tokenize(
            request.getCardNumber(), TokenType.CARD_NUMBER);

        // 이후 모든 처리는 토큰으로
        Order order = createOrder(request, cardToken);

        // PG사 결제 시에만 원본 카드번호 역조회 (권한 체크 포함)
        if (needActualCardNumber(order)) {
            String actualCard = tokenizationService.detokenize(cardToken)
                .orElseThrow(() -> new IllegalStateException("토큰 오류"));
            return pgService.charge(actualCard, order.getAmount());
        }

        return PaymentResult.success(order.getId());
    }
}
```

### 데이터 보존 기간과 삭제

개인정보보호법은 목적 달성 후 PII의 즉시 파기를 요구한다. "파기"는 단순 DELETE가 아닌 복구 불가능한 삭제를 의미한다.

```java
// 개인정보 파기 서비스
@Service
public class PersonalDataDeletionService {

    @Transactional
    public void deleteUserData(Long userId) {
        UserProfile profile = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));

        // 1. 암호화 키 삭제 (키 삭제 = 데이터 접근 불가 = 사실상 삭제)
        kmsService.scheduleKeyDeletion(profile.getDekId(), 7); // 7일 후 KMS 키 삭제

        // 2. 민감 필드 덮어쓰기 (DB에서 즉시 복구 불가)
        profile.anonymize();
        userRepository.save(profile);

        // 3. 감사 로그 (삭제 사실만 기록, PII 미포함)
        auditLog.record(AuditEvent.USER_DATA_DELETED, userId);
    }
}

@Entity
public class UserProfile {

    public void anonymize() {
        this.residentNumber = null;
        this.phoneNumber = null;
        this.email = "deleted_" + this.id + "@deleted.invalid"; // 고유성 유지용
        this.name = "탈퇴한 사용자";
        this.deletedAt = LocalDateTime.now();
    }
}
```

### PII 처리 안티패턴

```java
// 안티패턴 1: 불필요한 PII 수집
// 이벤트 참가 신청에 주민등록번호 요구 → 이름+연락처로 충분

// 안티패턴 2: 로그에 PII 그대로 출력
log.info("사용자 로그인: email={}, password={}", user.getEmail(), rawPassword);
// 올바른 방법:
log.info("사용자 로그인: userId={}", user.getId()); // email도 민감 정보

// 안티패턴 3: PII를 URL 파라미터로 전달
// GET /users?email=user@example.com → URL은 서버 로그, 브라우저 히스토리에 남음
// POST 바디 또는 헤더로 전달할 것

// 안티패턴 4: 마스킹된 데이터를 원본인 척 저장
// DB에 "010-****-1234" 저장 → 원본 복구 불가, 마스킹은 표시용으로만

// 안티패턴 5: 보존 기간 미설정
// @Column private LocalDateTime createdAt; // 만료 정책 없음
// 개인정보 만료일 컬럼과 자동 파기 배치 필수

// 안티패턴 6: 외부 로깅 서비스에 PII 전송
// Datadog/Sentry에 사용자 이메일, 카드번호 포함 이벤트 전송
// 전송 전 PII 필드 반드시 제거/마스킹
```

---

## 마무리: 데이터 보안의 핵심 원칙

데이터 보안을 한 문장으로 요약하면 **"침해를 가정하고 설계하라(Assume Breach)"**다. 경계 방어만으로는 부족하다. 공격자가 내부망에 진입했을 때, DB dump를 가져갔을 때, 로그를 탈취했을 때도 데이터가 안전하도록 여러 계층에서 보호해야 한다.

비밀번호는 bcrypt 또는 Argon2id로 해싱하고 충분한 cost를 사용하라.

저장 데이터는 AES-256-GCM으로 암호화하되 IV를 절대 재사용하지 마라.

전송 데이터는 TLS 1.3으로 보호하고 내부 서비스 간에도 예외 없이 적용하라.

암호화 키는 KMS에서 관리하고 코드와 데이터로부터 분리하라.

PII는 최소한만 수집하고, 목적 달성 후 안전하게 파기하라.

이 원칙들은 규제 준수를 위한 체크박스가 아니다. 사용자가 우리 서비스에 맡긴 신뢰를 지키는 엔지니어링 책임이다.

---

## 참고 자료

1. **OWASP Password Storage Cheat Sheet** — bcrypt/Argon2 파라미터 권고 기준: https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
2. **NIST SP 800-57 Part 1** — Recommendation for Key Management: https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final
3. **AWS Key Management Service Documentation** — Envelope Encryption 패턴: https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html
4. **RFC 5116** — An Interface and Algorithms for Authenticated Encryption (AES-GCM 표준): https://www.rfc-editor.org/rfc/rfc5116
5. **개인정보 보호법 (2023년 개정)** — 개인정보보호위원회: https://www.pipc.go.kr/np/cop/bbs/selectBoardList.do?bbsId=BS217
6. **Argon2 Reference Implementation** — PHC 우승 알고리즘 스펙: https://github.com/P-H-C/phc-winner-argon2/blob/master/argon2-specs.pdf
