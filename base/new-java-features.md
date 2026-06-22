Ниже — **рабочие примеры кода для каждой фичи Java 17+** с банковским контекстом. Каждый пример можно скопировать и использовать как шпаргалку.

---

## 1. RECORDS (Java 16/17)

**Иммутабельный DTO для транзакции:**

```java
public record Transaction(
    String id,
    String accountId,
    BigDecimal amount,
    LocalDateTime date,
    TransactionType type
) {
    // Можно добавлять кастомные методы
    public boolean isCredit() {
        return type == TransactionType.CREDIT;
    }
}

public enum TransactionType {
    CREDIT, DEBIT
}

// Использование:
Transaction tx = new Transaction("tx123", "ACC001", BigDecimal.valueOf(100), LocalDateTime.now(), CREDIT);
String accountId = tx.accountId(); // геттер
```

---

## 2. SWITCH EXPRESSIONS (Java 14/17)

**Маппинг статуса платежа:**

```java
public String getStatusMessage(TransactionStatus status) {
    return switch (status) {
        case PENDING -> "Обработка...";
        case SUCCESS -> "Платёж выполнен";
        case FAILED -> "Ошибка платежа";
        case CANCELLED -> "Отменён пользователем";
        default -> "Неизвестный статус";
    };
}

// С блоками и yield:
public BigDecimal calculateFee(TransactionType type, BigDecimal amount) {
    return switch (type) {
        case CREDIT -> {
            BigDecimal fee = amount.multiply(BigDecimal.valueOf(0.01));
            yield fee;  // return внутри блока
        }
        case DEBIT -> {
            BigDecimal fee = amount.multiply(BigDecimal.valueOf(0.02));
            yield fee;
        }
    };
}
```

---

## 3. TEXT BLOCKS (Java 15/17)

**SQL-запрос с форматированием:**

```java
public List<Transaction> findRecentTransactions(String accountId, LocalDateTime fromDate) {
    String query = """
        SELECT t.id, t.amount, t.date, t.type
        FROM transaction t
        WHERE t.account_id = :accountId
          AND t.date >= :fromDate
        ORDER BY t.date DESC
        LIMIT 100
        """;
    
    return entityManager.createQuery(query, Transaction.class)
        .setParameter("accountId", accountId)
        .setParameter("fromDate", fromDate)
        .getResultList();
}

// JSON для внешнего API:
String jsonRequest = """
    {
        "accountId": "%s",
        "amount": %s,
        "currency": "RUB",
        "timestamp": "%s"
    }
    """.formatted(accountId, amount, LocalDateTime.now());
```

---

## 4. PATTERN MATCHING for INSTANCEOF (Java 16/17)

**Обработка разных типов событий:**

```java
public void handleEvent(Object event) {
    if (event instanceof PaymentSuccessEvent success) {
        // success — уже типизированная переменная
        log.info("Платёж {} успешен на сумму {}", success.id(), success.amount());
        notifyUser(success);
    } else if (event instanceof PaymentFailedEvent failed) {
        log.warn("Платёж {} провален: {}", failed.id(), failed.reason());
        refund(failed);
    } else if (event instanceof FraudAlertEvent fraud) {
        log.error("Фрод на счёте {}: {}", fraud.accountId(), fraud.message());
        lockAccount(fraud.accountId());
    } else {
        log.warn("Неизвестное событие: {}", event);
    }
}

// Использование в Stream:
transactions.stream()
    .filter(tx -> tx instanceof PaymentSuccessEvent)
    .map(tx -> (PaymentSuccessEvent) tx)  // старый каст
    .toList();

// С pattern matching можно упростить
transactions.stream()
    .map(tx -> tx instanceof PaymentSuccessEvent success ? success : null)
    .filter(Objects::nonNull)
    .toList();
```

---

## 5. PATTERN MATCHING for SWITCH (Java 21)

**Обработка иерархии sealed классов:**

```java
// 1. Объявляем sealed иерархию
public sealed interface PaymentEvent 
    permits PaymentSuccessEvent, PaymentFailedEvent, FraudAlertEvent {
    String id();
}

public record PaymentSuccessEvent(String id, BigDecimal amount) implements PaymentEvent {}
public record PaymentFailedEvent(String id, String reason) implements PaymentEvent {}
public record FraudAlertEvent(String id, String accountId, String message) implements PaymentEvent {}

// 2. Используем pattern matching в switch
public String handlePaymentEvent(PaymentEvent event) {
    return switch (event) {
        case PaymentSuccessEvent e -> "✅ Успех: " + e.amount() + " руб.";
        case PaymentFailedEvent e -> "❌ Ошибка: " + e.reason();
        case FraudAlertEvent e -> "🚨 Фрод на счёте " + e.accountId() + ": " + e.message();
        // default не нужен, т.к. все варианты перечислены
    };
}

// С guards (дополнительными условиями):
public BigDecimal calculateBonus(PaymentEvent event) {
    return switch (event) {
        case PaymentSuccessEvent e when e.amount().compareTo(BigDecimal.valueOf(1000)) > 0 
            -> e.amount().multiply(BigDecimal.valueOf(0.05)); // 5% бонус для крупных платежей
        case PaymentSuccessEvent e 
            -> e.amount().multiply(BigDecimal.valueOf(0.01)); // 1% бонус для всех остальных
        default -> BigDecimal.ZERO;
    };
}
```

---

## 6. SEALED CLASSES (Java 17)

**Ограниченная иерархия для статусов:**

```java
// sealed interface с 3 разрешёнными реализациями
public sealed interface AccountStatus 
    permits ActiveStatus, BlockedStatus, ClosedStatus {
    String getDescription();
    boolean isOperable();
}

// Каждая реализация — record или класс
public record ActiveStatus(LocalDateTime openedAt) implements AccountStatus {
    @Override
    public String getDescription() {
        return "Счёт активен с " + openedAt;
    }
    
    @Override
    public boolean isOperable() {
        return true;
    }
}

public record BlockedStatus(String reason, LocalDateTime blockedAt) implements AccountStatus {
    @Override
    public String getDescription() {
        return "Заблокирован: " + reason + " (" + blockedAt + ")";
    }
    
    @Override
    public boolean isOperable() {
        return false;
    }
}

public record ClosedStatus(LocalDateTime closedAt) implements AccountStatus {
    @Override
    public String getDescription() {
        return "Закрыт " + closedAt;
    }
    
    @Override
    public boolean isOperable() {
        return false;
    }
}

// Использование:
AccountStatus status = new BlockedStatus("Подозрительная активность", LocalDateTime.now());
System.out.println(status.getDescription()); // "Заблокирован: Подозрительная активность..."
```

---

## 7. HTTP CLIENT (Java 11+)

**Асинхронный вызов банковского шлюза:**

```java
public class BankGatewayClient {
    private final HttpClient httpClient = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(5))
        .followRedirects(HttpClient.Redirect.NORMAL)
        .build();
    
    // Асинхронный запрос
    public CompletableFuture<String> processPayment(String accountId, BigDecimal amount) {
        String json = """
            {
                "accountId": "%s",
                "amount": %s
            }
            """.formatted(accountId, amount);
        
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://bank-gateway.com/api/v1/charge"))
            .header("Content-Type", "application/json")
            .header("Authorization", "Bearer " + getToken())
            .timeout(Duration.ofSeconds(10))
            .POST(HttpRequest.BodyPublishers.ofString(json))
            .build();
        
        return httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString())
            .thenApply(HttpResponse::body)
            .exceptionally(ex -> {
                log.error("Ошибка вызова шлюза", ex);
                return "ERROR";
            });
    }
}
```

---

## 8. VIRTUAL THREADS (Java 21)

**Обработка пачки транзакций с виртуальными потоками:**

```java
public class PaymentProcessor {
    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
    
    public void processBatch(List<Transaction> transactions) {
        List<CompletableFuture<Void>> futures = transactions.stream()
            .map(tx -> CompletableFuture.runAsync(() -> {
                try {
                    processTransaction(tx); // I/O-операция
                } catch (Exception e) {
                    log.error("Ошибка обработки {}", tx.id(), e);
                }
            }, executor))
            .toList();
        
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .join();
    }
    
    private void processTransaction(Transaction tx) {
        // Вызов БД, внешнего API — I/O
        accountService.apply(tx);
    }
}

// Веб-контроллер с виртуальными потоками в Spring (Java 21):
@RestController
public class AccountController {
    @GetMapping("/accounts/{id}")
    public CompletableFuture<Account> getAccount(@PathVariable Long id) {
        // Tomcat уже использует виртуальные потоки (если настроен)
        return CompletableFuture.supplyAsync(
            () -> accountService.findById(id),
            Executors.newVirtualThreadPerTaskExecutor()
        );
    }
}
```

**Настройка Spring Boot для виртуальных потоков:**
```properties
# application.properties
spring.threads.virtual.enabled=true
server.tomcat.threads.max=100000  # виртуальные потоки почти безлимитны
```

---

## 9. SEQUENCED COLLECTIONS (Java 21)

**Работа с историей транзакций:**

```java
public class TransactionHistory {
    // SequencedCollection — добавляет getFirst() и getLast()
    private final SequencedCollection<Transaction> transactions;
    
    public TransactionHistory(List<Transaction> txs) {
        // Если у вас List, он уже реализует SequencedCollection
        this.transactions = txs;
    }
    
    public Transaction getLatestTransaction() {
        return transactions.getLast();
    }
    
    public Transaction getEarliestTransaction() {
        return transactions.getFirst();
    }
    
    public void addTransaction(Transaction tx) {
        transactions.addLast(tx); // вместо add
    }
    
    public void removeLatest() {
        transactions.removeLast();
    }
}

// Для Map:
SequencedMap<String, BigDecimal> balances = new LinkedHashMap<>();
balances.put("ACC001", BigDecimal.valueOf(100));
balances.put("ACC002", BigDecimal.valueOf(200));

// Получить первый и последний элементы (по порядку вставки)
String firstKey = balances.firstEntry().getKey();    // "ACC001"
String lastKey = balances.lastEntry().getKey();      // "ACC002"
```

---

## 10. OPTIONAL.STREAM() (Java 9+)

**Преобразование списка Optional в список значений:**

```java
public class AccountService {
    private final Map<String, Account> accountCache = new HashMap<>();
    
    public List<Account> getActiveAccounts(List<String> accountIds) {
        return accountIds.stream()
            .map(this::findAccount)        // String -> Optional<Account>
            .flatMap(Optional::stream)     // убираем пустые Optional
            .filter(Account::isActive)
            .collect(Collectors.toList());
    }
    
    private Optional<Account> findAccount(String id) {
        return Optional.ofNullable(accountCache.get(id));
    }
}
```

---

## 11. FILES API (современный)

**Чтение/запись файлов для отчётов:**

```java
public class ReportService {
    private static final Path REPORTS_DIR = Path.of("/var/reports");
    
    public String generateDailyReport(LocalDate date) throws IOException {
        String report = """
            Отчёт за %s
            ==================
            Всего транзакций: %d
            Общая сумма: %.2f руб.
            Количество счетов: %d
            """.formatted(date, getCount(date), getTotal(date), getAccounts(date));
        
        // Запись
        Path reportFile = REPORTS_DIR.resolve("report_%s.txt".formatted(date));
        Files.writeString(reportFile, report);
        
        // Чтение
        return Files.readString(reportFile);
    }
}
```

---

## 12. МОДУЛЬНАЯ СИСТЕМА (Java 9+)

**module-info.java для банковского сервиса:**

```java
module com.bank.account.service {
    requires spring.boot;
    requires spring.boot.autoconfigure;
    requires spring.data.jpa;
    requires org.hibernate.orm.core;
    requires java.sql;
    requires org.slf4j;
    
    // Экспортируем только нужные пакеты
    exports com.bank.account.service.api;
    exports com.bank.account.service.dto;
    
    // Открываем для reflection (JPA)
    opens com.bank.account.service.entity to org.hibernate.orm.core;
    opens com.bank.account.service.config to spring.core;
}
```

---

## ИТОГОВАЯ ШПАРГАЛКА (КОД)

```java
// 1. Record
record Transaction(String id, BigDecimal amount) {}

// 2. Switch Expression
String msg = switch(status) { case SUCCESS -> "OK"; default -> "FAIL"; };

// 3. Text Block
String sql = """ SELECT * FROM accounts """;

// 4. Pattern Matching instanceof
if (obj instanceof Account acc) { process(acc); }

// 5. Pattern Matching switch
switch(event) { case SuccessEvent e -> handle(e); }

// 6. Sealed Classes
sealed interface Event permits SuccessEvent, FailedEvent {}

// 7. HttpClient
client.sendAsync(request, BodyHandlers.ofString());

// 8. Virtual Threads
Executors.newVirtualThreadPerTaskExecutor();

// 9. Sequenced Collections
list.getFirst(); list.getLast();

// 10. Optional.stream()
optional.stream().flatMap(Optional::stream);

// 11. Files
Files.readString(Path.of("file.txt"));
```

---

Теперь у тебя есть готовый код по каждой фиче. Скопируй в IDE, поиграйся — и на собеседовании будешь отвечать уверенно! 🚀
