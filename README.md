# java-master

A Claude Code skill for writing idiomatic Java + Spring Boot code across all LTS versions.

## What it covers

- **All Java LTS versions:** 8, 11, 17, 21, 25 — with version-specific patterns for each feature
- **Spring Boot:** 2.x, 3.x, and 4.x — including migration paths between versions
- **Layered architecture** — package structure, naming conventions, dependency rules
- **DTOs** — Records (Java 17+) and immutable classes (Java 8/11)
- **Domain modeling** — Sealed interfaces, pattern matching switch, pattern matching instanceof
- **Concurrency** — Virtual threads (Java 21+), Scoped Values (Java 25), ThreadLocal, CompletableFuture
- **HTTP clients** — RestClient, @ImportHttpServices (SB4), @HttpServiceProxyFactory (SB3), RestTemplate (SB2)
- **Persistence** — Spring Data JPA, projections, @Transactional, N+1 avoidance
- **Validation** — Bean Validation, custom constraints, global error handling
- **Testing** — @WebMvcTest, @DataJpaTest, @ExtendWith(MockitoExtension), Testcontainers
- **Security** — Spring Security stateless JWT, SecurityFilterChain, WebSecurityConfigurerAdapter (SB2)

## Version compatibility matrix

| Feature | Java 8 | Java 11 | Java 17 | Java 21 | Java 25 |
|---|---|---|---|---|---|
| Records | ❌ | ❌ | ✅ | ✅ | ✅ |
| Sealed classes | ❌ | ❌ | ✅ | ✅ | ✅ |
| Pattern matching switch | ❌ | ❌ | ❌ | ✅ | ✅ |
| Virtual threads | ❌ | ❌ | ❌ | ✅ | ✅ |
| Scoped Values | ❌ | ❌ | ❌ | Preview | ✅ |

| Feature | SB 2.x | SB 3.x | SB 4.x |
|---|---|---|---|
| RestTemplate | ✅ | deprecated | ❌ |
| RestClient | ❌ | ✅ (3.2+) | ✅ |
| @ImportHttpServices | ❌ | ❌ | ✅ |
| Jackson 3 | ❌ | ❌ | ✅ |

## Installation

### CLI (recomendado — Claude Code, Gemini CLI, GitHub Copilot, Cursor y más)

```bash
npx skills add zentaury/java-master -g
```

Instala globalmente y enlaza automáticamente a todos los agentes compatibles.

### Claude Desktop App

1. Descarga el ZIP desde [Releases](https://github.com/zentaury/java-master/releases) o clona el repo y comprime la carpeta
2. Abre Claude Desktop → **Settings → Extensions → Install from file**
3. Selecciona el ZIP

### Manual (git clone)

```bash
git clone https://github.com/zentaury/java-master.git ~/.agents/skills/java-master
```

## Usage

La skill se activa automáticamente cuando Claude detecta una tarea de Java o Spring Boot. También puedes invocarla explícitamente con `/java-master`.

## License

MIT
