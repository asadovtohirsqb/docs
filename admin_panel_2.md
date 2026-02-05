Sizning savolingiz juda o‘rinli: agar admin panelda **provodkaIsDone** yoki **paymentIsDone** bo‘yicha filter qo‘yilsa, nima bo‘ladi va bu holatni qanday to‘g‘ri hal qilish kerak?

Keling, muammoni va yechimlarni bosqichma-bosqich ko‘rib chiqaylik.

### Muammo nima?

- `provodkaIsDone` va `paymentIsDone` field’lari **Transaction** jadvalidan keladi (alohida database).
- Asosiy ro‘yxat va pagination esa **OrderParam** jadvalidan olinadi (boshqa database).
- Agar bu ikki boolean bo‘yicha filter qo‘yilsa, quyidagi muammolar paydo bo‘ladi:

1. **Asosiy query’da bu field’lar yo‘q** → ularni WHERE shartida ishlatib bo‘lmaydi.
2. **Agar oldindan filter qilmasak**, biz avval 20 ta (masalan page size) OrderParam olamiz, keyin ularning transaction’larini tekshiramiz va filterdan o‘tkazamiz → natijada sahifada 20 ta emas, balki 5–10 ta yoki hatto 0 ta qolishi mumkin.
3. **Total count va page soni noto‘g‘ri bo‘ladi** → frontend’da pagination buziladi (masalan “Showing 1–10 of 500” deb ko‘rsatiladi, lekin faqat 3 ta ko‘rinadi).

### Yechim variantlari (realistik darajada)

#### Variant 1: Post-filter (hozirgi kodda qilayotgan usul) – oddiy, lekin cheklovlari bor

Hozirgi holatda siz shunday qilyapsiz:

- OrderParam’larni filter + pageable bilan olasiz
- Har biriga tegishli Transaction’larni batch olib, `paymentIsDone` va `provodkaIsDone` ni hisoblaysiz
- Keyin DTO’larni filterdan o‘tkazasiz

**Afzalliklari:**
- Kod sodda
- Tez yoziladi va ishlaydi

**Kamchiliklari:**
- Sahifada ko‘rinadigan elementlar soni har doim page size’dan kam yoki teng bo‘ladi (ko‘pincha kam)
- Total pages va total elements noto‘g‘ri (frontend’da chiroyli ko‘rinmaydi)
- Agar filter qattiq bo‘lsa (masalan faqat `provodkaIsDone = true`), foydalanuvchi ko‘p sahifani aylantirishi mumkin, lekin natija kam chiqadi

**Qachon ishlatish mumkin?**  
Agar bu filterlar kamdan-kam ishlatilsa yoki foydalanuvchilar odatda bu filterlarni qo‘ymasa.

#### Variant 2: Ikki bosqichli yondashuv (eng real va to‘g‘ri yechim)

Bu usul ko‘p real loyihalarda (ayniqsa alohida DB’larda) qo‘llaniladi.

1. **Birinchi bosqich**: faqat `paymentIsDone` va `provodkaIsDone` bo‘yicha filter bo‘lsa, avval **Transaction** jadvalidan kerakli `operationId` larni topamiz.

```java
List<Long> filteredOperationIds = transactionRepository.findFilteredOperationIds(
    search.paymentIsDone(),
    search.provodkaIsDone(),
    search.createdFrom(),
    search.createdTo()  // agar transaction date ham filter bo'lsa
);
```

Masalan:

```java
@Query("SELECT DISTINCT t.operationId FROM Transaction t " +
       "WHERE (:paymentDone IS NULL OR t.status = :doneStatus) " +
       "AND (:provodkaDone IS NULL OR EXISTS (SELECT 1 FROM Transaction p WHERE p.operationId = t.operationId AND p.status = :provodkaStatus))")
List<Long> findFilteredOperationIds(
    @Param("paymentDone") Boolean paymentDone,
    @Param("provodkaDone") Boolean provodkaDone,
    @Param("doneStatus") TransactionStatus doneStatus,
    @Param("provodkaStatus") TransactionStatus provodkaStatus
);
```

2. **Ikkinchi bosqich**: OrderParam query’sida `operationId IN (...)` shartini qo‘shamiz.

```java
Specification<OrderParam> spec = (root, query, cb) -> {
    if (filteredOperationIds != null && !filteredOperationIds.isEmpty()) {
        return root.get("operationId").in(filteredOperationIds);
    }
    return cb.conjunction();
};
```

3. **Natija**: pagination va total count to‘g‘ri bo‘ladi, chunki filter asosiy query’da qo‘llanilgan.

**Afzalliklari:**
- Sahifa to‘g‘ri to‘ldiriladi
- Total count va page soni real
- Foydalanuvchi tajribasi yaxshi

**Kamchiliklari:**
- Agar operationId’lar juda ko‘p bo‘lsa (masalan 10000+), `IN` sharti sekinlashishi mumkin → limit qo‘yish yoki pagination’ni qisman qayta hisoblash kerak bo‘ladi.

#### Variant 3: Hybrid yondashuv (eng muvozanatli)

```text
Agar paymentIsDone yoki provodkaIsDone filter qo‘yilgan bo‘lsa:
  → Variant 2 (Transaction’dan operationId’larni oldindan olish)
Aks holda:
  → Variant 1 (post-filter)
```

Bu kodda shunday ko‘rinadi:

```java
List<Long> preFilteredOperationIds = null;
if (search.paymentIsDone() != null || search.provodkaIsDone() != null) {
    preFilteredOperationIds = transactionRepository.findFilteredOperationIds(...);
}

Specification<OrderParam> spec = buildSpecification(search)
    .and((root, query, cb) -> {
        if (preFilteredOperationIds != null && !preFilteredOperationIds.isEmpty()) {
            return root.get("operationId").in(preFilteredOperationIds);
        }
        return cb.conjunction();
    });

Page<OrderParam> orderPage = orderParamRepository.findAll(spec, search.pageable());

// Agar pre-filter bo‘lgan bo‘lsa → post-filter kerak emas
if (preFilteredOperationIds == null) {
    // post-filter qilish (Variant 1)
    dtos = dtos.stream().filter(...).toList();
}
```

### Tavsiya (qisqa va amaliy)

| Holat                                | Tavsiya qilingan yechim             | Izoh                                                                 |
|--------------------------------------|--------------------------------------|----------------------------------------------------------------------|
| Filter’lar kamdan-kam ishlatiladi   | Variant 1 (post-filter)             | Tez va sodda                                                         |
| Admin panelda bu filterlar muhim    | Variant 2 yoki Hybrid (Variant 3)   | To‘g‘ri pagination va foydalanuvchi tajribasi uchun                  |
| Ma’lumotlar juda katta hajmda       | Variant 2 + limit (masalan 5000 ta) | IN shartiga cheklov qo‘yish                                          |

### Yakuniy maslahat

Eng real va foydalanuvchiga qulay yechim – **Variant 3 (hybrid)**:

- Agar `paymentIsDone` yoki `provodkaIsDone` filter qo‘yilgan bo‘lsa → avval Transaction’dan operationId’larni topib, OrderParam’da `IN` bilan ishlatish.
- Aks holda → oddiy post-filter.

Bu usul bilan sizning admin panelingizda sahifalar to‘g‘ri to‘ldiriladi va total count real bo‘ladi.

Agar kodni yozishda yordam kerak bo‘lsa (masalan Transaction repository query’si yoki hybrid logic), aniqroq yozib beraman.
