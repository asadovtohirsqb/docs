### Senior Darajada Yechim: Ikki Alohida Database’dan Ma’lumotlarni Birlashtirish, Filter va Pageable Qo‘llash

Salom, Tohir_sqb! Savolingizni tushundim: Sizda **card_delivery** (OrderParam va Branch) va **sqb_mobile** (Transaction) alohida database’lar mavjud. Siz ularni birlashtirib, `OrderParamDTOForAdminPanel` DTO’sini qaytarishni xohlaysiz, unda filter (masalan, status, createdAt, paymentIsDone, provodkaIsDone) va pageable qo‘llanilishi kerak.

Bu muammoni senior darajada hal qilish uchun quyidagi arxitekturani taklif qilaman:

- **Multiple DataSources** sozlash (Spring Boot’da standart yechim).
- **Alohida Repository’lar** va **Entity Manager’lar** har DB uchun.
- **Service’da Aggregation**: OrderParam’larni asosiy query bo‘yicha olish (JOIN Branch bilan), keyin Transaction’larni batch’da olish (in-memory join).
- **Dinamik Filter**: `Specification` orqali (OrderParam va Branch uchun), Transaction field’lari uchun **post-filter** (in-memory, chunki alohida DB).
- **Pageable**: Faqat OrderParam query’sida qo‘llaniladi (asosiy entity bo‘lgani uchun). Transaction filter’lari page count’ni o‘zgartirishi mumkin, shuning uchun **custom Page** yaratish kerak (masalan, `PageImpl`).
- **Performance Optimizations**: Batch fetching, caching (agar kerak bo‘lsa), va N+1 muammosini oldini olish.
- **Error Handling va Transactions**: Alohida DB’lar bo‘lgani uchun distributed transactions (2PC) ishlatish mumkin, lekin read-only query uchun oddiy sinxron yetarli.

Bu yechim **microservices** yoki **distributed systems** da keng qo‘llaniladi va Spring Data JPA + Custom Logic bilan amalga oshiriladi.

#### 1. Multiple DataSources Configuration

`application.yml` da 2 ta DB sozlang:

```yaml
spring:
  datasource:
    card-delivery:
      url: jdbc:postgresql://host1:5432/card_delivery
      username: user1
      password: pass1
      driver-class-name: org.postgresql.Driver
    sqb-mobile:
      url: jdbc:postgresql://host2:5432/sqb_mobile
      username: user2
      password: pass2
      driver-class-name: org.postgresql.Driver
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: none
```

Keyin **2 ta Configuration class** yaratish (har DB uchun):

- **CardDeliveryConfig.java** (OrderParam va Branch uchun):

```java
@Configuration
@EnableJpaRepositories(basePackages = "uz.sqb.joyda.carddeliveryservice.repository.card_delivery")
@EnableTransactionManagement
@PropertySource("classpath:application.yml")
public class CardDeliveryConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.card-delivery")
    public DataSource cardDeliveryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean cardDeliveryEntityManagerFactory(
            EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(cardDeliveryDataSource())
                .packages("uz.sqb.joyda.carddeliveryservice.domain.card_delivery")
                .persistenceUnit("cardDelivery")
                .build();
    }

    @Bean
    @Primary
    public PlatformTransactionManager cardDeliveryTransactionManager(
            EntityManagerFactory cardDeliveryEntityManagerFactory) {
        return new JpaTransactionManager(cardDeliveryEntityManagerFactory);
    }
}
```

- **SqbMobileConfig.java** (Transaction uchun):

```java
@Configuration
@EnableJpaRepositories(basePackages = "uz.sqb.joyda.carddeliveryservice.repository.sqb_mobile")
@EnableTransactionManagement
@PropertySource("classpath:application.yml")
public class SqbMobileConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.sqb-mobile")
    public DataSource sqbMobileDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean sqbMobileEntityManagerFactory(
            EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(sqbMobileDataSource())
                .packages("uz.sqb.joyda.carddeliveryservice.domain.sqb_mobile")
                .persistenceUnit("sqbMobile")
                .build();
    }

    @Bean
    public PlatformTransactionManager sqbMobileTransactionManager(
            EntityManagerFactory sqbMobileEntityManagerFactory) {
        return new JpaTransactionManager(sqbMobileEntityManagerFactory);
    }
}
```

#### 2. Repository’lar

- **OrderParamRepository** (card_delivery package’da):

```java
@Repository
public interface OrderParamRepository extends JpaRepository<OrderParam, Long> {

    // Branch bilan JOIN uchun projection
    @Query("SELECT op.id AS id, op.userId AS userId, ... , b.localCode AS bxmCode " +  // barcha DTO field'lari
           "FROM OrderParam op LEFT JOIN Branch b ON op.branchId = b.id " +
           "WHERE ...")  // filterlar bu yerda
    Page<OrderParamProjection> findAllWithBranch(Pageable pageable, /* filter params */);

    // Projection interface
    interface OrderParamProjection {
        Long getId();
        Long getUserId();
        // boshqa field'lar...
        String getBxmCode();  // branch.localCode
    }
}
```

- **TransactionRepository** (sqb_mobile package’da):

```java
@Repository
public interface TransactionRepository extends JpaRepository<Transaction, Long> {

    // Batch olish uchun
    List<Transaction> findByOperationIdIn(List<Long> operationIds);
}
```

#### 3. Filter va Search DTO

Filter uchun maxsus DTO yaratish:

```java
public record OrderSearchDTO(
        OrderStatus status,
        LocalDateTime createdFrom,
        LocalDateTime createdTo,
        Boolean paymentIsDone,
        Boolean provodkaIsDone,
        // boshqa filter'lar: phone, pnfl va h.k.
        Pageable pageable
) {}
```

#### 4. Service’da Aggregation Logic

**OrderServiceImpl**:

```java
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {

    private final OrderParamRepository orderParamRepository;
    private final TransactionRepository transactionRepository;

    @Override
    public Page<OrderParamDTOForAdminPanel> findAllOrders(OrderSearchDTO search) {
        // Step 1: OrderParam va Branch JOIN bilan olish (filter va pageable qo'llash)
        Specification<OrderParam> spec = buildSpecification(search);  // dinamik filter
        Page<OrderParam> orderPage = orderParamRepository.findAll(spec, search.pageable());

        // Step 2: Operation ID'larni to'plash
        List<Long> operationIds = orderPage.getContent().stream()
                .map(OrderParam::getOperationId)
                .filter(Objects::nonNull)
                .toList();

        // Step 3: Transaction'larni batch olish
        Map<Long, List<Transaction>> transactionsMap = transactionRepository.findByOperationIdIn(operationIds)
                .stream()
                .collect(Collectors.groupingBy(Transaction::getOperationId));

        // Step 4: DTO'larni qurish va Transaction filter'lari qo'llash (post-filter)
        List<OrderParamDTOForAdminPanel> dtos = orderPage.getContent().stream()
                .map(op -> mapToDTO(op, transactionsMap.getOrDefault(op.getOperationId(), List.of())))
                .filter(dto -> applyTransactionFilters(dto, search))  // paymentIsDone va provodkaIsDone filter
                .toList();

        // Step 5: Custom Page qaytarish (post-filter tufayli count o'zgarishi mumkin)
        return new PageImpl<>(dtos, search.pageable(), dtos.size());  // real holatda total count'ni qayta hisoblash mumkin
    }

    private Specification<OrderParam> buildSpecification(OrderSearchDTO search) {
        return Specification.where(
                statusEquals(search.status())
        ).and(createdBetween(search.createdFrom(), search.createdTo()))
        // boshqa OrderParam filter'lari: phone, pnfl va h.k.
        ;
    }

    private Specification<OrderParam> statusEquals(OrderStatus status) {
        return (root, query, cb) -> status == null ? cb.conjunction() : cb.equal(root.get("status"), status);
    }

    private Specification<OrderParam> createdBetween(LocalDateTime from, LocalDateTime to) {
        return (root, query, cb) -> {
            if (from == null && to == null) return cb.conjunction();
            if (from == null) return cb.lessThanOrEqualTo(root.get("createdAt"), to);
            if (to == null) return cb.greaterThanOrEqualTo(root.get("createdAt"), from);
            return cb.between(root.get("createdAt"), from, to);
        };
    }

    private OrderParamDTOForAdminPanel mapToDTO(OrderParam op, List<Transaction> transactions) {
        // Branch ma'lumotlari allaqachon op dan keladi (JOIN orqali)
        boolean paymentDone = transactions.stream().anyMatch(t -> t.getStatus() == TransactionStatus.DONE /* va boshqa shartlar */);
        boolean provodkaDone = transactions.stream().anyMatch(t -> t.getStatus() == TransactionStatus.DONE /* va server shartlari */);

        return new OrderParamDTOForAdminPanel(
                op.getId(),
                op.getUserId(),
                // phone, pnfl, fio - agar user table bo'lsa join qilish kerak, aks holda alohida service
                op.getCardTypeId().toString(),  // productCode
                op.getTitle(),  // product
                op.getContractId().toString(),
                op.getStatus(),
                op.getBranch().getLocalCode(),  // bxmCode (Branch dan)
                op.getCardActionType(),
                op.getCardAmount(),
                paymentDone,
                provodkaDone,
                op.getDeliveryType(),
                op.getCreatedAt()
        );
    }

    private boolean applyTransactionFilters(OrderParamDTOForAdminPanel dto, OrderSearchDTO search) {
        if (search.paymentIsDone() != null && !search.paymentIsDone().equals(dto.paymentIsDone())) return false;
        if (search.provodkaIsDone() != null && !search.provodkaIsDone().equals(dto.provodkaIsDone())) return false;
        return true;
    }
}
```

#### 5. Controller Misoli

```java
@GetMapping("/admin/orders")
public BaseResponse<Page<OrderParamDTOForAdminPanel>> getOrders(@Valid OrderSearchDTO search) {
    return BaseResponse.ok(orderService.findAllOrders(search));
}
```

#### 6. Performance va Optimizations

- **N+1 muammosi**: Batch `findByOperationIdIn` orqali hal qilindi.
- **Filter Optimization**: Transaction filter’lari kam bo‘lsa, post-filter yetarli. Agar ko‘p bo‘lsa, avval Transaction’dan filter qilib, operation_id’larni olish va OrderParam’da `WHERE operationId IN (...)` qilish mumkin.
- **Caching**: `RedisCacheUtil` dan foydalanib, butun page’ni cache qilish mumkin (`@Cacheable` bilan yoki util orqali).
- **Async**: Agar DB’lar sekin bo‘lsa, Transaction olishni `@Async` bilan qilish mumkin (CompletableFuture).
- **Error Handling**: `DataAccessException` ni tutib, custom error qaytarish.

Bu yechim **scalable** va **maintainable** — agar DB’lar ko‘payib ketsa, microservices’ga o‘tish oson.

Agar qo‘shimcha detallar (masalan phone/pnfl user table’dan olish) yoki kodni test qilish kerak bo‘lsa, yozing!
