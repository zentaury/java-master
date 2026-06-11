---
name: java-master
description: Use when writing Java code with Spring Boot — covers all LTS versions (Java 8, 11, 17, 21, 25) and Spring Boot 2.x, 3.x, 4.x. Includes layered architecture, DTOs, concurrency, HTTP clients, persistence, validation, testing, and security with version-specific guidance for each feature.
---

# Java + Spring Boot — Senior Engineer Guide (All LTS Versions)

## Version Compatibility Matrix

| Feature | Java 8 | Java 11 | Java 17 | Java 21 | Java 25 |
|---|---|---|---|---|---|
| Lambda, Stream, Optional | ✅ | ✅ | ✅ | ✅ | ✅ |
| `var` (local type inference) | ❌ | ✅ | ✅ | ✅ | ✅ |
| Text blocks (`"""`) | ❌ | ❌ | ✅ (15+) | ✅ | ✅ |
| Records | ❌ | ❌ | ✅ (16+) | ✅ | ✅ |
| Sealed classes | ❌ | ❌ | ✅ | ✅ | ✅ |
| Pattern matching `instanceof` | ❌ | ❌ | ✅ (16+) | ✅ | ✅ |
| Pattern matching `switch` | ❌ | ❌ | ❌ | ✅ | ✅ |
| Virtual threads | ❌ | ❌ | ❌ | ✅ | ✅ |
| Scoped Values | ❌ | ❌ | ❌ | Preview | ✅ (finalized) |
| Unnamed variables (`_`) | ❌ | ❌ | ❌ | ❌ | ✅ (22+) |
| Flexible constructor bodies | ❌ | ❌ | ❌ | ❌ | ✅ |
| Structured Concurrency | ❌ | ❌ | ❌ | Preview | Preview |

| Feature | SB 2.x | SB 3.x | SB 4.x |
|---|---|---|---|
| Paquetes `javax.*` | ✅ | ❌ (jakarta) | ❌ (jakarta) |
| Paquetes `jakarta.*` | ❌ | ✅ | ✅ |
| RestTemplate | ✅ | ✅ (deprecated) | ❌ |
| WebClient (reactivo) | ✅ | ✅ | ✅ |
| RestClient | ❌ | ✅ (3.2+) | ✅ |
| `@HttpServiceProxyFactory` | ❌ | ✅ | ❌ |
| `@ImportHttpServices` | ❌ | ❌ | ✅ |
| Jackson 2 | ✅ | ✅ | compat flag |
| Jackson 3 | ❌ | ❌ | ✅ |
| `@ConfigurationProperties` record | ✅ | ✅ | ✅ |
| Virtual threads (auto) | ❌ | ✅ (3.2+, Java 21) | ✅ |

---

## 1. Layered Architecture (aplica a todas las versiones)

```
com.company.project/
├── controller/          <- HTTP: routing, status codes, request/response
├── service/             <- Business logic interfaces
│   └── impl/            <- Implementations
├── repository/          <- Spring Data JPA interfaces
├── model/               <- JPA entities (@Entity)
├── dto/                 <- Objetos de transferencia (records en Java 17+, clases en 8/11)
│   ├── request/
│   └── response/
├── config/              <- Beans, security config
├── exception/           <- Custom exceptions + @ControllerAdvice
├── mapper/              <- Entity <-> DTO conversion
└── util/                <- Static helpers
```

**Reglas de dependencia (todas las versiones):**
- Controller → Service (nunca al revés)
- Service → Repository (nunca Repository → Service)
- Controller NUNCA accede a Repository directamente
- Entidades JPA NUNCA salen de la capa de servicio — siempre mapear a DTO

---

## 2. DTOs — Records vs Clases

### `[Java 17+]` Records (preferido — inmutable, sin boilerplate)

```java
public record CreateUserRequest(
    String name,
    String email,
    String role
) {}

public record UserResponse(Long id, String name, String email) {}
```

### `[Java 8/11]` Clase con constructor inmutable (sin Lombok)

```java
public final class CreateUserRequest {
    private final String name;
    private final String email;
    private final String role;

    public CreateUserRequest(String name, String email, String role) {
        this.name  = name;
        this.email = email;
        this.role  = role;
    }

    public String getName()  { return name; }
    public String getEmail() { return email; }
    public String getRole()  { return role; }
}
```

> **Nota:** Si el proyecto usa Lombok en Java 8/11, `@Value` produce el equivalente inmutable. Úsalo si ya está en el classpath; no lo agregues solo para eso.

---

## 3. Modelado de dominio — Sealed vs Alternativas

### `[Java 17+]` Sealed interfaces + records para resultados

```java
public sealed interface EmissionResult
    permits EmissionResult.Accepted,
            EmissionResult.Rejected,
            EmissionResult.Failed {

    record Accepted(String receiptSeal, String generationCode) implements EmissionResult {}
    record Rejected(String errorCode, String description)      implements EmissionResult {}
    record Failed(String message, Throwable cause)             implements EmissionResult {}
}

// Java 21+ — pattern matching switch (exhaustivo, sin cast manual)
String message = switch (result) {
    case EmissionResult.Accepted a -> "Aceptado: " + a.generationCode();
    case EmissionResult.Rejected r -> "Rechazado: " + r.description();
    case EmissionResult.Failed f   -> "Error: " + f.message();
};
```

### `[Java 8/11]` Clase abstracta sellada por convención

```java
// No hay sealed en Java 8/11 — simular con constructor package-private
public abstract class EmissionResult {
    private EmissionResult() {}   // impide subclases fuera del paquete

    public static final class Accepted extends EmissionResult {
        public final String receiptSeal;
        public final String generationCode;
        public Accepted(String receiptSeal, String generationCode) {
            this.receiptSeal    = receiptSeal;
            this.generationCode = generationCode;
        }
    }
    public static final class Rejected extends EmissionResult {
        public final String errorCode;
        public final String description;
        public Rejected(String errorCode, String description) {
            this.errorCode   = errorCode;
            this.description = description;
        }
    }
    public static final class Failed extends EmissionResult {
        public final String message;
        public final Throwable cause;
        public Failed(String message, Throwable cause) {
            this.message = message;
            this.cause   = cause;
        }
    }
}

// Java 8/11 — instanceof + cast (no hay pattern matching)
if (result instanceof EmissionResult.Accepted) {
    EmissionResult.Accepted a = (EmissionResult.Accepted) result;
    System.out.println("Aceptado: " + a.generationCode);
} else if (result instanceof EmissionResult.Rejected) {
    EmissionResult.Rejected r = (EmissionResult.Rejected) result;
    System.out.println("Rechazado: " + r.description);
}

// Java 16+ — pattern matching instanceof (menos verbose)
if (result instanceof EmissionResult.Accepted a) {
    System.out.println("Aceptado: " + a.generationCode);
}
```

---

## 4. Concurrencia

### `[Java 21+]` Virtual Threads (preferido para I/O intensivo)

```yaml
# application.yml — Spring Boot 3.2+ / 4.x con Java 21+
spring:
  threads:
    virtual:
      enabled: true
```

```java
// Crear virtual thread manualmente
Thread.ofVirtual().name("invoice-worker").start(() -> processInvoice(inv));

// Executor para grupos
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> signInvoice(inv));
    executor.submit(() -> notifyClient(inv));
}
// Evitar synchronized con virtual threads — usar ReentrantLock
```

### `[Java 8+]` Thread pools (Java 8–20)

```java
// Executor service clásico
ExecutorService executor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);
try {
    Future<SignedData>  signFuture  = executor.submit(() -> signerClient.sign(inv));
    Future<AuditResult> auditFuture = executor.submit(() -> auditService.log(inv));
    return new ProcessedResult(signFuture.get(), auditFuture.get());
} finally {
    executor.shutdown();
}

// CompletableFuture (Java 8+) — preferido sobre Future.get() bloqueante
CompletableFuture<SignedData> signFuture = CompletableFuture
    .supplyAsync(() -> signerClient.sign(inv), executor);
```

### `[Java 21+]` Scoped Values — reemplazo de ThreadLocal

```java
// Finalizados en Java 25 (JEP 506); preview desde Java 21
public class TenantContext {
    public static final ScopedValue<String> SCHEMA = ScopedValue.newInstance();
}

ScopedValue.where(TenantContext.SCHEMA, "company_001")
    .run(() -> invoiceService.process(request));

String schema = TenantContext.SCHEMA.get();
```

### `[Java 8+]` ThreadLocal — usar en Java 8–20

```java
// Válido en Java 8–20; evitar con millones de virtual threads
public class TenantContext {
    private static final ThreadLocal<String> SCHEMA = new ThreadLocal<>();

    public static void set(String schema)  { SCHEMA.set(schema); }
    public static String get()             { return SCHEMA.get(); }
    public static void clear()             { SCHEMA.remove(); }   // CRÍTICO — limpiar siempre en finally
}

// En filtro/interceptor
try {
    TenantContext.set(extractSchema(request));
    chain.doFilter(request, response);
} finally {
    TenantContext.clear();   // evitar memory leak
}
```

### `[Java 25, PREVIEW]` Structured Concurrency — solo con `--enable-preview`

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var signerTask = scope.fork(() -> signerClient.sign(invoice));
    var auditTask  = scope.fork(() -> auditService.log(invoice));
    scope.join().throwIfFailed();
    return new ProcessedResult(signerTask.get(), auditTask.get());
}
```

---

## 5. Clientes HTTP

### `[SB 4.x]` RestClient + @ImportHttpServices (Spring Framework 7)

```java
// Bean de RestClient
@Bean
RestClient restClient(RestClient.Builder builder) {
    return builder
        .baseUrl("${signer.base-url}")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .defaultStatusHandler(HttpStatusCode::isError, (req, res) ->
            { throw new SignerException("HTTP " + res.getStatusCode()); })
        .build();
}

// Interfaz declarativa
@HttpExchange("/api/invoices")
public interface ExternalInvoiceClient {
    @GetExchange("/{id}")
    InvoiceResponse getById(@PathVariable String id);

    @PostExchange
    InvoiceResponse create(@RequestBody CreateInvoiceRequest body);
}

// Registrar con @ImportHttpServices
@Configuration
@ImportHttpServices(group = ExternalApiGroup.class, types = { ExternalInvoiceClient.class })
public class HttpClientsConfig {
    @Bean
    RestClientHttpServiceGroupConfigurer externalApiConfigurer() {
        return groups -> groups.forGroup(ExternalApiGroup.class, spec ->
            spec.restClient(builder -> builder.baseUrl("${external.api.base-url}")));
    }
}
```

### `[SB 3.2+]` RestClient + @HttpServiceProxyFactory

```java
// RestClient con interfaz declarativa en SB 3.x
@Bean
ExternalInvoiceClient externalInvoiceClient(RestClient.Builder builder) {
    RestClient client = builder.baseUrl("${external.api.base-url}").build();
    HttpServiceProxyFactory factory = HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(client))
        .build();
    return factory.createClient(ExternalInvoiceClient.class);
}
// La interfaz @HttpExchange es la misma — solo cambia el registro
```

### `[SB 2.x / SB 3.x]` RestTemplate (SB 2.x) — deprecated en SB 3.x

```java
@Bean
RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
        .rootUri("${signer.base-url}")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .errorHandler(new DefaultResponseErrorHandler())
        .build();
}

// Uso en servicio
SignedResponse response = restTemplate.postForObject("/sign", request, SignedResponse.class);
```

### `[Java 11+ / SB 2.x+]` java.net.http.HttpClient (sin Spring)

```java
// Cliente HTTP nativo de Java 11 — útil en módulos sin Spring
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create(baseUrl + "/sign"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(objectMapper.writeValueAsString(body)))
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
```

---

## 6. Manejo de errores — @RestControllerAdvice

### `[SB 3.x / 4.x]` ProblemDetail RFC 7807

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        var problem = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        problem.setTitle("Validation Failed");
        problem.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(e -> Map.of("field", e.getField(), "message", e.getDefaultMessage()))
            .toList());
        return problem;
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }
}
```

### `[SB 2.x]` ResponseEntity con cuerpo de error personalizado

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());
        return ResponseEntity
            .status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(new ErrorResponse("Validation Failed", errors));
    }
}

// DTO de error (Java 8/11: clase; Java 17+: record)
public record ErrorResponse(String message, List<String> errors) {}
```

---

## 7. Controllers (todas las versiones)

```java
@RestController
@RequestMapping("/api/v1/invoices")
public class InvoiceController {

    private final InvoiceService invoiceService;

    // Constructor injection — nunca @Autowired en campo
    public InvoiceController(InvoiceService invoiceService) {
        this.invoiceService = invoiceService;
    }

    @PostMapping
    ResponseEntity<InvoiceResponse> create(@Valid @RequestBody CreateInvoiceRequest request) {
        return ResponseEntity.accepted().body(invoiceService.create(request));
    }

    @GetMapping("/{id}")
    InvoiceResponse getById(@PathVariable String id) {
        return invoiceService.findById(id);
    }
}
```

---

## 8. Validación — Bean Validation (todas las versiones)

### Constraints en records `[Java 17+]` o clases `[Java 8+]`

```java
// Java 17+ — record
public record CreateUserRequest(
    @NotBlank(message = "Name is required") String name,
    @Email(message = "Must be a valid email") @NotBlank String email,
    @Size(min = 8) String password
) {}

// Java 8/11 — clase
public final class CreateUserRequest {
    @NotBlank(message = "Name is required")
    private final String name;

    @Email @NotBlank
    private final String email;

    // constructor + getters...
}
```

### Constraint personalizado

```java
@Documented
@Constraint(validatedBy = NitValidator.class)
@Target({ FIELD })
@Retention(RUNTIME)
public @interface ValidNit {
    String message() default "Invalid NIT format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class NitValidator implements ConstraintValidator<ValidNit, String> {
    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        return value != null && value.matches("\\d{4}-\\d{6}-\\d{3}-\\d");
    }
}
```

---

## 9. Persistencia — Spring Data JPA (todas las versiones)

### Proyecciones con record `[Java 17+]` o interfaz `[Java 8+]`

```java
// Java 17+ — record projection
public record InvoiceSummary(String code, String status, LocalDateTime createdAt) {}

// Java 8/11 — interface projection (Spring Data la implementa en runtime)
public interface InvoiceSummary {
    String getCode();
    String getStatus();
    LocalDateTime getCreatedAt();
}

// Repositorio (mismo en ambos casos)
@Query("""
    SELECT new com.company.dto.response.InvoiceSummary(i.code, i.status, i.createdAt)
    FROM Invoice i WHERE i.status = :status
""")
List<InvoiceSummary> findSummaryByStatus(@Param("status") String status);

// Con interface projection, usar @Query con SELECT i.code, i.status...
// Spring Data infiere los getters automáticamente
```

### @Transactional — service layer only

```java
@Service
@Transactional(readOnly = true)
public class InvoiceServiceImpl implements InvoiceService {

    @Override
    @Transactional
    public InvoiceResponse create(CreateInvoiceRequest request) { ... }

    @Override
    public InvoiceResponse findById(String id) { ... }  // hereda readOnly
}
```

### N+1 — JOIN FETCH o @EntityGraph

```java
@Query("SELECT i FROM Invoice i JOIN FETCH i.customer WHERE i.company.id = :companyId")
List<Invoice> findWithCustomerByCompany(@Param("companyId") Long companyId);

@EntityGraph(attributePaths = {"customer", "establishment"})
List<Invoice> findByStatus(String status);
```

### Entidad JPA (todas las versiones)

```java
@Entity
@Table(name = "invoices")
public class Invoice {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "generation_code", nullable = false, unique = true)
    private String generationCode;

    @Enumerated(EnumType.STRING)
    private InvoiceStatus status;

    public Invoice() {}

    public Long getId()                  { return id; }
    public String getGenerationCode()    { return generationCode; }
    public InvoiceStatus getStatus()     { return status; }
    public void setStatus(InvoiceStatus s) { this.status = s; }
}
```

---

## 10. Configuración Spring Boot

### @ConfigurationProperties con record `[Java 17+ / SB 2.6+]`

```java
@ConfigurationProperties(prefix = "signer")
public record SignerProperties(String baseUrl, Duration timeout, int maxRetries) {}

@EnableConfigurationProperties(SignerProperties.class)
```

### @ConfigurationProperties con clase `[Java 8+ / SB 2.x+]`

```java
@ConfigurationProperties(prefix = "signer")
@Validated
public class SignerProperties {

    @NotBlank
    private String baseUrl;
    private Duration timeout = Duration.ofSeconds(30);
    private int maxRetries = 3;

    // getters + setters
}
```

### Jackson 3 `[SB 4.x]`

```java
// SB4: paquete cambió de com.fasterxml.jackson -> tools.jackson
// Para librerías que requieran Jackson 2:
// spring.jackson.use-jackson2-defaults=true en application.yml
```

### Dependencias Gradle por versión

```groovy
// SB 3.x / 4.x
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'

    // SB 3.2+ / 4.x
    implementation 'org.springframework.boot:spring-boot-starter-restclient'

    // SB 4.x
    implementation 'org.springframework.boot:spring-boot-starter-opentelemetry'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
}

// SB 2.x — agregar si se usa cliente HTTP declarativo
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'  // alternativa a RestClient
}
```

---

## 11. Testing — JUnit 5, Mockito, Testcontainers

### Controller slice `[@WebMvcTest]` — todas las versiones SB

```java
@WebMvcTest(InvoiceController.class)
class InvoiceControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean  InvoiceService invoiceService;

    @Test
    void create_returns202_whenRequestIsValid() throws Exception {
        given(invoiceService.create(any())).willReturn(new InvoiceResponse(1L, "INV-001"));

        mockMvc.perform(post("/api/v1/invoices")
                .contentType(APPLICATION_JSON)
                .content("""{"code":"INV-001","amount":100.00}"""))
            .andExpect(status().isAccepted())
            .andExpect(jsonPath("$.id").value(1L));
    }

    @Test
    void create_returns422_whenBodyIsEmpty() throws Exception {
        mockMvc.perform(post("/api/v1/invoices")
                .contentType(APPLICATION_JSON).content("{}"))
            .andExpect(status().isUnprocessableEntity());
    }
}
```

### Service unit test — `@ExtendWith(MockitoExtension.class)`

```java
@ExtendWith(MockitoExtension.class)
class InvoiceServiceTest {

    @Mock InvoiceRepository repository;
    @Mock SignerClient       signerClient;

    InvoiceServiceImpl service;

    @BeforeEach
    void setUp() {
        service = new InvoiceServiceImpl(repository, signerClient);
    }

    @Test
    void create_callsSignerAndPersists() {
        given(signerClient.sign(any())).willReturn(new SignedData("SEAL-123"));
        given(repository.save(any())).willAnswer(inv -> inv.getArgument(0));

        InvoiceResponse response = service.create(new CreateInvoiceRequest("INV-001", BigDecimal.TEN));

        verify(repository).save(any());
        assertThat(response.receiptSeal()).isEqualTo("SEAL-123");
    }
}
```

### Repository slice — `@DataJpaTest`

```java
@DataJpaTest
class InvoiceRepositoryTest {

    @Autowired InvoiceRepository repository;

    @Test
    void findSummaryByStatus_returnsOnlyMatching() {
        repository.save(buildInvoice("ACCEPTED"));
        repository.save(buildInvoice("REJECTED"));

        List<InvoiceSummary> result = repository.findSummaryByStatus("ACCEPTED");

        assertThat(result).hasSize(1)
            .first().extracting(InvoiceSummary::status).isEqualTo("ACCEPTED");
    }
}
```

### Integration con Testcontainers — `[SB 3.1+ nativo; SB 2.x con dependencia]`

```java
@SpringBootTest
@Testcontainers
class InvoiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void configureProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired InvoiceService invoiceService;

    @Test
    void fullFlow_createAndRetrieve() {
        var created = invoiceService.create(new CreateInvoiceRequest("INV-001", BigDecimal.TEN));
        var found   = invoiceService.findById(created.id().toString());
        assertThat(found.code()).isEqualTo("INV-001");
    }
}
```

### Estrategia de testing

| Tipo | Anotación | Contexto | Para qué |
|---|---|---|---|
| Unit | `@ExtendWith(MockitoExtension.class)` | Ninguno | Lógica de servicio |
| Controller | `@WebMvcTest` | Solo MVC | HTTP, validación |
| Repository | `@DataJpaTest` | Solo JPA | Queries, proyecciones |
| Integration | `@SpringBootTest` + Testcontainers | Completo | Flujos cross-layer |

---

## 12. Spring Security

### `[SB 3.x / 4.x]` Stateless JWT (API REST)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health").permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthFilter(), UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### `[SB 2.x]` WebSecurityConfigurerAdapter (deprecated en SB 3.x)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeRequests()
                .antMatchers("/api/v1/auth/**").permitAll()
                .anyRequest().authenticated();
    }

    @Bean
    @Override
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 13. Anti-Patterns — Por versión

| ❌ Evitar | ✅ Moderno | Versión mínima |
|---|---|---|
| `new Thread(...).start()` | `Thread.ofVirtual().start(...)` | Java 21 |
| `ExecutorService` para I/O | Virtual threads con `newVirtualThreadPerTaskExecutor` | Java 21 |
| `ThreadLocal` sin `remove()` | `ThreadLocal` con `finally { ctx.clear() }` | Java 8 |
| `ThreadLocal` con virtual threads | `ScopedValue` | Java 25 |
| `RestTemplate` | `RestClient` | SB 3.2 |
| `FeignClient` | `@ImportHttpServices` + `@HttpExchange` | SB 4.x |
| `FeignClient` | `@HttpServiceProxyFactory` + `@HttpExchange` | SB 3.x |
| `@Autowired` en campo | Constructor injection | Siempre |
| POJO con setters para DTO | `record` (inmutable) | Java 17 |
| `instanceof` + cast explícito | Pattern matching `instanceof` | Java 16 |
| `instanceof` chain | Pattern matching `switch` | Java 21 |
| `@Value("${prop}")` por campo | `@ConfigurationProperties` con record/clase | SB 2.x |
| Entidad JPA fuera del servicio | Mapear siempre a DTO antes del controller | Siempre |
| `@Transactional` en controller | `@Transactional` solo en servicio | Siempre |
| Lazy loading en loop | `JOIN FETCH` o `@EntityGraph` | Siempre |
| `@SpringBootTest` para todo | Slice tests primero | SB 2.x |
| `BindingResult` en controller | `@Valid` + `@RestControllerAdvice` global | Siempre |
| `WebSecurityConfigurerAdapter` | `SecurityFilterChain` bean | SB 3.x |

---

## Quick Checklist — Antes de hacer commit

**Siempre (Java 8+):**
- [ ] Constructor injection — nunca `@Autowired` en campos
- [ ] Controllers solo manejan HTTP — lógica en servicios
- [ ] Servicios devuelven DTOs — nunca entidades JPA al controller
- [ ] `@Transactional(readOnly = true)` en servicio, override para escrituras
- [ ] `@Valid` en todos los `@RequestBody`
- [ ] `@RestControllerAdvice` centraliza manejo de errores
- [ ] Sin N+1: `JOIN FETCH` o `@EntityGraph`
- [ ] Unit tests con `@ExtendWith(MockitoExtension.class)` para servicios
- [ ] `@WebMvcTest` para capa de controllers
- [ ] `@DataJpaTest` para queries de repositorio
- [ ] `ThreadLocal.remove()` siempre en `finally`

**Java 17+:**
- [ ] DTOs son `record`, no clases con getters
- [ ] Resultados de dominio modelados con `sealed interface` + `record`
- [ ] Pattern matching con `instanceof`

**Java 21+:**
- [ ] Pattern matching con `switch` en lugar de cadenas de `instanceof`
- [ ] `spring.threads.virtual.enabled=true` en application.yml
- [ ] `ScopedValue` para contexto compartido (si proyecto usa `--enable-preview` o espera Java 25)

**SB 3.2+ / 4.x:**
- [ ] `RestClient` para llamadas HTTP salientes
- [ ] `@ConfigurationProperties` con `record` para configuración tipada

**SB 4.x:**
- [ ] `@ImportHttpServices` para clientes HTTP declarativos
- [ ] Imports de `jakarta.*` (no `javax.*`)
