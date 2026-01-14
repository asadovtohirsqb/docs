## ❗ Asosiy haqiqat (buni bilish shart)

👉 **Java’da `record` (yoki class) fieldlari soni**

* **compile-time’da qat’iy**
* **`application.properties` orqali dinamik ko‘paytirib bo‘lmaydi**

❌ Ya’ni bunday qilish **imkonsiz**:

```properties
pinfl.count=10
```

```java
// ❌ buni runtime’da yaratib bo‘lmaydi
String pinfl1;
String pinfl2;
...
String pinfl10;
```

---

## ✅ TO‘G‘RI VA PROFESSIONAL YECHIM

### 🔥 Variant 1 (ENG TAVSIYA ETILADI): `List<String>` yoki `Map`

### 1️⃣ `List<String>` (agar faqat ketma-ket bo‘lsa)

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import java.util.List;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record CalcRequest(
        VehicleDetails vehicle,
        ApplicantDetails applicant,
        CostDetails cost,
        DriverDetails details
) {

    public record DriverDetails(
            Boolean driverNumberRestriction,
            List<String> pinfls
    ) {}
}
```

### JSON chiqishi:

```json
{
  "details": {
    "driverNumberRestriction": true,
    "pinfls": [
      "123",
      "456",
      "789"
    ]
  }
}
```

➡ `application.properties`:

```properties
osago.driver.pinfl.count=10
```

➡ Siz **List size** ni shu qiymat bilan nazorat qilasiz.

---

### 2️⃣ `Map<String, String>` (agar API `pinfl1`, `pinfl2` deb talab qilsa)

⚠️ Agar **tashqi API aynan `pinfl1`, `pinfl2` nomlarini talab qilsa** — mana **ENG TO‘G‘RI YO‘L**:

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import java.util.Map;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record DriverDetails(
        Boolean driverNumberRestriction,
        Map<String, String> pinfls
) {}
```

### JSON:

```json
{
  "details": {
    "driverNumberRestriction": true,
    "pinfl1": "123",
    "pinfl2": "456",
    "pinfl3": "789"
  }
}
```

👉 Buning uchun **custom serializer** ishlatiladi.

---

## 🧠 ENG PRO YECHIM (Map + Custom Serializer) 🏆

### 1️⃣ DTO

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public record DriverDetails(
        Boolean driverNumberRestriction,
        Map<Integer, String> pinfls
) {}
```

---

### 2️⃣ Custom Serializer

```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.JsonSerializer;
import com.fasterxml.jackson.databind.SerializerProvider;

import java.io.IOException;
import java.util.Map;

public class PinflSerializer extends JsonSerializer<Map<Integer, String>> {

    @Override
    public void serialize(
            Map<Integer, String> value,
            JsonGenerator gen,
            SerializerProvider serializers
    ) throws IOException {

        for (Map.Entry<Integer, String> entry : value.entrySet()) {
            if (entry.getValue() != null) {
                gen.writeStringField(
                    "pinfl" + entry.getKey(),
                    entry.getValue()
                );
            }
        }
    }
}
```

---

### 3️⃣ Serializer’ni ulash

```java
import com.fasterxml.jackson.databind.annotation.JsonSerialize;

public record DriverDetails(
        Boolean driverNumberRestriction,

        @JsonSerialize(using = PinflSerializer.class)
        Map<Integer, String> pinfls
) {}
```

---

### 4️⃣ application.properties

```properties
osago.pinfl.max-count=10
```

Service’da:

```java
@Value("${osago.pinfl.max-count}")
private int maxPinflCount;
```

---

## 🎯 Qaysi variantni tanlash kerak?

| Holat                | Tavsiya                     |
| -------------------- | --------------------------- |
| API sizniki          | `List<String>`              |
| API tashqi va qat’iy | `Map + Custom Serializer` ✅ |
| Bank / insurance API | `Map + Serializer` 🏆       |
| Record + xavfsizlik  | Ha                          |

---

## 🧠 Xulosa (ochiq aytaman)

> **Field’larni sonini record ichida dinamik qilish mumkin emas**
> To‘g‘ri yechim — **List yoki Map**, kerak bo‘lsa **custom serializer**

---
