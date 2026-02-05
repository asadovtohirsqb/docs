**Jakarta Bean Validation (JSR-380)** standartida belgilangan va quyidagi vazifalarni bajaradi:

```java
String message() default "INVALID_DISTRICT_ID. Tuman topilmadi.";
Class<?>[] groups() default {};
Class<? extends Payload>[] payload() default {};
```

### 1. `String message() default "INVALID_DISTRICT_ID. Tuman topilmadi.";`

**Vazifasi:**  
Agar validatsiya muvaffaqiyatsiz bo‘lsa (ya’ni tuman topilmasa yoki ID null bo‘lsa), qaytariladigan **xato xabari**ni belgilaydi.

- Bu xabar odatda API javobida yoki frontend’da foydalanuvchiga ko‘rsatiladi.
- `default` qiymati bilan belgilangan bo‘lsa, siz annotatsiyani ishlatganda xabarni o‘zgartirmasangiz, shu matn ishlatiladi.
- Misol: controller’da xato bo‘lsa, javobda shunday chiqadi:

```json
{
  "status": "ERROR",
  "message": "INVALID_DISTRICT_ID. Tuman topilmadi."
}
```

**O‘zgartirish mumkin:**
```java
@PathVariable @ValidDistrictId(message = "Bunday tuman mavjud emas!") Long districtId
```

### 2. `Class<?>[] groups() default {};`

**Vazifasi:**  
Validatsiyani **guruhlash** imkonini beradi. Ya’ni bir xil obyekt yoki parametr turli holatlarda turli qoidalarga ko‘ra tekshirilishi mumkin.

- `groups` bo‘sh bo‘lsa → validatsiya har doim bajariladi (default holat).
- Agar guruh ishlatilsangiz, faqat belgilangan guruh faollashtirilganda validatsiya ishlaydi.

**Misol:**

```java
public interface Create {}
public interface Update {}

@ValidDistrictId(groups = {Create.class, Update.class})
private Long districtId;
```

Controller’da:

```java
@Validated(Create.class)
@PostMapping
public void create(@RequestBody Dto dto) { ... }

@Validated(Update.class)
@PutMapping
public void update(@RequestBody Dto dto) { ... }
```

Bu holatda `districtId` faqat `Create` yoki `Update` guruhlari faol bo‘lgandagina tekshiriladi.

Sizning loyihangizda hozircha guruhlar ishlatilmayotgani uchun `default {}` bo‘lishi to‘g‘ri va yetarli.

### 3. `Class<? extends Payload>[] payload() default {};`

**Vazifasi:**  
Validatsiya xatosiga **qo‘shimcha metadata** (ma’lumot) biriktirish uchun ishlatiladi.

- Bu juda kam ishlatiladi va ko‘pincha maxsus holatlarda kerak bo‘ladi.
- `Payload` orqali xatoga qo‘shimcha kontekst yoki tur (masalan severity level, error code, client-side action) qo‘shish mumkin.

**Misol:**

```java
public interface ErrorSeverity {
    interface Warning {}
    interface Critical {}
}

@ValidDistrictId(payload = ErrorSeverity.Critical.class)
```

Keyin exception handler’da:

```java
if (violation.getConstraintDescriptor().getPayload().contains(ErrorSeverity.Critical.class)) {
    // juda muhim xato — log qilish, alert yuborish va h.k.
}
```

Sizning loyihangizda hozircha `payload` ishlatilmayapti va deyarli barcha loyihalarda bo‘sh qoldiriladi.

### Qisqa xulosa (har birining asosiy vazifasi)

| Parametr       | Asosiy vazifasi                                      | Sizning loyihangizda kerakmi? | Odatda o‘zgartiriladimi? |
|----------------|------------------------------------------------------|-------------------------------|---------------------------|
| `message()`    | Xato bo‘lganda chiqadigan xabar matni                | Ha, foydali                   | Ha (ko‘pincha o‘zgartiriladi) |
| `groups()`     | Validatsiyani guruhlash (Create/Update/Delete va h.k.) | Hozircha yo‘q                 | Kamdan-kam                |
| `payload()`    | Xatoga qo‘shimcha metadata biriktirish               | Hozircha yo‘q                 | Juda kam                  |
