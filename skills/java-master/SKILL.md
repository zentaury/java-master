---
name: java-master
description: Use when writing Java code with Spring Boot — covers all LTS versions (Java 8, 11, 17, 21, 25) and Spring Boot 2.x, 3.x, 4.x. Includes layered architecture, DTOs, exception hierarchy, mappers, pagination, concurrency, HTTP clients, persistence, migrations, validation, testing, and security with version-specific guidance for each feature.
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
| FeignClient (Spring Cloud) | ✅ | ✅ | ❌ |
| Jackson 2 | ✅ | ✅ | compat flag |
| Jackson 3 | ❌ | ❌ | ✅ |
| `@ConfigurationProperties` record | ✅ | ✅ | ✅ |
| Virtual threads (auto) | ❌ | ✅ (3.2+, Java 21) | ✅ |

---

## Herramientas Opcionales — Preguntar ANTES de generar código

**IMPORTANTE:** Antes de generar cualquier código para un proyecto nuevo, pregunta al usuario:

1. **Migraciones de BD:** ¿El proyecto usará migraciones de base de datos?
   - **Flyway** — migraciones en SQL versionado (`V1__`, `V2__`…), simple y directo
   - **Liquibase** — changelogs en YAML/XML/SQL, más flexible para multi-DB
   - **Ninguna** — gestión manual o proyecto sin BD relacional

2. **Cliente HTTP declarativo:** ¿Además de RestClient, el proyecto usará FeignClient?
   - **Sí** — proyecto en SB 2.x/3.x con Spring Cloud, o equipo que ya lo usa
   - **No** — usar RestClient / @HttpExchange (moderno, sin dependencia extra)

Usa las respuestas para incluir o excluir las secciones correspondientes de este skill.

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
├── mapper/              <- Entity <-> DTO conversion (MapStruct)
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
    public static final class Rejected extends EmissionResult { ... }
    public static final class Failed   extends EmissionResult { ... }
}

// Java 8/11 — instanceof + cast
if (result instanceof EmissionResult.Accepted) {
    EmissionResult.Accepted a = (EmissionResult.Accepted) result;
}

// Java 16+ — pattern matching instanceof
if (result instanceof EmissionResult.Accepted a) {
    System.out.println(a.generationCode);
}
```

---

## 4. Jerarquía de Excepciones (todas las versiones)

### Diseño base — siempre unchecked (extends RuntimeException)

```java
// Base en exception/ — nunca instanciar directamente
public abstract class BusinessException extends RuntimeException {

    private final String errorCode;

    protected BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    protected BusinessException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}
```

### Excepciones específicas (en exception/)

```java
// 404 — recurso no encontrado
public class ResourceNotFoundException extends BusinessException {
    public ResourceNotFoundException(String resource, Object id) {
        super("RESOURCE_NOT_FOUND",
              resource + " with id '" + id + "' not found");
    }
}

// 409 — conflicto de estado o duplicado
public class ConflictException extends BusinessException {
    public ConflictException(String message) {
        super("CONFLICT", message);
    }
}

// 422 — regla de negocio violada
public class BusinessRuleException extends BusinessException {
    public BusinessRuleException(String errorCode, String message) {
        super(errorCode, message);
    }
}

// 403 — acción no permitida para ese usuario/rol
public class ForbiddenOperationException extends BusinessException {
    public ForbiddenOperationException(String message) {
        super("FORBIDDEN_OPERATION", message);
    }
}
```

### Uso en servicios

```java
// Good — lanzar directamente, sin try-catch innecesario
public InvoiceResponse findById(String id) {
    return repository.findById(id)
        .map(mapper::toResponse)
        .orElseThrow(() -> new ResourceNotFoundException("Invoice", id));
}

public void cancel(String id) {
    Invoice invoice = repository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Invoice", id));

    if (invoice.getStatus() == InvoiceStatus.PAID) {
        throw new BusinessRuleException("INVOICE_ALREADY_PAID",
            "Cannot cancel a paid invoice");
    }
    invoice.setStatus(InvoiceStatus.CANCELLED);
}

// Bad — checked exceptions para lógica de negocio
public InvoiceResponse findById(String id) throws InvoiceNotFoundException { ... }
```

### @RestControllerAdvice — conectar con la jerarquía

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // SB 3.x / 4.x — ProblemDetail RFC 7807
    @ExceptionHandler(ResourceNotFoundException.class)
    ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        var problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setProperty("errorCode", ex.getErrorCode());
        return problem;
    }

    @ExceptionHandler(BusinessRuleException.class)
    ProblemDetail handleBusinessRule(BusinessRuleException ex) {
        var problem = ProblemDetail.forStatusAndDetail(HttpStatus.UNPROCESSABLE_ENTITY, ex.getMessage());
        problem.setProperty("errorCode", ex.getErrorCode());
        return problem;
    }

    @ExceptionHandler(ConflictException.class)
    ProblemDetail handleConflict(ConflictException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        var problem = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        problem.setTitle("Validation Failed");
        problem.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(e -> Map.of("field", e.getField(), "message", e.getDefaultMessage()))
            .toList());
        return problem;
    }

    // SB 2.x — ResponseEntity
    @ExceptionHandler(ResourceNotFoundException.class)
    ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }
}
```

---

## 5. Concurrencia

### `[Java 21+]` Virtual Threads (preferido para I/O intensivo)

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

```java
Thread.ofVirtual().name("invoice-worker").start(() -> processInvoice(inv));

try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> signInvoice(inv));
    executor.submit(() -> notifyClient(inv));
}
// Evitar synchronized con virtual threads — usar ReentrantLock
```

### `[Java 8+]` Thread pools (Java 8–20)

```java
ExecutorService executor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

// CompletableFuture (Java 8+) — preferido sobre Future.get() bloqueante
CompletableFuture<SignedData> signFuture = CompletableFuture
    .supplyAsync(() -> signerClient.sign(inv), executor);
```

### `[Java 21+]` Scoped Values — reemplazo de ThreadLocal

```java
public class TenantContext {
    public static final ScopedValue<String> SCHEMA = ScopedValue.newInstance();
}

ScopedValue.where(TenantContext.SCHEMA, "company_001")
    .run(() -> invoiceService.process(request));
```

### `[Java 8+]` ThreadLocal — usar en Java 8–20

```java
public class TenantContext {
    private static final ThreadLocal<String> SCHEMA = new ThreadLocal<>();

    public static void set(String schema)  { SCHEMA.set(schema); }
    public static String get()             { return SCHEMA.get(); }
    public static void clear()             { SCHEMA.remove(); }
}

// En filtro/interceptor — CRÍTICO: siempre limpiar en finally
try {
    TenantContext.set(extractSchema(request));
    chain.doFilter(request, response);
} finally {
    TenantContext.clear();
}
```

---

## 6. Clientes HTTP

### `[SB 4.x]` RestClient + @ImportHttpServices

```java
@Bean
RestClient restClient(RestClient.Builder builder) {
    return builder
        .baseUrl("${signer.base-url}")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .defaultStatusHandler(HttpStatusCode::isError, (req, res) ->
            { throw new SignerException("HTTP " + res.getStatusCode()); })
        .build();
}

@HttpExchange("/api/invoices")
public interface ExternalInvoiceClient {
    @GetExchange("/{id}")
    InvoiceResponse getById(@PathVariable String id);

    @PostExchange
    InvoiceResponse create(@RequestBody CreateInvoiceRequest body);
}

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
@Bean
ExternalInvoiceClient externalInvoiceClient(RestClient.Builder builder) {
    RestClient client = builder.baseUrl("${external.api.base-url}").build();
    return HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(client))
        .build()
        .createClient(ExternalInvoiceClient.class);
}
```

### `[SB 2.x / SB 3.x]` RestTemplate — deprecated en SB 3.x

```java
@Bean
RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
        .rootUri("${signer.base-url}")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .build();
}
```

### `[OPCIONAL]` FeignClient — SB 2.x / 3.x con Spring Cloud

> Usar solo si el proyecto ya tiene Spring Cloud o el equipo lo prefiere.
> En SB 4.x no está soportado — usar `@ImportHttpServices`.

```groovy
// build.gradle
implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
```

```java
// Habilitar en la clase principal o en @Configuration
@SpringBootApplication
@EnableFeignClients
public class Application { ... }

// Declarar el cliente
@FeignClient(name = "signer", url = "${signer.base-url}")
public interface SignerClient {

    @PostMapping("/sign")
    SignedResponse sign(@RequestBody SignRequest request);

    @GetMapping("/status/{id}")
    SignStatus getStatus(@PathVariable String id);
}

// Configuración por cliente (timeout, interceptores)
@FeignClient(
    name     = "external-api",
    url      = "${external.api.base-url}",
    configuration = FeignConfig.class
)
public interface ExternalApiClient { ... }

@Configuration
public class FeignConfig {
    @Bean
    RequestInterceptor authInterceptor(@Value("${external.api.token}") String token) {
        return template -> template.header("Authorization", "Bearer " + token);
    }
}

// Inyección normal — Spring gestiona el proxy
@Service
public class InvoiceServiceImpl implements InvoiceService {
    private final SignerClient signerClient;

    public InvoiceServiceImpl(SignerClient signerClient) {
        this.signerClient = signerClient;
    }
}
```

### `[Java 11+ / SB 2.x+]` java.net.http.HttpClient (sin Spring)

```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .build();

HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create(baseUrl + "/sign"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(objectMapper.writeValueAsString(body)))
    .build();

HttpResponse<String> response = client.send(req, HttpResponse.BodyHandlers.ofString());
```

---

## 7. Controllers (todas las versiones)

```java
@RestController
@RequestMapping("/api/v1/invoices")
public class InvoiceController {

    private final InvoiceService invoiceService;

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

    @GetMapping
    Page<InvoiceResponse> findAll(
        @RequestParam(required = false) String status,
        Pageable pageable
    ) {
        return invoiceService.findAll(status, pageable);
    }
}
```

---

## 8. Validación — Bean Validation (todas las versiones)

```java
// Java 17+ — record
public record CreateUserRequest(
    @NotBlank(message = "Name is required") String name,
    @Email @NotBlank String email,
    @Size(min = 8) String password
) {}

// Java 8/11 — clase con campos anotados
public final class CreateUserRequest {
    @NotBlank private final String name;
    @Email @NotBlank private final String email;
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

### Proyecciones

```java
// Java 17+ — record projection
public record InvoiceSummary(String code, String status, LocalDateTime createdAt) {}

// Java 8/11 — interface projection
public interface InvoiceSummary {
    String getCode();
    String getStatus();
    LocalDateTime getCreatedAt();
}

@Query("""
    SELECT new com.company.dto.response.InvoiceSummary(i.code, i.status, i.createdAt)
    FROM Invoice i WHERE i.status = :status
""")
List<InvoiceSummary> findSummaryByStatus(@Param("status") String status);
```

### @Transactional — service layer only

```java
@Service
@Transactional(readOnly = true)
public class InvoiceServiceImpl implements InvoiceService {

    @Override
    @Transactional
    public InvoiceResponse create(CreateInvoiceRequest request) { ... }
}
```

### N+1 — JOIN FETCH o @EntityGraph

```java
@Query("SELECT i FROM Invoice i JOIN FETCH i.customer WHERE i.company.id = :companyId")
List<Invoice> findWithCustomerByCompany(@Param("companyId") Long companyId);

@EntityGraph(attributePaths = {"customer", "establishment"})
List<Invoice> findByStatus(String status);
```

### Entidad JPA

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
    public Long getId()                    { return id; }
    public String getGenerationCode()      { return generationCode; }
    public InvoiceStatus getStatus()       { return status; }
    public void setStatus(InvoiceStatus s) { this.status = s; }
}
```

---

## 10. Paginación (todas las versiones SB)

### Repository

```java
// Spring Data infiere la query automáticamente
Page<Invoice> findByStatus(String status, Pageable pageable);

// Con @Query
@Query("SELECT i FROM Invoice i WHERE i.status = :status")
Page<Invoice> findByStatusPaged(@Param("status") String status, Pageable pageable);
```

### Service

```java
// Interface
Page<InvoiceResponse> findAll(String status, Pageable pageable);

// Implementation
@Override
public Page<InvoiceResponse> findAll(String status, Pageable pageable) {
    return repository.findByStatus(status, pageable)
        .map(mapper::toResponse);
}
```

### Controller — Spring resuelve Pageable del query string

```java
// GET /api/v1/invoices?page=0&size=10&sort=createdAt,desc
@GetMapping
Page<InvoiceResponse> findAll(
    @RequestParam(required = false) String status,
    Pageable pageable
) {
    return invoiceService.findAll(status, pageable);
}
```

### DTO de página personalizado (cuando no quieres exponer el tipo Page de Spring)

```java
// Java 17+ — record
public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean last
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isLast()
        );
    }
}

// Uso en controller
@GetMapping
PageResponse<InvoiceResponse> findAll(Pageable pageable) {
    return PageResponse.from(invoiceService.findAll(pageable));
}
```

### Configuración de límites `[SB 2.x+]`

```yaml
spring:
  data:
    web:
      pageable:
        default-page-size: 20
        max-page-size: 100
```

---

## 11. Mapper — MapStruct (todas las versiones Java 8+)

### Dependencia

```groovy
// build.gradle
implementation 'org.mapstruct:mapstruct:1.5.5.Final'
annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'

// Si también usas Lombok — el orden importa
annotationProcessor 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
```

### Mapper básico

```java
// MapStruct genera la implementación en tiempo de compilación
@Mapper(componentModel = "spring")
public interface InvoiceMapper {

    InvoiceResponse toResponse(Invoice invoice);

    Invoice toEntity(CreateInvoiceRequest request);

    List<InvoiceResponse> toResponseList(List<Invoice> invoices);
}
```

### Mapper con transformaciones

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    // Renombrar campo
    @Mapping(source = "fullName", target = "name")

    // Ignorar campo (p.ej. auto-generado)
    @Mapping(target = "id", ignore = true)

    User toEntity(CreateUserRequest request);

    // Formato de fecha
    @Mapping(source = "createdAt", target = "registeredDate",
             dateFormat = "dd/MM/yyyy")
    UserResponse toResponse(User user);
}
```

### Mapper con lógica personalizada

```java
@Mapper(componentModel = "spring", uses = { AddressMapper.class })
public interface InvoiceMapper {

    @Mapping(target = "customerName",
             expression = "java(invoice.getCustomer().getFullName())")
    @Mapping(target = "totalFormatted",
             expression = "java(formatAmount(invoice.getTotal()))")
    InvoiceResponse toResponse(Invoice invoice);

    default String formatAmount(BigDecimal amount) {
        return "$" + amount.setScale(2, RoundingMode.HALF_UP);
    }
}
```

### Inyección en servicio

```java
@Service
@Transactional(readOnly = true)
public class InvoiceServiceImpl implements InvoiceService {

    private final InvoiceRepository repository;
    private final InvoiceMapper     mapper;

    public InvoiceServiceImpl(InvoiceRepository repository, InvoiceMapper mapper) {
        this.repository = repository;
        this.mapper     = mapper;
    }

    @Override
    @Transactional
    public InvoiceResponse create(CreateInvoiceRequest request) {
        Invoice saved = repository.save(mapper.toEntity(request));
        return mapper.toResponse(saved);
    }

    @Override
    public Page<InvoiceResponse> findAll(String status, Pageable pageable) {
        return repository.findByStatus(status, pageable).map(mapper::toResponse);
    }
}
```

---

## 12. Migraciones de BD `[OPCIONAL]`

> Incluir esta sección solo si el usuario eligió Flyway o Liquibase.

### `[OPCIONAL — Flyway]` Migraciones SQL versionadas

```groovy
implementation 'org.flywaydb:flyway-core'
// Para PostgreSQL en SB 3.x+:
implementation 'org.flywaydb:flyway-database-postgresql'
```

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true   # útil al agregar Flyway a proyecto existente
```

```
src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__create_invoices_table.sql
└── V3__add_status_index.sql
```

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id         BIGSERIAL    PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP    NOT NULL DEFAULT NOW()
);

-- V2__create_invoices_table.sql
CREATE TABLE invoices (
    id              BIGSERIAL    PRIMARY KEY,
    generation_code VARCHAR(50)  NOT NULL UNIQUE,
    status          VARCHAR(20)  NOT NULL,
    user_id         BIGINT       NOT NULL REFERENCES users(id),
    created_at      TIMESTAMP    NOT NULL DEFAULT NOW()
);

-- V3__add_status_index.sql
CREATE INDEX idx_invoices_status ON invoices(status);
```

**Convenciones Flyway:**
- `V{versión}__{descripción}.sql` — migraciones versionadas (solo avanzan)
- `R__{descripción}.sql` — migraciones repetibles (se re-ejecutan si cambian)
- `U{versión}__{descripción}.sql` — undo (requiere Flyway Teams)
- Nunca modificar un script ya aplicado en producción — crear un nuevo `V{n+1}`

---

### `[OPCIONAL — Liquibase]` Changelogs YAML

```groovy
implementation 'org.liquibase:liquibase-core'
```

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-create-users.yaml
  - include:
      file: db/changelog/changes/002-create-invoices.yaml
```

```yaml
# db/changelog/changes/001-create-users.yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-users
      author: zentaury
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: name
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: created_at
                  type: TIMESTAMP
                  defaultValueComputed: NOW()
                  constraints:
                    nullable: false
```

**Flyway vs Liquibase:**

| | Flyway | Liquibase |
|---|---|---|
| Formato | SQL puro | YAML / XML / JSON / SQL |
| Simplicidad | ✅ Más simple | Más verboso |
| Multi-DB | Limitado | ✅ Mejor soporte |
| Rollback | Solo Teams (pago) | ✅ Incorporado |
| Popularidad SB | ✅ Más común | Menos común |

---

## 13. Configuración Spring Boot

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
    private Duration timeout    = Duration.ofSeconds(30);
    private int maxRetries      = 3;

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

    // MapStruct
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'

    // OPCIONAL — Flyway
    implementation 'org.flywaydb:flyway-core'

    // OPCIONAL — Liquibase
    // implementation 'org.liquibase:liquibase-core'

    // OPCIONAL — FeignClient (SB 2.x/3.x + Spring Cloud)
    // implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
}
```

---

## 14. Testing — JUnit 5, Mockito, Testcontainers

### Controller slice `[@WebMvcTest]`

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
    @Mock InvoiceMapper     mapper;

    InvoiceServiceImpl service;

    @BeforeEach
    void setUp() {
        service = new InvoiceServiceImpl(repository, mapper);
    }

    @Test
    void create_mapsAndPersists() {
        var request  = new CreateInvoiceRequest("INV-001", BigDecimal.TEN);
        var entity   = new Invoice();
        var response = new InvoiceResponse(1L, "INV-001");

        given(mapper.toEntity(request)).willReturn(entity);
        given(repository.save(entity)).willReturn(entity);
        given(mapper.toResponse(entity)).willReturn(response);

        assertThat(service.create(request)).isEqualTo(response);
        verify(repository).save(entity);
    }
}
```

### Repository slice — `@DataJpaTest`

```java
@DataJpaTest
class InvoiceRepositoryTest {

    @Autowired InvoiceRepository repository;

    @Test
    void findByStatus_returnsPaged() {
        repository.save(buildInvoice("ACCEPTED"));
        repository.save(buildInvoice("REJECTED"));

        Page<Invoice> result = repository.findByStatus("ACCEPTED", PageRequest.of(0, 10));

        assertThat(result.getTotalElements()).isEqualTo(1);
        assertThat(result.getContent().get(0).getStatus()).isEqualTo("ACCEPTED");
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
| Repository | `@DataJpaTest` | Solo JPA | Queries, proyecciones, paginación |
| Integration | `@SpringBootTest` + Testcontainers | Completo | Flujos cross-layer |

---

## 15. Spring Security

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

    @Bean @Override
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 16. Anti-Patterns — Por versión

| ❌ Evitar | ✅ Moderno | Versión mínima |
|---|---|---|
| `new Thread(...).start()` | `Thread.ofVirtual().start(...)` | Java 21 |
| `ExecutorService` para I/O | Virtual threads `newVirtualThreadPerTaskExecutor` | Java 21 |
| `ThreadLocal` sin `remove()` | `ThreadLocal` con `finally { ctx.clear() }` | Java 8 |
| `ThreadLocal` con virtual threads | `ScopedValue` | Java 25 |
| `RestTemplate` | `RestClient` | SB 3.2 |
| `FeignClient` en SB 4.x | `@ImportHttpServices` + `@HttpExchange` | SB 4.x |
| `FeignClient` en SB 3.x | `@HttpServiceProxyFactory` + `@HttpExchange` | SB 3.x |
| `@Autowired` en campo | Constructor injection | Siempre |
| POJO con setters para DTO | `record` (inmutable) | Java 17 |
| `instanceof` + cast explícito | Pattern matching `instanceof` | Java 16 |
| `instanceof` chain | Pattern matching `switch` | Java 21 |
| `@Value("${prop}")` por campo | `@ConfigurationProperties` | SB 2.x |
| Mapeo manual entity↔DTO | MapStruct `@Mapper` | Java 8+ |
| Checked exceptions en negocio | `extends RuntimeException` | Siempre |
| `Optional.get()` sin check | `orElseThrow(() -> new ResourceNotFoundException(...))` | Siempre |
| Entidad JPA fuera del servicio | Mapear siempre a DTO antes del controller | Siempre |
| `@Transactional` en controller | `@Transactional` solo en servicio | Siempre |
| Lazy loading en loop | `JOIN FETCH` o `@EntityGraph` | Siempre |
| `@SpringBootTest` para todo | Slice tests primero | SB 2.x |
| `BindingResult` en controller | `@Valid` + `@RestControllerAdvice` global | Siempre |
| `WebSecurityConfigurerAdapter` | `SecurityFilterChain` bean | SB 3.x |
| Migraciones SQL manuales | Flyway o Liquibase | Siempre |

---

## Quick Checklist — Antes de hacer commit

**Siempre (Java 8+):**
- [ ] Constructor injection — nunca `@Autowired` en campos
- [ ] Controllers solo manejan HTTP — lógica en servicios
- [ ] Servicios devuelven DTOs — nunca entidades JPA al controller
- [ ] Excepciones de negocio extienden `BusinessException` (unchecked)
- [ ] `ResourceNotFoundException` para 404, `BusinessRuleException` para 422
- [ ] MapStruct `@Mapper` en lugar de mapeo manual
- [ ] `@Transactional(readOnly = true)` en servicio, override para escrituras
- [ ] `@Valid` en todos los `@RequestBody`
- [ ] `@RestControllerAdvice` centraliza manejo de errores
- [ ] Sin N+1: `JOIN FETCH` o `@EntityGraph`
- [ ] Paginación con `Pageable` — nunca traer toda la tabla
- [ ] Unit tests con `@ExtendWith(MockitoExtension.class)` para servicios
- [ ] `@WebMvcTest` para capa de controllers
- [ ] `@DataJpaTest` para queries de repositorio
- [ ] `ThreadLocal.remove()` siempre en `finally`

**Java 17+:**
- [ ] DTOs son `record`, no clases con getters
- [ ] `PageResponse<T>` record para envolver `Page<T>` en respuestas
- [ ] Resultados de dominio modelados con `sealed interface` + `record`

**Java 21+:**
- [ ] Pattern matching con `switch`
- [ ] `spring.threads.virtual.enabled=true` en application.yml
- [ ] `ScopedValue` para contexto compartido

**SB 3.2+ / 4.x:**
- [ ] `RestClient` para llamadas HTTP salientes
- [ ] `@ConfigurationProperties` con `record` para configuración tipada

**SB 4.x:**
- [ ] `@ImportHttpServices` para clientes HTTP declarativos
- [ ] Imports de `jakarta.*` (no `javax.*`)

**OPCIONAL — si el proyecto usa migraciones:**
- [ ] Scripts Flyway nombrados `V{n}__{descripcion}.sql` — nunca modificar los ya aplicados
- [ ] ChangeSets Liquibase con `id` único e `author`
- [ ] Scripts de test contra BD real con Testcontainers
