# Spring Boot & Backend — Lý thuyết sâu + Ví dụ 3 cấp độ

---

## 1. IoC, DI & SPRING CONTAINER

### Lý thuyết & Bản chất
**IoC (Inversion of Control):** Thay vì object tự tạo dependencies của mình (`new Service()`), Spring Container quản lý việc tạo và wire dependencies. "Inversion" vì control đã bị đảo ngược — không phải code mình mà là framework kiểm soát lifecycle.

**DI (Dependency Injection):** Cơ chế cụ thể để implement IoC. Framework "inject" dependencies vào object thay vì object tự tạo.

**Spring Container (ApplicationContext):** Đọc cấu hình (annotations/XML/Java config) → tạo Beans → inject dependencies → quản lý lifecycle.

**Bean Scope:**
- `@Singleton` (default): 1 instance dùng chung toàn app → stateless!
- `@Prototype`: instance mới mỗi lần inject
- `@RequestScope`: 1 instance per HTTP request
- `@SessionScope`: 1 instance per HTTP session

---

### Ví dụ 1 — Cơ bản: 3 cách DI
```java
@Service
public class OrderService {

    // Constructor Injection — BEST PRACTICE
    // ✅ Immutable (final fields), testable, fails fast nếu thiếu dependency
    private final OrderRepository orderRepo;
    private final PaymentService paymentService;

    public OrderService(OrderRepository orderRepo, PaymentService paymentService) {
        this.orderRepo = orderRepo;
        this.paymentService = paymentService;
    }
    // Lombok shortcut: @RequiredArgsConstructor
}

@Service
public class AnotherService {

    // Field Injection — AVOID
    // ❌ Không test được mà không dùng Spring context, không immutable
    @Autowired
    private UserRepository userRepo;

    // Setter Injection — cho optional dependencies
    // ❌ Ít dùng trong practice
    @Autowired(required = false)
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

---

### Ví dụ 2 — Trung cấp: @Configuration + Conditional Beans
```java
// Java-based configuration
@Configuration
public class AppConfig {

    // Manual bean definition — dùng khi không thể dùng @Component (third-party classes)
    @Bean
    public ObjectMapper objectMapper() {
        return Jackson2ObjectMapperBuilder.json()
            .serializationInclusion(JsonInclude.Include.NON_NULL)
            .build();
    }

    // Conditional: chỉ create bean nếu property được set
    @Bean
    @ConditionalOnProperty(name = "app.email.enabled", havingValue = "true")
    public EmailService emailService() {
        return new SmtpEmailService();
    }

    // Profile-based beans
    @Bean
    @Profile("production")
    public CacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory).build();
    }

    @Bean
    @Profile("!production") // development và test
    public CacheManager inMemoryCacheManager() {
        return new ConcurrentMapCacheManager();
    }
}

// @Qualifier khi có nhiều implementation
public interface NotificationService {
    void send(String message);
}

@Service("email")
public class EmailNotificationService implements NotificationService { ... }

@Service("sms")
public class SmsNotificationService implements NotificationService { ... }

@Service
@RequiredArgsConstructor
public class UserService {
    @Qualifier("email") // chọn implementation cụ thể
    private final NotificationService notificationService;
}
```

---

### Ví dụ 3 — Nâng cao: Custom Scope + BeanPostProcessor
```java
// Custom annotation để inject cụ thể
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Qualifier("email")
public @interface EmailNotification {}

// Dùng
@EmailNotification
private final NotificationService notificationService;

// BeanPostProcessor: can thiệp vào lifecycle của bean
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Trước khi @PostConstruct
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Sau @PostConstruct — có thể wrap bean với proxy (cách AOP hoạt động)
        if (bean instanceof Service) {
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    log.info("Calling {}.{}", beanName, method.getName());
                    return method.invoke(bean, args);
                }
            );
        }
        return bean;
    }
}
```

---

## 2. REST API & EXCEPTION HANDLING

### Lý thuyết & Bản chất
**REST (Representational State Transfer):** Architectural style dùng HTTP methods để thao tác với **resources** (danh từ). Key constraints: stateless, uniform interface, layered system.

**HTTP Methods:**
- `GET`: đọc, idempotent, safe
- `POST`: tạo mới, không idempotent
- `PUT`: thay thế hoàn toàn, idempotent
- `PATCH`: cập nhật một phần, idempotent
- `DELETE`: xóa, idempotent

**Status Codes quan trọng:** 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 500 Internal Server Error.

---

### Ví dụ 1 — Cơ bản: CRUD Controller
```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Validated
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public ResponseEntity<Page<ProductDTO>> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") @Max(100) int size,
        @RequestParam(required = false) String category,
        @SortDefault(sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable
    ) {
        return ResponseEntity.ok(productService.findAll(category, pageable));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDTO> getProduct(@PathVariable String id) {
        return ResponseEntity.ok(productService.findById(id));
        // findById throws EntityNotFoundException → handled by @ControllerAdvice
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductDTO createProduct(
        @Valid @RequestBody CreateProductRequest request,
        @AuthenticationPrincipal UserDetails currentUser
    ) {
        return productService.create(request, currentUser.getUsername());
    }

    @PatchMapping("/{id}")
    public ProductDTO updateProduct(
        @PathVariable String id,
        @Valid @RequestBody UpdateProductRequest request
    ) {
        return productService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteProduct(@PathVariable String id) {
        productService.delete(id);
    }
}
```

---

### Ví dụ 2 — Trung cấp: Global Exception Handler
```java
// Standard error response
@Getter
@Builder
public class ApiError {
    private final int status;
    private final String error;
    private final String message;
    private final String path;
    private final Instant timestamp;
    private final Map<String, String> fieldErrors; // cho validation errors
}

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // 404 Not Found
    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiError handleNotFound(EntityNotFoundException ex, HttpServletRequest req) {
        return ApiError.builder()
            .status(404)
            .error("Not Found")
            .message(ex.getMessage())
            .path(req.getRequestURI())
            .timestamp(Instant.now())
            .build();
    }

    // 409 Conflict
    @ExceptionHandler(DuplicateResourceException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ApiError handleConflict(DuplicateResourceException ex, HttpServletRequest req) {
        return buildError(409, "Conflict", ex.getMessage(), req);
    }

    // 400 Validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
        Map<String, String> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "Invalid",
                (existing, replacement) -> existing // giữ lỗi đầu tiên nếu duplicate
            ));

        return ApiError.builder()
            .status(400)
            .error("Validation Failed")
            .message("Request contains invalid fields")
            .path(req.getRequestURI())
            .timestamp(Instant.now())
            .fieldErrors(fieldErrors)
            .build();
    }

    // 500 fallback
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiError handleAll(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception for {}: {}", req.getRequestURI(), ex.getMessage(), ex);
        return buildError(500, "Internal Server Error", "An unexpected error occurred", req);
    }
}
```

---

### Ví dụ 3 — Nâng cao: Request DTOs + MapStruct
```java
// Request DTO với Bean Validation
public record CreateProductRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    String name,

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    @Digits(integer = 10, fraction = 2, message = "Price format invalid")
    BigDecimal price,

    @NotBlank(message = "Category is required")
    String category,

    @Size(max = 2000)
    String description
) {}

// Custom validator
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
@RequiredArgsConstructor
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    private final UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) return true; // @NotBlank handles null
        return !userRepository.existsByEmail(email);
    }
}

// MapStruct — code generation cho entity ↔ DTO mapping
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface ProductMapper {
    ProductDTO toDto(Product product);
    Product toEntity(CreateProductRequest request);

    @Mapping(target = "updatedAt", expression = "java(java.time.Instant.now())")
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntityFromRequest(UpdateProductRequest request, @MappingTarget Product product);

    List<ProductDTO> toDtoList(List<Product> products);
}
```

---

## 3. SPRING DATA JPA & DATABASE

### Lý thuyết & Bản chất
**JPA (Jakarta Persistence API):** Specification cho ORM. Hibernate là implementation phổ biến nhất.

**Entity Lifecycle:**
```
Transient → Managed (persist()) → Detached (evict/close) → Removed (remove())
                ↓ flush()
            Synchronized with DB
```

**N+1 Query Problem:** Query lấy N entities → N thêm queries để load association của mỗi entity. Phổ biến nhất với lazy loading.

---

### Ví dụ 1 — Cơ bản: Entity + Repository
```java
@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_product_category", columnList = "category"),
    @Index(name = "idx_product_sku", columnList = "sku", unique = true),
})
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(nullable = false, unique = true)
    private String sku;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ProductStatus status = ProductStatus.ACTIVE;

    @ManyToOne(fetch = FetchType.LAZY) // LAZY by default cho @ManyToOne best practice
    @JoinColumn(name = "category_id")
    private Category category;

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<ProductImage> images = new ArrayList<>();

    @CreatedDate
    @Column(updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;
}

// Repository
public interface ProductRepository extends JpaRepository<Product, String> {

    // Derived query — Spring tự generate query từ method name
    Optional<Product> findBySku(String sku);
    List<Product> findByCategoryAndStatus(Category category, ProductStatus status);
    boolean existsBySku(String sku);

    // JPQL
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max AND p.status = 'ACTIVE'")
    Page<Product> findByPriceRange(@Param("min") BigDecimal min,
                                    @Param("max") BigDecimal max,
                                    Pageable pageable);

    // Native SQL (dùng khi cần DB-specific features)
    @Query(value = "SELECT * FROM products WHERE MATCH(name, description) AGAINST (:query IN BOOLEAN MODE)",
           nativeQuery = true)
    List<Product> fullTextSearch(@Param("query") String query);
}
```

---

### Ví dụ 2 — Trung cấp: N+1 Problem & Solutions
```java
// SETUP
@Entity
public class Order {
    @Id Long id;
    @ManyToOne(fetch = FetchType.LAZY)
    User user;
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    List<OrderItem> items;
}

// N+1 PROBLEM
List<Order> orders = orderRepository.findAll(); // 1 query
for (Order order : orders) {
    System.out.println(order.getUser().getName()); // N queries (lazy load)
    // Tổng: 1 + N queries!
}

// SOLUTION 1: JOIN FETCH trong JPQL
@Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.user LEFT JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithDetails(@Param("status") OrderStatus status);
// 1 query với JOIN → load tất cả cùng lúc

// SOLUTION 2: @EntityGraph
@EntityGraph(attributePaths = {"user", "items", "items.product"})
List<Order> findByStatus(OrderStatus status);
// Spring tự generate JOIN FETCH

// SOLUTION 3: DTO Projection — chỉ lấy fields cần thiết
public interface OrderSummary {
    Long getId();
    String getUserName();   // Spring Data tự map từ user.name
    BigDecimal getTotal();
    LocalDateTime getCreatedAt();
}

@Query("SELECT o.id as id, o.user.name as userName, o.total as total, o.createdAt as createdAt FROM Order o")
List<OrderSummary> findAllSummaries();
// Hiệu quả nhất: chỉ select columns cần thiết

// SOLUTION 4: Batch Size (cho nhiều associations)
@BatchSize(size = 25)
@OneToMany(mappedBy = "order")
List<OrderItem> items;
// Thay vì N queries, dùng WHERE id IN (...) → batch của 25
```

---

### Ví dụ 3 — Nâng cao: @Transactional sâu + Specifications
```java
@Service
@Transactional(readOnly = true) // class-level: mọi method đều readOnly
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepo;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;

    // Override class-level annotation
    @Transactional // readOnly = false
    public Order createOrder(CreateOrderRequest request, String userId) {
        // Toàn bộ method này trong 1 transaction
        // Nếu bất kỳ exception nào → rollback tất cả

        List<OrderItem> items = request.items().stream()
            .map(item -> {
                // Kiểm tra và reserve inventory (cùng transaction!)
                inventoryService.reserve(item.productId(), item.quantity());
                return OrderItem.of(item.productId(), item.quantity(), item.price());
            })
            .toList();

        Order order = orderRepo.save(Order.builder()
            .userId(userId)
            .items(items)
            .status(OrderStatus.PENDING)
            .total(calculateTotal(items))
            .build());

        // Nếu payment fail → exception → rollback order + inventory reservation
        paymentService.charge(order.getId(), order.getTotal(), request.paymentMethod());

        order.setStatus(OrderStatus.CONFIRMED);
        return orderRepo.save(order);
    }

    // @Transactional pitfalls
    // 1. Self-invocation: method này gọi @Transactional method khác trong cùng class
    //    → proxy bị bypass → transaction của method được gọi KHÔNG start!
    //    FIX: inject self hoặc tách ra service khác

    // 2. Checked exceptions không rollback by default
    @Transactional(rollbackFor = Exception.class) // rollback cả checked exceptions
    public void riskyOperation() throws Exception { ... }

    // 3. @Transactional trên private method → KHÔNG WORK (proxy-based AOP)
}

// Specifications — dynamic queries (Criteria API wrapper)
public class ProductSpec {
    public static Specification<Product> hasCategory(String category) {
        return (root, query, cb) ->
            category == null ? null : cb.equal(root.get("category").get("name"), category);
    }

    public static Specification<Product> hasPriceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("price"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("price"), min);
            return cb.between(root.get("price"), min, max);
        };
    }

    public static Specification<Product> isActive() {
        return (root, query, cb) -> cb.equal(root.get("status"), ProductStatus.ACTIVE);
    }
}

// Dùng
Page<Product> products = productRepo.findAll(
    Specification.where(ProductSpec.isActive())
        .and(ProductSpec.hasCategory(filterCategory))
        .and(ProductSpec.hasPriceBetween(minPrice, maxPrice)),
    pageable
);
```

---

## 4. SPRING SECURITY + JWT

### Lý thuyết & Bản chất
Spring Security hoạt động như một chuỗi **Filter Chain**. Mỗi request đi qua nhiều filter trước khi đến controller. JWT filter extract và validate token → lưu authentication vào `SecurityContextHolder`.

**Authentication vs Authorization:**
- **Authentication (AuthN):** Bạn là ai? (verify identity)
- **Authorization (AuthZ):** Bạn được làm gì? (verify permissions)

---

### Ví dụ 1 — Cơ bản: JWT Filter + Security Config
```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        try {
            final String token = authHeader.substring(7);
            final String username = jwtService.extractUsername(token);

            // Chỉ authenticate nếu chưa có authentication trong context
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isTokenValid(token, userDetails)) {
                    UsernamePasswordAuthenticationToken auth =
                        new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities()
                        );
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (JwtException ex) {
            // Token invalid → không set authentication → filter chain tiếp tục
            // Controller sẽ nhận 401 từ Spring Security
        }

        filterChain.doFilter(request, response);
    }
}

// Security Config
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // cho @PreAuthorize, @PostAuthorize
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable) // stateless API → không cần CSRF
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/api/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> {
                    res.setStatus(401);
                    res.setContentType("application/json");
                    res.getWriter().write("{\"error\":\"Unauthorized\"}");
                })
                .accessDeniedHandler((req, res, e) -> {
                    res.setStatus(403);
                    res.setContentType("application/json");
                    res.getWriter().write("{\"error\":\"Forbidden\"}");
                })
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

---

### Ví dụ 2 — Trung cấp: JWT Service + Auth Controller
```java
@Service
@RequiredArgsConstructor
public class JwtService {

    @Value("${app.jwt.secret}")
    private String secretKey;

    @Value("${app.jwt.access-expiration:900000}") // 15 phút
    private long accessExpiration;

    @Value("${app.jwt.refresh-expiration:604800000}") // 7 ngày
    private long refreshExpiration;

    public String generateAccessToken(UserDetails user) {
        Map<String, Object> claims = new HashMap<>();
        if (user instanceof AppUserDetails u) {
            claims.put("role", u.getRole());
            claims.put("userId", u.getUserId());
        }
        return buildToken(claims, user.getUsername(), accessExpiration);
    }

    public String generateRefreshToken(UserDetails user) {
        return buildToken(Map.of(), user.getUsername(), refreshExpiration);
    }

    private String buildToken(Map<String, Object> extraClaims, String subject, long expiration) {
        return Jwts.builder()
            .claims(extraClaims)
            .subject(subject)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey())
            .compact();
    }

    public boolean isTokenValid(String token, UserDetails user) {
        final String username = extractUsername(token);
        return username.equals(user.getUsername()) && !isTokenExpired(token);
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    private <T> T extractClaim(String token, Function<Claims, T> resolver) {
        return resolver.apply(Jwts.parser().verifyWith(getSigningKey()).build()
            .parseSignedClaims(token).getPayload());
    }

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey));
    }
}
```

---

### Ví dụ 3 — Nâng cao: Refresh Token + Method Security
```java
// @PreAuthorize — method-level security
@Service
@RequiredArgsConstructor
public class PostService {

    private final PostRepository postRepo;

    @PreAuthorize("hasRole('ADMIN') or @postService.isOwner(#postId, authentication.name)")
    public void deletePost(String postId) {
        postRepo.deleteById(postId);
    }

    @PostAuthorize("returnObject.authorId == authentication.principal.userId or hasRole('ADMIN')")
    public Post getPost(String postId) {
        return postRepo.findById(postId).orElseThrow();
    }

    // Bean được reference trong SpEL
    public boolean isOwner(String postId, String username) {
        return postRepo.existsByIdAndAuthorUsername(postId, username);
    }
}

// Refresh token rotation
@Service
@Transactional
@RequiredArgsConstructor
public class AuthService {

    private final RefreshTokenRepository refreshTokenRepo;
    private final JwtService jwtService;

    public AuthResponse refreshTokens(String refreshToken) {
        // Tìm token trong DB
        RefreshToken storedToken = refreshTokenRepo.findByToken(refreshToken)
            .orElseThrow(() -> new InvalidTokenException("Invalid refresh token"));

        // Check expired
        if (storedToken.getExpiresAt().isBefore(Instant.now())) {
            refreshTokenRepo.delete(storedToken);
            throw new TokenExpiredException("Refresh token expired, please login again");
        }

        // Token rotation: xóa token cũ, tạo token mới
        // Nếu cùng 1 refresh token được dùng 2 lần → có thể bị stolen
        refreshTokenRepo.delete(storedToken);

        UserDetails user = userDetailsService.loadUserByUsername(storedToken.getUsername());
        String newAccessToken = jwtService.generateAccessToken(user);
        String newRefreshToken = jwtService.generateRefreshToken(user);

        refreshTokenRepo.save(RefreshToken.builder()
            .token(newRefreshToken)
            .username(user.getUsername())
            .expiresAt(Instant.now().plusSeconds(7 * 24 * 3600))
            .build());

        return new AuthResponse(newAccessToken, newRefreshToken);
    }

    public void logout(String username) {
        // Xóa tất cả refresh tokens của user → force re-login trên mọi device
        refreshTokenRepo.deleteAllByUsername(username);
    }
}
```

---

## TỔNG KẾT — Backend Quick Reference

| Concept | Key Point | Pitfall |
|---------|-----------|---------|
| DI | Constructor injection, immutable | Field injection không testable |
| @Transactional | ACID, rollback on RuntimeException default | Self-invocation, private methods, checked exceptions |
| N+1 | 1 query + N queries cho associations | Dùng JOIN FETCH, EntityGraph, hoặc DTO projection |
| JWT | Stateless, có thể verify mà không cần DB | Không thể revoke ngay (dùng short expiry + refresh rotation) |
| @PreAuthorize | SpEL expression, method security | Phải enable với @EnableMethodSecurity |
| Specifications | Dynamic queries, reusable predicates | Có thể tạo cartesian product nếu không cẩn thận với joins |
