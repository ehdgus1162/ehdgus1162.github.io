---
title: TDD를 적용한 Password (VO) 단위 개발 과정
date: 2025-04-23 15:41:39 +0900
categories: [Java, TDD, Clean Code]
tags: [TDD, Test-Driven Development, Value Object, Domain-Driven Design, Java, Unit Testing, JUnit5, 입문]
toc: true
permalink: /posts/java/:title/
---

# 비밀번호 관리 시스템 설계 및 구현 - TDD와 DDD 적용하기

안녕하세요! 이번 포스트에서는 DDD(도메인 주도 설계)와 TDD(테스트 주도 개발) 방식을 활용하여 안전한 비밀번호 처리 시스템을 구현한 경험을 공유하려고 합니다. 또한 SOLID 원칙과 다형성이 어떻게 적용되었는지도 자세히 살펴보겠습니다.

## 1. 프로젝트 소개

웰니스 스트리밍 서비스 "MindfulMe(숨터)" 프로젝트에서 사용자 비밀번호를 안전하게 관리하기 위한 시스템을 구현했습니다. 이 시스템의 핵심 요구사항은 다음과 같습니다:

- 비밀번호 정책 검증 (8자 이상, 대문자, 숫자, 특수문자 포함)
- 안전한 비밀번호 암호화 처리
- 비밀번호 저장 및 검증 시 원본 노출 방지
- 다양한 예외 상황 처리와 명확한 오류 메시지 제공

## 2. 도메인 주도 설계(DDD) 적용

DDD는 복잡한 도메인을 효과적으로 모델링하기 위한 접근 방식입니다. 비밀번호 관리 시스템에서는 다음과 같이 DDD 원칙을 적용했습니다.

### 2.1. 값 객체(Value Object) 설계

비밀번호는 고유 식별자 없이 그 값으로만 의미를 갖는 값 객체로 모델링했습니다:

```java
@Getter
public class Password {
    private final String value;
    private final boolean encrypted;

    public Password(String value, PasswordPolicy policy) {
        if (value == null) {
            throw new PasswordPolicyViolationException("비밀번호는 필수 값입니다");
        }

        // 정책 검증 위임
        policy.validate(value);
        this.value = value;
        this.encrypted = false;
    }

    // 암호화된 비밀번호 생성 팩토리 메서드
    public static Password ofEncrypted(String encryptedValue) {
        if (encryptedValue == null || encryptedValue.isBlank()) {
            throw new IllegalArgumentException("암호화된 비밀번호는 필수 값입니다");
        }
        return new Password(encryptedValue, true);
    }

    // 암호화된 비밀번호를 위한 private 생성자
    private Password(String value, boolean encrypted) {
        this.value = value;
        this.encrypted = encrypted;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Password password = (Password) o;
        return Objects.equals(value, password.value);
    }

    @Override
    public int hashCode() {
        return Objects.hash(value);
    }

    @Override
    public String toString() {
        return encrypted ? "[ENCRYPTED]" : "[RAW]";
    }
}
```

값 객체의 특징:
- **불변성(Immutability)**: 모든 필드가 final로 선언되어 생성 후 변경 불가
- **값 기반 동등성**: equals()와 hashCode()가 값을 기준으로 구현
- **자가 유효성 검증**: 생성 시점에 모든 제약조건 검증
- **표현적 풍부함**: toString() 메서드가 비밀번호를 직접 노출하지 않고 상태만 표시

### 2.2. 도메인 서비스와 정책 분리

비밀번호 정책과 암호화는 별도의 도메인 서비스로 분리했습니다:

```java
public interface PasswordPolicy {
    void validate(String password) throws PasswordPolicyViolationException;
    boolean isValid(String password);
}

public interface PasswordEncoder {
    String encrypt(String rawPassword) throws PasswordEncryptionException;
    boolean matches(String rawPassword, String encodedPassword) throws PasswordEncryptionException;
}
```

이러한 분리는 다음과 같은 이점을 제공합니다:
- 비밀번호 관련 책임의 명확한 분리
- 교체 가능한 정책과 암호화 알고리즘
- 명시적인 도메인 언어 사용으로 코드 가독성 향상

## 3. 테스트 주도 개발(TDD) 과정

TDD는 "Red(실패) → Green(성공) → Refactor(개선)" 사이클을 반복하며 개발하는 방법론입니다. 비밀번호 시스템 구현에 TDD를 다음과 같이 적용했습니다.

### 3.1. RED 단계: 실패하는 테스트 작성

먼저 `Password` 클래스의 요구사항을 테스트 코드로 작성했습니다:

```java
@Test
@DisplayName("유효한 비밀번호로 Password 객체 생성 성공")
void createPasswordWithValidFormat() {
    // given
    String validPassword = "Password123!";

    // when
    Password password = new Password(validPassword, passwordPolicy);

    // then
    assertThat(password.getValue()).isEqualTo(validPassword);
    assertThat(password.isEncrypted()).isFalse();
}

@Test
@DisplayName("비밀번호가 null이면 예외 발생")
void throwExceptionForNullPassword() {
    // given
    String nullPassword = null;

    // when & then
    assertThatThrownBy(() -> new Password(nullPassword, passwordPolicy))
            .isInstanceOf(PasswordPolicyViolationException.class)
            .hasMessageContaining("비밀번호는 필수 값입니다");
}

@Test
@DisplayName("비밀번호 정책 위반 시 예외 발생 - 8글자 미만")
void throwExceptionForShortPassword() {
    // given
    String shortPassword = "Abc12!";

    // when & then
    assertThatThrownBy(() -> new Password(shortPassword, passwordPolicy))
            .isInstanceOf(PasswordPolicyViolationException.class)
            .hasMessageContaining("비밀번호는 8자 이상이어야 합니다");
}
```

초기 테스트 개발 시에는 Mock 객체를 사용하여 의존성을 격리했습니다:

```java
@BeforeEach
void setUp() {
    // PasswordPolicy 목 객체 생성
    passwordPolicy = mock(PasswordPolicy.class);
}

@Test
void someTest() {
    // when
    // 유효한 비밀번호에 대해 validate 메서드가 예외를 던지지 않도록 설정
    doNothing().when(passwordPolicy).validate(validPassword);
    
    // 또는 예외 발생 모킹
    doThrow(new PasswordPolicyViolationException("비밀번호는 8자 이상이어야 합니다"))
            .when(passwordPolicy).validate(shortPassword);
}
```

이와 같이 모든 비즈니스 요구사항과 예외 케이스에 대한 테스트를 작성했습니다.

### 3.2. GREEN 단계: 테스트 통과 구현

실패하는 테스트를 통과시키기 위한 최소한의 구현을 진행했습니다:

```java
public class DefaultPasswordPolicy implements PasswordPolicy {
    private static final int MIN_LENGTH = 8;
    private static final Pattern UPPERCASE_PATTERN = Pattern.compile("[A-Z]");
    private static final Pattern DIGIT_PATTERN = Pattern.compile("[0-9]");
    private static final Pattern SPECIAL_CHAR_PATTERN = Pattern.compile("[!@#$%^&*(),.?\":{}|<>]");

    @Override
    public void validate(String password) throws PasswordPolicyViolationException {
        List<String> violations = new ArrayList<>();

        if (password == null) {
            throw new PasswordPolicyViolationException("비밀번호는 필수 값입니다");
        }

        if (password.length() < MIN_LENGTH) {
            violations.add("비밀번호는 8자 이상이어야 합니다");
        }

        if (!UPPERCASE_PATTERN.matcher(password).find()) {
            violations.add("비밀번호는 적어도 하나의 대문자를 포함해야 합니다");
        }

        if (!DIGIT_PATTERN.matcher(password).find()) {
            violations.add("비밀번호는 적어도 하나의 숫자를 포함해야 합니다");
        }

        if (!SPECIAL_CHAR_PATTERN.matcher(password).find()) {
            violations.add("비밀번호는 적어도 하나의 특수문자를 포함해야 합니다");
        }

        if (!violations.isEmpty()) {
            throw new PasswordPolicyViolationException(violations);
        }
    }

    @Override
    public boolean isValid(String password) {
        try {
            validate(password);
            return true;
        } catch (PasswordPolicyViolationException e) {
            return false;
        }
    }
}
```

BCrypt 알고리즘을 활용한 암호화 구현체도 작성했습니다:

```java
public class BCryptPasswordEncoderImpl implements PasswordEncoder {
    private final BCryptPasswordEncoder encoder;

    public BCryptPasswordEncoderImpl() {
        this.encoder = new BCryptPasswordEncoder();
    }

    public BCryptPasswordEncoderImpl(int strength) {
        this.encoder = new BCryptPasswordEncoder(strength);
    }

    @Override
    public String encrypt(String rawPassword) throws PasswordEncryptionException {
        try {
            return encoder.encode(rawPassword);
        } catch (Exception e) {
            throw new PasswordEncryptionException("비밀번호 암호화 중 오류가 발생했습니다", e);
        }
    }

    @Override
    public boolean matches(String rawPassword, String encodedPassword) throws PasswordEncryptionException {
        try {
            return encoder.matches(rawPassword, encodedPassword);
        } catch (Exception e) {
            throw new PasswordEncryptionException("비밀번호 검증 중 오류가 발생했습니다", e);
        }
    }
}
```

### 3.3. REFACTOR 단계: 코드 개선

테스트를 통과한 후, 코드 품질을 개선하기 위한 리팩토링을 진행했습니다. 특히 `DefaultPasswordPolicy` 클래스에서 하드코딩된 상수와 검증 로직을 개선했습니다:

```java
public class DefaultPasswordPolicy implements PasswordPolicy {
    // 설정 가능한 상수값들을 모아둔 내부 클래스
    public static class PolicyConfig {
        private final int minLength;
        private final Pattern uppercasePattern;
        private final Pattern digitPattern;
        private final Pattern specialCharPattern;

        // 기본 설정값을 사용하는 생성자
        public PolicyConfig() {
            this(8, "[A-Z]", "[0-9]", "[!@#$%^&*(),.?\":{}|<>]");
        }

        // 커스텀 설정값을 사용하는 생성자
        public PolicyConfig(int minLength, String uppercaseRegex, 
                            String digitRegex, String specialCharRegex) {
            this.minLength = minLength;
            this.uppercasePattern = Pattern.compile(uppercaseRegex);
            this.digitPattern = Pattern.compile(digitRegex);
            this.specialCharPattern = Pattern.compile(specialCharRegex);
        }
    }

    private final PolicyConfig config;

    // 기본 설정을 사용하는 생성자
    public DefaultPasswordPolicy() {
        this(new PolicyConfig());
    }

    // 커스텀 설정을 사용하는 생성자
    public DefaultPasswordPolicy(PolicyConfig config) {
        this.config = config;
    }

    @Override
    public void validate(String password) throws PasswordPolicyViolationException {
        if (password == null) {
            throw new PasswordPolicyViolationException("비밀번호는 필수 값입니다");
        }

        List<String> violations = new ArrayList<>();
        
        // 각 검증 규칙을 함수로 분리하여 가독성 향상
        checkLength(password, violations);
        checkUppercase(password, violations);
        checkDigit(password, violations);
        checkSpecialChar(password, violations);

        if (!violations.isEmpty()) {
            throw new PasswordPolicyViolationException(violations);
        }
    }

    // 각 검증 규칙을 별도 메서드로 분리
    private void checkLength(String password, List<String> violations) {
        if (password.length() < config.minLength) {
            violations.add(String.format("비밀번호는 %d자 이상이어야 합니다", config.minLength));
        }
    }

    private void checkUppercase(String password, List<String> violations) {
        // 구현 내용...
    }

    private void checkDigit(String password, List<String> violations) {
        // 구현 내용...
    }

    private void checkSpecialChar(String password, List<String> violations) {
        // 구현 내용...
    }

    @Override
    public boolean isValid(String password) {
        try {
            validate(password);
            return true;
        } catch (PasswordPolicyViolationException e) {
            return false;
        }
    }
}
```

리팩토링을 통해 달성한 개선 사항:
- 코드 가독성 및 유지보수성 향상
- 설정 변경 유연성 제공
- 메서드 분리를 통한 단일 책임 준수
- 동적 오류 메시지 지원

## 4. SOLID 원칙 적용

### 4.1. 단일 책임 원칙(SRP: Single Responsibility Principle)

각 클래스는 하나의 책임만 가집니다:
- `Password`: 비밀번호 값 객체 표현
- `PasswordPolicy`: 비밀번호 정책 검증
- `PasswordEncoder`: 비밀번호 암호화 및 검증
- `PasswordPolicyViolationException`: 정책 위반 예외 처리

### 4.2. 개방-폐쇄 원칙(OCP: Open-Closed Principle)

시스템은 확장에는 열려 있고, 수정에는 닫혀 있습니다:
- 새로운 비밀번호 정책은 `PasswordPolicy` 인터페이스를 구현하면 됨
- 새로운 암호화 알고리즘은 `PasswordEncoder` 인터페이스를 구현하면 됨
- 기존 코드 수정 없이 새로운 기능 추가 가능

### 4.3. 리스코프 치환 원칙(LSP: Liskov Substitution Principle)

하위 타입은 상위 타입을 대체할 수 있어야 합니다:
- `DefaultPasswordPolicy`는 `PasswordPolicy` 인터페이스의 계약을 준수
- `BCryptPasswordEncoderImpl`은 `PasswordEncoder` 인터페이스의 계약을 준수

### 4.4. 인터페이스 분리 원칙(ISP: Interface Segregation Principle)

인터페이스는 클라이언트에 필요한 메서드만 제공합니다:
- `PasswordPolicy`는 검증 관련 메서드만 포함
- `PasswordEncoder`는 암호화 관련 메서드만 포함

### 4.5. 의존성 역전 원칙(DIP: Dependency Inversion Principle)

고수준 모듈은 저수준 모듈에 의존하지 않고, 둘 다 추상화에 의존합니다:
- `Password` 클래스는 구체적인 `DefaultPasswordPolicy`가 아닌 `PasswordPolicy` 인터페이스에 의존
- 서비스 계층에서도 `PasswordEncoder` 인터페이스를 통해 의존성 주입 가능

## 5. 다형성 활용

다형성은 객체 지향 프로그래밍의 핵심 개념으로, 이 시스템에서는 다음과 같이 활용되었습니다:

### 5.1. 인터페이스를 통한 다형성

`PasswordPolicy` 인터페이스를 사용하면 다양한 비밀번호 정책을 쉽게 적용할 수 있습니다:

```java
// 기본 정책 사용
PasswordPolicy defaultPolicy = new DefaultPasswordPolicy();
Password password1 = new Password("MyPassword123!", defaultPolicy);

// 커스텀 정책 사용
PasswordPolicy customPolicy = new DefaultPasswordPolicy(
    new DefaultPasswordPolicy.PolicyConfig(10, "[A-Z]", "[0-9]", "[!@#$%^&*]")
);
Password password2 = new Password("MyStrongerPassword123#", customPolicy);

// 기업용 정책 사용 (예시)
PasswordPolicy enterprisePolicy = new EnterprisePasswordPolicy(); // 다른 구현체
Password password3 = new Password("EnterpriseP@ssw0rd", enterprisePolicy);
```

### 5.2. 암호화 알고리즘의 다형성

`PasswordEncoder` 인터페이스를 통해 다양한 암호화 알고리즘을 적용할 수 있습니다:

```java
// BCrypt 알고리즘 사용
PasswordEncoder bcryptEncoder = new BCryptPasswordEncoderImpl();
String encrypted1 = bcryptEncoder.encrypt("MyPassword123!");

// 다른 강도의 BCrypt 사용
PasswordEncoder strongerBcrypt = new BCryptPasswordEncoderImpl(12); // 더 강력한 해싱
String encrypted2 = strongerBcrypt.encrypt("MyPassword123!");

// 다른 알고리즘 사용 (예시)
PasswordEncoder argon2Encoder = new Argon2PasswordEncoderImpl(); // 다른 구현체
String encrypted3 = argon2Encoder.encrypt("MyPassword123!");
```

### 5.3. 예외 처리의 다형성

예외 클래스의 계층 구조를 통해 세분화된 오류 처리가 가능합니다:

```java
try {
    Password password = new Password(userInput, passwordPolicy);
    // 비밀번호 처리...
} catch (PasswordPolicyViolationException e) {
    // 정책 위반 예외 처리
    List<String> violations = e.getViolations();
    showValidationErrors(violations);
} catch (PasswordEncryptionException e) {
    // 암호화 예외 처리
    logError("암호화 오류: " + e.getMessage(), e.getCause());
} catch (RuntimeException e) {
    // 다른 예외 처리
    logUnexpectedError(e);
}
```

## 6. 전체 시스템 동작 흐름

비밀번호 관리 시스템의 주요 시나리오는 다음과 같습니다:

### 6.1. 사용자 등록 시 비밀번호 처리 흐름

```java
// 1. 사용자 입력 비밀번호
String userInputPassword = "MyP@ssword123";

// 2. 비밀번호 정책 검증 및 Password 객체 생성
PasswordPolicy policy = new DefaultPasswordPolicy();
Password rawPassword = new Password(userInputPassword, policy);

// 3. 비밀번호 암호화
PasswordEncoder encoder = new BCryptPasswordEncoderImpl();
String encryptedValue = encoder.encrypt(rawPassword.getValue());

// 4. 암호화된 비밀번호 객체 생성
Password securePassword = Password.ofEncrypted(encryptedValue);

// 5. 사용자 객체에 비밀번호 설정 및 저장
User user = new User(email, username, securePassword);
userRepository.save(user);
```

### 6.2. 로그인 시 비밀번호 검증 흐름

```java
// 1. 사용자 로그인 시도
String loginEmail = "user@example.com";
String loginPassword = "MyP@ssword123";

// 2. 사용자 조회
User user = userRepository.findByEmail(new Email(loginEmail))
    .orElseThrow(() -> new UserNotFoundException("사용자를 찾을 수 없습니다"));

// 3. 저장된 암호화 비밀번호 가져오기
String storedEncryptedPassword = user.getPassword().getValue();

// 4. 비밀번호 일치 검증
PasswordEncoder encoder = new BCryptPasswordEncoderImpl();
boolean isMatch = encoder.matches(loginPassword, storedEncryptedPassword);

// 5. 인증 결과 처리
if (isMatch) {
    // 로그인 성공 처리
} else {
    // 로그인 실패 처리
}
```

## 7. 테스트 코드와 실제 구현 비교

초기 개발 단계에서는 모킹을 활용하여 의존성을 제어했지만, 최종 구현에서는 실제 구현체를 사용하도록 변경했습니다:

### 초기 테스트 코드 (모킹 활용)
```java
@Test
@DisplayName("비밀번호 정책 위반 시 예외 발생 - 대문자 없음")
void throwExceptionForPasswordWithoutUppercase() {
    // given
    String noUppercasePassword = "password123!";

    // when
    doThrow(new PasswordPolicyViolationException("비밀번호는 적어도 하나의 대문자를 포함해야 합니다"))
            .when(passwordPolicy).validate(noUppercasePassword);

    // then
    assertThatThrownBy(() -> new Password(noUppercasePassword, passwordPolicy))
            .isInstanceOf(PasswordPolicyViolationException.class)
            .hasMessageContaining("비밀번호는 적어도 하나의 대문자를 포함해야 합니다");

    verify(passwordPolicy).validate(noUppercasePassword);
}
```

### 최종 테스트 코드 (실제 구현체 사용)
```java
@Test
@DisplayName("비밀번호 정책 위반 시 예외 발생 - 대문자 없음")
void throwExceptionForPasswordWithoutUppercase() {
    // given
    String noUppercasePassword = "password123!";
    PasswordPolicy passwordPolicy = new DefaultPasswordPolicy();

    // when & then
    assertThatThrownBy(() -> new Password(noUppercasePassword, passwordPolicy))
            .isInstanceOf(PasswordPolicyViolationException.class)
            .hasMessageContaining("비밀번호는 적어도 하나의 대문자를 포함해야 합니다");
}
```

이러한 변화는 테스트가 실제 구현 동작을 더 정확하게 반영하면서도, 의존성의 결합도를 낮게 유지하는 방식으로 발전했음을 보여줍니다.

## 8. 결론

TDD와 DDD 방식으로 구현한 비밀번호 관리 시스템은 다음과 같은 이점을 제공합니다:

### 8.1. 품질 측면
- 높은 테스트 커버리지로 코드 안정성 보장
- 비즈니스 규칙이 명확하게 표현된 도메인 모델
- SOLID 원칙을 준수한 확장 가능한 설계
- 다형성을 활용한 유연한 시스템 구조

### 8.2. 개발 프로세스 측면
- 요구사항을 테스트로 먼저 명확하게 정의
- 작은 단계로 나누어 점진적 개발 진행
- 지속적인 리팩토링을 통한 코드 품질 개선
- 도메인 전문가와 개발자 간 소통 향상

### 8.3. 유지보수 측면
- 도메인 개념을 명확하게 표현하는 코드 구조
- 책임이 분리된 모듈로 변경 영향 범위 최소화
- 교체 가능한 컴포넌트로 기능 진화 용이
- 테스트를 통한 회귀 오류 방지

이 시스템은 단순한 비밀번호 처리를 넘어, 객체 지향 설계의 핵심 원칙을 실천하는 좋은 예시가 되었습니다. 특히 값 객체, 도메인 서비스, 다형성 등의 개념을 실제 코드로 구현하면서 이론과 실무를 연결하는 경험이 되었습니다.

앞으로도 TDD와 DDD 접근법을 활용하여 더 복잡한 도메인 문제도 명확하고 유지보수하기 쉬운 코드로 구현할 수 있기를 기대합니다.