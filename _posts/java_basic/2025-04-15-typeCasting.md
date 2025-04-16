---
title: 3. Java의 타입 변환
date: 2025-04-16 17:07:39 +0900
categories: [Java, Java-basic]
tags: [Java, basic, variable]
toc: true
permalink: /posts/java/:title/
---

# 타입 변환

## 1. 자동 타입 변환

**자동 타입 변환<sup>promotion</sup>**은 작은 범위의 타입이 더 큰 범위의 타입에 대입될 때 자동으로 발생합니다.

기본 타입을 허용 범위 순으로 나열하면 다음과 같습니다.
<div style="display: flex; justify-content: center; align-items: center; flex-wrap: wrap; gap: 12px; padding: 24px; border-radius: 8px;  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05); font-family: 'Arial', sans-serif;">
    <div style="background-color: #333; color: white; padding: 8px 16px; border-radius: 4px; font-weight: 500;">byte</div>
    <div style="color: #777; font-weight: 500;">→</div>
    <div style="background-color: #333; color: white; padding: 8px 16px; border-radius: 4px; font-weight: 500;">short</div>
    <div style="color: #777; font-weight: 500;">→</div>
    <div style="background-color: #333; color: white; padding: 8px 16px; border-radius: 4px; font-weight: 500;">int</div>
    <div style="color: #777; font-weight: 500;">→</div>
    <div style="background-color: #333; color: white; padding: 8px 16px; border-radius: 4px; font-weight: 500;">long</div>
    <div style="color: #777; font-weight: 500;">→</div>
    <div style="background-color: #333; color: white; padding: 8px 16px; border-radius: 4px; font-weight: 500;">float</div>
    <div style="color: #777; font-weight: 500;">→</div>
    <div style="background-color: #333; color: white; padding: 8px 16px; border-radius: 4px; font-weight: 500;">double</div>
</div>


`int` 타입이 `byte` 타입보다 허용 범위가 크기 때문에 다음 코드는 자동 타입 변환이 됩니다.
```java
byte byteValue = 10;
int intValue = byteValue; // 자동 타입 변환
```

정수 타입이 실수 타입으로 대입될 경우엔 무조건 자동 타입 변환이 됩니다.

```java
int intValue = 100;
float floatValue = intValue;    // 100.0f
double doubleValue = intValue;  // 100.0

long longValue = 200;
float floatValue2 = longValue;  // 200.0f
double doubleValue2 = longValue; // 200.0
```

`char` 타입의 경우 int 타입으로 자동 변환되면 유니코드 값이 int 타입에 대입됩니다.

```java
char charValue = 'A';
int intValue = charValue;    // 유니코드: 65

char koreanChar = '가';
int koreanValue = koreanChar; // 유니코드: 44032
```

> **예외** char 타입보다 허용 범위가 작은 byte 타입은 char 타입으로 자동 변환할 수 없습니다. char 타입의 허용 범위는 0~65535로 음수를 포함하지 않는데, byte 타입은 -128~127로 음수를 포함하기 때문입니다.
> ```java
> byte byteValue = 65;
> char charValue = byteValue; // 컴파일 오류 발생
> 
> // 이 경우 명시적 형변환 필요
> char charValue = (char) byteValue;
> ```
## 2. 강제 타입 변환

큰 허용 범위 타입을 작은 허용 범위 타입으로 쪼개어 저장하는 것을 **강제 타입 변환<sup>casting</sup>**이라고 합니다. 강제 타입 변환은 캐스팅 연산자로 괄호()를 사용하는데, 타입은 쪼갤 단위입니다.

### int → byte 변환

`int`(4byte)에서 `byte`(1byte)로 변환할 때는 상위 3바이트가 손실되고 마지막 1바이트만 저장됩니다.

![alt text](assets/lib/img/posts/casting2.png)

```java
int intValue = 10;
byte byteValue = (byte) intValue; // 강제 타입 변환

int intValue2 = 200;
byte byteValue2 = (byte) intValue2; // byte 범위(-128~127)를 초과해 데이터 손실 발생
System.out.println(byteValue2); // -56 출력 (200-256=-56)
```


### long → int 변환

`long`(8byte)에서 `int`(4byte)로 변환할 때는 상위 4바이트가 손실됩니다.

```java
long longValue = 300;
int intValue = (int) longValue; // 강제 타입 변환

long longValue2 = 2147483648L; // int 최대값(2147483647) 초과
int intValue2 = (int) longValue2; // 오버플로우 발생
System.out.println(intValue2); // -2147483648 출력
```

### int → char 변환

`int`(4byte)에서 `char`(2byte)로 변환할 때는 상위 2바이트가 손실되고, 하위 2바이트만 유지됩니다.

```java
int intValue = 65;
char charValue = (char) intValue; // 'A'로 변환

int intValue2 = 44032;
char charValue2 = (char) intValue2; // '가'로 변환

// 음수값은 char로 변환 시 다른 문자가 됨(유니코드 인코딩 방식에 따라)
int negativeInt = -1;
char charValue3 = (char) negativeInt;
System.out.println(charValue3); // '￿' 출력 (유니코드 65535)
```

### 실수 → 정수 변환

실수에서 정수로 변환할 때는 소수점 이하 부분이 **버려집니다**(반올림이 아님).

```java
double doubleValue = 3.14;
int intValue = (int) doubleValue; // 3으로 변환 (소수점 이하 버림)

float floatValue = 9.9f;
int intValue2 = (int) floatValue; // 9로 변환
```
> **주의사항**: 강제 타입 변환 시 데이터 손실 가능성을 항상 염두에 두어야 합니다. 특히 큰 값을 작은 범위 타입으로 변환할 때나 실수를 정수로 변환할 때 예상치 못한 값이 나올 수 있습니다.

## 3. 연산식에서 자동 타입 변환
