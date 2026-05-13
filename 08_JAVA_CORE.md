# Java Core — Lý thuyết sâu + Ví dụ 3 cấp độ

---

## 1. OOP — 4 TÍNH CHẤT

### Lý thuyết & Bản chất

**OOP (Object-Oriented Programming)** xoay quanh 4 tính chất cốt lõi:

| Tính chất | Bản chất 1 câu |
|-----------|----------------|
| **Encapsulation** | Gói dữ liệu và hành vi vào object, ẩn chi tiết nội bộ |
| **Inheritance** | Class con kế thừa fields + methods từ class cha |
| **Polymorphism** | Cùng 1 interface, nhiều hành vi khác nhau tùy implementation |
| **Abstraction** | Che giấu implementation, chỉ expose những gì cần thiết |

---

### Ví dụ 1 — Cơ bản: Encapsulation + Inheritance
```java
// Encapsulation: private fields, public getters/setters
public class BankAccount {
    private final String accountNumber; // immutable
    private BigDecimal balance;
    private final List<String> transactions = new ArrayList<>();

    public BankAccount(String accountNumber, BigDecimal initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    // Controlled access — không cho set balance trực tiếp
    public BigDecimal getBalance() { return balance; }

    public void deposit(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("Amount must be positive");
        balance = balance.add(amount);
        transactions.add("DEPOSIT: +" + amount);
    }

    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(balance) > 0)
            throw new IllegalStateException("Insufficient funds");
        balance = balance.subtract(amount);
        transactions.add("WITHDRAW: -" + amount);
    }

    // Trả defensive copy — không cho sửa internal list
    public List<String> getTransactions() {
        return Collections.unmodifiableList(transactions);
    }
}

// Inheritance
public class SavingsAccount extends BankAccount {
    private final double interestRate;

    public SavingsAccount(String number, BigDecimal balance, double rate) {
        super(number, balance); // gọi constructor cha
        this.interestRate = rate;
    }

    public void applyInterest() {
        BigDecimal interest = getBalance().multiply(BigDecimal.valueOf(interestRate));
        deposit(interest); // dùng method của cha
    }
}
```

---

### Ví dụ 2 — Trung cấp: Polymorphism + Abstract class vs Interface
```java
// Abstract class: có thể có implementation, state, constructor
// Dùng khi: các subclasses chia sẻ code chung
public abstract class Shape {
    private String color;

    public Shape(String color) { this.color = color; }

    // Abstract method: subclass BẮT BUỘC override
    public abstract double area();
    public abstract double perimeter();

    // Concrete method: subclass KHÔNG cần override
    public String describe() {
        return String.format("%s %s: area=%.2f", color, getClass().getSimpleName(), area());
    }
}

// Interface: không có state (chỉ constants), Java 8+ có default methods
// Dùng khi: define contract, unrelated classes implement cùng behavior
public interface Drawable {
    void draw(Canvas canvas); // abstract by default
    default void drawWithBorder(Canvas canvas) { // default method (Java 8+)
        draw(canvas);
        drawBorder(canvas);
    }
}

public interface Resizable {
    void resize(double factor);
}

// Class có thể implements nhiều interfaces nhưng chỉ extends 1 class
public class Circle extends Shape implements Drawable, Resizable {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override public double area() { return Math.PI * radius * radius; }
    @Override public double perimeter() { return 2 * Math.PI * radius; }
    @Override public void draw(Canvas canvas) { canvas.drawCircle(radius); }
    @Override public void resize(double factor) { radius *= factor; }
}

// Polymorphism: xử lý nhiều types qua chung interface
List<Shape> shapes = List.of(
    new Circle("red", 5),
    new Rectangle("blue", 4, 6),
    new Triangle("green", 3, 4, 5)
);

// Gọi area() → mỗi shape tính theo cách riêng
double totalArea = shapes.stream()
    .mapToDouble(Shape::area) // method reference
    .sum();
```

---

### Ví dụ 3 — Nâng cao: Interface vs Abstract, Diamond problem, Composition over Inheritance
```java
// Diamond problem với interfaces (Java 8+)
interface A { default String hello() { return "Hello from A"; } }
interface B extends A { default String hello() { return "Hello from B"; } }
interface C extends A { default String hello() { return "Hello from C"; } }

class D implements B, C {
    // Compiler lỗi: "inherits unrelated defaults"
    // PHẢI override để resolve
    @Override
    public String hello() {
        return B.super.hello(); // chọn rõ ràng implementation nào
    }
}

// Composition over Inheritance: prefer HAS-A over IS-A
// BAD: inheritance cho code reuse
class LoggingUserService extends UserService { // tight coupling
    @Override
    public User findById(Long id) {
        log.info("Finding user {}", id);
        return super.findById(id);
    }
}

// GOOD: composition
@Service
@RequiredArgsConstructor
public class LoggingUserService implements UserServicePort {
    private final UserService delegate;  // HAS-A
    private final Logger log;

    @Override
    public User findById(Long id) {
        log.info("Finding user {}", id);
        User user = delegate.findById(id);
        log.info("Found user: {}", user.getEmail());
        return user;
    }
}

// Sealed classes (Java 17+) — restrict inheritance
public sealed class Result<T> permits Success, Failure {
    public abstract boolean isSuccess();
}
public final class Success<T> extends Result<T> {
    private final T value;
    public boolean isSuccess() { return true; }
    public T getValue() { return value; }
}
public final class Failure<T> extends Result<T> {
    private final Exception error;
    public boolean isSuccess() { return false; }
    public Exception getError() { return error; }
}

// Pattern matching với switch (Java 21)
String describe(Result<?> result) {
    return switch (result) {
        case Success<?> s -> "Got: " + s.getValue();
        case Failure<?> f -> "Error: " + f.getError().getMessage();
    };
}
```

---

## 2. GENERICS

### Lý thuyết & Bản chất
Generics cho phép viết **type-safe code** hoạt động với nhiều types mà không cần casting. Compiler kiểm tra types tại **compile time** thay vì runtime.

**Type Erasure:** Java xóa type parameters khi compile — `List<String>` và `List<Integer>` đều thành `List` tại runtime. Đây là lý do Java generics có một số hạn chế so với TypeScript.

**Bounded wildcards:**
- `<T extends Foo>` — upper bound: T là Foo hoặc subtype
- `<? super Foo>` — lower bound: ? là Foo hoặc supertype
- `<?>` — unbounded wildcard

**PECS rule** (Producer Extends, Consumer Super):
- `List<? extends T>` — đọc từ (producer)
- `List<? super T>` — ghi vào (consumer)

---

### Ví dụ 1 — Cơ bản: Generic class + method
```java
// Generic class
public class Pair<A, B> {
    private final A first;
    private final B second;

    public Pair(A first, B second) {
        this.first = first;
        this.second = second;
    }

    public A getFirst() { return first; }
    public B getSecond() { return second; }

    // Generic method trong non-generic class
    public static <T> List<T> repeat(T value, int times) {
        return Collections.nCopies(times, value);
    }

    // Swap type — type inference
    public Pair<B, A> swap() {
        return new Pair<>(second, first);
    }
}

// Generic method với bounded type
public static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

max(3, 5);           // 5
max("apple", "banana"); // "banana"
```

---

### Ví dụ 2 — Trung cấp: Wildcards + PECS
```java
// Upper bounded wildcard: đọc từ list (producer)
public static double sumList(List<? extends Number> list) {
    // List<Integer>, List<Double>, List<Long> đều OK
    return list.stream().mapToDouble(Number::doubleValue).sum();
}

// Lower bounded wildcard: ghi vào list (consumer)
public static void addNumbers(List<? super Integer> list) {
    // List<Integer>, List<Number>, List<Object> đều OK
    list.add(1);
    list.add(2);
    list.add(3);
}

// PECS trong action
public static <T> void copy(List<? extends T> src, List<? super T> dest) {
    // src là producer → extends
    // dest là consumer → super
    dest.addAll(src);
}

List<Integer> ints = List.of(1, 2, 3);
List<Number> nums = new ArrayList<>();
copy(ints, nums); // ✅

// Generic Stack implementation
public class Stack<T> {
    private final Deque<T> storage = new ArrayDeque<>();

    public void push(T item) { storage.push(item); }
    public T pop() {
        if (isEmpty()) throw new NoSuchElementException("Stack is empty");
        return storage.pop();
    }
    public T peek() { return storage.peek(); }
    public boolean isEmpty() { return storage.isEmpty(); }

    // Wildcard: accept Stack<Integer>, Stack<Double>...
    public void pushAll(Stack<? extends T> other) {
        while (!other.isEmpty()) push(other.pop());
    }
}
```

---

### Ví dụ 3 — Nâng cao: Generic với Reflection + Type Token
```java
// Type erasure problem: không thể làm new T() hoặc T.class
// Solution: Type Token pattern

public class TypeSafeContainer {
    private final Map<Class<?>, Object> map = new HashMap<>();

    public <T> void put(Class<T> type, T instance) {
        map.put(type, instance);
    }

    public <T> T get(Class<T> type) {
        return type.cast(map.get(type)); // safe cast với Class object
    }
}

TypeSafeContainer container = new TypeSafeContainer();
container.put(String.class, "Hello");
container.put(Integer.class, 42);
String s = container.get(String.class); // type-safe, no ClassCastException

// Generic Result type (như Either monad)
public sealed interface Result<T, E extends Exception>
    permits Result.Ok, Result.Err {

    record Ok<T, E extends Exception>(T value) implements Result<T, E> {}
    record Err<T, E extends Exception>(E error) implements Result<T, E> {}

    static <T, E extends Exception> Result<T, E> of(T value) {
        return new Ok<>(value);
    }
    static <T, E extends Exception> Result<T, E> fail(E error) {
        return new Err<>(error);
    }

    default boolean isOk() { return this instanceof Ok; }

    @SuppressWarnings("unchecked")
    default <R> Result<R, E> map(Function<T, R> fn) {
        return switch (this) {
            case Ok<T, E> ok -> Result.of(fn.apply(ok.value()));
            case Err<T, E> err -> (Result<R, E>) err;
        };
    }
}
```

---

## 3. COLLECTIONS FRAMEWORK

### Lý thuyết & Bản chất

```
Collection
├── List (ordered, allows duplicates)
│   ├── ArrayList   — dynamic array, O(1) get, O(n) insert
│   ├── LinkedList  — doubly linked, O(1) add/remove head/tail, O(n) get
│   └── Vector      — synchronized ArrayList (legacy)
│
├── Set (no duplicates)
│   ├── HashSet     — hash table, O(1) avg, unordered
│   ├── LinkedHashSet — hash + linked list, insertion order
│   └── TreeSet     — red-black tree, sorted, O(log n)
│
└── Queue / Deque
    ├── ArrayDeque  — resizable array, fast stack/queue
    ├── PriorityQueue — min-heap, O(log n) offer/poll
    └── LinkedList  — implements Deque

Map (key-value)
├── HashMap         — hash table, O(1) avg, unordered
├── LinkedHashMap   — insertion order
├── TreeMap         — sorted by key, O(log n)
├── Hashtable       — synchronized (legacy)
└── ConcurrentHashMap — thread-safe, segment locking
```

---

### Ví dụ 1 — Cơ bản: Chọn đúng collection
```java
// Khi nào dùng gì?

// ArrayList: truy cập random, thêm cuối nhiều
List<String> names = new ArrayList<>();
names.add("Alice");       // O(1) amortized
names.get(0);             // O(1)
names.add(0, "Zoe");      // O(n) — shift elements

// LinkedList: thêm/xóa đầu/giữa nhiều
Deque<Task> taskQueue = new LinkedList<>();
taskQueue.addFirst(urgentTask); // O(1)
taskQueue.pollLast();           // O(1)

// HashSet: kiểm tra exists nhanh
Set<String> visited = new HashSet<>();
visited.add("page1");    // O(1)
visited.contains("page1"); // O(1) avg

// TreeSet: cần sorted + range queries
TreeSet<Integer> sorted = new TreeSet<>();
sorted.add(3); sorted.add(1); sorted.add(2);
sorted.first();              // 1
sorted.headSet(3);           // {1, 2} — elements < 3
sorted.subSet(1, 3);         // {1, 2}

// PriorityQueue: luôn xử lý element "nhỏ nhất" trước
PriorityQueue<Task> pq = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
);
pq.offer(new Task("low", 3));
pq.offer(new Task("urgent", 1));
pq.poll(); // lấy Task với priority = 1 (min-heap)
```

---

### Ví dụ 2 — Trung cấp: HashMap internals + equals/hashCode contract
```java
// HashMap hoạt động như thế nào?
// 1. key.hashCode() → bucket index
// 2. Nếu collision → linked list (Java 7) hoặc red-black tree khi > 8 nodes (Java 8+)
// 3. key.equals() để tìm đúng entry trong bucket

// equals/hashCode CONTRACT:
// - Nếu a.equals(b) == true → a.hashCode() == b.hashCode() (BẮT BUỘC)
// - Nếu a.hashCode() == b.hashCode() không nhất thiết a.equals(b) (collision OK)

public class Student {
    private final String id;
    private String name;

    // IDE-generated hoặc Lombok @EqualsAndHashCode
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Student other)) return false;
        return Objects.equals(id, other.id); // chỉ dùng id để compare
    }

    @Override
    public int hashCode() {
        return Objects.hash(id); // nhất quán với equals
    }
}

// BUG: dùng mutable field trong hashCode → item "mất" trong HashMap
Student s = new Student("S001", "Alice");
Map<Student, Integer> grades = new HashMap<>();
grades.put(s, 90);
s.setName("Bob"); // nếu name trong hashCode → bucket thay đổi!
grades.get(s);    // null! (tìm ở bucket mới, không thấy)

// Giải pháp: chỉ dùng immutable fields trong equals/hashCode
// Hoặc dùng id (primary key) — best practice

// Merging, computing
Map<String, Integer> wordCount = new HashMap<>();
// BAD:
// if (!map.containsKey(word)) map.put(word, 0);
// map.put(word, map.get(word) + 1);

// GOOD: merge
words.forEach(word ->
    wordCount.merge(word, 1, Integer::sum) // key, value, remapping fn
);
// GOOD: compute
wordCount.computeIfAbsent("java", k -> new ArrayList<>()).add("occurrence");
```

---

### Ví dụ 3 — Nâng cao: ConcurrentHashMap + Collections utilities
```java
// Thread-safe alternatives
// Hashtable: synchronized toàn bộ → chậm
// Collections.synchronizedMap(): synchronized toàn bộ
// ConcurrentHashMap: segment locking → nhanh hơn nhiều

ConcurrentHashMap<String, Integer> concurrent = new ConcurrentHashMap<>();
// Atomic operations
concurrent.putIfAbsent("key", 1);
concurrent.computeIfAbsent("list", k -> 0);
concurrent.merge("count", 1, Integer::sum); // atomic increment

// ConcurrentHashMap KHÔNG allow null key/value (HashMap cho phép null)

// Unmodifiable vs Immutable
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

List<String> unmodifiable = Collections.unmodifiableList(list);
// unmodifiable.add("d"); // UnsupportedOperationException
list.add("d"); // list gốc vẫn thay đổi được!
System.out.println(unmodifiable.size()); // 4 — view thay đổi theo!

List<String> immutable = List.of("a", "b", "c"); // Java 9+
// immutable.add("d"); // UnsupportedOperationException
// Thực sự immutable — không có reference đến mutable source

// Comparable vs Comparator
// Comparable: natural ordering, implement trong class
public class Employee implements Comparable<Employee> {
    private String name;
    private int salary;

    @Override
    public int compareTo(Employee other) {
        return this.name.compareTo(other.name); // sort by name by default
    }
}

// Comparator: external ordering, flexible
Comparator<Employee> bySalary = Comparator.comparingInt(Employee::getSalary);
Comparator<Employee> byNameThenSalary = Comparator
    .comparing(Employee::getName)
    .thenComparingInt(Employee::getSalary)
    .reversed();

employees.sort(byNameThenSalary);
```

---

## 4. FUNCTIONAL PROGRAMMING — STREAMS & LAMBDAS

### Lý thuyết & Bản chất
Java 8 giới thiệu **Stream API** và **Lambda expressions**, mang functional programming vào Java.

**Stream:** Không phải data structure — là **pipeline xử lý data** theo kiểu lazy. Stream không lưu data, không modify source.

**3 loại operations:**
- **Intermediate** (lazy, return Stream): `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `peek`
- **Terminal** (trigger execution, return result): `collect`, `forEach`, `reduce`, `count`, `anyMatch`, `findFirst`
- **Short-circuit**: `findFirst`, `anyMatch`, `allMatch`, `noneMatch`, `limit` — dừng sớm khi đủ

**Functional Interfaces quan trọng:**

| Interface | Method | Dùng cho |
|-----------|--------|---------|
| `Predicate<T>` | `boolean test(T t)` | `filter()` |
| `Function<T,R>` | `R apply(T t)` | `map()` |
| `Consumer<T>` | `void accept(T t)` | `forEach()` |
| `Supplier<T>` | `T get()` | lazy init |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | `reduce()` |
| `UnaryOperator<T>` | `T apply(T t)` | `map()` khi same type |

---

### Ví dụ 1 — Cơ bản: Stream operations
```java
List<Employee> employees = List.of(
    new Employee("Alice", "IT", 80000),
    new Employee("Bob", "IT", 90000),
    new Employee("Carol", "HR", 60000),
    new Employee("Dave", "HR", 70000),
    new Employee("Eve", "IT", 85000)
);

// Filter + map + collect
List<String> itNames = employees.stream()
    .filter(e -> "IT".equals(e.getDepartment()))
    .filter(e -> e.getSalary() > 80000)
    .map(Employee::getName)           // method reference
    .sorted()
    .collect(Collectors.toList());    // [Bob, Eve]

// Statistics
OptionalDouble avgSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .average();

DoubleSummaryStatistics stats = employees.stream()
    .mapToDouble(Employee::getSalary)
    .summaryStatistics();
// count, sum, min, max, average

// Short-circuit: dừng sớm
boolean hasHighEarner = employees.stream()
    .anyMatch(e -> e.getSalary() > 100_000); // false, nhưng không scan hết

Optional<Employee> firstIT = employees.stream()
    .filter(e -> "IT".equals(e.getDepartment()))
    .findFirst(); // trả về ngay khi tìm thấy đầu tiên
```

---

### Ví dụ 2 — Trung cấp: Collectors + groupingBy + reduce
```java
// groupingBy — phổ biến nhất trong interview
Map<String, List<Employee>> byDepartment = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));
// { "IT": [Alice, Bob, Eve], "HR": [Carol, Dave] }

// groupingBy + downstream collector
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));
// { "IT": 85000.0, "HR": 65000.0 }

Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));

// partitioningBy — chia 2 nhóm
Map<Boolean, List<Employee>> partition = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 75000));
// { true: [Alice, Bob, Eve], false: [Carol, Dave] }

// toMap
Map<String, Double> salaryMap = employees.stream()
    .collect(Collectors.toMap(
        Employee::getName,
        Employee::getSalary,
        (existing, replacement) -> existing // merge fn khi duplicate key
    ));

// reduce — custom aggregation
Optional<Employee> highestPaid = employees.stream()
    .reduce((e1, e2) -> e1.getSalary() > e2.getSalary() ? e1 : e2);

// flatMap — flatten nested collections
List<String> allSkills = employees.stream()
    .flatMap(e -> e.getSkills().stream()) // Employee có List<String> skills
    .distinct()
    .sorted()
    .collect(Collectors.toList());
```

---

### Ví dụ 3 — Nâng cao: Custom Collector + Parallel Stream + Method References
```java
// Custom Collector
// Ví dụ: collect thành ImmutableList (như Guava)
public class ImmutableListCollector<T>
        implements Collector<T, List<T>, List<T>> {

    @Override
    public Supplier<List<T>> supplier() {
        return ArrayList::new;
    }

    @Override
    public BiConsumer<List<T>, T> accumulator() {
        return List::add;
    }

    @Override
    public BinaryOperator<List<T>> combiner() {
        return (l1, l2) -> { l1.addAll(l2); return l1; };
    }

    @Override
    public Function<List<T>, List<T>> finisher() {
        return Collections::unmodifiableList; // wrap khi xong
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Set.of(); // không IDENTITY_FINISH vì cần finisher
    }
}

// Parallel Stream — xử lý song song trên ForkJoinPool.commonPool()
long count = IntStream.range(0, 1_000_000)
    .parallel()
    .filter(n -> isPrime(n))
    .count();
// Tự động chia work cho multiple cores

// QUAN TRỌNG: Parallel stream không phải lúc nào cũng nhanh hơn
// Chi phí: splitting data, merging results, thread coordination
// Nên dùng khi: data lớn, computation nặng, thứ tự không quan trọng
// Không nên dùng khi: I/O-bound operations, có side effects, data nhỏ

// Method references — 4 loại
// 1. Static method: ClassName::staticMethod
Function<String, Integer> parse = Integer::parseInt;

// 2. Instance method of particular object: instance::method
String prefix = "Hello ";
Function<String, String> greet = prefix::concat;

// 3. Instance method of arbitrary object: ClassName::instanceMethod
Function<String, String> upper = String::toUpperCase;
Comparator<String> cmp = String::compareTo;

// 4. Constructor: ClassName::new
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<String, StringBuilder> sbFactory = StringBuilder::new;
```

---

## 5. EXCEPTION HANDLING

### Lý thuyết & Bản chất

```
Throwable
├── Error         — JVM-level, không nên catch (OutOfMemoryError, StackOverflowError)
└── Exception
    ├── Checked   — phải handle (IOException, SQLException)
    │   (compile-time check, khai báo trong throws clause)
    └── RuntimeException (Unchecked) — không bắt buộc handle
        (NullPointerException, ArrayIndexOutOfBoundsException, IllegalArgumentException...)
```

**Khi nào dùng loại nào?**
- **Checked**: caller CÓ THỂ recover (file not found → dùng default, network error → retry)
- **Unchecked**: programming errors, không expect caller recover (null arg → fix code)

---

### Ví dụ 1 — Cơ bản: try-catch-finally + try-with-resources
```java
// try-with-resources (Java 7+) — AutoCloseable tự đóng
public String readFile(String path) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(path));
         // Nhiều resources: đóng theo thứ tự ngược
         InputStream is = new FileInputStream(path)) {

        return reader.lines().collect(Collectors.joining("\n"));
    }
    // reader.close() được gọi tự động dù có exception hay không
}

// Multi-catch (Java 7+)
try {
    process(data);
} catch (IOException | SQLException e) {
    log.error("Data error", e);
    throw new DataProcessingException("Failed to process", e); // wrap và rethrow
} catch (Exception e) {
    log.error("Unexpected error", e);
    throw e; // rethrow nguyên
} finally {
    cleanup(); // luôn chạy, kể cả khi có return trong try
}

// Exception chaining — giữ root cause
public User findUser(String email) {
    try {
        return userRepository.findByEmail(email);
    } catch (DataAccessException e) {
        // Wrap DB exception thành domain exception, giữ cause
        throw new UserNotFoundException("User not found: " + email, e);
    }
}
```

---

### Ví dụ 2 — Trung cấp: Custom exceptions + Exception hierarchy
```java
// Custom exception hierarchy
public abstract class AppException extends RuntimeException {
    private final String errorCode;

    protected AppException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    protected AppException(String message, String errorCode, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

// Domain-specific exceptions
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resource, Object id) {
        super(String.format("%s with id '%s' not found", resource, id), "RESOURCE_NOT_FOUND");
    }
}

public class BusinessRuleException extends AppException {
    public BusinessRuleException(String message) {
        super(message, "BUSINESS_RULE_VIOLATION");
    }
}

public class DuplicateResourceException extends AppException {
    public DuplicateResourceException(String resource, String field, Object value) {
        super(String.format("%s with %s '%s' already exists", resource, field, value),
              "DUPLICATE_RESOURCE");
    }
}

// Dùng
@Service
public class UserService {
    public User findById(Long id) {
        return userRepo.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    public User create(CreateUserRequest req) {
        if (userRepo.existsByEmail(req.email())) {
            throw new DuplicateResourceException("User", "email", req.email());
        }
        if (req.age() < 18) {
            throw new BusinessRuleException("User must be at least 18 years old");
        }
        return userRepo.save(User.from(req));
    }
}
```

---

### Ví dụ 3 — Nâng cao: Optional + functional error handling
```java
// Optional — tránh NullPointerException, thay thế null return
public Optional<User> findByEmail(String email) {
    return userRepo.findByEmail(email); // trả Optional, không null
}

// Chain operations an toàn
String city = findByEmail("alice@example.com")
    .map(User::getAddress)          // Optional<Address>
    .map(Address::getCity)          // Optional<String>
    .filter(c -> !c.isEmpty())
    .orElse("Unknown");             // giá trị mặc định

// orElseGet: lazy evaluation — chỉ gọi khi Optional empty
String expensive = opt.orElseGet(() -> computeExpensiveDefault());

// orElseThrow
User user = findByEmail(email)
    .orElseThrow(() -> new ResourceNotFoundException("User", email));

// ifPresent / ifPresentOrElse (Java 9)
findByEmail(email).ifPresentOrElse(
    user -> sendWelcomeEmail(user),
    () -> log.warn("User not found: {}", email)
);

// Khi KHÔNG dùng Optional:
// - Không dùng làm field trong class (không serializable tốt)
// - Không dùng làm method parameter (dùng @Nullable hoặc overload)
// - Không dùng cho collection (trả List rỗng thay vì Optional<List>)

// Functional error handling với Either-like pattern
public Result<User, String> createUser(CreateUserRequest req) {
    if (userRepo.existsByEmail(req.email()))
        return Result.fail("Email already exists");
    if (req.age() < 18)
        return Result.fail("Must be 18+");

    User user = userRepo.save(User.from(req));
    return Result.ok(user);
}
```

---

## 6. CONCURRENCY

### Lý thuyết & Bản chất
Java là **multi-threaded** — nhiều threads có thể chạy song song trên nhiều CPU cores. Điều này tạo ra 3 vấn đề chính:

1. **Race condition**: Nhiều threads modify shared data → kết quả không đoán trước
2. **Visibility**: Thread A write, Thread B không thấy giá trị mới (CPU caching)
3. **Atomicity**: Operation tưởng là 1 bước nhưng thực ra nhiều bước (count++ = read + increment + write)

**Giải pháp:**
- `synchronized`: mutual exclusion + visibility
- `volatile`: chỉ visibility (không atomicity)
- `java.util.concurrent.atomic.*`: atomic operations
- `java.util.concurrent`: high-level concurrency utilities

---

### Ví dụ 1 — Cơ bản: synchronized + volatile
```java
// Race condition
public class Counter {
    private int count = 0;

    // BUG: count++ không atomic (read-modify-write)
    public void increment() { count++; }
    public int get() { return count; }
}

// FIX 1: synchronized method (implicit lock trên `this`)
public class SynchronizedCounter {
    private int count = 0;

    public synchronized void increment() { count++; } // chỉ 1 thread tại 1 thời điểm
    public synchronized int get() { return count; }
}

// FIX 2: AtomicInteger — lock-free, nhanh hơn synchronized
public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() { count.incrementAndGet(); } // CAS operation
    public int get() { return count.get(); }
}

// volatile: đảm bảo visibility, không đảm bảo atomicity
// Dùng cho flag, singleton (double-checked locking)
public class StopableTask implements Runnable {
    private volatile boolean running = true; // volatile: các thread thấy giá trị mới nhất

    public void stop() { running = false; }

    @Override
    public void run() {
        while (running) {
            doWork();
        }
    }
}
```

---

### Ví dụ 2 — Trung cấp: ExecutorService + CompletableFuture
```java
// ExecutorService: thread pool management
ExecutorService pool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

// Submit tasks
Future<String> future = pool.submit(() -> {
    Thread.sleep(1000);
    return "Result";
});

String result = future.get(); // blocking — đợi kết quả
String result2 = future.get(5, TimeUnit.SECONDS); // timeout

// Shutdown gracefully
pool.shutdown();
pool.awaitTermination(10, TimeUnit.SECONDS);

// CompletableFuture (Java 8+) — non-blocking async
CompletableFuture<User> userFuture = CompletableFuture
    .supplyAsync(() -> userRepo.findById(1L))    // chạy async
    .thenApply(user -> enrichUser(user))          // transform
    .thenApply(user -> {
        sendWelcomeEmail(user);
        return user;
    })
    .exceptionally(ex -> {
        log.error("Failed", ex);
        return User.anonymous();                   // fallback
    });

// Combine multiple futures — tương đương Promise.all
CompletableFuture<User> userF = fetchUserAsync(id);
CompletableFuture<List<Order>> ordersF = fetchOrdersAsync(id);

CompletableFuture<UserProfile> profile = userF.thenCombine(ordersF,
    (user, orders) -> new UserProfile(user, orders)
);

// allOf — đợi tất cả
CompletableFuture.allOf(userF, ordersF)
    .thenRun(() -> System.out.println("All done"));
```

---

### Ví dụ 3 — Nâng cao: ReentrantLock + Deadlock prevention
```java
// ReentrantLock: linh hoạt hơn synchronized
public class BankTransfer {
    // Dùng cùng lock ordering để tránh deadlock
    private static final Comparator<Account> LOCK_ORDER =
        Comparator.comparingLong(Account::getId);

    public void transfer(Account from, Account to, BigDecimal amount) {
        // Luôn acquire lock theo thứ tự nhất định → tránh deadlock
        Account first = LOCK_ORDER.compare(from, to) < 0 ? from : to;
        Account second = first == from ? to : from;

        first.getLock().lock();
        try {
            second.getLock().lock();
            try {
                if (from.getBalance().compareTo(amount) < 0)
                    throw new InsufficientFundsException();
                from.debit(amount);
                to.credit(amount);
            } finally {
                second.getLock().unlock();
            }
        } finally {
            first.getLock().unlock();
        }
    }
}

// ReadWriteLock: nhiều reader OR 1 writer
public class Cache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    public V get(K key) {
        lock.readLock().lock(); // nhiều threads có thể đọc cùng lúc
        try {
            return map.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void put(K key, V value) {
        lock.writeLock().lock(); // chỉ 1 thread ghi, block tất cả reader
        try {
            map.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
}

// Semaphore: giới hạn concurrent access
public class ConnectionPool {
    private final Semaphore semaphore;
    private final Deque<Connection> pool;

    public ConnectionPool(int maxConnections) {
        semaphore = new Semaphore(maxConnections, true); // fair
        pool = new ConcurrentLinkedDeque<>();
        // init connections...
    }

    public Connection acquire() throws InterruptedException {
        semaphore.acquire(); // block nếu không còn permit
        return pool.poll();
    }

    public void release(Connection conn) {
        pool.push(conn);
        semaphore.release(); // trả permit
    }
}
```

---

## 7. JAVA MODERN FEATURES (Java 8 → Java 21)

### Lý thuyết & Bản chất
Java phát triển liên tục. Interviewer hay hỏi về những feature quan trọng từ Java 8 trở đi.

---

### Ví dụ 1 — Cơ bản: Java 8 essentials
```java
// --- Java 8 ---

// 1. Default methods trong interface
interface Greeting {
    String greet(String name);
    default String greetLoudly(String name) {
        return greet(name).toUpperCase(); // dùng abstract method
    }
}

// 2. Functional interfaces + @FunctionalInterface
@FunctionalInterface
interface Transformer<T, R> {
    R transform(T input);
    // Chỉ 1 abstract method → có thể dùng lambda
}
Transformer<String, Integer> length = String::length;

// 3. LocalDate / LocalDateTime (thay Date)
LocalDate date = LocalDate.now();
LocalDate birthday = LocalDate.of(1995, Month.MARCH, 15);
long age = ChronoUnit.YEARS.between(birthday, date);
LocalDate nextWeek = date.plusWeeks(1);

// ZonedDateTime cho timezone-aware
ZonedDateTime now = ZonedDateTime.now(ZoneId.of("Asia/Ho_Chi_Minh"));

// 4. Map.forEach / Map.getOrDefault / Map.computeIfAbsent
map.forEach((k, v) -> System.out.println(k + ": " + v));
int value = map.getOrDefault("missing", 0);
map.computeIfAbsent("key", k -> new ArrayList<>()).add("value");
```

---

### Ví dụ 2 — Trung cấp: Java 9-16
```java
// --- Java 9 ---
// Collection factory methods
List<String> list = List.of("a", "b", "c");           // immutable
Set<String> set = Set.of("x", "y");
Map<String, Integer> map = Map.of("one", 1, "two", 2);
Map<String, Integer> map2 = Map.ofEntries(
    Map.entry("one", 1), Map.entry("two", 2)
);

// --- Java 11 ---
// String methods
"  hello  ".strip();             // Unicode-aware trim
"".isBlank();                    // true
"hello\nworld".lines().toList(); // Stream<String>
"ha".repeat(3);                  // "hahaha"

// --- Java 14 ---
// Records — immutable data class
public record Point(double x, double y) {
    // Compact constructor — validation
    public Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("Coordinates must be positive");
    }
    // Custom method
    public double distanceTo(Point other) {
        return Math.sqrt(Math.pow(x - other.x, 2) + Math.pow(y - other.y, 2));
    }
}
// Tự động generate: constructor, getters, equals, hashCode, toString

// --- Java 16 ---
// Pattern matching instanceof (stable)
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s.toUpperCase()); // s đã cast, không cần (String) obj
}
```

---

### Ví dụ 3 — Nâng cao: Java 17-21
```java
// --- Java 17 ---
// Sealed classes (stable)
public sealed interface Shape permits Circle, Rectangle, Triangle {}
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}

// --- Java 21 ---
// Pattern matching for switch (stable)
double area = switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t -> 0.5 * t.base() * t.height();
};

// Guarded patterns
String describe = switch (obj) {
    case Integer i when i < 0 -> "negative int";
    case Integer i when i == 0 -> "zero";
    case Integer i -> "positive int: " + i;
    case String s when s.isEmpty() -> "empty string";
    case String s -> "string: " + s;
    case null -> "null";
    default -> "other: " + obj.getClass().getSimpleName();
};

// Virtual Threads (Project Loom) — Java 21
// Thread nhẹ, JVM-managed, không map 1-1 với OS thread
// Cho phép hàng triệu threads → perfect cho I/O-bound workloads
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = IntStream.range(0, 1_000_000)
        .mapToObj(i -> executor.submit(() -> callExternalAPI(i)))
        .toList();
    // 1 triệu virtual threads — không block OS threads khi I/O wait
}

// Text Blocks (Java 15, stable)
String json = """
        {
            "name": "Alice",
            "age": 30
        }
        """; // Tự động strip indentation
```

---

## 8. DESIGN PATTERNS

### Lý thuyết & Bản chất
Design patterns là **giải pháp tái sử dụng** cho các vấn đề lập trình phổ biến. Chia 3 nhóm:

- **Creational**: cách tạo objects (Singleton, Builder, Factory, Abstract Factory, Prototype)
- **Structural**: cách compose objects (Adapter, Decorator, Proxy, Facade, Composite)
- **Behavioral**: cách objects giao tiếp (Strategy, Observer, Command, Iterator, Template Method)

---

### Ví dụ 1 — Cơ bản: Builder + Singleton
```java
// Builder — khi constructor có nhiều tham số optional
public class EmailMessage {
    private final String from;
    private final String to;
    private final String subject;
    private final String body;
    private final List<String> cc;
    private final List<String> bcc;
    private final boolean html;

    private EmailMessage(Builder builder) {
        this.from = builder.from;
        this.to = builder.to;
        this.subject = builder.subject;
        this.body = builder.body;
        this.cc = List.copyOf(builder.cc);
        this.bcc = List.copyOf(builder.bcc);
        this.html = builder.html;
    }

    public static class Builder {
        private final String from;
        private final String to;
        private String subject = "";
        private String body = "";
        private List<String> cc = new ArrayList<>();
        private List<String> bcc = new ArrayList<>();
        private boolean html = false;

        public Builder(String from, String to) {
            this.from = Objects.requireNonNull(from);
            this.to = Objects.requireNonNull(to);
        }

        public Builder subject(String subject) { this.subject = subject; return this; }
        public Builder body(String body) { this.body = body; return this; }
        public Builder cc(String... addresses) { cc.addAll(Arrays.asList(addresses)); return this; }
        public Builder html(boolean html) { this.html = html; return this; }
        public EmailMessage build() { return new EmailMessage(this); }
    }
}

EmailMessage email = new EmailMessage.Builder("alice@example.com", "bob@example.com")
    .subject("Hello")
    .body("<h1>Hi Bob!</h1>")
    .html(true)
    .cc("carol@example.com")
    .build();

// Singleton — thread-safe với initialization-on-demand holder
public class DatabaseConnection {
    private DatabaseConnection() {} // prevent instantiation

    private static class Holder {
        // Loaded only when Holder is accessed → thread-safe by classloader
        private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    }

    public static DatabaseConnection getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

### Ví dụ 2 — Trung cấp: Strategy + Observer
```java
// Strategy — algorithm family, interchangeable
@FunctionalInterface
public interface SortStrategy<T> {
    List<T> sort(List<T> items, Comparator<T> comparator);
}

public class DataProcessor<T> {
    private SortStrategy<T> sortStrategy;

    public void setSortStrategy(SortStrategy<T> strategy) {
        this.sortStrategy = strategy;
    }

    public List<T> process(List<T> data, Comparator<T> comparator) {
        return sortStrategy.sort(data, comparator);
    }
}

// Swap strategies at runtime
processor.setSortStrategy((items, cmp) -> {
    List<T> sorted = new ArrayList<>(items);
    Collections.sort(sorted, cmp); // merge sort
    return sorted;
});

// Observer (Event-driven)
public interface EventListener<T> {
    void onEvent(T event);
}

public class EventBus {
    private final Map<Class<?>, List<EventListener<?>>> listeners = new ConcurrentHashMap<>();

    @SuppressWarnings("unchecked")
    public <T> void subscribe(Class<T> eventType, EventListener<T> listener) {
        listeners.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                 .add(listener);
    }

    @SuppressWarnings("unchecked")
    public <T> void publish(T event) {
        List<EventListener<?>> eventListeners = listeners.getOrDefault(
            event.getClass(), Collections.emptyList()
        );
        eventListeners.forEach(l -> ((EventListener<T>) l).onEvent(event));
    }
}

// Dùng
eventBus.subscribe(UserCreatedEvent.class, event -> sendWelcomeEmail(event.getUser()));
eventBus.subscribe(UserCreatedEvent.class, event -> initUserProfile(event.getUser()));
eventBus.publish(new UserCreatedEvent(user));
```

---

### Ví dụ 3 — Nâng cao: Decorator + Proxy (AOP-like)
```java
// Decorator — thêm behavior mà không thay đổi interface
public interface DataSource {
    void writeData(String data);
    String readData();
}

// Base implementation
public class FileDataSource implements DataSource {
    private final String filename;
    public void writeData(String data) { /* write to file */ }
    public String readData() { return /* read from file */ null; }
}

// Decorator abstract base
public abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrappee;
    public DataSourceDecorator(DataSource source) { this.wrappee = source; }
    public void writeData(String data) { wrappee.writeData(data); }
    public String readData() { return wrappee.readData(); }
}

// Encryption decorator
public class EncryptionDecorator extends DataSourceDecorator {
    public EncryptionDecorator(DataSource source) { super(source); }

    @Override
    public void writeData(String data) {
        super.writeData(encrypt(data)); // encrypt trước khi ghi
    }

    @Override
    public String readData() {
        return decrypt(super.readData()); // decrypt sau khi đọc
    }
}

// Compression decorator
public class CompressionDecorator extends DataSourceDecorator {
    public CompressionDecorator(DataSource source) { super(source); }
    @Override public void writeData(String data) { super.writeData(compress(data)); }
    @Override public String readData() { return decompress(super.readData()); }
}

// Wrap theo thứ tự: compress → encrypt → write to file
DataSource source = new EncryptionDecorator(
    new CompressionDecorator(
        new FileDataSource("data.txt")
    )
);
source.writeData("Hello World"); // compress → encrypt → write
source.readData();               // read → decrypt → decompress
```

---

## TỔNG KẾT — Java Core Quick Reference

| Topic | Key Point | Hay bị hỏi |
|-------|-----------|-----------|
| OOP | 4 tính chất, abstract class vs interface | Khi nào dùng interface vs abstract |
| Generics | Type erasure, PECS, wildcards | `? extends` vs `? super` |
| Collections | Chọn đúng structure theo use case | HashMap internals, equals/hashCode |
| Streams | Lazy, intermediate vs terminal, parallel | groupingBy, flatMap, custom collector |
| Exception | Checked vs unchecked, chaining | Khi nào dùng loại nào |
| Concurrency | Race condition, volatile, atomic, lock | Deadlock, CompletableFuture |
| Modern Java | Records, sealed, pattern matching, virtual threads | Java version mới nhất bạn dùng |
| Design Patterns | Creational, structural, behavioral | Builder, Strategy, Observer, Decorator |
