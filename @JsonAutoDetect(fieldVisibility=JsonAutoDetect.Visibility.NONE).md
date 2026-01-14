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
