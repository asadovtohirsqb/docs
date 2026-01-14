# @JsonAutoDetect(fieldVisibility=JsonAutoDetect.Visibility.NONE)

## 1️⃣ Real case: Bank API (xavfsizlik sabab)

### ❗ Muammo

`User` entity ichida **maxfiy fieldlar** bor:

* `password`
* `pinCode`
* `internalId`

Lekin API response’da **faqat kerakli ma’lumotlar** chiqishi kerak.

---

## 2️⃣ Entity (yomon variant ❌)

```java
@Entity
public class User {

    @Id
    private Long id;

    private String fullName;
    private String phone;

    private String password;   // ❌ chiqib ketadi
    private String pinCode;    // ❌ chiqib ketadi

    // getters/setters
}
```

➡ Jackson **getter bor bo‘lsa hammasini JSON’ga chiqaradi**
Bu bank API uchun **katta xato** ❌

---

## 3️⃣ To‘g‘ri yechim (QAT’IY nazorat) ✅

```java
@JsonAutoDetect(
    fieldVisibility = JsonAutoDetect.Visibility.NONE,
    getterVisibility = JsonAutoDetect.Visibility.NONE,
    setterVisibility = JsonAutoDetect.Visibility.NONE
)
@Entity
public class User {

    @Id
    @JsonProperty("id")
    private Long id;

    @JsonProperty("fullName")
    private String fullName;

    @JsonProperty("phone")
    private String phone;

    private String password;   // ❌ umuman ko‘rinmaydi
    private String pinCode;    // ❌ umuman ko‘rinmaydi
}
```

➡ JSON response:

```json
{
  "id": 1,
  "fullName": "Ali Valiyev",
  "phone": "+998901234567"
}
```

---

## 4️⃣ `@JsonIgnore` bilan solishtirish

### `@JsonIgnore` ❌ (kamroq xavfsiz)

```java
public class User {

    private String fullName;

    @JsonIgnore
    private String password;
}
```

⚠️ Xavf:

* Yangi field qo‘shilsa → **unutib qo‘yish mumkin**
* Getter qo‘shilsa → JSON’ga chiqib ketishi mumkin

---

### `@JsonAutoDetect(NONE)` ✅ (xavfsiz)

➡ Default: **hech narsa chiqmaydi**
➡ Faqat `@JsonProperty` bilan belgilanganlar chiqadi
➡ Human error deyarli yo‘q

---

## 5️⃣ Getter / Setter bilan ishlatish

Agar field emas, **method orqali** boshqarmoqchi bo‘lsangiz:

```java
@JsonAutoDetect(
    fieldVisibility = Visibility.NONE,
    getterVisibility = Visibility.NONE
)
public class CardDto {

    private String cardNumber;

    @JsonGetter("card")
    public String maskedCard() {
        return "**** **** **** " + cardNumber.substring(12);
    }
}
```

➡ JSON:

```json
{
  "card": "**** **** **** 1234"
}
```

---

## 6️⃣ DTO uchun ideal pattern 🏆

```java
@JsonAutoDetect(
    fieldVisibility = Visibility.NONE,
    getterVisibility = Visibility.NONE,
    setterVisibility = Visibility.NONE
)
public class OrderResponse {

    @JsonProperty("order_id")
    private Long id;

    @JsonProperty("status")
    private String status;

    @JsonProperty("created_at")
    private LocalDateTime createdAt;
}
```

➡ **Entity alohida**, **DTO alohida** → eng toza arxitektura

---

## 7️⃣ Qachon ishlatish shart?

✅ Bank / Payment / Card / PII
✅ Public REST API
✅ Compliance (PCI DSS, GDPR)
✅ Microservice response contract qat’iy bo‘lsa

❌ Internal tool
❌ Quick prototype

---

## 🧠 Xulosa (short)

| Yondashuv               | Xavfsizlik  | Tavsiya    |
| ----------------------- | ----------- | ---------- |
| Default Jackson         | ❌ past      | Yo‘q       |
| `@JsonIgnore`           | ⚠️ o‘rtacha | Kam        |
| `@JsonAutoDetect(NONE)` | ✅ yuqori    | ✅ **BEST** |

---


# 1️⃣ `@JsonAutoDetect` vs `@JsonView`

## 📌 Asosiy farq

| Narsa    | `@JsonAutoDetect`     | `@JsonView`                      |
| -------- | --------------------- | -------------------------------- |
| Maqsad   | **Qattiq xavfsizlik** | **Turli response ko‘rinishlari** |
| Default  | Hech narsa chiqmaydi  | Hammasi chiqadi                  |
| Control  | Compile-time          | Runtime                          |
| Use case | Bank / PCI / PII      | Admin vs User API                |

---

## 🧪 `@JsonView` real misol

### View’lar

```java
public class Views {
    public static class Public {}
    public static class Internal extends Public {}
}
```

### Model

```java
public class User {

    @JsonView(Views.Public.class)
    private String fullName;

    @JsonView(Views.Public.class)
    private String phone;

    @JsonView(Views.Internal.class)
    private String internalId;
}
```

### Controller

```java
@GetMapping("/users")
@JsonView(Views.Public.class)
public User getUser() {
    return userService.get();
}
```

➡ Public API:

```json
{
  "fullName": "Ali",
  "phone": "+99890..."
}
```

---

## ⚠️ Qachon yaramaydi?

* Sensitive data bo‘lsa
* Kimdir `@JsonView` ni unutsa
* Getter qo‘shib yuborsa

📌 **Bank API’da `@JsonAutoDetect` afzal**

---

# 2️⃣ Java 17 `record` + Jackson ✅

## 📌 Nega record?

* Immutable
* DTO uchun ideal
* Kam kod

---

### Record DTO

```java
public record CardResponse(
    String cardNumber,
    String status,
    LocalDateTime createdAt
) {}
```

➡ JSON avtomatik ishlaydi.

---

### Xavfsiz record (qat’iy control)

```java
@JsonAutoDetect(
    fieldVisibility = JsonAutoDetect.Visibility.NONE
)
public record CardResponse(

    @JsonProperty("card")
    String maskedCard,

    @JsonProperty("status")
    String status
) {}
```

---

### Custom logic bilan

```java
public record CardResponse(String cardNumber) {

    @JsonProperty("card")
    public String masked() {
        return "**** **** **** " + cardNumber.substring(12);
    }
}
```

---

# 3️⃣ Lombok + Jackson BEST PRACTICE 🏆

## ❌ Yomon variant

```java
@Data
public class User {
    private String password;
}
```

⚠️ Getter avtomatik chiqadi → JSON’da ko‘rinadi

---

## ✅ To‘g‘ri variant (Entity)

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Entity
@JsonAutoDetect(
    fieldVisibility = JsonAutoDetect.Visibility.NONE
)
public class User {

    @Id
    @JsonProperty("id")
    private Long id;

    @JsonProperty("fullName")
    private String fullName;

    private String password; // ❌
}
```

---

## ✅ DTO + Lombok

```java
@Getter
@AllArgsConstructor
@JsonAutoDetect(fieldVisibility = Visibility.NONE)
public class UserResponse {

    @JsonProperty("id")
    private final Long id;

    @JsonProperty("fullName")
    private final String fullName;
}
```

📌 **DTO’da `@Data` ishlatmang**
📌 `@Getter` + `final` → immutable

---

# 4️⃣ Entity → DTO Map (MapStruct) 🔥

## 📌 Nega MapStruct?

* Compile-time
* Fast
* Null-safe
* No reflection

---

## Dependency

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
```

---

## Mapper

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    @Mapping(target = "password", ignore = true)
    UserResponse toResponse(User user);
}
```

---

## Record DTO bilan

```java
public record UserResponse(Long id, String fullName) {}
```

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(User user);
}
```

---

## Custom mapping

```java
@Mapper(componentModel = "spring")
public interface CardMapper {

    @Mapping(
        target = "card",
        expression = "java(\"**** **** **** \" + card.getNumber().substring(12))"
    )
    CardResponse toDto(Card card);
}
```

---

# 🧠 Qoidalar (PRO level)

✅ Entity → tashqariga chiqmasin
✅ API → faqat DTO / record
✅ Sensitive data → `@JsonAutoDetect(NONE)`
✅ Lombok `@Data` → yo‘q
✅ MapStruct → ha
✅ Java 17 record → DTO uchun ideal

---

## 🎯 FINAL STACK (Bank-grade)

```text
Entity + JPA
↓
MapStruct
↓
Record DTO
↓
@JsonAutoDetect(NONE)
↓
REST API
```

---
