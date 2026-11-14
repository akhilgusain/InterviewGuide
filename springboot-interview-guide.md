# Spring Boot Interview Guide — 9 Years Experience (Senior / Lead Level)

> **How to use this file:** Each answer has a **one-line hook** (what to say first), the **detailed explanation**, and where useful a **memory trick**. In interviews, lead with the hook, then expand. Interviewers at 9-YOE level care less about definitions and more about *trade-offs, failure modes, and "have you actually operated this in production?"*
>
> **Golden rule for senior interviews:** Never answer only "what it is." Always add *why it exists*, *what breaks without it*, and *what you'd choose in production*.

---

## Table of Contents

1. [Spring Boot Core & Auto-Configuration](#1-spring-boot-core--auto-configuration)
2. [Spring Core: IoC, DI, Beans](#2-spring-core-ioc-di-beans)
3. [Configuration, Profiles & Externalized Properties](#3-configuration-profiles--externalized-properties)
4. [REST API Design & Exception Handling](#4-rest-api-design--exception-handling)
5. [Spring Data JPA & Hibernate](#5-spring-data-jpa--hibernate)
6. [Transaction Management](#6-transaction-management)
7. [Spring AOP](#7-spring-aop)
8. [Caching](#8-caching)
9. [Async, Scheduling & Concurrency](#9-async-scheduling--concurrency)
10. [Spring Security, JWT & OAuth2](#10-spring-security-jwt--oauth2)
11. [Actuator, Observability & Production Readiness](#11-actuator-observability--production-readiness)
12. [Testing](#12-testing)
13. [Microservices & Cloud Patterns](#13-microservices--cloud-patterns)
14. [Messaging (Kafka / RabbitMQ)](#14-messaging-kafka--rabbitmq)
15. [Performance Tuning & JVM](#15-performance-tuning--jvm)
16. [Spring Boot 3.x, Virtual Threads & Native Image](#16-spring-boot-3x-virtual-threads--native-image)
17. [Spring WebFlux vs Spring MVC](#17-spring-webflux-vs-spring-mvc)
18. [Scenario-Based Questions (40+)](#18-scenario-based-questions)
19. [System Design Rounds for Spring Boot Devs](#19-system-design-rounds)
20. [Rapid-Fire Recall Tables](#20-rapid-fire-recall-tables)
21. [Behavioural / Lead-Level Questions](#21-behavioural--lead-level-questions)
22. [Last-Minute Cheat Sheet](#22-last-minute-cheat-sheet)

---

## 1. Spring Boot Core & Auto-Configuration

### Q1. What problem does Spring Boot actually solve?

**Hook:** "Spring Boot didn't add new capability to Spring — it removed the configuration tax."

**Detailed:**
Classic Spring required XML/Java config for everything: DispatcherServlet, view resolvers, DataSource, EntityManagerFactory, transaction manager, Jackson converters. Two developers building the same app produced two very different setups.

Spring Boot gives four things:

| Pillar | What it does |
|---|---|
| **Starters** | Curated, version-aligned dependency bundles (`spring-boot-starter-web`) |
| **Auto-configuration** | Conditionally creates beans based on classpath + existing beans + properties |
| **Embedded server** | Tomcat/Jetty/Undertow inside the JAR → deployable artifact, no external container |
| **Production-ready features** | Actuator, metrics, health checks, externalized config |

**Memory trick:** **S-A-E-P** — *Starters, Autoconfig, Embedded, Production-ready*.

**Explained with an example:** to build a REST API with a database in classic Spring you had to add a dozen dependencies yourself (`spring-webmvc`, `jackson`, `hibernate`, `hikari`, `tomcat`...), hunt for versions that work together, then write config for each. With Boot you add two starters and write a controller:

```xml
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
```

- A **starter** is just a dependency descriptor ("a menu item") that pulls in a compatible set of libraries; the version numbers are managed centrally by `spring-boot-dependencies`, so you rarely write them.
- **Auto-configuration** then notices "Hibernate and a JDBC driver are on the classpath and `spring.datasource.url` is set" and creates `DataSource`, `EntityManagerFactory`, and the transaction manager for you, using sensible defaults you can override with properties or your own beans.
- **Embedded server** means `java -jar app.jar` starts the whole app; there's no separate Tomcat to install and configure, which fits containers.
- **Production-ready** = Actuator health/metrics endpoints, externalized config and graceful shutdown out of the box.

**Important framing:** Boot is *convention over configuration* — opinionated defaults, but never a lock-in: every default can be replaced. It sits on top of the Spring Framework (DI, MVC, data access), so all Spring knowledge still applies.

---

### Q2. Explain `@SpringBootApplication` in depth.

It is a meta-annotation combining three:

```java
@SpringBootConfiguration   // = @Configuration, marks this as a bean-definition source
@EnableAutoConfiguration   // triggers the auto-config machinery
@ComponentScan             // scans the current package and all sub-packages
```

**Senior-level points interviewers look for:**
- `@ComponentScan` defaults to the **package of the main class**. Put the main class in the root package (`com.company.app`), otherwise beans in sibling packages are silently not found — a classic "why is my bean null" bug.
- `@SpringBootConfiguration` is a specialization that also enables **proxyBeanMethods** semantics; only one such class should exist per app (tests find it via `@SpringBootTest`).
- You can exclude auto-configs: `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`.

**Plain-language breakdown:**
- A **meta-annotation** is an annotation that is itself made of other annotations — writing one `@SpringBootApplication` is exactly the same as writing all three above on the class. It exists purely to cut boilerplate.
- *Why a "bean is null / not found" happens:* component scanning starts at the main class's package and walks **downwards** only. If the main class is `com.company.app.Application` but a service lives in `com.company.shared.PaymentService`, it is never scanned. Fixes: move the main class up, or add `@SpringBootApplication(scanBasePackages = {"com.company.app", "com.company.shared"})`.
- *When you'd exclude an auto-config:* e.g. a library on the classpath pulls in JDBC, but this service has no database yet — Boot would fail at startup trying to build a `DataSource`. Excluding `DataSourceAutoConfiguration` (or setting `spring.autoconfigure.exclude=...` in properties) tells Boot "do not even try".
- **Follow-up probe:** *"Can you split the three annotations?"* Yes — put `@Configuration @EnableAutoConfiguration @ComponentScan("...")` on any class for finer control, but the combined annotation is the convention.

---

### Q3. How does auto-configuration actually work internally? (Very common at senior level)

**Step-by-step:**

1. `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`.
2. The selector reads candidate configuration class names from
   - **Spring Boot ≥ 2.7 / 3.x:** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
   - **Spring Boot ≤ 2.6:** `META-INF/spring.factories`
3. Candidates are **filtered** by `@Conditional` evaluation.
4. Ordering applied via `@AutoConfigureBefore`, `@AutoConfigureAfter`, `@AutoConfigureOrder`.
5. Surviving configuration classes register their beans.

**The `@Conditional` family — know these cold:**

| Annotation | Fires when |
|---|---|
| `@ConditionalOnClass` / `@ConditionalOnMissingClass` | Class present/absent on classpath |
| `@ConditionalOnBean` / `@ConditionalOnMissingBean` | Bean already defined or not |
| `@ConditionalOnProperty` | Property present with a value |
| `@ConditionalOnResource` | File exists (e.g. `classpath:schema.sql`) |
| `@ConditionalOnWebApplication` | Servlet / reactive / none |
| `@ConditionalOnExpression` | SpEL evaluates true |
| `@ConditionalOnJava` | JVM version range |

**The key insight:** `@ConditionalOnMissingBean` is what makes "your bean wins" work. Auto-config classes are processed **after** user configuration, so if you define your own `DataSource`, Boot's backs off.

**Debugging tool:** run with `--debug` or set `debug=true` → prints the **Condition Evaluation Report** showing *Positive matches*, *Negative matches*, and *Exclusions*. Mentioning this report scores points.

---

### Q4. How do you write your own auto-configuration / custom starter?

**Structure:** two modules — `xyz-spring-boot-autoconfigure` (the config) and `xyz-spring-boot-starter` (just dependencies). Naming convention: never prefix with `spring-boot` (that's reserved by the Spring team) — use `mycompany-spring-boot-starter`.

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnClass(AuditClient.class)
@ConditionalOnProperty(prefix = "audit", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean          // let users override
    public AuditClient auditClient(AuditProperties props) {
        return new AuditClient(props.getEndpoint(), props.getTimeout());
    }
}
```

Then register it in
`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
com.mycompany.audit.AuditAutoConfiguration
```

Add `spring-boot-configuration-processor` as an optional dependency so IDE auto-completion works for your properties.

**Real-world framing:** "In my last role we built an internal starter that wired logging correlation IDs, standard exception handlers, Actuator tags, and a Kafka audit publisher. Twelve services dropped ~400 lines of boilerplate each."

---

### Q5. Fat JAR vs WAR — and how does the executable JAR work?

- Spring Boot repackages into a **fat/uber JAR** with a nested layout: your classes in `BOOT-INF/classes`, dependencies in `BOOT-INF/lib`.
- `MANIFEST.MF` `Main-Class` is `JarLauncher` (Spring Boot's own), and `Start-Class` is your `@SpringBootApplication` class.
- `JarLauncher` installs a custom `LaunchedURLClassLoader` that can read nested JARs — plain Java cannot load a JAR inside a JAR.
- Nested JARs are stored **uncompressed** (STORED) so they can be memory-mapped and read without extraction.

**WAR still makes sense when:** corporate policy mandates a shared app server, or you need container-managed resources (JNDI DataSource). Then extend `SpringBootServletInitializer` and set Tomcat scope to `provided`.

**Why a "fat JAR" at all:** a normal JAR contains only your classes and needs the dependency JARs on the classpath (`java -cp a.jar:b.jar:...`). Boot bundles everything into **one runnable file** so deployment is "copy one file, run `java -jar`". The catch: the standard JVM class loader can't read a JAR nested inside another JAR, so Boot's own `JarLauncher` is the `Main-Class`; it sets up the special class loader first, then calls your `Start-Class` main method. (Contrast with a *shaded* JAR, which unpacks and merges all classes into one flat JAR and can cause duplicate-file conflicts.) Run it with `java -jar app.jar`, or build with `mvn package` (the `spring-boot-maven-plugin` `repackage` goal produces it).

**Why layered JARs help Docker:** Docker caches each image layer and only rebuilds layers whose files changed. Your code changes on every commit, but third-party dependencies rarely change. Separating them into layers means a code change only re-uploads a few KB instead of the whole 80 MB fat JAR.

**Layered JARs (Boot 2.3+)** for Docker: `layers.idx` splits dependencies / spring-boot-loader / snapshot-deps / application so Docker layer caching only rebuilds the thin app layer.

```dockerfile
FROM eclipse-temurin:21-jre AS builder
COPY app.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre
COPY --from=builder dependencies/ ./
COPY --from=builder spring-boot-loader/ ./
COPY --from=builder snapshot-dependencies/ ./
COPY --from=builder application/ ./
ENTRYPOINT ["java","org.springframework.boot.loader.launch.JarLauncher"]
```

---

### Q6. Explain the Spring Boot startup sequence.

1. `SpringApplication.run()` → creates `SpringApplication`, deduces `WebApplicationType` (SERVLET / REACTIVE / NONE) from the classpath.
2. Loads `ApplicationContextInitializer`s and `ApplicationListener`s from `spring.factories`.
3. Prepares `Environment` → property sources resolved, profiles activated, `ApplicationEnvironmentPreparedEvent` fired.
4. Prints the banner, creates the `ApplicationContext`.
5. Loads bean definitions (your `@Configuration` + auto-configs).
6. `refresh()` → BeanFactoryPostProcessors → BeanPostProcessors → instantiate singletons → embedded server starts.
7. Calls `ApplicationRunner` / `CommandLineRunner` beans.
8. Fires `ApplicationReadyEvent`.

**What those steps mean in plain language:**
- *Why deduce `WebApplicationType`?* Boot has to know whether to start a servlet container (Spring MVC on the classpath), a Netty server (WebFlux), or no server at all (batch/CLI app). It simply checks which classes exist (e.g. `DispatcherServlet` vs `DispatcherHandler`).
- *`Environment`* is Spring's view of configuration: all property sources (files, env vars, CLI args) plus the active profiles. It is built **before** beans exist so that `@ConditionalOnProperty` and `@Profile` can be evaluated.
- *`refresh()`* is the heart of the container: first `BeanFactoryPostProcessor`s edit bean definitions (e.g. resolve `${...}` placeholders), then `BeanPostProcessor`s are registered, then all non-lazy singletons are created, and finally the embedded Tomcat starts accepting traffic (that is why a failing bean prevents the port from ever opening).
- *Runners* are hooks that run **after** the context is ready but before `run()` returns:

```java
@Component
class CacheWarmer implements ApplicationRunner {
    public void run(ApplicationArguments args) { /* load reference data */ }
}
```

`CommandLineRunner` gets raw `String[]` args; `ApplicationRunner` gets parsed `--key=value` options. Use `@Order` when you have several.

**Event order to memorize:** Starting → EnvironmentPrepared → ContextPrepared → ContextLoaded → **Started** → **Ready** → (Failed on error).

**Memory trick:** *"Start Environment, Prepare Context, Load, Start, Ready."*

**Practical use:** do warm-up work (cache priming, connection pool preheating) in `ApplicationReadyEvent`, **not** in a `@PostConstruct` — the context isn't fully up in `@PostConstruct` and failures there are harder to diagnose.

---

## 2. Spring Core: IoC, DI, Beans

### Q7. Constructor vs setter vs field injection — which and why?

**Answer: constructor injection, always, for mandatory dependencies.**

| Reason | Explanation |
|---|---|
| **Immutability** | Fields can be `final` → thread-safe, no reassignment |
| **Fail fast** | Missing dependency = context fails at startup, not `NullPointerException` at 2 AM |
| **Testability** | `new OrderService(mockRepo)` — no reflection, no Spring needed |
| **Design feedback** | A 9-argument constructor screams "this class does too much." Field injection hides that |
| **No circular deps** | Constructor cycles fail at startup; field-injection cycles silently "work" via proxies |

Since Spring 4.3, `@Autowired` is **optional** on a single constructor.

Use setter injection only for genuinely optional dependencies; use field injection essentially only in tests.

---

### Q8. Bean scopes.

| Scope | Lifetime |
|---|---|
| `singleton` (default) | One per container |
| `prototype` | New instance per injection/lookup — **Spring does not manage its destruction** |
| `request` | Per HTTP request |
| `session` | Per HTTP session |
| `application` | Per ServletContext |
| `websocket` | Per WebSocket session |

**The classic trap:** injecting a `prototype` into a `singleton` gives you **one** instance forever — the singleton is wired once. Fixes:
1. `ObjectProvider<T>` / `ObjectFactory<T>` and call `.getObject()` per use (preferred, no CGLIB).
2. `@Lookup` method injection.
3. Scoped proxy: `@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)`.

```java
@Service
public class ReportService {
    private final ObjectProvider<ReportBuilder> builders;   // prototype
    ReportService(ObjectProvider<ReportBuilder> builders) { this.builders = builders; }

    public Report build() { return builders.getObject().render(); }  // fresh each time
}
```

---

### Q9. Bean lifecycle in order.

```
Instantiate → Populate properties (DI) → BeanNameAware/BeanFactoryAware/ApplicationContextAware
→ BeanPostProcessor.postProcessBeforeInitialization
→ @PostConstruct → InitializingBean.afterPropertiesSet() → custom init-method
→ BeanPostProcessor.postProcessAfterInitialization   ← AOP proxies created HERE
→ [BEAN IN USE]
→ @PreDestroy → DisposableBean.destroy() → custom destroy-method
```

**Memory trick:** *"Create, Inject, Aware, Before, Init, After — Use — PreDestroy, Destroy."*

**In plain language:** Spring first calls your constructor, then injects dependencies, then lets the bean "meet" the container through optional `*Aware` interfaces (e.g. `ApplicationContextAware` hands you the context, `BeanNameAware` your bean name). `BeanPostProcessor`s are Spring's extension hooks that see **every** bean right before and right after initialization — this is how `@Autowired`, `@Value`, and AOP proxies are implemented. Your own `@PostConstruct` is the best place for "dependencies are injected, now validate/prepare" logic; `@PreDestroy` for releasing resources (closing a client, flushing a buffer) on shutdown.

**Pitfall:** `prototype` beans never get their destroy callbacks called by Spring — the container hands them over and forgets them.

**Why the proxy detail matters:** because proxies are created in `postProcessAfterInitialization`, `@Transactional`/`@Async`/`@Cacheable` **do not work inside `@PostConstruct`** — the proxy doesn't exist yet.

---

### Q10. How do you resolve ambiguity when multiple beans of the same type exist?

Precedence order: `@Qualifier` > `@Primary` > by-name match of the field/parameter.

```java
@Bean @Primary  PaymentGateway stripe() { ... }
@Bean("paypal") PaymentGateway paypal() { ... }

// consumer
OrderService(@Qualifier("paypal") PaymentGateway gw) { ... }
```

Advanced: **custom qualifier annotations** (cleaner than string names, refactor-safe):

```java
@Qualifier @Retention(RUNTIME) public @interface Paypal {}
```

Or inject **all implementations** as `List<PaymentGateway>` / `Map<String, PaymentGateway>` (map key = bean name) — the standard Strategy-pattern approach. Combine with `@Order` to control list order.

```java
@Service
public class PaymentRouter {
    private final Map<String, PaymentGateway> gateways;   // "stripe" -> bean
    public void pay(String provider, Money m) {
        gateways.getOrDefault(provider, gateways.get("stripe")).charge(m);
    }
}
```

---

### Q11. Circular dependencies — how do you detect and fix them?

Spring resolves *setter/field* cycles using the **three-level cache** (`singletonObjects`, `earlySingletonObjects`, `singletonFactories`) by exposing a half-built bean. Constructor cycles cannot be resolved → `BeanCurrentlyInCreationException`.

**What a circular dependency is:** `A` needs `B` and `B` needs `A`. With constructor injection neither can be built first, so Spring gives up. With setter/field injection Spring can create `A` with no dependencies filled yet, stash that half-built `A` in an "early reference" cache (level 3 builds it, level 2 holds it, level 1 holds finished singletons), build `B` using the early `A`, then finish `A`. It "works", but `B` may hold a not-yet-initialized `A`, which causes subtle bugs and hides a design smell.

```java
// Smell: OrderService <-> InventoryService
// Fix: extract StockReservation (used by both), so the dependency becomes a one-way graph
@Service class StockReservation { /* shared logic */ }
@Service class OrderService     { OrderService(StockReservation s) {} }
@Service class InventoryService { InventoryService(StockReservation s) {} }
```

**Since Spring Boot 2.6, circular references are prohibited by default.** `spring.main.allow-circular-references=true` is an escape hatch, not a fix.

**Proper fixes (in order of preference):**
1. **Extract a third collaborator** holding the shared logic — this is almost always the right answer and shows design maturity.
2. Publish an `ApplicationEvent` instead of calling back.
3. `@Lazy` on one injection point (injects a proxy) — tactical only.
4. `ObjectProvider` for on-demand lookup.

---

### Q12. `BeanFactoryPostProcessor` vs `BeanPostProcessor`.

| | BeanFactoryPostProcessor | BeanPostProcessor |
|---|---|---|
| Operates on | Bean **definitions** (metadata) | Bean **instances** |
| Runs | Before any bean instantiation | Around each bean's initialization |
| Example | `PropertySourcesPlaceholderConfigurer`, `ConfigurationClassPostProcessor` | `AutowiredAnnotationBeanPostProcessor`, `AnnotationAwareAspectJAutoProxyCreator` |

**Memory trick:** *Factory* processor edits the **blueprint**; *Bean* processor edits the **building**.

**Why two hooks exist:** Spring builds beans in two phases — (1) read all bean *definitions* (class name, scope, constructor args) into a registry, (2) create *instances* from them. Each phase needs its own extension point. A `BeanFactoryPostProcessor` can change a definition before any object exists (e.g. replace `${db.url}` with the real value); a `BeanPostProcessor` can wrap or modify each finished object (e.g. return a transactional proxy instead of the raw service).

```java
@Component
class TimingBeanPostProcessor implements BeanPostProcessor {
    public Object postProcessAfterInitialization(Object bean, String name) {
        // could return a wrapper/proxy here instead of the original bean
        return bean;
    }
}
```

**Pitfall:** a `BeanFactoryPostProcessor` declared as a non-static `@Bean` method forces its enclosing config class to be created very early, so beans there miss other post-processing (you'll see "not eligible for getting processed by all BeanPostProcessors" in logs). Declare such `@Bean` methods `static`.

---

### Q13. `@Component` vs `@Bean`.

- `@Component` (+ `@Service`, `@Repository`, `@Controller`): classpath-scanned, you own the class.
- `@Bean`: method inside `@Configuration`, used for **third-party classes** you can't annotate, or when construction needs logic/conditions.
- `@Repository` additionally enables **exception translation** — vendor `SQLException`s become Spring's `DataAccessException` hierarchy.

**Why the difference matters:** `@Component` annotates a *class* (Spring creates it for you); `@Bean` annotates a *factory method* (you create the object and hand it to Spring). You cannot add `@Component` to a library class like `ObjectMapper` or `RestTemplate`, so you write a factory:

```java
@Configuration
class HttpConfig {
    @Bean
    RestTemplate restTemplate(RestTemplateBuilder b) {
        return b.connectTimeout(Duration.ofSeconds(2)).readTimeout(Duration.ofSeconds(5)).build();
    }
}
```

`@Service` / `@Controller` carry no extra behaviour over `@Component` (they only document the layer, and `@Controller` is detected by Spring MVC); `@Repository` is the one with real extra behaviour (exception translation).

**`proxyBeanMethods`:** in a `@Configuration` class, calling one `@Bean` method from another returns the *same singleton* because Spring CGLIB-proxies the config class. Setting `@Configuration(proxyBeanMethods = false)` ("lite mode") skips that proxy for faster startup — then inter-method calls create new objects, so pass dependencies as method parameters instead.

---

## 3. Configuration, Profiles & Externalized Properties

### Q14. Property source precedence (highest → lowest) — know the top few.

1. Devtools global settings (`~/.config/spring-boot`)
2. `@TestPropertySource` / test `properties` attribute
3. **Command-line arguments** (`--server.port=9090`)
4. `SPRING_APPLICATION_JSON`
5. ServletConfig / ServletContext params
6. JNDI (`java:comp/env`)
7. **Java System properties** (`-Dserver.port=`)
8. **OS environment variables**
9. Profile-specific `application-{profile}.yml` (outside jar, then inside)
10. `application.yml` (outside jar, then inside)
11. `@PropertySource`
12. **Default properties** (`SpringApplication.setDefaultProperties`)

**Memory trick — "CJ ES AD":** **C**ommand-line → **J**SON → **S**ystem props → **E**nv vars → **A**pplication yml → **D**efaults. In practice, just remember: *command line beats system props beats env vars beats files, and profile-specific beats plain.*

**Relaxed binding:** `spring.datasource.url` ≡ `SPRING_DATASOURCE_URL` ≡ `spring.datasource.URL`. That's how 12-factor container env vars work.

**Why precedence exists and how to use it:** the same property can be defined in many places (jar, server file, env var, CLI). Spring keeps all of them in an ordered list of *property sources* and the **first source containing the key wins**. This lets you ship sane defaults inside the jar and override per environment without rebuilding. Example — `application.yml` in the jar says `server.port: 8080`:

```bash
java -jar app.jar                                  # 8080 (file)
SERVER_PORT=9000 java -jar app.jar                 # 9000 (env var beats file)
SERVER_PORT=9000 java -Dserver.port=9100 -jar app.jar   # 9100 (system prop beats env)
java -jar app.jar --server.port=9200               # 9200 (CLI beats everything below it)
```

**Pitfalls / probes:** (1) Env-var names can't contain `.` or `-`, so `my.api-key` becomes `MY_APIKEY` (relaxed binding strips dashes). (2) Lists in env vars use indexes: `MY_HOSTS_0=a`, `MY_HOSTS_1=b`. (3) A file placed next to the jar (`./config/application.yml`) overrides the one packaged inside — handy for ops overrides.

---

### Q15. `@Value` vs `@ConfigurationProperties` — which do you use?

**`@ConfigurationProperties`, for anything non-trivial.**

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Binding | Single value, SpEL supported | Whole hierarchical group |
| Relaxed binding | ❌ Exact name only | ✅ |
| Validation | Manual | ✅ JSR-380 via `@Validated` |
| Type-safe / testable | Strings everywhere | POJO, IDE-completable |
| Metadata / docs | No | Yes (configuration-processor) |

```java
@Validated
@ConfigurationProperties(prefix = "payment")
public record PaymentProperties(
        @NotBlank String apiKey,
        @NotNull @DurationUnit(ChronoUnit.SECONDS) Duration timeout,
        @Min(1) @Max(10) int maxRetries,
        Map<String, String> providerUrls,
        List<String> allowedCurrencies) {}
```

Enable with `@EnableConfigurationProperties(PaymentProperties.class)` or `@ConfigurationPropertiesScan` on the main class.
**Big win:** invalid config fails the app at **startup**, not on first request.

---

### Q16. Profiles — and what's the modern way to do multi-document YAML?

```yaml
spring:
  config:
    activate:
      on-profile: prod          # Boot 2.4+ replaced the old `spring.profiles`
  datasource:
    url: jdbc:postgresql://prod-db:5432/app
---
spring:
  config:
    activate:
      on-profile: dev
```

- Activate: `--spring.profiles.active=prod,eu-west` or `SPRING_PROFILES_ACTIVE`.
- `spring.profiles.include` / `spring.profiles.group` (2.4+) to compose: `spring.profiles.group.prod=prod-db,prod-mq,metrics`.
- `@Profile("!test")` on beans; supports `&`, `|`, `!`.
- `spring.config.import: optional:configserver:` / `optional:file:./extra.yml` (2.4+) — the modern replacement for `bootstrap.yml`.

**What a profile is:** a named switch ("dev", "prod", "test") that turns on a subset of beans (`@Profile`) and/or config files (`application-prod.yml`). It exists so one build artifact can behave differently per environment. In the multi-document YAML above, `---` separates documents, and each document with `on-profile` is only applied when that profile is active; the document without it is the common baseline.

```java
@Service @Profile("!prod")          // active in every profile except prod
class LoggingEmailSender implements EmailSender { /* prints instead of sending */ }

@Service @Profile("prod")
class SmtpEmailSender implements EmailSender { /* real SMTP */ }
```

`optional:` in `spring.config.import` means "don't fail startup if that source is missing" (useful locally when no Config Server is running). **Pitfall:** if no profile is active Boot uses the `default` profile — forgetting `SPRING_PROFILES_ACTIVE` in a container silently runs prod with dev settings (or vice-versa).

**Senior nuance:** don't use profiles for *environment values* (URLs, credentials) — use externalized config/Vault/ConfigMaps and keep profiles for *behavioural* variation (e.g. `@Profile("!prod")` for a stub email sender). Profile-per-environment inside the jar means rebuilding to change a URL, breaking build-once-deploy-anywhere.

---

### Q17. How do you manage secrets?

Never in `application.yml` in Git. Options in order of maturity:
1. **HashiCorp Vault** (`spring-cloud-starter-vault-config`) with dynamic DB credentials + lease renewal.
2. **AWS Secrets Manager / Parameter Store**, GCP Secret Manager, Azure Key Vault.
3. **Kubernetes Secrets** mounted as files + `spring.config.import: optional:configtree:/etc/secrets/`.
4. **Jasypt** encrypted properties (`ENC(...)`) — legacy, key still has to live somewhere.
5. `spring-cloud-config-server` with encryption keys.

Also mention: rotate without redeploy via `@RefreshScope` + `/actuator/refresh`, and scrub secrets from Actuator with `management.endpoint.env.show-values=NEVER` (sanitize keys like `password`, `secret`, `key`, `token`).

**Why this matters & how each option works:**
- *The problem:* anything committed to Git lives in history forever, is visible to everyone with repo access, and ends up in build logs/images. Secrets should be injected at **runtime** and be **rotatable**.
- **Vault dynamic credentials:** instead of one long-lived DB password, Vault creates a short-lived DB user per app instance and revokes it after the *lease* (time-to-live) expires; the app renews the lease while running. A leaked credential is therefore useless within hours.
- **Kubernetes Secret as files (configtree):** each file in the mounted directory becomes a property — file `/etc/secrets/db.password` becomes `db.password`. Preferred over env vars because env vars leak easily via `/proc`, crash dumps, and `docker inspect`.
- **Jasypt:** the value in Git is `ENC(xyz)` and is decrypted with a master password supplied at runtime — it only moves the secret problem to the master key, hence "legacy".
- *`@RefreshScope`:* beans annotated with it are re-created when `/actuator/refresh` is called, so they pick up a new secret without a restart. Caveat: only beans that read the value at creation benefit; a long-lived connection pool must be rebuilt too.
- **Probe:** *"A secret was committed by accident — what now?"* Rotate it immediately (removing from Git history is not enough), then purge history and add a pre-commit scanner (gitleaks).

---

## 4. REST API Design & Exception Handling

### Q18. `@Controller` vs `@RestController`; `@RequestBody` vs `@ModelAttribute`.

- `@RestController` = `@Controller` + `@ResponseBody` on every method → return values go through `HttpMessageConverter` (Jackson) into the body instead of being resolved as view names.
- `@RequestBody` → deserializes the request body (JSON/XML). `@ModelAttribute` → binds query/form params to an object. `@RequestParam` → single param. `@PathVariable` → URI template segment.
- `ResponseEntity<T>` when you need to control **status + headers** (201 with `Location`, 304, caching headers).

**Explained:**
- *Why `@Controller` returns views:* in classic MVC the return value `"orders"` is a **view name** (e.g. a Thymeleaf template). A REST API wants the object itself written as JSON, so `@ResponseBody` tells Spring "skip view resolution, serialize the return value". `@RestController` just applies that to every method.
- *Which binder is used:* `@RequestBody` uses an `HttpMessageConverter` chosen by `Content-Type` (JSON → Jackson). `@ModelAttribute` (the default for plain objects with no annotation) binds form fields / query string keys to setters by name; it can't read a JSON body.

```java
@GetMapping("/orders/{id}")                        // /orders/42?expand=items
Order get(@PathVariable long id, @RequestParam(defaultValue = "none") String expand) { ... }

@PostMapping("/orders")
ResponseEntity<Order> create(@RequestBody @Valid CreateOrder req) {
    Order o = service.create(req);
    return ResponseEntity.created(URI.create("/orders/" + o.id())).body(o);   // 201 + Location header
}
```

**Probe:** *"What status for create vs update vs delete?"* — create 201 (+`Location`), update 200 or 204, delete 204, validation error 400, not found 404, conflict 409.

---

### Q19. Global exception handling — how do you design it?

`@RestControllerAdvice` + a consistent error contract. Since Boot 3, prefer **RFC 7807 `ProblemDetail`**.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    ProblemDetail notFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource not found");
        pd.setType(URI.create("https://api.acme.com/errors/not-found"));
        pd.setProperty("traceId", MDC.get("traceId"));   // links log to response
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail invalid(MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(FieldError::getField, f ->
                     Optional.ofNullable(f.getDefaultMessage()).orElse("invalid"), (a,b) -> a));
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Validation failed");
        pd.setProperty("fieldErrors", errors);
        return pd;
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail unexpected(Exception ex) {
        log.error("Unhandled exception", ex);              // log stack trace server-side
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "Something went wrong");  // never leak internals
    }
}
```

**Senior points:**
- Extend `ResponseEntityExceptionHandler` to reuse Spring's handling of the 15+ built-in MVC exceptions.
- Ordering: most specific exception wins; use `@Order` on multiple advices; scope with `@RestControllerAdvice(basePackages = "...")`.
- **Never** return stack traces or SQL messages to clients — map to a stable error code and correlate via `traceId`.
- Exceptions thrown in **filters** bypass `@ControllerAdvice` (they're outside DispatcherServlet) — handle those in the filter or via `ErrorController` / Security's `AuthenticationEntryPoint`.

---

### Q20. Validation — how deep can you go?

```java
@PostMapping
ResponseEntity<Void> create(@Valid @RequestBody CreateOrderRequest req) { ... }

record CreateOrderRequest(
    @NotBlank @Size(max = 64) String customerId,
    @NotEmpty @Valid List<LineItem> items,          // @Valid = cascade into nested objects
    @Positive BigDecimal total,
    @Email String notifyEmail) {}
```

- `@Valid` (Bean Validation) vs `@Validated` (Spring) — `@Validated` supports **validation groups** and enables **method-level validation** on `@Service` beans (needs `@Validated` on the class).
- Validating path/query params: `@Validated` on the controller class + `@Min(1) @PathVariable long id` → throws `ConstraintViolationException`, which you must map separately (it's a 500 by default!).
- **Custom validator:**

```java
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target(FIELD) @Retention(RUNTIME)
public @interface UniqueEmail { String message() default "email already registered"; ... }
```

- **Cross-field validation:** class-level constraint (e.g. `@EndAfterStart` on the DTO).
- Group example: `@NotNull(groups = Update.class) Long id;` then `@Validated(Update.class)`.

---

### Q21. API versioning strategies.

| Strategy | Example | Trade-off |
|---|---|---|
| URI path | `/api/v1/orders` | Simplest, most visible, cache-friendly — most common in practice |
| Query param | `/orders?version=1` | Easy but pollutes params |
| Custom header | `X-API-Version: 1` | Clean URLs, harder to test/cache |
| Content negotiation | `Accept: application/vnd.acme.v1+json` | Most RESTful, least developer-friendly |

**What to actually say:** "We used URI versioning at the gateway for major versions, and kept minor changes backward-compatible — additive fields only, tolerant reader on the consumer side. We only bump the version for breaking changes, and we run N and N-1 in parallel with a documented sunset date (`Deprecation` and `Sunset` headers)."

**Why versioning exists:** once clients (mobile apps you can't force-update, partner systems) depend on your JSON shape, renaming or removing a field breaks them. A version lets old and new contracts live side-by-side. *Breaking* = removing/renaming a field, changing a type or meaning, making an optional field required. *Non-breaking* = adding an optional field or a new endpoint. A **tolerant reader** is a consumer that ignores unknown fields (in Jackson: `spring.jackson.deserialization.fail-on-unknown-properties=false`, which is Boot's default).

```java
@RestController @RequestMapping("/api/v1/orders")     // v1 contract frozen
class OrderControllerV1 { ... }

@RestController @RequestMapping("/api/v2/orders")     // v2 adds breaking change
class OrderControllerV2 { ... }
```

Both controllers can delegate to the same service; only the DTO mapping differs. Spring Framework 7 / Boot 4 adds first-class API versioning support, but until then versioning is done manually as above.

---

### Q22. Idempotency in REST APIs.

- GET/PUT/DELETE are idempotent by spec; **POST is not**.
- For payments/orders: require an **`Idempotency-Key` header**. Store `key → (status, response)` in Redis/DB with a TTL, inside the same transaction as the business write (or via unique constraint). On repeat, return the stored response instead of re-executing.
- Use a **unique constraint** as the ultimate guard (e.g. unique on `(client_ref)`), catch `DataIntegrityViolationException` and treat as success.

**What idempotent means:** calling the operation once or ten times leaves the system in the same state. It matters because networks fail: a client times out after sending "charge ₹500", doesn't know if it succeeded, and retries — without protection the customer is charged twice. GET/PUT/DELETE are naturally safe to repeat (PUT replaces with the same data; DELETE of something already gone is still "gone"); POST creates a new thing each time, so you must add idempotency yourself.

**How the `Idempotency-Key` flow works:**
1. Client generates a UUID per *logical* operation and sends `Idempotency-Key: 7c9e...` on the request (and reuses the **same** key on retries).
2. Server tries to insert the key (unique column) in the same transaction as the business write.
3. Insert succeeds → first time: process, save the response against the key.
4. Insert fails with a duplicate → replay: return the saved response (same status and body) without re-running the logic.

```java
try {
    idempotencyRepo.save(new IdempotencyRecord(key, requestHash));   // unique(key)
    Order o = orderService.place(req);
    idempotencyRepo.complete(key, o);
    return ResponseEntity.status(201).body(o);
} catch (DataIntegrityViolationException dup) {
    return idempotencyRepo.findResponse(key);                         // replay saved result
}
```

**Pitfalls:** store a hash of the request body and reject the same key with a *different* payload (422); handle the "first request still in progress" case (return 409 or wait); expire keys with a TTL (e.g. 24h).

---

### Q23. How do you handle large responses / pagination?

- `Pageable` + `Page<T>`: `GET /orders?page=0&size=20&sort=createdAt,desc`.
- `Page` runs a **second COUNT query** — expensive on big tables. Use `Slice<T>` (only "hasNext") or **keyset/cursor pagination** (`WHERE (created_at, id) < (:lastTs, :lastId) ORDER BY created_at DESC, id DESC LIMIT 20`) for deep pagination and infinite scroll — O(1) instead of O(offset).
- Cap page size server-side (`spring.data.web.pageable.max-page-size=100`) or an attacker requests `size=1000000`.
- For very large exports: streaming (`StreamingResponseBody`, `ResponseBodyEmitter`) or async job + presigned S3 URL. Don't build a 500 MB list in heap.

**Why offset pagination gets slow:** `LIMIT 20 OFFSET 100000` makes the database walk and throw away 100,000 rows before returning 20 — cost grows with the page number, and concurrent inserts can cause skipped/duplicate rows between pages. **Keyset (cursor) pagination** remembers the last seen sort key and asks "give me rows after this one", so the index jumps straight to the right place regardless of depth. The trade-off: you can't jump to "page 57", only next/previous — fine for feeds and infinite scroll, not for numbered pagers.

```java
// client sends the cursor returned by the previous page
List<Order> next(Instant lastTs, long lastId) {
    return repo.findNextPage(lastTs, lastId, PageRequest.of(0, 20));
}
// response: { "items": [...], "nextCursor": "2024-05-01T10:00:00Z_9812" }
```

Keyset needs a **deterministic, unique sort** (hence `created_at` + `id` as tie-breaker) and a matching composite index on those columns.

---

### Q24. `RestTemplate` vs `WebClient` vs `RestClient` vs Feign.

| Client | Notes |
|---|---|
| `RestTemplate` | Blocking, **in maintenance mode** — no new features, still supported |
| `WebClient` | Reactive, non-blocking; works in MVC too (call `.block()` sparingly) |
| `RestClient` (Boot 3.2+) | **Synchronous, fluent** — the modern replacement for RestTemplate |
| `@HttpExchange` interface clients | Declarative, framework-native (`HttpServiceProxyFactory`) |
| OpenFeign | Declarative, Spring Cloud; feature-rich but extra dependency |

Always configure: connect + **read timeouts** (defaults are infinite → thread exhaustion), connection pooling, retries with jitter, and a circuit breaker.

**Why this matters:** every outbound HTTP call blocks a thread while it waits. If a downstream service hangs and you have no timeout, threads pile up until Tomcat's pool (default 200) is exhausted and *your* service stops answering — a cascading failure. *Connect timeout* = how long to wait to establish the TCP connection; *read timeout* = how long to wait for response bytes; *connection-request timeout* = how long to wait for a free connection from the pool. *Jitter* is random delay added to retry back-offs so many clients don't retry in lock-step and hammer a recovering service. A *circuit breaker* (Resilience4j) stops calling a failing service for a while and fails fast.

**Choosing:** new blocking code on Boot 3.2+ → `RestClient`; reactive stack → `WebClient`; many internal service-to-service calls with a clean interface → `@HttpExchange`/Feign. A typical declarative client:

```java
@HttpExchange("/payments")
interface PaymentApi {
    @PostExchange Receipt charge(@RequestBody ChargeRequest req);
}
```

```java
@Bean
RestClient paymentClient(RestClient.Builder builder) {
    var factory = new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(2_000);
    factory.setConnectionRequestTimeout(1_000);
    return builder.baseUrl("https://payments.internal")
                  .requestFactory(factory)
                  .defaultHeader("X-Client", "order-service")
                  .build();
}
```

---

### Q25. CORS — how do you configure it properly?

**What CORS is and why it exists:** browsers enforce the *same-origin policy* — JavaScript on `https://app.acme.com` may not read responses from a different origin (scheme + host + port) like `https://api.acme.com`, so a malicious site can't silently call your bank API using the user's cookies. **CORS (Cross-Origin Resource Sharing)** is the opt-in: the server sends `Access-Control-Allow-Origin` etc. headers telling the browser which origins are trusted. It is enforced **by the browser only** — curl/Postman/server-to-server calls ignore it, so it is not an authentication mechanism.

**How it works:** for "non-simple" requests (JSON body, `Authorization` header, PUT/DELETE) the browser first sends a **preflight** `OPTIONS` request asking "may I send this?". The server replies with allowed origins/methods/headers; if OK, the browser sends the real request. `Max-Age` lets the browser cache that answer so it isn't repeated every call.

```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    var cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("https://app.acme.com"));   // never "*" with credentials
    cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE"));
    cfg.setAllowedHeaders(List.of("Authorization","Content-Type","Idempotency-Key"));
    cfg.setAllowCredentials(true);
    cfg.setMaxAge(3600L);                                      // cache preflight
    var src = new UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/api/**", cfg);
    return src;
}
```

Gotcha: with Spring Security you must call `http.cors(Customizer.withDefaults())`, otherwise the security filter chain rejects the preflight `OPTIONS` before MVC's CORS handling runs. `allowCredentials(true)` + `allowedOrigins("*")` is illegal — use `allowedOriginPatterns`.

---

## 5. Spring Data JPA & Hibernate

### Q26. How does Spring Data JPA create repository implementations at runtime?

`@EnableJpaRepositories` (auto-configured) scans for `Repository` interfaces, and `RepositoryFactoryBean` creates a **JDK dynamic proxy** per interface. Each call is routed by a chain of `QueryLookupStrategy`s:
1. **Declared query** — `@Query` annotation or named query (`User.findByEmail` in `orm.xml`).
2. **Derived query** — parse the method name (`findByStatusAndCreatedAtAfter`) into a criteria query.
3. **Base implementation** — `SimpleJpaRepository` for `save`, `findById`, `delete`, etc.

Derived-query keywords worth knowing: `findFirst10By`, `findDistinctBy`, `countBy`, `existsBy`, `deleteBy`, `...OrderByCreatedAtDesc`, `IgnoreCase`, `Between`, `In`, `Containing`, `StartingWith`, `IsNull`, `Not`.

**Senior note:** long derived names (`findByAAndBAndCOrderByD`) are a smell — switch to `@Query`, Specifications, or QueryDSL for anything complex.

**In plain language:** you only write an *interface*; you never write `implements`. At startup Spring Data scans for interfaces extending `Repository`/`JpaRepository`, and for each one generates a **proxy object** (a stand-in object created at runtime that intercepts method calls) and registers it as a bean — that is what gets injected. When you call a method, the proxy decides how to answer it:
- It's a CRUD method from the parent interface (`save`, `findById`) → delegated to `SimpleJpaRepository`, a ready-made class that uses `EntityManager`.
- It has `@Query` → run that JPQL/SQL.
- Otherwise → the **method name is parsed** at startup into a query: `findByStatusAndCreatedAtAfter(status, date)` → `WHERE status = ? AND created_at > ?`. A typo in a property name (`findByStatuss`) fails the app at **startup**, not at first call — a nice fail-fast property.

```java
interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmailIgnoreCase(String email);          // derived
    @Query("select u from User u where u.lastLogin < :cutoff")   // explicit JPQL
    List<User> findInactive(Instant cutoff);
}
```

**Why proxies:** the same mechanism lets Spring add cross-cutting behaviour for free — every repository method is automatically transactional (reads are `readOnly`), and persistence exceptions are translated to `DataAccessException`.

---

### Q27. `save()` vs `saveAndFlush()`; how does `save()` decide insert vs update?

`SimpleJpaRepository.save()`:
```java
if (entityInformation.isNew(entity)) em.persist(entity);   // INSERT
else return em.merge(entity);                              // SELECT + UPDATE
```
`isNew()` is true when the `@Id` is null (or version is null with `@Version`), unless the entity implements `Persistable` where you override `isNew()`.

**Consequences to mention:**
- **Assigned IDs (e.g. UUID / natural keys)** make `isNew()` false → `merge()` fires an extra SELECT for every save. Fix: implement `Persistable<UUID>` with a transient `isNew` flag, or use `@Version`.
- Inside a transaction, if the entity is **already managed**, you don't need `save()` at all — dirty checking flushes changes automatically at commit. Many devs call `save()` redundantly.
- `saveAndFlush()` forces the SQL immediately (useful before a native query that must see the row, or to surface constraint violations early), but breaks batching.

**Terms explained:** *persist* = "this is a brand-new entity, schedule an INSERT". *merge* = "I'm handing you a possibly-detached object; load the current row, copy my state onto it, and update" (hence the extra SELECT). *Flush* = send the pending SQL to the DB now; *commit* = end the transaction. Normally Hibernate delays SQL until commit (write-behind) so it can batch statements; `saveAndFlush` defeats that.

```java
@Transactional
void rename(long id, String name) {
    User u = repo.findById(id).orElseThrow();   // managed entity
    u.setName(name);                            // no save() needed — dirty checking issues UPDATE at commit
}
```

```java
@Entity class Order implements Persistable<UUID> {
    @Id UUID id = UUID.randomUUID();
    @Transient boolean isNew = true;
    public boolean isNew() { return isNew; }
    @PostPersist @PostLoad void markNotNew() { isNew = false; }
    public UUID getId() { return id; }
}
```

---

### Q28. What is the N+1 select problem and how do you fix it? (Almost guaranteed question)

**Problem:** you fetch 100 orders (1 query), then touch `order.getItems()` on each → 100 more queries. Latency explodes linearly.

**Fixes, best to worst:**

| Fix | Use when |
|---|---|
| `JOIN FETCH` in JPQL | You know the access pattern for that use case |
| `@EntityGraph` on the repository method | Same, declaratively, and reusable |
| `@BatchSize(size = 25)` / `hibernate.default_batch_fetch_size` | Fetches children in `IN (...)` batches — turns N+1 into N/25+1. **Best global safety net** |
| Projections (DTO / interface) | Read-only screens — fetch only needed columns |
| Two queries + manual stitching | Multiple collections (avoids cartesian product) |
| `FetchType.EAGER` on the mapping | ❌ Never — global cost, can't be turned off per query |

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = {"items", "customer"})
    List<Order> findByStatus(OrderStatus status);

    @Query("""
           select distinct o from Order o
           join fetch o.items i
           where o.createdAt > :since
           """)
    List<Order> findRecentWithItems(Instant since);
}
```

**Critical gotcha:** `JOIN FETCH` + `Pageable` → Hibernate warns `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory` and pages **in memory** after loading everything. Fix: paginate IDs first, then fetch by `id IN (:ids)`; or use `@BatchSize`.

**Detection:** enable `spring.jpa.properties.hibernate.generate_statistics=true`, use **Hypersistence Utils / datasource-proxy** to assert query counts in tests, or `p6spy`. Say: "We added a test that fails the build if a critical endpoint exceeds N queries."

---

### Q29. LazyInitializationException — root cause and the *right* fix.

Cause: touching a lazy association after the `EntityManager`/session closed (i.e. after the transaction ended — typically in the controller or during JSON serialization).

**Wrong fixes:** `spring.jpa.open-in-view=true` (it's ON by default and Spring even warns about it — it holds the connection for the whole request, hides N+1 into the view layer, and exhausts the pool under load), or switching everything to EAGER.

**Right fixes:**
1. Fetch what you need inside the transaction: `JOIN FETCH` / `@EntityGraph`.
2. Map to a **DTO inside the service layer** while the session is open — never return entities from controllers.
3. Set `spring.jpa.open-in-view=false` **explicitly** so the failure appears in dev/test, not production.

**Strong senior statement:** "We disable open-in-view on every service and return DTOs; the transactional boundary is the service, and the web layer never touches a managed entity."

**What actually happens:** a `@ManyToOne`/`@OneToMany` marked LAZY isn't loaded with the parent; Hibernate puts a **proxy placeholder** in the field and runs the SELECT the first time you touch it. That SELECT needs an open Hibernate session (bound to the transaction). Once the `@Transactional` service method returns, the session is closed, so touching the placeholder later throws.

```java
@Transactional(readOnly = true)
public OrderDto get(long id) {
    Order o = repo.findById(id).orElseThrow();
    return new OrderDto(o.getNumber(), o.getItems().size());   // lazy collection touched while session open ✔
}
// Controller returning the Order entity directly → Jackson calls getItems() after tx ended → LazyInitializationException
```

**Why open-in-view (OSIV) is tempting but harmful:** it keeps the session (and so a DB connection) open until the HTTP response is written — including template rendering, JSON serialization and slow network writes. Lazy loading "just works", but each request holds a pooled connection far longer, and N+1 queries hide inside serialization. Boot logs a warning at startup for this reason.

---

### Q30. Entity mapping best practices (things interviewers probe).

- **Always** `@ManyToOne(fetch = LAZY)` — `@ManyToOne`/`@OneToOne` are EAGER by default, which is the sneakiest source of N+1.
- Bidirectional relations: owning side has the FK; the inverse side uses `mappedBy`. Add helper methods (`addItem`) to keep both sides in sync.
- `@OneToMany(cascade = ALL, orphanRemoval = true)` only for true parent–child aggregates. `CascadeType.REMOVE` on a `@ManyToMany` is dangerous.
- `equals`/`hashCode`: use the **business/natural key** or a UUID assigned at construction — never the generated `@Id` (null before persist → breaks `HashSet` semantics).
- ID generation: `IDENTITY` disables JDBC batch inserts (Hibernate must hit the DB per row). Use `SEQUENCE` with a **pooled optimizer** (`allocationSize = 50`) for bulk-insert throughput.
- `@Enumerated(EnumType.STRING)` — never ORDINAL (reordering the enum silently corrupts data).
- Use `Instant`/`OffsetDateTime`, store UTC.
- Prefer `Set` over `List` for `@ManyToMany` (List causes delete-all + re-insert on modification).
- `@DynamicUpdate` when tables are wide and updates are narrow.

**Why each rule exists:**
- *EAGER defaults:* `@ManyToOne` loads the parent row immediately by default. Load 100 `OrderItem`s and Hibernate also fires queries for their `Order`s — that is N+1 you never wrote. Override with `fetch = LAZY` and fetch explicitly when needed.
- *`mappedBy`:* in a bidirectional relationship the database has only one foreign key column; the side that owns it (the "many" side with `@JoinColumn`) writes it. The other side says `mappedBy = "order"` meaning "I'm just a mirror". If you only add an item to the parent's list but never set `item.setOrder(parent)`, the FK is saved as NULL — hence the helper method:

```java
@OneToMany(mappedBy = "order", cascade = ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
public void addItem(OrderItem i) { items.add(i); i.setOrder(this); }
```

- *`orphanRemoval`:* removing an item from the parent's list deletes the row. *Cascade* = "apply persist/remove from parent to children automatically" — only sensible when the child cannot exist without the parent (order → order lines), not for shared entities (a `Category` used by many products).
- *`equals/hashCode` on generated ID:* the ID is `null` until saved, so the hash changes after `persist` and the object gets "lost" inside a `HashSet`. Use a UUID/natural key set at construction.
- *IDENTITY vs SEQUENCE:* with IDENTITY the DB generates the ID during the INSERT, so Hibernate must execute each insert immediately to learn the ID and can't batch. A SEQUENCE with `allocationSize=50` lets Hibernate reserve 50 IDs in one call and batch inserts.
- *ORDINAL enums:* stored as 0,1,2; inserting a new constant in the middle makes old rows silently mean something else. STRING stores the name.
- *`List` in `@ManyToMany`:* Hibernate can't identify individual rows in a bag, so any change deletes all link rows and re-inserts them; a `Set` allows targeted deletes.
- *`@DynamicUpdate`:* by default the UPDATE lists every column; with it Hibernate only includes changed ones (small cost to compute the diff, benefit on wide tables or concurrent writers of different columns).

---

### Q31. First-level vs second-level cache.

- **L1 (persistence context)**: mandatory, per-`EntityManager`/transaction, guarantees identity (`==` for the same row in one tx). Not shared, not configurable.
- **L2 (SessionFactory-wide)**: opt-in, needs a provider (Ehcache, Infinispan, Hazelcast, Redis via Redisson). Enable with `hibernate.cache.use_second_level_cache=true` + `@Cache(usage = READ_WRITE)` on the entity.
- **Query cache** is separate (`use_query_cache`) and often a net negative — it's invalidated by any write to the touched tables.

**Senior take:** L2 fits small, read-mostly reference data (countries, tariff plans). For anything else, an explicit application-level cache (`@Cacheable` on a service returning DTOs) is easier to reason about and to invalidate — especially in a multi-instance deployment where L2 needs distributed invalidation.

**Explained simply:** the **persistence context** (L1) is a per-transaction map of "entity type + id → object". Ask for `User#5` twice in the same transaction and the second `find` returns the same Java object without a query; that same map is how dirty checking knows what changed. It's gone when the transaction/EntityManager closes. **L2** survives across transactions and is shared by all sessions of one app instance, so `findById` can be answered from memory without touching the DB — but it can serve **stale** data if another instance or a native SQL update changes the row, which is why it needs a provider with invalidation.

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)   // Hibernate annotation, cache this entity in L2
class Country { @Id String code; String name; }
```

`READ_ONLY` = fastest, data never changes; `READ_WRITE` = soft locks to keep cache consistent with updates; `NONSTRICT_READ_WRITE` = tolerates brief staleness. **Probe:** *"Two pods, one updates a row — what does the other see?"* Only a distributed/invalidating L2 (or a short TTL) gives the right answer.

---

### Q32. Optimistic vs pessimistic locking.

```java
@Entity class Account {
    @Id Long id;
    @Version Long version;      // optimistic: UPDATE ... WHERE id=? AND version=?
    BigDecimal balance;
}
```
- **Optimistic** (`@Version`): no DB locks, high concurrency; conflict → `OptimisticLockingFailureException`/`ObjectOptimisticLockingFailureException` at flush. Handle by **retrying** the whole transaction (e.g. Spring Retry) or returning 409 to the client. Default choice for web apps.
- **Pessimistic** (`@Lock(LockModeType.PESSIMISTIC_WRITE)` → `SELECT ... FOR UPDATE`): serializes writers, risks lock waits/deadlocks; use for short critical sections like decrementing inventory. Always set `jakarta.persistence.lock.timeout`.

**Alternative that avoids locking entirely:** an atomic DB statement —
`UPDATE inventory SET qty = qty - :n WHERE sku = :sku AND qty >= :n` and check the affected-row count. Mentioning this shows real production experience.

**The problem being solved — "lost update":** two users load the same account (balance 100). A withdraws 30 and saves 70; B deposits 50 and saves 150 — A's change vanishes because B overwrote it from stale data. Locking prevents that.

**How optimistic locking works:** nobody is blocked. Each row has a `version`. Hibernate runs `UPDATE account SET balance=?, version=2 WHERE id=? AND version=1`. If someone else already bumped it to 2, **0 rows are updated**, Hibernate detects that and throws the exception — you then reload and retry. It's "optimistic" because it assumes conflicts are rare and only checks at the end. **Pessimistic** assumes conflicts are likely, so it locks the row on read (`SELECT ... FOR UPDATE`) and other writers wait until you commit.

```java
@Retryable(retryFor = ObjectOptimisticLockingFailureException.class, maxAttempts = 3)
@Transactional
public void withdraw(long id, BigDecimal amt) { /* load, check, change — whole tx is retried */ }
```

**Pitfall:** the retry must wrap the *entire transaction* (the `@Retryable` proxy must be outside `@Transactional`), otherwise you retry inside an already-rollback-only transaction.

---

### Q33. JPA batch inserts — why is my bulk insert slow?

```yaml
spring.jpa.properties:
  hibernate.jdbc.batch_size: 50
  hibernate.order_inserts: true
  hibernate.order_updates: true
  hibernate.batch_versioned_data: true
spring.datasource.hikari.data-source-properties:
  reWriteBatchedInserts: true      # PostgreSQL: rewrites into multi-row INSERT
```
Plus: **don't use `GenerationType.IDENTITY`** (kills batching), and `flush()` + `clear()` every batch to stop the persistence context growing (which makes dirty checking O(n²) and eventually OOMs).

For genuinely large loads: `JdbcTemplate.batchUpdate`, `COPY`/`LOAD DATA`, or Spring Batch. "We moved a 2-million-row nightly load from JPA to JdbcTemplate batching and went from 40 minutes to 90 seconds."

**Why it's slow without these settings:** by default JPA sends one `INSERT` per row, each a network round-trip (1M rows = 1M round-trips). *JDBC batching* groups many statements into one network call (`batch_size: 50` → 50 rows per trip). `order_inserts/updates` sorts statements by table so that consecutive ones can actually be grouped (otherwise parent/child inserts interleave and break batches). The persistence context also keeps every saved entity in memory and checks each for changes at flush — so after each batch you `flush()` (send SQL) and `clear()` (forget the objects).

```java
@Transactional
public void importAll(List<Customer> rows) {
    for (int i = 0; i < rows.size(); i++) {
        em.persist(rows.get(i));
        if (i % 50 == 49) { em.flush(); em.clear(); }   // match hibernate.jdbc.batch_size
    }
}
```

Verify batching really happens by enabling datasource-proxy/`hibernate.generate_statistics` and checking the batch count — a silently disabled batch (IDENTITY id, or entity with `@Version`+`batch_versioned_data` off) is the usual culprit.

---

### Q34. Projections and DTOs.

```java
// 1. Interface projection (closed) — Spring Data generates the proxy, selects only these columns
interface OrderSummary { String getOrderNumber(); BigDecimal getTotal(); }

// 2. Class/record projection via constructor expression
@Query("select new com.acme.OrderDto(o.orderNumber, o.total) from Order o where o.status = :s")
List<OrderDto> findSummaries(OrderStatus s);

// 3. Dynamic projection
<T> List<T> findByStatus(OrderStatus s, Class<T> type);
```
Benefits: fewer columns, no persistence-context overhead, no dirty checking, no lazy-loading surprises. **Read endpoints should almost always use projections.**

**What a projection is:** instead of loading the whole entity (all columns + managed state), you ask for just the fields the screen needs — e.g. a list page needing order number and total shouldn't load 30 columns plus lazy relations. The generated SQL becomes `select order_number, total from orders ...`. *Interface projections* are proxies that read from the result tuple; *record/class projections* use a constructor; *dynamic projections* let the caller choose the shape (`findByStatus(s, OrderSummary.class)`). **Pitfalls:** an *open* interface projection (using `@Value("#{...}")` SpEL) loads the full entity anyway; DTO constructor expressions can't be used with nested collections — fetch those separately.

---

### Q35. Flyway vs Liquibase; how do you handle schema changes safely?

- Both give versioned, repeatable, auditable migrations. Flyway = SQL-first, simpler; Liquibase = XML/YAML changelogs, DB-agnostic, better rollback support.
- `spring.jpa.hibernate.ddl-auto` must be **`validate` or `none` in production** — never `update`/`create-drop`. `update` never drops or alters safely and silently diverges across environments.
- **Zero-downtime pattern (expand/contract):**
  1. Add nullable column + backfill (batched) — old code unaffected.
  2. Deploy code that writes both old and new.
  3. Migrate readers to the new column.
  4. Drop the old column in a later release.
- Never combine a long-running `ALTER` with a deployment window; test migrations against a production-sized copy; make them idempotent and forward-only.

**How migration tools work:** you add numbered scripts (`V1__init.sql`, `V2__add_phone.sql`) to `src/main/resources/db/migration`. On startup Flyway compares them with a history table (`flyway_schema_history`) in the DB, runs only the not-yet-applied ones in order, and stores a **checksum** of each — editing an already-applied script makes startup fail, which protects you from environments drifting apart. Every environment (dev → prod) therefore reaches the identical schema by the identical path, in Git, reviewable.

```sql
-- V7__add_customer_phone.sql  (step 1 of expand/contract: additive & nullable)
ALTER TABLE customer ADD COLUMN phone VARCHAR(20);
```

**Why expand/contract:** during a rolling deployment old and new app versions run **at the same time** against one database. Renaming a column in one step breaks whichever version isn't updated yet. By only *adding* first, and *removing* in a later release once no running code uses it, both versions always work. **Pitfalls:** `ALTER TABLE ... ADD COLUMN ... DEFAULT` or index creation can lock large tables (use `CREATE INDEX CONCURRENTLY` on PostgreSQL); a failed multi-statement migration on MySQL isn't rolled back (DDL auto-commits).

---

## 6. Transaction Management

### Q36. How does `@Transactional` work under the hood?

Spring creates an **AOP proxy** around the bean. On invocation, `TransactionInterceptor` asks the `PlatformTransactionManager` (JPA → `JpaTransactionManager`) to begin/join a transaction, binds the `EntityManager`/`Connection` to the current thread via `TransactionSynchronizationManager` (a `ThreadLocal`), runs the method, then commits or rolls back.

**Therefore three famous limitations:**
1. **Self-invocation doesn't work** — calling `this.otherTransactionalMethod()` bypasses the proxy, so no new transaction. Fix: move it to another bean, self-inject via `@Lazy`/`ObjectProvider`, use `TransactionTemplate`, or `AopContext.currentProxy()` (needs `exposeProxy`).
2. **Only public methods** are advised with the default JDK/CGLIB proxying (private/protected/final silently do nothing). AspectJ weaving lifts this.
3. Thread-bound: a `@Transactional` context does **not** propagate to `@Async` threads or child threads.

**In plain language:** a *transaction* groups several DB operations so they all succeed or none do (e.g. debit one account and credit another). Rather than make you write `begin/commit/rollback` try/catch code everywhere, Spring wraps your bean in a **proxy** — a generated subclass/wrapper object that other beans actually receive. Calls go *through the proxy*, which starts the transaction, calls your real method, and commits if it returned normally or rolls back if it threw.

```java
// What the proxy effectively does around your method:
tm.begin();
try { target.placeOrder(req); tm.commit(); }
catch (RuntimeException e) { tm.rollback(); throw e; }
```

The connection is stored in a `ThreadLocal` (a variable with one separate value per thread) so every repository call in that thread automatically uses the same connection/transaction — that's why a new thread doesn't see it. **Rollback rule to remember:** by default only **unchecked** exceptions (`RuntimeException`, `Error`) trigger rollback; a checked exception commits unless you set `rollbackFor = Exception.class`. **Self-invocation example:**

```java
@Service class OrderService {
    public void process() { this.audit(); }          // 'this' = raw object, not proxy → REQUIRES_NEW ignored
    @Transactional(propagation = REQUIRES_NEW)
    public void audit() { ... }
}
```

---

### Q37. Explain propagation levels — with when-to-use.

| Propagation | Behaviour | Use case |
|---|---|---|
| `REQUIRED` (default) | Join existing, else create | 95% of cases |
| `REQUIRES_NEW` | Suspend outer, always start new | **Audit/logging that must survive an outer rollback** |
| `NESTED` | Savepoint inside current tx | Partial rollback (JDBC only, not JTA) |
| `SUPPORTS` | Join if exists, else non-transactional | Read helpers |
| `NOT_SUPPORTED` | Suspend any tx | Long-running report that shouldn't hold a connection |
| `MANDATORY` | Must already be in a tx, else throw | Internal method that must never be called standalone |
| `NEVER` | Throw if a tx exists | Guard code |

**Critical gotcha with `REQUIRES_NEW`:** it uses a **second DB connection** while holding the first → with a pool of 10 and 10 concurrent such calls you get a **deadlock on the pool**. Also, if the inner tx commits and the outer rolls back, you have committed data referencing rolled-back rows.

**Interview-winning example:** "For failure auditing we initially used `REQUIRES_NEW`, then moved to publishing an event and writing the audit record after commit — or better, the transactional outbox — because `REQUIRES_NEW` doubled our pool usage."

**What "propagation" means:** it answers "what should happen when a `@Transactional` method is called by code that's *already* inside a transaction?" Join it, start a separate one, or refuse. Example scenario for the two most-asked:

```java
@Transactional                                   // REQUIRED: outer transaction T1
public void placeOrder(Order o) {
    orderRepo.save(o);
    auditService.record("ORDER_ATTEMPT", o);     // see below
    paymentService.charge(o);                    // throws → T1 rolls back
}

@Service class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)   // own tx T2, own connection
    public void record(String what, Order o) { auditRepo.save(new Audit(what, o.getId())); }
}
```

With `REQUIRED` on `record`, the audit row would join T1 and vanish when `charge` fails. With `REQUIRES_NEW`, T1 is *suspended*, T2 commits independently, then T1 resumes — the audit row survives the rollback. (Remember: this only works if `record` lives in a **different bean**, otherwise the self-invocation limitation applies.) `NESTED` instead creates a **savepoint** (a bookmark inside the same transaction) so only the inner part can be rolled back while the outer continues.

---

### Q38. Isolation levels and the anomalies they prevent.

| Isolation | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| READ_UNCOMMITTED | ✅ possible | ✅ | ✅ |
| READ_COMMITTED | ❌ | ✅ | ✅ |
| REPEATABLE_READ | ❌ | ❌ | ✅ (MySQL InnoDB prevents via gap locks) |
| SERIALIZABLE | ❌ | ❌ | ❌ |

Defaults: PostgreSQL & Oracle → READ_COMMITTED; MySQL InnoDB → REPEATABLE_READ.
**Memory trick:** *"Dirty → Non-repeatable → Phantom"* is the order of severity, and each higher isolation kills one more.

Note: `@Transactional(isolation = ...)` is ignored by some DBs/managers; raising isolation costs concurrency. Prefer explicit locking or atomic SQL over SERIALIZABLE.

**What the anomalies mean (with a concrete story):** isolation defines how much one running transaction can see of another's work.
- **Dirty read:** T1 reads a row that T2 changed but *hasn't committed*. If T2 rolls back, T1 acted on data that never existed (e.g. saw a balance of ₹1000 that was reverted).
- **Non-repeatable read:** T1 reads a row, T2 *updates and commits* it, T1 reads the same row again and gets a different value within one transaction.
- **Phantom read:** T1 runs `SELECT ... WHERE amount > 100` and gets 5 rows; T2 *inserts and commits* a matching row; T1 re-runs the query and gets 6 — a new "phantom" row appeared.

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Report monthEnd() {
    var total1 = repo.sumPaid();      // same snapshot ...
    var total2 = repo.sumPaid();      // ... so this equals total1 even if other txs commit in between
    return new Report(total1, total2);
}
```

**Why not always SERIALIZABLE:** it makes transactions behave as if they ran one after another, so the DB uses more locks or aborts conflicting transactions (serialization failures you must retry), reducing throughput. READ_COMMITTED + `@Version`/atomic updates covers most real needs. PostgreSQL implements REPEATABLE_READ via snapshots (MVCC), so its anomalies differ slightly from the textbook table.

---

### Q39. When does `@Transactional` roll back?

By default: **unchecked** exceptions (`RuntimeException`, `Error`) roll back; **checked** exceptions **commit**. This surprises everyone.

```java
@Transactional(rollbackFor = Exception.class,
               noRollbackFor = NotificationFailedException.class)
```

Other traps:
- Catching the exception inside the method → no rollback (Spring never sees it). If you must catch, call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.
- Once a transaction is marked rollback-only, a nested `REQUIRED` "success" still fails the outer commit with `UnexpectedRollbackException`.
- `readOnly = true` on read methods: sets the Hibernate flush mode to MANUAL (skips dirty checking → faster) and hints the driver/replica. Big cheap win on read paths.
- `@Transactional` on a private method or on a class not managed by Spring = silently no-op.

**Why the checked/unchecked rule:** it follows the old EJB convention — unchecked exceptions signal programming/system errors (unrecoverable → roll back), checked exceptions signal expected business conditions the caller may handle (so state is kept). Most teams find this surprising, so they set `rollbackFor = Exception.class` or use only unchecked business exceptions.

```java
@Transactional
public void transfer(long from, long to, BigDecimal amt) throws InsufficientFundsException { // checked
    accounts.debit(from, amt);       // already written
    accounts.credit(to, amt);        // throws InsufficientFundsException → COMMITS the debit! (money lost)
}
```

*Swallowed-exception trap:* `try { service2.work(); } catch (Exception e) { log.warn(...); }` — if `service2.work()` is `@Transactional(REQUIRED)` and threw a RuntimeException, the shared transaction was already marked rollback-only; your `catch` hides the error but the outer `commit` then fails with `UnexpectedRollbackException`. Use `REQUIRES_NEW` for the part that is allowed to fail independently.

---

### Q40. How do you coordinate transactions with external side effects (email, Kafka, REST)?

Never call an external system inside a transaction and assume consistency. Two patterns:

**1. `TransactionalEventListener` (simple, at-most-once):**
```java
@Transactional
public void placeOrder(Order o) {
    repo.save(o);
    events.publishEvent(new OrderPlaced(o.getId()));   // published, not yet handled
}

@TransactionalEventListener(phase = AFTER_COMMIT)
void onPlaced(OrderPlaced e) { kafka.send("orders", e); }  // runs only if commit succeeded
```
Risk: if the app crashes between commit and send, the message is lost.

**2. Transactional Outbox (at-least-once, the production answer):**
Write the event to an `outbox` table **in the same transaction** as the business data; a poller or CDC (Debezium) publishes it and marks it sent. Consumers must be idempotent. This is the standard answer for "how do you keep DB and Kafka consistent without 2PC?"

---

## 7. Spring AOP

### Q41. AOP basics + terminology.

- **Aspect** — the module of cross-cutting concern; **Join point** — a point of execution (in Spring AOP: always a method execution); **Advice** — the action (`@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`); **Pointcut** — the predicate selecting join points; **Weaving** — linking, done at **runtime via proxies** in Spring.
- **JDK dynamic proxy** if the bean implements an interface; **CGLIB subclass proxy** otherwise. Spring Boot forces CGLIB by default (`spring.aop.proxy-target-class=true`). CGLIB cannot proxy `final` classes/methods.

```java
@Aspect @Component
public class TimingAspect {
    @Around("@annotation(com.acme.Timed) || within(com.acme..service..*)")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long t = System.nanoTime();
        try { return pjp.proceed(); }
        finally { metrics.record(pjp.getSignature().toShortString(), System.nanoTime() - t); }
    }
}
```

**Real uses:** method timing/metrics, audit logging, retry, feature flags, multi-tenancy resolution, `@PreAuthorize`. **Anti-use:** business logic hidden in aspects — impossible to trace.

**Order:** multiple aspects on one method → control with `@Order` (lower = outer).

**What AOP is for, plainly:** some logic (logging, timing, security checks, transactions) is needed in *hundreds* of methods but isn't business logic. Copy-pasting it is error-prone; AOP lets you write it once and declare *where* it applies. Spring does this by handing callers a **proxy** that runs your *advice* before/after delegating to the real bean — the target class is untouched.

**Terms with an example** — "log every call in the service layer":
- *Aspect:* the `TimingAspect` class. *Pointcut:* the expression `within(com.acme..service..*)` = "any method in any service package". *Join point:* one specific call such as `OrderService.place()`. *Advice:* the `time(...)` method body. *Weaving:* wiring advice into the target — in Spring, at runtime via the proxy (AspectJ can instead weave at compile/load time).
- Advice types: `@Before` (before method), `@AfterReturning` (only on success), `@AfterThrowing` (on exception), `@After` (finally-like), `@Around` (wraps everything; can change args/return value or skip the call — the most powerful, and the only one that can *measure* the call). Always call `pjp.proceed()` in `@Around` or the target never runs.

**Pitfalls (same as `@Transactional`, since it is also AOP):** self-invocation bypasses the aspect; only Spring-managed beans are advised; `final`/private methods can't be proxied by CGLIB; aspects can't see field access or constructor calls (use AspectJ for that). **Probe:** *"JDK proxy vs CGLIB?"* JDK proxy implements the same interfaces (so you must inject by interface); CGLIB subclasses the class (so injecting the concrete class works).

---

## 8. Caching

### Q42. Spring's caching abstraction.

`@EnableCaching` creates a proxy-based interceptor over `CacheManager`.

```java
@Cacheable(cacheNames = "products", key = "#id", unless = "#result == null")
Product find(Long id) { ... }

@CachePut(cacheNames = "products", key = "#p.id")     // always executes, updates cache
Product update(Product p) { ... }

@CacheEvict(cacheNames = "products", key = "#id")
void delete(Long id) { ... }

@CacheEvict(cacheNames = "products", allEntries = true)
void reloadAll() { ... }

@Caching(evict = { @CacheEvict("products"), @CacheEvict("catalog") })
void bulkChange() { ... }
```

- `@Cacheable` = look first, skip method on hit. `@CachePut` = always run, then store. `condition` (pre-invocation, on args) vs `unless` (post-invocation, can see `#result`).
- Same proxy limitations as `@Transactional`: **self-invocation and private methods don't cache.**
- Custom `KeyGenerator` when the default (`SimpleKey` of all params) isn't right — and remember the default key ignores nothing, so adding a param changes keys.

**What and why:** caching stores the result of an expensive call (DB query, remote API) so repeat calls with the same arguments return instantly. Spring's abstraction means you annotate the method and don't write cache plumbing; the backing store (Caffeine, Redis, Ehcache) is swappable through `CacheManager`. The proxy checks the cache **before** your method runs; on a miss it runs the method and stores the return value under a key built from the arguments (`#id` is a SpEL expression referring to the parameter named `id` — compile with `-parameters`, which Boot's build plugins enable).

```java
service.find(5L);   // MISS → runs method, caches Product#5
service.find(5L);   // HIT  → method body not executed
service.update(p);  // @CachePut refreshes entry so next find() sees new data
service.delete(5L); // @CacheEvict removes it
```

**Pitfalls:** caching a mutable object means callers can change the cached instance; `null` results are cached by default unless you add `unless = "#result == null"`; without a TTL/size limit the default `ConcurrentMapCache` grows forever (configure Caffeine/Redis with expiry); `@Cacheable` on a method that throws caches nothing.

---

### Q43. Local (Caffeine) vs distributed (Redis) — how do you choose?

| | Caffeine (local) | Redis (distributed) |
|---|---|---|
| Latency | ~ns, in-heap | ~ms, network hop |
| Consistency across pods | ❌ each pod has its own copy | ✅ single source |
| Capacity | Bounded by heap | Large, external |
| Failure impact | None | Cache outage can take down the app if not degraded gracefully |

**Choose local** for small, hot, tolerant-of-staleness data (config, feature flags) with short TTL.
**Choose Redis** when invalidation must be global (a price change must be visible from every pod immediately).
**Two-tier (L1 Caffeine + L2 Redis + pub/sub invalidation)** for read-heavy hot keys.

**Reasoning:** the trade-off is *speed vs. consistency*. A local cache lives inside each JVM, so reads are nanoseconds, but with 5 pods each has its own copy: update a price on pod A and pods B–E keep serving the old price until their TTL expires. Redis is one shared copy, so every pod sees the same value, at the cost of a network round-trip (~0.5–2 ms) and a new dependency to operate. In two-tier setups, a write evicts the key in Redis and publishes an "invalidate key X" message (Redis pub/sub) so every pod drops its local copy.

```java
@Bean CacheManager cacheManager() {
    var c = new CaffeineCacheManager("config");
    c.setCaffeine(Caffeine.newBuilder().maximumSize(10_000).expireAfterWrite(Duration.ofMinutes(5)));
    return c;
}
```

**Pitfall:** treat Redis as an optimization, not a hard dependency — wrap calls with timeouts and a fallback to the DB (a custom `CacheErrorHandler`) so a Redis outage degrades to "slower" rather than "down".

```yaml
spring.cache.type: redis
spring.data.redis.host: redis
spring.cache.redis.time-to-live: 10m
spring.cache.redis.cache-null-values: false
```

---

### Q44. Cache problems you must be able to name.

| Problem | What happens | Fix |
|---|---|---|
| **Stampede / thundering herd** | Hot key expires, 1000 requests all hit the DB | `@Cacheable(sync = true)`, Caffeine `refreshAfterWrite`, mutex/single-flight, probabilistic early refresh |
| **Avalanche** | Many keys expire at the same instant (e.g. bulk warm-up) | Add random jitter to TTLs |
| **Penetration** | Requests for non-existent keys always miss | Cache negative results with a short TTL, or a Bloom filter |
| **Staleness** | Cache and DB diverge | Short TTL + explicit eviction on write; evict **after commit**, not inside the tx |
| **Big keys / memory** | OOM or Redis eviction storms | Size limits, `maximumSize`/`maxmemory-policy allkeys-lru`, cache DTOs not entities |

**Also:** never cache entities (they carry lazy proxies and identity semantics) — cache immutable DTOs, and version your cache key prefix so a deploy with a changed DTO shape doesn't deserialize garbage.

**Quick intuition for the names:** *Stampede* — one popular key (say, homepage banners) expires; in the next millisecond 1000 requests all miss and all run the same expensive query. `@Cacheable(sync = true)` makes only the first caller compute while the rest wait for its result. *Avalanche* — you warmed 100k keys at startup with the same 10-minute TTL, so they all expire together and the DB gets hit with everything at once; jitter (`ttl + random(0..60s)`) spreads the expiry. *Penetration* — an attacker requests IDs that don't exist (`/products/-1`); nothing is ever cached because there's no value, so every request reaches the DB; caching "not found" for 30–60 s or a **Bloom filter** (a compact probabilistic set that can say "definitely absent") blocks them.

---

## 9. Async, Scheduling & Concurrency

### Q45. `@Async` — how do you use it correctly?

```java
@Configuration @EnableAsync
class AsyncConfig implements AsyncConfigurer {
    @Bean("emailExecutor")
    ThreadPoolTaskExecutor emailExecutor() {
        var ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(8);
        ex.setMaxPoolSize(16);
        ex.setQueueCapacity(500);
        ex.setThreadNamePrefix("email-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        ex.setWaitForTasksToCompleteOnShutdown(true);
        ex.setAwaitTerminationSeconds(30);          // graceful shutdown
        return ex;
    }
    @Override public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, m, params) -> log.error("Async failure in {}", m, ex);
    }
}

@Async("emailExecutor")
public CompletableFuture<Void> send(String to) { ... }
```

**Gotchas to volunteer:**
1. **Never rely on the default executor.** Pre-Boot-3.2 it was a `SimpleAsyncTaskExecutor` that creates an unbounded number of threads (no pooling) — a production landmine. Always name your own.
2. Same proxy rule: `@Async` on a self-invoked or private method does nothing.
3. `void` return → exceptions are swallowed unless you register an `AsyncUncaughtExceptionHandler`. Prefer `CompletableFuture<T>`.
4. **Nothing propagates automatically:** SecurityContext (use `DelegatingSecurityContextAsyncTaskExecutor`), MDC/traceId (use a `TaskDecorator`), transactions (a new thread = new transaction), request-scoped beans.
5. **Understand the queue semantics:** a `ThreadPoolTaskExecutor` only grows from core → max **after the queue is full**. So `queueCapacity=Integer.MAX_VALUE` means `maxPoolSize` is never reached.

```java
ex.setTaskDecorator(runnable -> {
    var ctx = MDC.getCopyOfContextMap();
    return () -> { MDC.setContextMap(ctx); try { runnable.run(); } finally { MDC.clear(); } };
});
```

**Plain-language:** `@Async` makes a method return immediately while its body runs on **another thread** from a pool — good for slow side tasks (sending email, calling a slow partner API) that the HTTP response doesn't need to wait for. Spring achieves it with a proxy (like `@Transactional`) that submits your method as a task to an executor. Pool terms: *core size* = threads kept alive; *queue* = waiting room for tasks when all core threads are busy; *max size* = extra threads only created when the queue is full; *rejection policy* = what to do when queue and max are both full — `CallerRunsPolicy` runs the task on the calling (request) thread, which naturally slows the producer down (back-pressure) instead of dropping work. *MDC* (Mapped Diagnostic Context) is the per-thread map the logger uses for fields like `traceId`; since a new thread starts with an empty one, the `TaskDecorator` above copies it across so async log lines remain correlated.

**Example of the queue gotcha:** core=8, max=16, queue=500: tasks 1–8 get threads, tasks 9–508 wait in the queue, only task 509 onward triggers threads 9–16. Tasks 1–508 never use more than 8 threads — so a "burst" doesn't scale out as people expect.

---

### Q46. `@Scheduled` — and how do you run it safely in a multi-instance deployment?

```java
@Scheduled(fixedDelay = 30_000, initialDelay = 10_000)   // waits for previous to finish
@Scheduled(fixedRate  = 30_000)                          // fires regardless (can overlap/pile up)
@Scheduled(cron = "0 0 2 * * *", zone = "Asia/Kolkata")  // 2 AM daily
```

**The #1 real-world issue:** the default scheduler is a **single-threaded** `ThreadPoolTaskScheduler` — one slow job blocks every other job. Fix: `spring.task.scheduling.pool.size=5` or a custom `TaskScheduler`.

**The #2 issue:** with 3 replicas, the job runs **3 times**. Solutions:
- **ShedLock** (`@SchedulerLock`) — DB/Redis-based lock, simplest and most common.
- Quartz in clustered mode (JDBC job store) — heavier, supports misfire policies and persistence.
- Kubernetes `CronJob` running a one-shot task — moves scheduling out of the app.
- Leader election (Spring Integration / Zookeeper / Consul).

Always make jobs **idempotent** and **re-entrant-safe**, add timeouts, and emit metrics (last success timestamp) so you can alert on a job that stopped running.

**Differences explained:** `fixedDelay` = wait N ms *after the previous run finishes* (never overlaps); `fixedRate` = start every N ms measured from the previous *start* (if a run takes longer than N, runs queue up or overlap on a multi-thread scheduler); `cron` = calendar-based (`second minute hour day-of-month month day-of-week`, so `0 0 2 * * *` = 02:00:00 every day). Always specify `zone`, otherwise it uses the server's timezone, which differs between laptop and container (usually UTC). `@EnableScheduling` must be present on a configuration class or nothing fires.

**Why duplicates happen:** `@Scheduled` is purely **in-process** — each JVM has its own timer and knows nothing about other pods. ShedLock fixes this by making the job take a named lock (a row in a `shedlock` table or a Redis key) before running; pods that fail to get it skip that tick.

```java
@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "nightlyReport", lockAtMostFor = "PT30M", lockAtLeastFor = "PT1M")
public void nightlyReport() { ... }   // only one pod runs it; lock auto-expires after 30m if the pod dies
```

`lockAtMostFor` protects against a crashed node holding the lock forever; `lockAtLeastFor` prevents a fast job being re-run by another pod whose clock is slightly ahead.

---

### Q47. `CompletableFuture` orchestration in a Spring service.

```java
public OrderView load(Long id) {
    var order    = supplyAsync(() -> orderClient.get(id), ioPool);
    var customer = supplyAsync(() -> custClient.get(id),  ioPool);
    var shipping = supplyAsync(() -> shipClient.get(id),  ioPool);

    return allOf(order, customer, shipping)
        .orTimeout(2, TimeUnit.SECONDS)
        .thenApply(v -> new OrderView(order.join(), customer.join(), shipping.join()))
        .exceptionally(ex -> OrderView.degraded(id))     // partial response beats 500
        .join();
}
```
Key points: parallel fan-out turns 3×300 ms sequential into ~300 ms; use a **dedicated bounded pool** (not `ForkJoinPool.commonPool`, which is shared and sized to CPUs); always add timeouts; design graceful degradation.

**Reading the code:** a `CompletableFuture` is a promise of a value that will be filled in later by another thread. `supplyAsync(task, pool)` starts the task on the given pool and returns immediately, so all three remote calls run *simultaneously*. `allOf(...)` creates a future that completes when all three are done. `orTimeout` fails it if it takes over 2 s, `thenApply` transforms the result, and `exceptionally` supplies a fallback if anything failed or timed out. `join()` waits for the final value (like `get()` but without checked exceptions). Inside `thenApply`, `order.join()` returns instantly because `allOf` already guaranteed completion.

**Why not the common pool:** blocking I/O calls inside `commonPool` (about *CPU-count − 1* threads) can starve every other parallel stream/future in the JVM. An I/O pool can be larger (I/O waits don't use CPU) and is isolated so one slow dependency can't consume everything.

**Pitfalls:** `thenApply` on an already-failed future skips the lambda — errors surface only at `join()` (wrapped in `CompletionException`); `orTimeout` doesn't cancel the underlying HTTP call, so also set client timeouts; context (security, MDC, transactions) is **not** propagated into these threads. On Java 21+ virtual threads or `StructuredTaskScope` (preview) can express this more simply.

---

### Q48. How do you make a shared bean thread-safe?

Singletons are shared across all request threads. Rules:
- Keep controllers/services **stateless**; put per-request state in method locals or parameters.
- If you need mutable state: `AtomicLong`/`LongAdder`, `ConcurrentHashMap`, `computeIfAbsent`, immutable objects, or `synchronized`/`ReentrantLock` on the narrowest section.
- Not thread-safe and commonly misused as fields: `SimpleDateFormat` (use `DateTimeFormatter`), `Random` (use `ThreadLocalRandom`), non-thread-safe `ObjectMapper` configuration mutation (the mapper itself is thread-safe once configured).
- Use `@Scope("request")` or `ThreadLocal` only with a cleanup filter — with pooled threads, forgetting to clear a `ThreadLocal` leaks data **across users** (a real security incident pattern).

**Why this is an issue:** Spring creates **one** instance of each singleton, and Tomcat runs many request threads through it at the same time. If that single instance has a mutable field, all users read and write the same variable — a *race condition* (outcome depends on timing).

```java
@Service
class BadCounter {
    private int count = 0;                          // shared by every request
    int next() { return ++count; }                  // ++ is read-modify-write, not atomic → lost updates
}
@Service
class GoodCounter {
    private final AtomicLong count = new AtomicLong();
    long next() { return count.incrementAndGet(); } // atomic
}
```

Worse example: a service storing `private User currentUser` as a field — request A's thread overwrites it while request B is mid-way, so B acts as A (a data leak). Pass such state as method parameters. `SimpleDateFormat` keeps internal mutable state while parsing, so sharing one instance gives garbled dates under concurrency; `DateTimeFormatter` is immutable and safe. **Pooled-thread leak:** Tomcat reuses threads, so a `ThreadLocal` set for user A and not cleared (`remove()` in a `finally`) is still there when user B's request is served on the same thread.

---

## 10. Spring Security, JWT & OAuth2

### Q49. Explain the Spring Security filter chain.

An ordered chain of servlet filters inserted via `DelegatingFilterProxy` → `FilterChainProxy` → `SecurityFilterChain`. Key filters in order:

1. `SecurityContextHolderFilter` — loads/persists the `SecurityContext` (was `SecurityContextPersistenceFilter`)
2. `CorsFilter`
3. `CsrfFilter`
4. `LogoutFilter`
5. `UsernamePasswordAuthenticationFilter` / your `JwtAuthenticationFilter`
6. `BearerTokenAuthenticationFilter` (resource server)
7. `RequestCacheAwareFilter`, `SecurityContextHolderAwareRequestFilter`
8. `AnonymousAuthenticationFilter` — gives an anonymous token so authorization can uniformly deny
9. `ExceptionTranslationFilter` — turns `AuthenticationException` → 401 via `AuthenticationEntryPoint`, `AccessDeniedException` → 403
10. `AuthorizationFilter` (was `FilterSecurityInterceptor`) — the final authorization decision

**Core objects:** `Authentication` (principal + credentials + authorities), `SecurityContext` → held in `SecurityContextHolder` (`ThreadLocal` by default), `AuthenticationManager` → `ProviderManager` → `AuthenticationProvider`s (e.g. `DaoAuthenticationProvider` → `UserDetailsService` + `PasswordEncoder`).

**Memory trick:** *Context → CORS → CSRF → Authenticate → Anonymous → Translate → Authorize.*

**What this means in plain language:** Spring Security is not magic inside controllers — it's a **chain of servlet filters** (small interceptors that see every HTTP request before the controller does). `DelegatingFilterProxy` is a bridge so a normal servlet filter registered with Tomcat can delegate to a Spring bean; `FilterChainProxy` is that bean and picks the first `SecurityFilterChain` whose `securityMatcher` matches the URL. Every request then flows through the filters in order; any filter can stop the request (e.g. return 401) before it reaches your code.

**Walk through one request** with `Authorization: Bearer <jwt>`:
1. `SecurityContextHolderFilter` prepares an empty `SecurityContext` (the per-request holder of "who is the current user", stored in a `ThreadLocal`).
2. Your JWT filter reads the header, validates the signature/expiry, builds an `Authentication` (principal + authorities) and puts it in the context.
3. `AuthorizationFilter` checks the rules (`hasRole("ADMIN")` etc.) against that `Authentication`.
4. If no one authenticated, `AnonymousAuthenticationFilter` has put an anonymous token there, so rule checks fail → `ExceptionTranslationFilter` returns **401**; if authenticated but lacking permission → **403**.
5. The controller runs; the context is cleared at the end.

**Vocabulary:** *Authentication* = proving who you are (login/token); *Authorization* = deciding what you may do. *principal* = the user identity object; *authorities* = granted permissions/roles. `AuthenticationManager` is the entry point that asks one or more `AuthenticationProvider`s "can you verify these credentials?" — e.g. `DaoAuthenticationProvider` loads the user via `UserDetailsService` and compares the password hash via `PasswordEncoder`. To debug ordering, set `logging.level.org.springframework.security=DEBUG` — it prints the filter list per request.

---

### Q50. Modern (Boot 3 / Security 6) configuration — `WebSecurityConfigurerAdapter` is gone.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity          // replaces @EnableGlobalMethodSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http, JwtAuthFilter jwtFilter) throws Exception {
        return http
            .securityMatcher("/api/**")
            .csrf(AbstractHttpConfigurer::disable)             // stateless API + Bearer token
            .cors(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(a -> a
                .requestMatchers("/api/public/**", "/actuator/health/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/orders").hasAuthority("SCOPE_orders:write")
                .anyRequest().authenticated())
            .exceptionHandling(e -> e.authenticationEntryPoint(problemDetailEntryPoint()))
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .headers(h -> h.frameOptions(FrameOptionsConfig::deny)
                           .httpStrictTransportSecurity(hsts -> hsts.maxAgeInSeconds(31536000)))
            .build();
    }

    @Bean PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();  // {bcrypt} prefixed
    }
}
```
Mention: **multiple `SecurityFilterChain` beans** with `@Order` + `securityMatcher` for different path groups (API vs admin UI vs actuator).

**What changed and why:** older code extended `WebSecurityConfigurerAdapter` and overrode `configure(HttpSecurity)`. Spring Security 5.7 deprecated it and 6 removed it in favour of **declaring a `SecurityFilterChain` bean** — simpler to compose, to test, and to have several of them. Reading the config above top to bottom:
- `csrf(disable)` — CSRF (Cross-Site Request Forgery: a malicious site makes the user's browser send an authenticated request automatically via cookies) is only a risk when authentication rides on cookies. A stateless API using an `Authorization` header token isn't vulnerable the same way, so CSRF protection is turned off; **keep it on** for cookie/session-based browser apps.
- `STATELESS` — never create an `HttpSession`; every request must carry its own credentials, which is what lets you scale pods horizontally with no sticky sessions.
- `authorizeHttpRequests` — rules are evaluated **top to bottom, first match wins**, so put specific rules before `anyRequest()`. Forgetting `anyRequest().authenticated()` can leave unmatched endpoints open.
- `addFilterBefore(jwtFilter, ...)` — slots your custom token filter into the chain at the right position.
- `DelegatingPasswordEncoder` — stores hashes as `{bcrypt}$2a$...`, so you can later migrate to a stronger algorithm and old hashes still verify.
- `@EnableMethodSecurity` activates `@PreAuthorize` annotations on methods (separate from URL rules, a second line of defence).

---

### Q51. `hasRole` vs `hasAuthority`; `@PreAuthorize` vs `@Secured`.

- `hasRole("ADMIN")` automatically prepends `ROLE_` → matches authority `ROLE_ADMIN`. `hasAuthority("ROLE_ADMIN")` is the literal check. Mixing these up is the most common Spring Security bug.
- `@PreAuthorize("hasRole('ADMIN') and #order.ownerId == authentication.name")` — SpEL, supports method args, the modern choice. `@PostAuthorize` filters after execution; `@PreFilter`/`@PostFilter` filter collections.
- `@Secured("ROLE_ADMIN")` — legacy, no SpEL. `@RolesAllowed` — JSR-250.
- For domain-object-level rules, use ACLs or a custom `PermissionEvaluator`: `@PreAuthorize("hasPermission(#id, 'Order', 'read')")`.

**Definitions:** an *authority* is any permission string attached to the logged-in user (`orders:write`, `ROLE_ADMIN`); a *role* is just an authority whose name starts with `ROLE_`. `hasRole("ADMIN")` is shorthand for `hasAuthority("ROLE_ADMIN")`. So if your JWT scopes map to authorities like `SCOPE_orders:write`, you must use `hasAuthority("SCOPE_orders:write")`; `hasRole` would look for `ROLE_SCOPE_...` and silently deny.

```java
@EnableMethodSecurity                              // required, otherwise annotations are ignored
@Service
class OrderService {
    @PreAuthorize("hasRole('ADMIN') or #order.ownerId == authentication.name")
    public void cancel(Order order) { ... }       // runs BEFORE the method; denial → AccessDeniedException (403)

    @PostAuthorize("returnObject.ownerId == authentication.name")
    public Order get(long id) { ... }             // method runs first, result checked afterwards
}
```

**Why method security as well as URL rules:** URL rules are coarse ("only admins reach `/admin/**`"); method rules express *business* rules such as "a user may cancel only their own order" and still apply when the service is called from a scheduler, a message listener or another controller. Same proxy rule as `@Transactional`: self-invocation bypasses the check.

---

### Q52. Design JWT authentication end to end.

**Flow:** `/login` validates credentials → issue a **short-lived access token (5–15 min)** + a **long-lived refresh token (days, rotating, stored server-side/hashed)** → client sends `Authorization: Bearer <jwt>` → a filter validates signature, `exp`, `iss`, `aud` and populates the `SecurityContext`.

```java
public class JwtAuthFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String header = req.getHeader(HttpHeaders.AUTHORIZATION);
        if (header != null && header.startsWith("Bearer ")) {
            try {
                var claims = jwtService.parse(header.substring(7));       // verifies signature + exp
                var auth = new UsernamePasswordAuthenticationToken(
                        claims.getSubject(), null, jwtService.authorities(claims));
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException ex) {
                SecurityContextHolder.clearContext();   // let ExceptionTranslationFilter emit 401
            }
        }
        chain.doFilter(req, res);
    }
}
```

**What a JWT is:** a JSON Web Token is a compact string `header.payload.signature` (each part Base64URL-encoded). The payload holds *claims* — `sub` (subject/user id), `exp` (expiry time), `iss` (issuer), `aud` (intended audience), plus custom ones like roles. The signature is produced with a key so the server can check nobody altered the payload. The payload is **encoded, not encrypted** — anyone can read it, so never put secrets in it. The big appeal: services can validate it **without a DB/session lookup**, which suits stateless, horizontally scaled APIs.

**Why two tokens:** because a JWT can't be recalled once issued, the access token is kept short-lived (minutes). To avoid forcing the user to log in every 10 minutes, a longer-lived **refresh token** (stored server-side, hashed) is exchanged for a new access token. *Rotation* = each refresh issues a new refresh token and invalidates the old; *reuse detection* = if an already-used refresh token appears again, it was probably stolen, so revoke the whole token family.

**In the filter above:** `OncePerRequestFilter` guarantees it runs once per request; it extracts the token, verifies it, builds an `Authentication` and stores it in the `SecurityContext`. If the token is bad it simply leaves the context empty, and the later `ExceptionTranslationFilter`/entry point answers 401.

**Talking points that separate seniors:**
- **Use RS256/ES256 (asymmetric)** in a microservices estate: the auth server holds the private key; services verify with the public key from a JWKS endpoint. HS256 means sharing a secret with every service.
- **You cannot revoke a stateless JWT** → keep TTL short, maintain a denylist (`jti` in Redis until natural expiry) for logout/compromise, and support refresh-token rotation with reuse detection.
- **Never store JWTs in `localStorage`** if you can avoid it (XSS-readable). Prefer `HttpOnly; Secure; SameSite=Strict` cookies — then you need CSRF protection again.
- Don't put PII or authorization data that changes often in claims — it's cached until expiry.
- Validate `alg` explicitly (guard against `alg: none` and key-confusion attacks).
- **When you have a real IdP (Keycloak/Auth0/Cognito), don't hand-roll:** use `spring-boot-starter-oauth2-resource-server` with `spring.security.oauth2.resourceserver.jwt.issuer-uri` — it handles JWKS rotation and validation for you.

---

### Q53. OAuth2 / OIDC essentials.

- **Roles:** Resource Owner (user), Client (app), Authorization Server, Resource Server.
- **Grants:** Authorization Code + **PKCE** (all interactive apps, including SPAs and mobile — implicit and password grants are deprecated), **Client Credentials** (service-to-service), Device Code (TVs/CLI), Refresh Token.
- **OIDC** adds an `id_token` (a JWT about the user) and the `userinfo` endpoint on top of OAuth2 (which is authorization only, not authentication).
- Spring modules: `oauth2-client` (login/act as client), `oauth2-resource-server` (validate tokens), `spring-authorization-server` (be the IdP).
- Service-to-service: `ClientRegistration` with `client_credentials` + `OAuth2AuthorizedClientManager`, propagated via a `WebClient`/`RestClient` filter.

**CSRF rule of thumb:** stateless token in an `Authorization` header → CSRF can be disabled. Cookie-based session/auth → CSRF **must** stay on.

**The four roles in a real example:** you click "Sign in with Google" on a photo-printing app. *You* = Resource Owner; the *photo app* = Client; *Google's login server* = Authorization Server; *Google Photos API* = Resource Server (it accepts the access token the auth server issued). The app never sees your Google password — it only gets a token with limited **scopes** (e.g. `photos.read`).

**Authorization Code + PKCE, step by step:**
1. App redirects the browser to the auth server with a `code_challenge` (hash of a random secret the app generates, the *code verifier*).
2. User logs in and consents; the auth server redirects back with a one-time **authorization code**.
3. App's backend/SPA exchanges the code **plus the original code verifier** for tokens; the server checks the hash matches.
4. PKCE (Proof Key for Code Exchange, pronounced "pixie") exists so that a stolen authorization code is useless to an attacker who doesn't know the verifier — essential for public clients (SPA, mobile) that can't keep a client secret.

**Client Credentials** has no user: service A authenticates with its own `client_id/secret` to get a token for calling service B (batch jobs, microservice calls).

```yaml
spring.security.oauth2.resourceserver.jwt.issuer-uri: https://login.acme.com/realms/main
# Spring fetches the issuer's public keys (JWKS) and validates every incoming JWT automatically
```

**OAuth2 vs OIDC in one line:** OAuth2 answers "may this app access that API?" (an access token); OIDC (OpenID Connect) adds "who is the user?" (an `id_token`). Interview trap: an access token must not be used as proof of login.

---

### Q54. Common security hardening you'd add to any service.

- BCrypt/Argon2 password hashing via `DelegatingPasswordEncoder` (never MD5/SHA + no salt).
- Rate limiting / brute-force lockout at the gateway (Bucket4j, Resilience4j, or Nginx/APIM).
- HTTPS only, HSTS, secure headers (CSP, X-Content-Type-Options, Referrer-Policy).
- Parameterized queries everywhere — the SQL injection risk in Spring Data is **native queries with string concatenation** and unvalidated `Sort`/dynamic ORDER BY.
- Actuator on a separate port/network, secured; never expose `/heapdump`, `/env`, `/threaddump` publicly.
- Dependency scanning (OWASP dependency-check, Snyk, `mvn versions:display-dependency-updates`) — Spring4Shell/Log4Shell lessons.
- Input validation + output encoding; mass-assignment protection (use explicit request DTOs, never bind entities directly to `@RequestBody`).
- Audit logging of authz failures; don't log tokens or PII.

**Why each item matters (one line each):**
- *Hashing:* BCrypt/Argon2 are deliberately slow and add a random **salt** per password, so a stolen DB can't be cracked with precomputed tables; MD5/SHA are fast, which helps attackers.
- *Rate limiting:* caps requests per user/IP (e.g. 5 login attempts/min) to stop credential stuffing and abuse.
- *HSTS* tells browsers to use HTTPS only for this domain; *CSP* (Content-Security-Policy) restricts where scripts can load from, limiting XSS damage; `X-Content-Type-Options: nosniff` stops browsers guessing file types.
- *SQL injection example:*

```java
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)   // ❌ concatenation: name = "x' OR '1'='1"
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)            // ✔ bound parameter
```

  Parameters can't be used for column names, so a user-controlled `sort=...` must be checked against an allow-list.
- *Actuator exposure:* `/heapdump` contains every object in memory — including passwords and tokens; `/env` can reveal config.
- *Mass assignment:* if you bind JSON straight onto an entity, a client can send `"role":"ADMIN"` or `"id":1` and set fields you never intended. A request DTO with only permitted fields prevents it.
- *Spring4Shell/Log4Shell* were library vulnerabilities, so automated dependency scanning in CI catches known-bad versions early.

---

## 11. Actuator, Observability & Production Readiness

### Q55. What do you actually use Actuator for?

`spring-boot-starter-actuator` exposes operational endpoints. Only `/health` and `/info` are exposed over HTTP by default.

| Endpoint | Purpose |
|---|---|
| `/health` | Liveness/readiness; aggregates `HealthIndicator`s (db, redis, disk, kafka) |
| `/metrics`, `/prometheus` | Micrometer metrics scraping |
| `/info` | Build/git info (`build-info` goal, `git-commit-id-plugin`) |
| `/env`, `/configprops` | Effective configuration + where each property came from |
| `/loggers` | **Change log levels at runtime** — no redeploy to debug prod |
| `/threaddump`, `/heapdump` | Diagnose hangs / memory leaks |
| `/httpexchanges`, `/mappings`, `/beans`, `/conditions` | Introspection |
| `/shutdown` | Graceful shutdown (disabled by default) |

```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus,loggers
  endpoint.health:
    show-details: when-authorized
    probes.enabled: true            # /health/liveness and /health/readiness
  server.port: 9090                 # separate port, not exposed to the internet
  metrics.tags.application: ${spring.application.name}
```

**Kubernetes tie-in (great answer):** map `livenessProbe` → `/actuator/health/liveness` (is the JVM alive? restart if not) and `readinessProbe` → `/actuator/health/readiness` (can it serve traffic? remove from the Service if not). **Don't put DB checks in liveness** — a brief DB blip would restart every pod and cause a cascading outage. Add `spring.lifecycle.timeout-per-shutdown-phase` + `server.shutdown=graceful` so in-flight requests finish during a rolling deploy.

Custom indicator:
```java
@Component
class PaymentGatewayHealth implements HealthIndicator {
    public Health health() {
        return client.ping()
            ? Health.up().withDetail("latencyMs", client.lastLatency()).build()
            : Health.down().withDetail("reason", "ping failed").build();
    }
}
```

---

### Q56. How do you instrument metrics with Micrometer?

Micrometer is the "SLF4J of metrics" — one API, many backends (Prometheus, Datadog, New Relic, CloudWatch, OTLP).

```java
@Timed(value = "order.place", percentiles = {0.5, 0.95, 0.99})
public Order place(...) { ... }

// programmatic
Counter.builder("orders.created").tag("channel", channel).register(registry).increment();
registry.gauge("queue.depth", queue, Queue::size);
Timer.Sample s = Timer.start(registry); ... s.stop(registry.timer("db.call"));
```

**The cardinality warning (a senior signal):** never tag with unbounded values — `userId`, `orderId`, raw URLs with IDs, email. Each unique tag combination is a separate time series and will blow up Prometheus. Use templated URIs (`/orders/{id}`).

**Golden signals to expose:** latency (p50/p95/p99), traffic (RPS), errors (rate by status), saturation (pool usage, heap, CPU, queue depth). Alert on **SLO burn rate**, not on raw CPU.

**Types explained:** a **Counter** only goes up (orders created, errors); a **Gauge** is a value that moves both ways and is sampled (queue depth, active connections); a **Timer** records duration *and* count (latency distribution); a **DistributionSummary** is like a Timer for non-time values (payload size). *Percentiles:* p95 = 95% of requests were faster than this; averages hide slow outliers, so you alert on p95/p99. *Tag* (a.k.a. label) = dimension such as `channel=web` that lets you slice one metric.

**How it flows:** your code → Micrometer `MeterRegistry` → a backend registry (e.g. Prometheus) → exposed at `/actuator/prometheus` → Prometheus scrapes it periodically (pull model) → Grafana graphs/alerts. Boot auto-registers JVM, Tomcat, HikariCP, and HTTP server metrics (`http.server.requests`) for free. **Cardinality example:** `tag("userId", id)` with 1M users = 1M time series per metric → Prometheus memory explodes; use `tag("tier", "gold")` instead. **SLO burn rate** = how fast you're consuming your error budget (e.g. a 99.9% SLO allows 0.1% errors; if you're erroring at 1%, you burn it 10× too fast) — more meaningful than "CPU > 80%".

---

### Q57. How do you do distributed tracing and log correlation?

- **Boot 3:** Micrometer Tracing (Brave or OpenTelemetry) replaced Spring Cloud Sleuth. Add `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp`, export to Tempo/Jaeger/Zipkin.
- A `traceId` and `spanId` are auto-propagated via W3C `traceparent` headers across `RestClient`/`WebClient`/Kafka/RabbitMQ and injected into **MDC** so every log line carries them.
- Log pattern: `%d %-5level [${spring.application.name},%X{traceId:-},%X{spanId:-}] %logger{36} - %msg%n`.
- Ship **structured JSON logs** (logstash-logback-encoder) to Loki/ELK; return the `traceId` in error responses so support can jump straight to the trace.
- Sampling: 100% in dev, 1–10% + tail-based sampling of errors in prod.

**Three pillars phrase:** *"Metrics tell you something is wrong, traces tell you where, logs tell you why."*

**The problem this solves:** one user click may pass through gateway → order-service → payment-service → Kafka → notification-service. When it's slow or fails, grepping five services' logs by timestamp is hopeless. **Distributed tracing** assigns the whole request one **`traceId`**, and each unit of work (an incoming call, an outgoing HTTP call, a DB query) is a **span** with its own `spanId`, parent, and duration. A tracing UI (Tempo/Jaeger/Zipkin) draws a waterfall showing e.g. "payment-service took 1.8 s of the 2 s".

**How it works:** the first service creates the `traceId`; when it calls another service it adds the W3C `traceparent` header (`00-<traceId>-<spanId>-01`); the next service reads that header and continues the same trace. Boot's auto-configured `RestClient.Builder`/`WebClient.Builder`/Kafka templates do this automatically **only if you build clients from the injected builder** (calling `new RestTemplate()` yourself loses propagation). *MDC* (Mapped Diagnostic Context) is the logger's per-thread key-value map; the tracing library puts `traceId`/`spanId` into it, so the log pattern above prints them on every line. Then: user reports error → copy `traceId` from the response/support screen → search logs for it across all services.

```yaml
management.tracing.sampling.probability: 0.1      # sample 10% of requests
management.otlp.tracing.endpoint: http://otel-collector:4318/v1/traces
```

**Sampling** exists because recording every span is costly; *head-based* sampling decides at the start (random 10%), *tail-based* (done in the collector) decides after seeing the whole trace so it can keep all slow/error traces.

---

## 12. Testing

### Q58. Test slices — which annotation when?

| Annotation | Loads | Use for |
|---|---|---|
| plain JUnit + Mockito | Nothing (no Spring) | **Unit tests of business logic — the majority** |
| `@WebMvcTest` | MVC layer only, mock the service | Controller mapping, validation, JSON, security rules |
| `@DataJpaTest` | JPA + embedded/Testcontainers DB, tx rolled back per test | Repository queries, mappings |
| `@JsonTest` | Jackson only | Serialization contracts |
| `@RestClientTest` | Client + `MockRestServiceServer` | Outbound HTTP clients |
| `@SpringBootTest` | Whole context | Wiring/integration; **slowest — use sparingly** |
| `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate` | Real server | End-to-end API tests |

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockitoBean OrderService service;              // @MockBean before Boot 3.4

    @Test void returns404WhenMissing() throws Exception {
        given(service.find(9L)).willThrow(new ResourceNotFoundException("no order"));
        mvc.perform(get("/api/orders/9"))
           .andExpect(status().isNotFound())
           .andExpect(jsonPath("$.detail").value("no order"));
    }
}
```

**Key insight to state:** every distinct combination of `@MockBean`/properties creates a **new application context**; Spring caches contexts by configuration, so minimizing variants is the single biggest test-suite speed win. "We cut CI from 22 to 7 minutes by consolidating test configurations so contexts were reused."

**Why slices exist:** a full `@SpringBootTest` starts the entire app (DB pool, Kafka, security...) — seconds per context. A *test slice* loads only the beans for one layer, so tests are faster and failures point to one layer. For example `@WebMvcTest` builds the controllers, `@ControllerAdvice`, filters, and Jackson, but **not** services/repositories — hence you supply fake ones with `@MockitoBean`, and test only HTTP concerns (URL mapping, status, JSON shape, validation, security) through `MockMvc` (an in-memory fake of the servlet container, no real port).

**Rule of thumb:** put business rules in plain classes tested by plain JUnit + Mockito (milliseconds, no Spring); use a slice when you're testing *framework behaviour* (does `@Valid` return 400? does this derived query return the right rows?). **Why "different `@MockBean` = new context":** Spring's context cache key includes the set of mocks, active profiles and properties; each new key means a fresh startup, so avoid sprinkling unique `@MockBean`s/`@TestPropertySource`s across test classes — put them in a shared base class or configuration.

---

### Q59. Testcontainers vs H2 for repository tests.

H2 is fast but **lies**: different SQL dialect, no native JSON/array types, different constraint and locking behaviour, no `FOR UPDATE SKIP LOCKED`. Bugs slip to production.

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIT {
    @Container @ServiceConnection                 // Boot 3.1+: auto-wires datasource props
    static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16-alpine");
}
```
`@ServiceConnection` removed all the old `@DynamicPropertySource` boilerplate. Reuse containers (`.withReuse(true)` + `testcontainers.reuse.enable=true`) and make them `static` so one container serves the class. Also mention: Testcontainers works for Kafka, Redis, LocalStack, WireMock — so integration tests use real infrastructure.

Also worth naming: **`@SpringBootTest` + `spring-boot-testcontainers` dev-mode support** (`SpringApplication.from(...).with(TestcontainersConfig.class)`) to run the app locally against real dependencies.

**What Testcontainers is:** a library that starts real Docker containers (Postgres, Kafka, Redis) from inside your JUnit test, waits until they're healthy, and tears them down afterwards. Your repository test therefore runs against the **same database engine as production**. `@ServiceConnection` tells Boot "read the container's random host/port/credentials and configure `spring.datasource.*` from it" — before 3.1 you had to copy these into properties by hand with `@DynamicPropertySource`.

**Concrete H2 lie:** a query using PostgreSQL's `jsonb` operators, `ILIKE`, or `SELECT ... FOR UPDATE SKIP LOCKED` (work-queue pattern) either fails on H2 or behaves differently; a Flyway migration using Postgres-specific DDL won't even run. Even H2's "PostgreSQL mode" only imitates a subset.

**Trade-offs/pitfalls:** needs Docker on the dev machine and in CI (the CI agent must allow it); start-up costs a few seconds per container — hence `static` containers and reuse; keep them **shared across test classes** (singleton container pattern) to avoid paying that cost per class.

---

### Q60. How do you test transactions, security, and async code?

- **Transactions:** `@Transactional` on a test rolls back afterwards. But be careful — it makes the test share the transaction with the code, so it can **hide** missing-transaction and lazy-loading bugs. For those, use `@Commit` or `TransactionTemplate` and assert in a fresh transaction.
- **Security:** `@WithMockUser(roles = "ADMIN")`, `@WithUserDetails`, or `SecurityMockMvcRequestPostProcessors.jwt()`; include `spring-security-test`.
- **Async/scheduled:** don't `Thread.sleep`. Use Awaitility: `await().atMost(5, SECONDS).untilAsserted(() -> verify(mailer).send(any()))`. Or make the executor synchronous in tests (`new SyncTaskExecutor()`).
- **External HTTP:** WireMock or `MockRestServiceServer` — never call real third parties in CI.
- **Contract testing:** Spring Cloud Contract or Pact so a provider can't break consumers silently. Mentioning consumer-driven contracts is a strong senior signal.
- **Test data:** builders/object mothers over shared SQL fixtures; `@Sql` for edge cases.

**Testing pyramid line:** "Many fast unit tests, a moderate number of slice tests, few end-to-end tests — and one smoke test per environment after deploy."

**Examples for each bullet:**

```java
// Transactions: test-level @Transactional hides a LazyInitializationException because the test's tx stays open.
@SpringBootTest class OrderServiceIT {
    @Autowired OrderService service; @Autowired TransactionTemplate tx;
    @Test void loadsItemsOutsideTx() {
        Long id = tx.execute(s -> service.create(...).getId());        // committed for real
        assertThatCode(() -> service.getDto(id)).doesNotThrowAnyException();
    }
}

// Security: pretend a user is logged in without real authentication
@Test @WithMockUser(roles = "USER")
void adminEndpointForbidden() throws Exception {
    mvc.perform(delete("/api/admin/orders/1")).andExpect(status().isForbidden());
}
```

- *Why not `Thread.sleep`:* too short = flaky test; too long = slow suite. **Awaitility** polls the condition until it's true or the timeout hits, so the test is as fast as the code allows.
- *Contract testing:* the consumer team writes "when I call `GET /orders/1` I expect these fields"; the provider's build verifies it still satisfies those contracts, so a renamed field fails the provider's CI instead of the consumer's production.
- *Object mother / builder:* a helper like `anOrder().withStatus(PAID).build()` creates valid test data with sensible defaults so each test only states what matters.

---

## 13. Microservices & Cloud Patterns

### Q61. Monolith vs microservices — when do you actually split?

**Say this:** "Microservices buy independent deployability and scaling at the cost of distributed-systems complexity — network failures, eventual consistency, distributed tracing, data duplication, and much harder local development. I'd start with a **modular monolith** with clean module boundaries and split only when there's a real driver: differing scaling profiles, differing release cadences, team autonomy at scale, or differing compliance/isolation requirements."

**Split by:** bounded context / business capability (Order, Payment, Inventory) — never by technical layer, and never so small that one feature touches five services. Each service owns its own database (no shared schema — that's a distributed monolith).

**Terms explained:** a *monolith* is one deployable containing everything; a *modular monolith* is still one deployable, but internally split into modules with enforced boundaries (separate packages/Maven modules, modules talk through interfaces or events, no reaching into each other's tables) — so you can later extract a module into a service cheaply. A **bounded context** (from Domain-Driven Design) is a part of the business where words have one clear meaning: "Customer" in Billing (payment terms) differs from "Customer" in Support (tickets), so each gets its own model. A **distributed monolith** is the worst of both: many services deployed separately but so coupled (shared DB, synchronous call chains) that you must release them together.

**What you pay for with microservices:** a method call becomes a network call that can time out; one business transaction spans several databases, so you need sagas/outbox and *eventual consistency* (data is consistent "after a short delay", not instantly); testing, tracing, versioning of APIs, and on-call load all grow. **Rule-of-thumb probes:** *"How small should a service be?"* — owned by one team and changeable without coordinating releases; *"Shared database?"* — no, share through APIs/events.

---

### Q62. Core Spring Cloud components.

| Concern | Modern choice | Legacy |
|---|---|---|
| Service discovery | Kubernetes DNS/Services, Consul, Eureka | Eureka + Ribbon |
| Client-side load balancing | Spring Cloud LoadBalancer | Ribbon (EOL) |
| API gateway | Spring Cloud Gateway (reactive), or Envoy/Istio/Kong | Zuul 1 (EOL) |
| Config | Spring Cloud Config Server, K8s ConfigMap, Vault | — |
| Resilience | **Resilience4j** | Hystrix (EOL) |
| Tracing | Micrometer Tracing + OTel | Sleuth + Zipkin |
| Messaging | Kafka / RabbitMQ, Spring Cloud Stream | — |

**Gateway responsibilities:** routing, authn/token validation, rate limiting, request/response transformation, canary routing, TLS termination, aggregation of cross-cutting concerns. Keep **business logic out** of the gateway.

**What each concern means:**
- **Service discovery:** instances come and go (autoscaling, redeploys) with changing IPs, so callers can't hard-code addresses. A registry maps a logical name (`payment-service`) to live instances. On Kubernetes this is built in — a `Service` gives a stable DNS name (`http://payment-service`) and K8s balances across pods, so Eureka is often unnecessary.
- **Client-side load balancing:** the *caller* picks one of several instances (round-robin etc.) instead of going through a central balancer. Spring Cloud LoadBalancer does this when you use `lb://payment-service` URIs.
- **API gateway:** the single front door for external clients — one entry point that routes `/orders/**` to order-service, validates tokens, and applies rate limits, so each service doesn't repeat that.
- **Config server:** central place for externalized config that services pull at startup (and refresh); on Kubernetes ConfigMaps/Secrets often play this role.
- **Resilience4j:** libraries for circuit breaker/retry/bulkhead (next question). **EOL** (end-of-life) means no longer maintained, so you'd migrate away.

```yaml
spring.cloud.gateway.routes:
  - id: orders
    uri: lb://order-service              # logical name resolved via discovery + load balancer
    predicates: [ Path=/api/orders/** ]
    filters: [ StripPrefix=1, RequestRateLimiter ]
```

**Probe:** *"Do you need Eureka on Kubernetes?"* — usually not; K8s DNS + readiness probes already give discovery and health-aware routing. Mention Spring Cloud Kubernetes if you need the Spring abstractions.

---

### Q63. Resilience patterns — Resilience4j in depth.

```yaml
resilience4j:
  circuitbreaker.instances.payments:
    slidingWindowType: COUNT_BASED
    slidingWindowSize: 50
    failureRateThreshold: 50
    slowCallDurationThreshold: 2s
    slowCallRateThreshold: 50
    waitDurationInOpenState: 30s
    permittedNumberOfCallsInHalfOpenState: 5
    minimumNumberOfCalls: 20
  retry.instances.payments:
    maxAttempts: 3
    waitDuration: 200ms
    enableExponentialBackoff: true
    retryExceptions: [java.io.IOException]
  timelimiter.instances.payments.timeoutDuration: 3s
  bulkhead.instances.payments.maxConcurrentCalls: 20
```
```java
@CircuitBreaker(name = "payments", fallbackMethod = "queueForLater")
@Retry(name = "payments")
@Bulkhead(name = "payments")
@TimeLimiter(name = "payments")
public CompletableFuture<Receipt> charge(Payment p) { ... }

private CompletableFuture<Receipt> queueForLater(Payment p, Throwable t) { ... }
```

**Circuit breaker states:** CLOSED → (failure rate exceeded) → **OPEN** (fail fast, no calls) → after wait → **HALF_OPEN** (limited probe calls) → CLOSED or OPEN.

**What each pattern protects against, plainly:**
- *Circuit breaker* — like an electrical breaker. While the downstream is healthy (CLOSED) calls pass through and results are recorded in a sliding window. If ≥50% of the last 50 calls fail or are slow (and at least 20 calls were seen, `minimumNumberOfCalls`), it trips OPEN: for 30 s calls fail **immediately** without even trying (so you don't waste threads waiting on a dead service and the dependency gets breathing room), and the `fallbackMethod` answers instead. After 30 s it lets 5 test calls through (HALF_OPEN); if they succeed it closes again.
- *Retry* — repeats a call that failed from a **transient** fault (network blip). Exponential backoff means waits grow (200 ms, 400 ms, ...) to avoid hammering.
- *Time limiter* — gives up waiting after N seconds so a caller thread isn't tied up.
- *Bulkhead* — named after ship compartments: limits how many concurrent calls go to one dependency (20 above), so if "payments" hangs it can only occupy 20 threads, not all 200 of Tomcat's, and unrelated endpoints keep working.

```java
private CompletableFuture<Receipt> queueForLater(Payment p, Throwable t) {
    // called when the breaker is open or the call failed: degrade instead of erroring
    pendingQueue.add(p);
    return CompletableFuture.completedFuture(Receipt.pending(p.id()));
}
```

The fallback must have the **same signature plus a `Throwable`** parameter and live in the same class. **Pitfall:** the annotations work through AOP proxies, so self-invocation bypasses them; and `@Retry` on a non-idempotent call can double-charge.

**Senior points:**
- **Decorator order matters:** Retry wraps CircuitBreaker wraps TimeLimiter wraps Bulkhead by default — so retries are counted as one logical call by the breaker. Getting this backwards means retries hammer a dying service.
- **Never retry non-idempotent operations** without an idempotency key, and never retry a 4xx.
- Always add **jitter** to backoff or all clients retry in lockstep (retry storm).
- **Bulkhead is what prevents cascading failure**: one slow downstream shouldn't consume every thread. Separate thread pools/connection pools per dependency.
- **Timeouts are the most important resilience control** and the most commonly forgotten.

---

### Q64. Distributed transactions — Saga pattern.

2PC/XA doesn't scale and couples services. Use a **Saga**: a sequence of local transactions, each with a **compensating action**.

| Style | How | Trade-off |
|---|---|---|
| **Choreography** | Services react to each other's events | Decoupled, no central component; hard to see the overall flow |
| **Orchestration** | A saga orchestrator issues commands and tracks state | Explicit, observable, testable; the orchestrator is a new component |

Order example: `OrderCreated` → reserve inventory → authorize payment → ship. If payment fails, emit compensations: release inventory, cancel order. Requirements: **idempotent handlers**, **outbox for reliable publishing**, a **state machine** for the saga (or a framework like Temporal/Camunda/Axon), and business-level handling of "semantic locks" (e.g. order in PENDING state).

Say explicitly: "You trade ACID for **eventual consistency**, so the product/UX must tolerate a PENDING state. That's a product decision, not just a technical one."

**Why not 2PC:** in a monolith one DB transaction gives ACID (all-or-nothing). With database-per-service, one business action spans several databases. **Two-phase commit (2PC/XA)** makes a coordinator ask all participants "can you commit?" then "commit!" — it works, but locks resources across services while waiting, any participant being down blocks everyone, and many brokers/NoSQL stores don't support it. A saga instead commits each step locally right away and, if a later step fails, *undoes the earlier business effect* with a **compensating transaction** (a new action that semantically reverses the first — e.g. "refund payment", not a technical DB rollback).

**Walk-through (orchestration):**
1. Order-service saves order `PENDING` and sends `ReserveInventory`.
2. Inventory replies `Reserved` → orchestrator sends `AuthorizePayment`.
3. Payment replies `Declined` → orchestrator sends `ReleaseInventory` (compensation) and marks the order `CANCELLED`.
4. If everything succeeds, the order becomes `CONFIRMED`.

```java
// choreography flavour: each service reacts to events
@KafkaListener(topics = "payment.declined")
void on(PaymentDeclined e) { inventory.release(e.orderId()); }   // idempotent compensation
```

**Pitfalls:** compensations can themselves fail (so they must be retried and idempotent); steps are visible to others mid-way (the "PENDING" state is a **semantic lock** telling readers the order isn't final); not every action can be undone (an email already sent — you send a correction instead).

---

### Q65. How do services communicate — sync vs async?

- **Sync (REST/gRPC):** simple, immediate consistency of the read, but creates temporal coupling — if the downstream is down, you're down; latency adds up across the chain.
- **Async (Kafka/Rabbit):** decoupled, resilient, absorbs bursts, enables replay; but eventual consistency, harder debugging, message ordering and duplicate handling.
- **Rule of thumb:** commands the user is waiting for → sync. Everything else (notifications, projections, analytics, cross-service state propagation) → async events.
- Avoid **chatty chains** (A→B→C→D): each hop multiplies failure probability and latency. Use CQRS read models / data replication via events instead of calling three services to render one page.

**Why chains are risky — a number:** if each service is 99.9% available, four synchronous hops in series give about 0.999⁴ ≈ 99.6% — roughly 4× the downtime — and latencies add (4 × 100 ms = 400 ms). **Temporal coupling** means caller and callee must both be up *at the same moment*; with a queue in between, the producer succeeds even if the consumer is temporarily down, and the consumer catches up later.

**Example of the rule of thumb:** "Place order" — user waits for the payment result → synchronous call to payment-service (with timeout/circuit breaker). "Send confirmation email", "update analytics", "notify warehouse" → publish an `OrderPlaced` event; each interested service consumes it independently. **CQRS** (Command Query Responsibility Segregation) = separate the write model from a denormalised read model; e.g. a "order summary" service keeps its own copy of customer name and product titles (updated from events), so rendering a page doesn't need three calls. Cost: that copy can be briefly stale, and you must handle duplicate/out-of-order events.

---

### Q66. Config Server and dynamic refresh.

- `spring-cloud-config-server` backed by Git → services pull config at startup via `spring.config.import: configserver:http://config:8888`.
- `@RefreshScope` beans are re-created on `POST /actuator/refresh`; Spring Cloud Bus can broadcast the refresh to all instances.
- **In Kubernetes** most teams skip Config Server: ConfigMaps + Secrets mounted as env/files, and a rolling restart on change (immutable config is easier to reason about and audit).
- Always keep config **environment-agnostic in the artifact** — build once, deploy anywhere.

**What problem it solves:** with many services × many environments, copying `application-prod.yml` into each repo means changes (e.g. a new timeout) require editing and redeploying dozens of services. A Config Server centralises the files (in a Git repo → versioned, reviewable, auditable) and serves them over HTTP by app name + profile (`/order-service/prod`). At startup each service loads them via `spring.config.import`, with precedence over local files.

```yaml
# order-service application.yml
spring:
  application.name: order-service
  config.import: "optional:configserver:http://config:8888"   # optional → app still starts if server is down
  profiles.active: prod
```

**Refresh flow:** edit config in Git → call `POST /actuator/refresh` (or push a Bus event) → beans annotated `@RefreshScope` (and `@ConfigurationProperties` beans) are rebuilt with new values **without a restart**. **Pitfalls:** things created at startup (a Hikari pool, server port, Kafka consumer concurrency) don't change on refresh; the Config Server becomes a critical dependency (run it highly available, keep `optional:` / fail-fast policy deliberate); secrets in Git must be encrypted (`{cipher}...`) or kept in Vault.

---

## 14. Messaging (Kafka / RabbitMQ)

### Q67. Kafka essentials a Spring Boot dev must know.

- **Topic → partitions**; partition is the unit of parallelism and **ordering** (order guaranteed *within* a partition only). Key choice determines partition → to keep per-customer ordering, key by `customerId`.
- **Consumer group:** each partition is consumed by exactly one consumer in a group. **Max useful consumers = partitions.**
- **Offsets:** commit after processing (`enable.auto.commit=false` + `AckMode.MANUAL_IMMEDIATE` or `RECORD`/`BATCH`) — auto-commit can lose or duplicate messages.
- **Delivery semantics:** at-least-once is the practical default → **consumers must be idempotent** (dedupe by message ID / natural key / `INSERT ... ON CONFLICT DO NOTHING`).
- **Rebalance** happens when consumers join/leave or `max.poll.interval.ms` is exceeded → keep processing fast or increase the interval / reduce `max.poll.records`.
- **Retries & DLT:** `DefaultErrorHandler` with `ExponentialBackOff` + `DeadLetterPublishingRecoverer`, or `@RetryableTopic` for non-blocking retries on separate topics.

```java
@KafkaListener(topics = "orders", groupId = "billing", concurrency = "3")
public void onOrder(@Payload OrderEvent e,
                    @Header(KafkaHeaders.RECEIVED_KEY) String key,
                    Acknowledgment ack) {
    if (processed.contains(e.eventId())) { ack.acknowledge(); return; }  // idempotency
    billingService.bill(e);
    ack.acknowledge();
}

@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<String,Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));
    var handler = new DefaultErrorHandler(recoverer, new ExponentialBackOffWithMaxRetries(3));
    handler.addNotRetryableExceptions(ValidationException.class);   // poison pill → straight to DLT
    return handler;
}
```

**Producer reliability:** `acks=all`, `enable.idempotence=true`, `retries=Integer.MAX_VALUE`, `max.in.flight.requests.per.connection<=5`, `min.insync.replicas=2`.

**Kafka in plain language:** Kafka is a durable, append-only **log** of messages split into **topics**. Each topic is cut into **partitions** (separate ordered logs spread across brokers) so many consumers can read in parallel. Every message in a partition has a sequential **offset** (its position). A **consumer group** is a team of consumers sharing a topic: Kafka assigns each partition to exactly one member, so with 6 partitions and 3 consumers each gets 2; a 7th consumer would sit idle (hence "max useful consumers = partitions"). Two different groups (billing, analytics) each get **every** message independently — this is the fan-out.

**Why ordering needs a key:** messages with the same key hash to the same partition, so all events for `customerId=42` stay in order; messages for different customers may interleave, and that's fine.

**Why manual offset commits:** the offset is the consumer's bookmark. If auto-commit saves the bookmark *before* your processing finishes and you crash, that message is lost; if you process and crash *before* committing, it's redelivered (duplicate). Committing after processing gives **at-least-once** — duplicates are possible, so the idempotency check in the code above is required.

**Rebalance:** when membership changes Kafka reassigns partitions; consumption pauses briefly. If your handler takes longer than `max.poll.interval.ms` (default 5 min) between polls, the broker assumes you're dead and kicks you out — leading to endless rebalances.

**DLT (dead-letter topic):** after N failed retries, a "poison pill" (a message that always fails, e.g. malformed JSON) is parked in `orders.DLT` so it doesn't block the partition, and you inspect/replay it later. **Producer settings:** `acks=all` waits for all in-sync replicas, `enable.idempotence` stops retries from creating duplicates, `min.insync.replicas=2` means a write needs at least two copies — together these survive a broker failure without losing acknowledged messages.

---

### Q68. Kafka vs RabbitMQ — which and why?

| | Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed **log**, consumers track offsets, messages retained | **Broker/queue**, message removed on ack |
| Strengths | Huge throughput, replay, event sourcing, stream processing, fan-out to many consumer groups | Flexible routing (exchanges: direct/topic/fanout/headers), per-message TTL/priority, delayed messages, simple RPC |
| Ordering | Per partition | Per queue (single consumer) |
| Use when | Event streaming, audit logs, analytics, multiple independent consumers, replay needed | Task queues, work distribution, complex routing, low-volume commands |

**Handling poison messages / DLQ in Rabbit:** `x-dead-letter-exchange`, retry queue with TTL for backoff, max-retry header counter.

**Understanding the difference (analogy):** Kafka is like a **newspaper archive** — every issue is kept for a retention period (days/weeks) and each reader keeps their own bookmark, so a new reader can start from the beginning or re-read yesterday. RabbitMQ is like a **post office** — a message is delivered to a queue, handed to one worker, and thrown away once acknowledged. Consequences: with Kafka you can *replay* (fix a bug, then reprocess last week's events; build a new consumer from history), while in Rabbit a consumed message is gone. Rabbit's **exchange** decides routing: `direct` = exact routing key, `topic` = wildcard patterns (`order.*.created`), `fanout` = copy to every bound queue, `headers` = match on headers.

```java
@RabbitListener(queues = "email.queue")
void send(EmailJob job) { mailer.send(job); }     // ack'd automatically on return; exception → requeue/DLQ per config
```

**Choosing:** "A thousand independent services need the same stream of order events, and analytics must replay history" → Kafka. "Distribute 50k email-sending jobs among workers, with priorities and delayed retries" → RabbitMQ. **Probe:** *"Why can't Kafka scale consumers beyond partitions?"* — one partition is read by one consumer in a group to preserve order.

---

### Q69. How do you guarantee exactly-once-ish processing?

True exactly-once end-to-end across a DB and a broker doesn't exist without transactions on both sides. Practical recipe:
1. **At-least-once delivery** from the broker.
2. **Idempotent consumers** — dedupe table keyed by `eventId` with a unique constraint, or make the operation naturally idempotent (`SET status='PAID'` rather than `balance = balance - x`).
3. **Transactional outbox** on the producer side so DB write + event publish can't diverge.
4. Kafka transactions (`transactional.id`, `read_committed`) for Kafka→Kafka pipelines.

**One-liner:** *"At-least-once delivery + idempotent consumer = effectively exactly-once."*

**Why true exactly-once is impossible across systems:** the consumer processes a message, writes to the DB, then must tell the broker "done". If it crashes *between* the DB write and the ack, the broker can't know, so it redelivers → duplicate. You can't make the DB write and the ack one atomic action (they're two systems), so you accept at-least-once and make duplicates harmless.

```java
@Transactional
void handle(PaymentEvent e) {
    try {
        processedEvents.save(new ProcessedEvent(e.eventId()));     // unique(event_id) — inside the SAME tx as the business change
    } catch (DataIntegrityViolationException dup) { return; }      // seen before → skip silently
    ledger.credit(e.accountId(), e.amount());
}
```

Because the dedupe row and the business update commit atomically, a redelivery either finds the row (skip) or the whole thing rolled back (retry cleanly). **Naturally idempotent vs not:** `status = 'PAID'` can be applied 5 times with the same result; `balance = balance - 100` cannot. **Outbox on the producer side** solves the mirror problem: "DB commit succeeded but the publish to Kafka failed" — write the event to a table in the same transaction and let a relay publish it. **Kafka transactions** (`transactional.id`, consumers on `read_committed`) make "consume → process → produce" atomic *within Kafka only*.

---

## 15. Performance Tuning & JVM

### Q70. HikariCP tuning — what matters?

```yaml
spring.datasource.hikari:
  maximum-pool-size: 20        # start from cores & DB limits, NOT "bigger is better"
  minimum-idle: 20             # = max for steady latency (avoids creation spikes)
  connection-timeout: 3000     # fail fast rather than queue forever
  idle-timeout: 600000
  max-lifetime: 1800000        # must be < DB/proxy idle kill time
  leak-detection-threshold: 20000
  validation-timeout: 3000
```
**Key insight (say this):** a *smaller* pool is often faster. The formula from HikariCP's own docs is roughly `connections ≈ (core_count × 2) + effective_spindle_count`. Too many connections cause DB context switching and lock contention. Also: `maximumPoolSize × instances` must stay below the DB's `max_connections` — with 20 pods × 20 connections you've asked Postgres for 400 connections. Add PgBouncer if you need more app instances.

**Pool exhaustion causes to name:** connections held across external HTTP calls inside `@Transactional`, `open-in-view` holding connections for the whole request, `REQUIRES_NEW` nesting, streaming a huge result set, missing timeouts, and long-running scheduled jobs.

**What a connection pool is and why:** opening a DB connection (TCP + auth + session setup) costs tens of milliseconds, so apps open a fixed set once and *lend* them to requests (`getConnection()` = borrow, `close()` = give back, not a real close). If all connections are lent out, the next request **waits** up to `connection-timeout` and then fails with "Connection is not available, request timed out after 3000ms" — the classic symptom of exhaustion.

**Why bigger isn't better:** the DB has finite CPU/disk; 200 simultaneous queries just interleave and each gets slower (context switching, lock contention), while 20 queries finishing fast and the rest queuing in the pool gives better overall throughput. Math example: 20 pods × `maximum-pool-size: 20` = 400 connections; Postgres's default `max_connections` is 100 — pods would fail to start. **PgBouncer** is a lightweight connection multiplexer in front of the DB that lets thousands of app connections share a few real ones.

**Setting meanings:** `max-lifetime` recycles connections before a firewall/DB silently kills idle ones (keep it a few minutes shorter than that limit); `leak-detection-threshold` logs a stack trace if a connection is held longer than 20 s — pointing at the code that forgot to release it or kept it during slow work. **Example leak:**

```java
@Transactional                       // connection borrowed here
public void process(Order o) {
    repo.save(o);
    restClient.post(...);            // 5s external call while still holding the connection
}                                    // → under load, pool empties while threads just wait on HTTP
```

Fix: do the external call before/after the transaction (or after commit via an event).

---

### Q71. How do you debug high latency in a Spring Boot service in production?

**Structured answer (very interviewable):**
1. **Confirm and scope** — which endpoint, which percentile, since when? Check dashboards: p99 latency, error rate, RPS.
2. **Layer isolation** — is it app, DB, downstream, or infra? Traces show where the time goes (`span` breakdown).
3. **Common culprits:**
   - DB: missing index, N+1, lock waits, plan regression after data growth → `EXPLAIN ANALYZE`, `pg_stat_statements`, slow query log.
   - Pool saturation → `hikaricp_connections_pending` metric.
   - GC pauses → GC logs, `jstat`; long pauses mean heap/allocation problems.
   - Thread pool saturation → `/actuator/threaddump`, Tomcat `threads.busy` vs `threads.max`.
   - A slow downstream without a timeout.
   - Cache hit-rate drop after deploy (key change).
   - Noisy neighbour / CPU throttling in K8s (`container_cpu_cfs_throttled_seconds`).
4. **Tools:** async-profiler / JFR flame graphs for CPU and allocation, Micrometer metrics, `arthas` for live tracing.
5. **Fix + prevent** — add a regression test, an alert, and a runbook entry.

**Terms and quick commands:** *p99* = the latency that 99% of requests are faster than (the slowest 1% are above it) — averages hide these. *GC pause* = the JVM stopping application threads to reclaim memory; frequent/long pauses show up as latency spikes. *Thread pool saturation* = all request threads busy, so new requests wait in a queue. A **flame graph** (from async-profiler/JFR) is a chart where wider bars mean more CPU time spent in that method — you read it top-down to find the hot method. `EXPLAIN ANALYZE` runs a SQL query and shows the planner's actual steps (a `Seq Scan` over millions of rows instead of an `Index Scan` is the classic smoking gun).

```bash
curl -s localhost:9090/actuator/metrics/hikaricp.connections.pending      # are requests waiting for a DB connection?
curl -s localhost:9090/actuator/threaddump | jq '.threads[] | select(.threadState=="BLOCKED") | .threadName'
jcmd <pid> JFR.start duration=60s filename=app.jfr                          # 60 s CPU/allocation recording
```

**Order of thinking:** start broad and cheap (dashboards, traces), narrow to a layer, then use heavier tools (profilers, dumps) only once you know where to look. Say that "traces first" is faster than guessing.

Then give a concrete story: "We had p99 spike from 200 ms to 4 s. The trace showed 90% in one repository call; a table had grown past the point where the planner chose a sequential scan. A composite index on `(tenant_id, status, created_at)` brought it back to 120 ms, and we added a query-count assertion test."

---

### Q72. How do you diagnose a memory leak / OOM?

- Enable `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps` **before** you need it; also `/actuator/heapdump`.
- Analyse with **Eclipse MAT** (dominator tree, leak suspects) or VisualVM/JFR.
- Classic Spring-app leaks: unbounded caches/`Map` fields, `ThreadLocal`s never cleared with pooled threads, listeners/`@EventListener` registrations never removed, `ClassLoader` leaks on redeploy, holding entities in a long-lived persistence context, unbounded queues in a `ThreadPoolTaskExecutor`, streaming large results into a `List`.
- In containers: use `-XX:MaxRAMPercentage=75` (not fixed `-Xmx`) so the JVM respects cgroup limits, and remember the JVM needs headroom beyond heap (metaspace, threads, direct buffers, code cache) or the **kernel OOM-kills the pod** with no heap dump.

**Useful flags:** `-XX:+UseG1GC` (default), `-Xss`, `-XX:MaxMetaspaceSize`, `-XX:+ExitOnOutOfMemoryError` (let the orchestrator restart a poisoned JVM).

**What a leak means in Java:** the GC only frees objects that are *unreachable*. A "leak" is memory you no longer need but something still references — so heap grows slowly until `OutOfMemoryError`. Typical symptom: the heap sawtooth (rise, GC drop) keeps a rising *floor* after each full GC, and GC runs more often until the app spends most time collecting.

**Example (looks innocent):**

```java
@Service
class PriceService {
    private final Map<String, Price> cache = new HashMap<>();     // never evicted, one entry per SKU/user/request id...
    Price get(String key) { return cache.computeIfAbsent(key, this::load); }
}
```
Fix: bounded cache with expiry (Caffeine `maximumSize` + `expireAfterWrite`).

**Investigation steps:** (1) watch heap-after-GC in metrics (`jvm.memory.used` / `jvm.gc.live.data.size`); (2) take a heap dump (before it dies, or auto on OOM); (3) in Eclipse MAT open the **Leak Suspects** report — the *dominator tree* shows which object keeps the most memory alive (e.g. one `HashMap` holding 80%); (4) follow the *path to GC roots* to find who references it. **Container note:** the pod can be OOM-killed (exit code 137) even if heap isn't full — heap is only part of the process memory; native/direct buffers, thread stacks (each ~1 MB) and metaspace live outside it, so use `MaxRAMPercentage=75` and leave headroom.

---

### Q73. How do you reduce Spring Boot startup time?

- Trim dependencies (each starter adds auto-config evaluation); `--debug` report shows the cost.
- `spring.main.lazy-initialization=true` (dev only — moves failures to first request).
- `@Configuration(proxyBeanMethods = false)`; avoid component-scanning huge packages; avoid classpath scanning of JARs.
- Exclude unused auto-configurations explicitly.
- Boot 3: **AOT processing** + **CDS/AppCDS**, or **GraalVM native image** (~50–100 ms startup, low memory) for serverless/scale-to-zero.
- Boot 3.2+ **Checkpoint/Restore (CRaC)** for near-instant restore.
- Avoid heavy work in `@PostConstruct`; defer to `ApplicationReadyEvent` and warm asynchronously.

**Why it matters:** faster HPA scale-out, faster rollbacks, cheaper serverless cold starts, faster CI.

**Where startup time goes and what each tip does:** on startup Spring scans classes, reads annotations, evaluates hundreds of `@Conditional`s, creates bean definitions, builds proxies via reflection/CGLIB, and the JVM itself loads and JIT-compiles classes. So:
- *Fewer starters/auto-configs* = fewer conditions evaluated and fewer beans built.
- *Lazy initialization* postpones bean creation until first use: faster startup, but the first request pays the cost and wiring errors show up at runtime — fine for local dev, risky in prod (use `@Lazy` on specific heavy beans instead).
- *AOT (Ahead-Of-Time) processing* does the annotation/condition analysis at **build time** and generates plain Java config code, so runtime does no scanning. *CDS/AppCDS* (Class Data Sharing) stores pre-parsed class metadata in an archive so the JVM loads classes faster. *GraalVM native image* compiles everything ahead of time into a native executable — starts in tens of milliseconds, uses far less memory, but builds slowly, has no JIT (lower peak throughput), and reflection/dynamic proxies need hints. *CRaC* (Coordinated Restore at Checkpoint) snapshots a warmed-up JVM and restores it, skipping both startup and warm-up.
- *`@PostConstruct` work* blocks the whole startup path — load caches in the background after `ApplicationReadyEvent`.

**Measure first:** run with `--debug`/`-Dspring.main.log-startup-info`, use the `/actuator/startup` endpoint (with `BufferingApplicationStartup`) to list the slowest beans — don't optimize blindly.

---

## 16. Spring Boot 3.x, Virtual Threads & Native Image

### Q74. What changed in Spring Boot 3?

- **Java 17 baseline** (3.2+ supports 21; Boot 3.5/Framework 6.2 era targets 17–25).
- **`javax.*` → `jakarta.*`** — the single biggest migration cost (`javax.persistence` → `jakarta.persistence`, servlet, validation). Use the OpenRewrite recipe or Migration Advisor.
- Spring Framework 6, Hibernate 6 (new query parser — some HQL/criteria behaviour and generated SQL changed).
- **Observability built in:** Micrometer Observation API + Micrometer Tracing (Sleuth removed).
- **RFC 7807 `ProblemDetail`** first-class.
- **GraalVM native image support** as a first-class citizen with AOT processing.
- `RestClient` (3.2), **virtual threads** support (3.2), `@ServiceConnection` for Testcontainers (3.1), Docker Compose support at dev time (3.1), `spring-boot-docker-compose`.
- Security 6: `WebSecurityConfigurerAdapter` removed; lambda DSL only; `requestMatchers` replaces `antMatchers`.
- Actuator: `/actuator` paths and property names tightened; `management.endpoints.web.exposure.include` still required.

**Explained — the items that cause real work:**
- **Java 17 baseline:** Boot 3 won't run on Java 8/11; you must upgrade the JDK and the build/CI images first. In exchange you get records, sealed classes, pattern matching, text blocks, and (on 21) virtual threads.
- **`javax` → `jakarta`:** Oracle transferred Java EE to the Eclipse Foundation, which was not allowed to keep the `javax` namespace; Jakarta EE 9+ renamed packages. Every import and every library using the old names breaks (`javax.servlet.http.HttpServletRequest` → `jakarta.servlet.http.HttpServletRequest`), including third-party libs that haven't released a Jakarta-compatible version. Typical migration path: first upgrade to the latest 2.7.x, fix deprecations, then run the OpenRewrite recipe `UpgradeSpringBoot_3_x` and compile.
- **Hibernate 6:** a rewritten query engine — stricter HQL, different SQL generation, changed ID generation defaults (`SEQUENCE` naming `<entity>_seq`), changed handling of some types. Always run the full repository test-suite (with Testcontainers) after upgrading.
- **Observation API:** one instrumentation API (`Observation`) that produces both metrics and traces; replaces Sleuth, which is why `spring-cloud-starter-sleuth` must be removed.
- **Security 6:** `antMatchers/mvcMatchers` → `requestMatchers`; `authorizeRequests` → `authorizeHttpRequests`; `@EnableGlobalMethodSecurity` → `@EnableMethodSecurity`; configuration is now a `SecurityFilterChain` bean (see Q50).
- **Docker Compose support (3.1):** with `spring-boot-docker-compose` on the classpath, `./mvnw spring-boot:run` starts services from your `compose.yaml` (Postgres, Kafka) and auto-configures connection details.

**Migration advice to give:** "Upgrade in two hops — latest 2.7 first, then 3.x; use `spring-boot-properties-migrator` to detect renamed properties; and run on a branch with the full integration suite." Boot 2.7 is out of OSS support, so staying is a security risk.

---

### Q75. Virtual threads (Project Loom) — what do they change for Spring Boot?

```yaml
spring.threads.virtual.enabled: true   # Boot 3.2+, needs Java 21
```
This makes Tomcat use a virtual-thread-per-request executor and `@Async`/`@Scheduled` use virtual threads.

**What it fixes:** the "thread-per-request doesn't scale" problem. Platform threads cost ~1 MB stack, so 200 threads is a normal Tomcat max; blocking I/O parks a real thread. Virtual threads are cheap (heap-allocated continuations), so you can have hundreds of thousands, and blocking a virtual thread *unmounts* it from its carrier thread.

**Result:** you get most of the scalability of WebFlux while keeping simple, debuggable, blocking, imperative code — and stack traces that make sense.

**Caveats to mention (this is what seniors get asked):**
- **`synchronized` blocks pin** the virtual thread to its carrier in early versions (JDK 21) — prefer `ReentrantLock` in hot paths. Improved in later JDKs (JEP 491).
- Thread pools become an anti-pattern for virtual threads; but you still need **semaphores/bulkheads** for backpressure — otherwise unbounded concurrency just moves the bottleneck to the DB connection pool.
- `ThreadLocal` still works but is expensive at scale; prefer `ScopedValue`.
- Don't use virtual threads for CPU-bound work.
- Connection pools and downstream limits, not threads, become your real ceiling — so size them deliberately.

**Concept in plain language:** a normal ("platform") Java thread is a thin wrapper over an OS thread: heavy (~1 MB stack) and limited to a few thousand. In a classic request, the thread spends most time *waiting* (for the DB, an HTTP call) yet is unavailable for other work. A **virtual thread** is a lightweight thread managed by the JVM: when it blocks on I/O, the JVM parks it, **unmounts** it from the small set of real *carrier* threads (about one per CPU core), and lets another virtual thread run there. When the I/O completes, it is mounted again and continues. You write ordinary blocking code, but the waiting no longer wastes an OS thread.

```java
// same blocking code as always — only the thread type changes
@GetMapping("/orders/{id}")
OrderView get(@PathVariable long id) {
    var order = orderRepo.findById(id).orElseThrow();      // blocks → carrier is freed meanwhile
    var rate  = fxClient.rate(order.currency());           // blocks → same
    return OrderView.of(order, rate);
}
```

**Pinning explained:** if a virtual thread blocks while inside a `synchronized` block (or a native call), it can't unmount, so it holds its carrier thread — with enough of these, throughput collapses back to "a few threads". That's why hot paths prefer `ReentrantLock` on JDK 21 (fixed for `synchronized` in JDK 24). **Why you still limit concurrency:** 10,000 concurrent requests can now run, but the DB pool has 20 connections — 9,980 virtual threads just wait on the pool; protect downstreams with a `Semaphore`/bulkhead and sensible timeouts. **Quick comparison:** virtual threads = "blocking style, scalable", WebFlux = "non-blocking style, scalable, steeper learning curve".

---

### Q76. GraalVM native image — trade-offs.

`mvn -Pnative native:compile` or `spring-boot:build-image`.
- **Pros:** ~50–100 ms startup, 2–5× lower memory, small attack surface — ideal for serverless, CLI, high-density deployments.
- **Cons:** closed-world assumption means **no runtime reflection/dynamic proxies/dynamic class loading** unless registered (Spring's AOT engine generates most hints; third-party libs may need `@RegisterReflectionForBinding` or `reflect-config.json`). Long build times (minutes), no JIT peak throughput, harder profiling/debugging, **profiles are largely fixed at build time**, and conditional beans are resolved during AOT.
- **Verdict to give:** "Great for functions and short-lived workloads; for long-running high-throughput services, JIT-compiled JVM usually wins on peak throughput, and I'd reach for CDS/CRaC before native."

**What "native image" is:** normally Java ships bytecode that the JVM interprets and then *JIT*-compiles (Just-In-Time, at runtime, based on observed hot spots). GraalVM's `native-image` compiles your app **ahead of time** into a standalone OS executable that contains only the code it can prove is reachable — no JVM warm-up, tiny memory, instant start. The price is the **closed-world assumption**: the compiler must know every class at build time. Reflection, dynamic proxies, resource loading and serialization are determined at runtime, so they must be declared as *hints*; Spring's AOT step analyses your beans and generates those hints and replaces runtime reflection with generated code.

```java
@RegisterReflectionForBinding(PartnerPayload.class)   // JSON (de)serialization of this type uses reflection
@Configuration class NativeHints { }
```

**What breaks in practice:** a library using reflection without published hints → works on the JVM, fails with `ClassNotFoundException`/missing-method at runtime in native; `@ConditionalOnProperty` and `@Profile` are evaluated when AOT runs, so changing them at runtime has no effect; a 5–10 minute CI build. **When to pick it:** AWS Lambda/serverless with cold starts, CLIs, scale-to-zero, many tiny sidecar-like services. **Probe:** *"Native vs virtual threads vs CRaC?"* — native optimises startup/memory, virtual threads optimise concurrency, CRaC keeps the JIT'd JVM and removes warm-up.

---

## 17. Spring WebFlux vs Spring MVC

### Q77. Compare them and say when you'd choose each.

| | Spring MVC | Spring WebFlux |
|---|---|---|
| Model | Thread-per-request, blocking | Event loop, non-blocking, reactive streams |
| Server | Tomcat (Servlet) | Netty (default), also Tomcat/Undertow |
| Return types | `T`, `ResponseEntity<T>` | `Mono<T>`, `Flux<T>` |
| Data access | JPA/JDBC (blocking) | R2DBC, reactive Mongo/Redis/Cassandra |
| Backpressure | No | Yes (Reactive Streams) |
| Debuggability | Easy stack traces | Hard — assembly vs subscription time |

**Choose WebFlux when:** very high concurrency with mostly I/O wait, streaming (SSE, chat, telemetry), an API gateway fanning out to many services, or you need backpressure.
**Choose MVC when:** relational DB with JPA, a team unfamiliar with reactive, blocking libraries in the stack — and note that **with virtual threads (Boot 3.2 + Java 21), MVC now scales well enough that WebFlux's main remaining edge is backpressure and streaming semantics.** That is exactly the answer a 9-YOE candidate should give in 2025+.

**Fatal mistake to call out:** one blocking JDBC call inside a WebFlux handler blocks an event-loop thread and can stall the entire application (there are only ~cores event-loop threads). If you must, wrap it: `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`.

Key operators: `map` vs `flatMap` (async), `zip` (parallel combine), `switchIfEmpty`, `onErrorResume`, `retryWhen(Retry.backoff(...))`, `timeout`, `subscribeOn`/`publishOn`. `Mono`/`Flux` are **lazy** — nothing happens until subscribed (the classic "my code didn't run" bug).

**What "reactive" means:** `Mono<T>` is a promise of 0 or 1 value, `Flux<T>` of 0..N values that arrive over time. Instead of a thread waiting for the DB, you *describe a pipeline* ("when the row arrives, map it to a DTO, then call service B, then respond") and a small fixed set of **event-loop** threads runs the steps whenever data is ready — one thread can juggle thousands of in-flight requests because no step blocks. **Backpressure** = the consumer tells the producer how many items it can handle (`request(n)`), so a fast source can't flood a slow consumer.

```java
@GetMapping("/orders/{id}")
Mono<OrderView> get(@PathVariable long id) {
    return orderRepo.findById(id)                                  // Mono<Order> (R2DBC)
        .flatMap(o -> fxClient.rate(o.currency())                  // async call returning Mono<Rate>
                              .map(r -> OrderView.of(o, r)))
        .switchIfEmpty(Mono.error(new ResourceNotFoundException("no order")))
        .timeout(Duration.ofSeconds(2));
}   // nothing runs until the framework subscribes to the returned Mono
```

`map` transforms a value synchronously; `flatMap` is used when the transformation itself returns a `Mono`/`Flux` (an async call). **Why debugging is hard:** stack traces show the event-loop internals, not the chain you wrote, because assembly (building the pipeline) and execution (subscribing) happen at different times; `Hooks.onOperatorDebug()` / `checkpoint()` help. **Why a JPA app should stay on MVC:** JPA/JDBC are blocking, so a reactive controller on top of them gives no benefit — you'd need R2DBC end-to-end.

---

## 18. Scenario-Based Questions

> These are where 9-YOE interviews are won or lost. Structure every answer as
> **① Clarify → ② Diagnose/Options → ③ Decide with trade-offs → ④ Prevent recurrence.**
> Say the trade-off out loud; a senior who names the downside of their own choice sounds far stronger than one who doesn't.

### S1. "Your API's p99 latency jumped from 200 ms to 3 s after last night's deploy. Walk me through it."

1. **Stabilise first:** is it all endpoints or one? If it's a clear regression and traffic is impacted, **roll back first, investigate after** — MTTR beats curiosity.
2. **Compare the deploy diff:** new query, new dependency call, changed cache key, changed JPA fetch mapping, a new `@Transactional` boundary, config change (pool size, timeouts).
3. **Data:** traces (where does the time go), `hikaricp_connections_pending` (pool starvation), GC pause metrics, DB slow-query log, downstream latency.
4. **Most common real cause in a Spring Boot deploy:** someone added a lazy association access or removed a `JOIN FETCH`, creating N+1; or a cache key changed so hit rate went from 95% to 0.
5. **Prevent:** query-count assertion tests, canary deploy with automatic rollback on SLO burn, latency alerts on p99 not average.

---

### S2. "Under load, your service throws `Unable to acquire JDBC Connection` / `HikariPool-1 - Connection is not available, request timed out`."

- **Immediate meaning:** all pool connections are checked out longer than `connection-timeout`. It's a *holding-time* problem far more often than a *pool-size* problem.
- **Root causes to check, in order:**
  1. An **HTTP/external call inside a `@Transactional` method** — a 5 s downstream call holds a DB connection for 5 s.
  2. `spring.jpa.open-in-view=true` holding a connection for the whole request including view rendering.
  3. `REQUIRES_NEW` nesting — each level takes another connection.
  4. Missing statement/lock timeouts → a blocked query holds forever.
  5. A leak (`leak-detection-threshold` will log the offending stack trace).
  6. Genuinely more concurrency than the pool + DB can serve.
- **Fix:** move remote calls **outside** the transaction; shrink transaction scope; set `open-in-view=false`; add `statement_timeout`/lock timeouts; right-size pool against DB `max_connections`; add PgBouncer if instance count is high; add a bulkhead/rate limit so overload sheds instead of queueing.
- **Prevent:** an ArchUnit/Sonar rule flagging remote clients called from `@Transactional` code, plus a dashboard alert on `hikaricp_connections_pending > 0` for 1 min.

---

### S3. "Two users update the same order simultaneously; the second overwrites the first (lost update). Fix it."

- Add `@Version` for **optimistic locking**. The losing update throws `ObjectOptimisticLockingFailureException`.
- Then decide the UX: return **409 Conflict** with the current state so the client can merge/re-submit, or auto-retry with `@Retryable(retryFor = OptimisticLockingFailureException.class, backoff = @Backoff(100))` if the operation is naturally re-runnable.
- For HTTP APIs, expose the version as an **ETag** and require `If-Match` — gives you conditional requests for free.
- Use pessimistic (`SELECT FOR UPDATE`) only for short, high-contention critical sections; use an atomic `UPDATE ... WHERE` when the operation is a simple delta.
- **Trade-off statement:** "Optimistic scales better and never deadlocks but pushes conflict handling to the caller; pessimistic is simpler for the caller but serializes and risks lock waits."

---

### S4. "You have 3 pods and a `@Scheduled` job that sends invoices. Customers got 3 invoices each. Fix it."

- Root cause: every instance runs the schedule independently.
- **Fix:** distributed lock. **ShedLock** is the pragmatic answer:
```java
@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "invoiceJob", lockAtMostFor = "30m", lockAtLeastFor = "5m")
public void sendInvoices() { ... }
```
- Alternatives: Quartz clustered, a Kubernetes `CronJob`, or leader election.
- **Also fix the deeper issue:** make it idempotent (mark `invoice_sent_at` in the same transaction / unique constraint on `(order_id, period)`) so even a double-run can't double-send. **Locks reduce the probability; idempotency removes the consequence.** Say both.
- Add monitoring: a gauge for "minutes since last successful run" and alert if it exceeds the expected interval.

---

### S5. "A downstream payment service starts responding in 30 s. Your whole app becomes unresponsive. Why, and how do you prevent it?"

- **Why:** thread-per-request + no timeout → every Tomcat worker (default max 200) ends up blocked on that call. Now even `/health` and unrelated endpoints queue. This is **cascading failure**.
- **Fix stack (name all five):**
  1. **Timeouts** — connect + read, tighter than your own SLA (e.g. 2 s).
  2. **Circuit breaker** — fail fast when the error/slow-call rate is high, so you stop feeding a dying service.
  3. **Bulkhead** — cap concurrent calls to that dependency (e.g. 20) so it can never consume all threads.
  4. **Fallback / graceful degradation** — queue the payment for async processing, or return a "pending" status.
  5. **Retry with backoff + jitter**, only for idempotent calls, and *inside* the breaker.
- Add virtual threads or async if you must hold many in-flight calls, but stress that **timeouts and bulkheads are the actual fix** — more threads just delays the collapse.

---

### S6. "Your Kafka consumer processed the same message twice and charged a customer twice."

- Kafka gives **at-least-once**; duplicates are expected (rebalance before commit, retry after partial failure, producer retry).
- **Fix:** idempotent consumer.
  - Dedupe table: `PRIMARY KEY (event_id)`; insert first, and if `DataIntegrityViolationException` → already processed, ack and skip.
  - Or make the write naturally idempotent: `UPDATE payment SET status='CHARGED' WHERE id=? AND status='PENDING'` and act on the affected-row count.
  - Or pass an idempotency key to the payment provider so *they* dedupe.
- Also: commit offsets **after** the DB transaction commits, use manual ack, and don't do the charge and the offset commit in an order that can lose either.
- **Prevent:** every consumer in the estate carries an `eventId` header, and the dedupe check is in a shared library/interceptor rather than each handler.

---

### S7. "You must save an order to Postgres and publish an `OrderPlaced` event to Kafka atomically. No 2PC. How?"

**Transactional Outbox:**
1. In one local DB transaction: insert the order **and** insert a row into `outbox(id, aggregate_id, type, payload, created_at, published_at)`.
2. A separate publisher — a polling job (`SELECT ... WHERE published_at IS NULL FOR UPDATE SKIP LOCKED LIMIT 100`) or **Debezium CDC** reading the WAL — publishes to Kafka and marks it published.
3. Consumers are idempotent because the publisher is at-least-once.

Mention the cheaper alternative and its weakness: `@TransactionalEventListener(AFTER_COMMIT)` publishing to Kafka is simple but loses the event if the process dies right after commit. Outbox trades a little latency and a table for durability. `SKIP LOCKED` is what makes the poller safely multi-instance.

**Why this works:** the real problem is two systems (Postgres, Kafka) that can't share one transaction, so "save then publish" can fail halfway (order saved, event never sent — or event sent for an order that rolled back). The outbox turns the *publish* into just another row written in the **same** local transaction as the order, so both are atomic. Publishing then becomes a separate, retryable step that can be repeated until it succeeds. `SKIP LOCKED` makes each poller instance skip rows already locked by another, so several instances can drain the table in parallel without double-picking.

```java
@Transactional
public void place(Order o) {
    orders.save(o);
    outbox.save(new OutboxEvent(o.getId(), "OrderPlaced", json.write(new OrderPlaced(o.getId()))));
}   // both rows commit together or not at all

@Scheduled(fixedDelay = 500)
@Transactional
void relay() {
    for (OutboxEvent e : outbox.lockNextBatch(100)) {       // SELECT ... FOR UPDATE SKIP LOCKED
        kafka.send("orders", e.aggregateId(), e.payload()).join();   // wait for broker ack
        e.markPublished();
    }
}
```

Crash between `send` and `markPublished` → the event is sent again next time (duplicate) — which is why consumers must be idempotent. **Probe:** *"Polling vs Debezium?"* — polling is simple but adds DB load/latency; CDC (Change Data Capture) tails the database's transaction log, giving lower latency without queries, at the cost of running Kafka Connect/Debezium.

---

### S8. "Requirement: an endpoint must respond in 200 ms but it aggregates data from 4 services, each taking 150 ms."

- Sequential = 600 ms. **Parallelise** with `CompletableFuture.allOf` / `Mono.zip` on a bounded pool → ~150–200 ms.
- Add `orTimeout` per call and **degrade gracefully**: return the core object with optional sections omitted plus a `partial: true` flag, rather than a 500.
- If it still doesn't fit: **pre-aggregate**. Build a **read model / materialized view** updated by events (CQRS), or cache the composite response with a short TTL. Then the endpoint is a single lookup.
- If two of the four calls are for rarely-used data, make them lazy/on-demand from the client.
- **Trade-off:** the read model is eventually consistent and is extra infrastructure to maintain; parallel fan-out keeps consistency but multiplies load on downstreams (4× RPS) and your latency is bounded by the slowest, so their p99 becomes your p99.

---

### S9. "After enabling `@Transactional` on a service method, you see `LazyInitializationException` in the controller. Explain and fix."

The transaction (and persistence context) ends when the service method returns, so the lazy proxy in the controller has no session. Fix by **mapping to a DTO inside the service** while the session is open, or `@EntityGraph`/`JOIN FETCH` to load what's needed. Set `spring.jpa.open-in-view=false` so this fails early in dev instead of being masked in prod. Emphasise the architectural rule: **entities never leave the service layer.**

**Why it appears right *after* adding `@Transactional`:** before, with open-in-view on (the default), the session lived for the whole HTTP request, so the controller could lazily load anything. Adding a transactional boundary (together with `open-in-view=false`, or a different entry point such as a Kafka listener that has no OSIV) makes the session end with the method. The lazy field is a placeholder that needs the session to fetch itself — outside, there's nothing to ask. Example:

```java
@Transactional(readOnly = true)
public OrderDto get(long id) {
    Order o = repo.findWithItemsById(id).orElseThrow();   // @EntityGraph("items") — loaded inside tx
    return mapper.toDto(o);                              // plain object; safe to serialize anywhere
}
```

Avoid the quick-and-dirty "fixes": making the association `EAGER` (every query now pays), or `Hibernate.initialize()` sprinkled in controllers (leaks persistence concerns into the web layer).

---

### S10. "Someone added `@Transactional` to a private method / calls it from within the same class and it isn't working."

Spring's proxy only intercepts external calls to public methods. Demonstrate the three fixes: extract to a separate bean (preferred, best design), self-injection via `@Lazy` or `ObjectProvider`, or use `TransactionTemplate` programmatically. Same explanation applies to `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize` — mention that generalisation, it shows you understand the mechanism rather than memorising a rule.

**Visualising why:** other beans hold a reference to the **proxy**, not your object. A call from a controller goes `controller → proxy (starts tx) → real service`. But inside the real service, `this` is the *raw* object, so `this.save()` jumps straight to the method, skipping the proxy and thus the transaction advice.

```java
@Service
class InvoiceService {
    private final TransactionTemplate tx;
    InvoiceService(PlatformTransactionManager tm) { this.tx = new TransactionTemplate(tm); }

    public void process(List<Invoice> list) {
        for (Invoice i : list) {
            tx.executeWithoutResult(s -> saveOne(i));   // each invoice in its own transaction, no proxy needed
        }
    }
    private void saveOne(Invoice i) { ... }
}
```

The `TransactionTemplate` option is often the cleanest for "per-item transaction in a loop" because it makes the boundary explicit in code. Common review catch: `@Transactional` on `private`/`final` methods compiles and "runs" but silently has no effect — add a test that asserts rollback behaviour.

---

### S11. "A single tenant's traffic spike is degrading service for everyone. What do you do?"

- **Rate limit per tenant** at the gateway (token bucket, e.g. Bucket4j/Redis) — 429 with `Retry-After`.
- **Bulkhead by tenant tier**: separate thread pools / connection pools / even separate pod deployments (cell-based architecture) for large tenants.
- **Fair queueing** for async work — per-tenant partitions or weighted consumption so one tenant can't fill the queue.
- **Load shedding**: reject early when saturated (queue depth threshold) instead of degrading everyone.
- Add **per-tenant metrics** so you can see it happening, and quotas in the contract.

**Terms explained:** a *tenant* is one customer organisation sharing your system (the "noisy neighbour" problem is one tenant consuming shared resources). A **token bucket** gives each tenant a bucket that refills at a steady rate (say 100 tokens/second); each request takes a token, and an empty bucket means "429 Too Many Requests" with a `Retry-After` header telling the client when to retry — it allows short bursts but caps sustained rate. **Bulkheading** = isolating resources so one tenant's overload can't drain shared ones (separate pools like ship compartments). A **cell-based architecture** runs several complete, independent copies of the stack and assigns tenants to cells, limiting the "blast radius" of any problem to one cell. **Load shedding** = when the system is already near saturation, deliberately reject the lowest-priority/excess requests *quickly* (503/429) instead of queuing them until everything times out.

**How to describe the order of action:** (1) detect which tenant via per-tenant metrics, (2) throttle that tenant immediately at the gateway, (3) protect others with bulkheads, (4) fix the structural cause (quotas, tier isolation).

---

### S12. "Design multi-tenancy in a Spring Boot app."

| Model | Isolation | Cost/ops |
|---|---|---|
| Shared schema + `tenant_id` column | Weakest — one bad query leaks data | Cheapest, easiest to scale to many tenants |
| Schema per tenant | Good | Migrations × N schemas |
| Database per tenant | Strongest, easy per-tenant backup/restore | Most expensive, connection-pool pressure |

Implementation for shared schema:
- Resolve tenant from JWT claim/subdomain/header in a **filter** → store in a `ThreadLocal`/`ScopedValue` **context holder** (clear it in a `finally` — otherwise a pooled thread leaks tenant A's context into tenant B's request, which is a data breach).
- Enforce automatically with **Hibernate filters** (`@FilterDef`/`@Filter`) or Postgres **Row-Level Security** — never rely on developers remembering `where tenant_id = ?`.
- For schema/DB-per-tenant: Hibernate's `MultiTenantConnectionProvider` + `CurrentTenantIdentifierResolver`, or `AbstractRoutingDataSource`.
- Don't forget tenant propagation into async threads, Kafka headers, and caches (**tenant must be part of every cache key**).

---

### S13. "Deploy with zero downtime while a DB migration adds a NOT NULL column."

Use **expand/contract**:
1. Release 1: add the column **nullable**, no code change.
2. Backfill in batches (throttled, avoid a table-lock/long transaction).
3. Release 2: code writes both old and new; add the NOT NULL constraint (in Postgres: `NOT VALID` then `VALIDATE CONSTRAINT` to avoid a long exclusive lock).
4. Release 3: code reads only the new column.
5. Release 4: drop the old column.

Plus: rolling deploy with readiness probes and `server.shutdown=graceful`, backward-compatible API/messages during the overlap (old and new pods run simultaneously — **your migration must be compatible with N-1 code**), and `ddl-auto: validate`.

---

### S14. "A `@Cacheable` method returns stale data after an update. Debug it."

Checklist to walk through:
1. Is the update path calling `@CacheEvict`/`@CachePut` at all?
2. Is the **key** the same in both places? Default `SimpleKeyGenerator` uses all params — `find(id)` and `evict(entity)` produce different keys. Use explicit `key = "#id"`.
3. Is the evict happening **before commit**? If the transaction rolls back you've evicted for nothing; worse, a concurrent read can repopulate from the old committed row. Evict **after commit** (`@TransactionalEventListener(AFTER_COMMIT)`) or use `@CacheEvict(beforeInvocation = false)` plus short TTL.
4. Is it a **local cache on multiple pods**? Pod A evicts, pods B and C still serve stale. → distributed cache or pub/sub invalidation.
5. Self-invocation → annotation not applied at all.
6. Missing `@EnableCaching`.

State the principle: **prefer short TTL + explicit invalidation, and accept a bounded staleness window that the product has agreed to.**

---

### S15. "Your service's memory grows steadily and pods get OOM-killed every 6 hours."

- Confirm shape: is it heap (`jvm_memory_used_bytes{area="heap"}` rising with GC unable to reclaim) or **off-heap/RSS** (heap flat but the container is killed → metaspace, threads, direct buffers, native libs)?
- Heap dump → MAT dominator tree → find the retaining root.
- Typical Spring culprits: unbounded `ConcurrentHashMap` used as a cache, `ThreadLocal` never cleared, growing persistence context in a long transaction/batch job, unbounded executor queue, event listeners accumulating, `List` accumulation while streaming a huge query.
- Off-heap culprits: too many threads × stack size, Netty direct buffers, unclosed streams, `MaxRAMPercentage` not set so the JVM assumed the host's RAM.
- Fixes: bound every cache (Caffeine `maximumSize` + TTL), clear `ThreadLocal` in a `finally`, `flush()/clear()` in batches, `-XX:MaxRAMPercentage=75`, `-XX:+HeapDumpOnOutOfMemoryError`.
- Prevent: a memory dashboard, container-limit alerts, and a nightly soak/load test.

---

### S16. "Requirement change: an existing sync REST call must become fire-and-forget because it's slow. How do you migrate safely?"

1. Keep the endpoint's contract: accept the request, validate synchronously (so clients still get 400 for bad input), persist a job row, return **202 Accepted** with a `Location: /jobs/{id}` for polling.
2. Publish the work to Kafka/Rabbit (via the outbox), process with a consumer, update job status.
3. Provide a status endpoint or callback/webhook/SSE for completion.
4. **Migration strategy:** feature flag per client, dual-run behind the flag, monitor, then flip. Version the endpoint if the response shape must change.
5. Call out the cost: the client now needs to handle asynchrony; you need job-status storage, retries, DLQ, and observability of the queue depth.

---

### S17. "How would you add rate limiting to a Spring Boot API?"

- **Preferred:** at the gateway/API management layer (Spring Cloud Gateway's `RequestRateLimiter` with Redis, Nginx, Kong, AWS API Gateway) — keeps the app simple and protects it before the request costs you anything.
- **In-app:** Bucket4j with Redis backing (distributed token bucket) or Resilience4j `RateLimiter`, applied via a filter or an annotation + aspect.
- Key by API key / tenant / user / IP; different limits per tier.
- Return **429** with `Retry-After`, `X-RateLimit-Limit/Remaining/Reset` headers.
- **Must be distributed** — an in-memory limiter on 5 pods means the effective limit is 5×.
- Combine with concurrency limits (bulkhead) and load shedding; rate limiting alone doesn't stop one slow expensive query from saturating you.

**What rate limiting is and how the token bucket works:** it caps how many requests a caller can make per time window, protecting the service from abuse, bugs (a client retry loop), and noisy neighbours. In a **token bucket** each caller has a bucket holding up to `capacity` tokens (allowed burst) that refills at `rate` tokens/sec; every request removes one token; no token → rejected with 429. Example: capacity 20, refill 10/s → a client can burst 20 requests instantly, then sustain 10/s. Other algorithms: *fixed window* (simple but allows double bursts at window edges) and *sliding window* (smoother, costlier).

```java
Bucket bucket = Bucket.builder()
    .addLimit(Bandwidth.builder().capacity(20).refillGreedy(10, Duration.ofSeconds(1)).build())
    .build();                                     // Bucket4j — back with Redis (proxy manager) to share across pods

if (!bucket.tryConsume(1)) {
    res.setStatus(429);
    res.setHeader("Retry-After", "1");
    return;
}
chain.doFilter(req, res);
```

Why *distributed*: with state in each pod's memory, a load balancer spreads one client's requests across 5 pods and each pod counts only its share, so the real limit is 5× what you configured; a shared store (Redis) keeps one counter per key. Why at the *gateway*: rejecting at the edge costs almost nothing, while rejecting deep in the app has already used a thread, a connection and parsing time. **Pitfall:** key on something hard to spoof/rotate (API key, authenticated user) — pure IP limits punish users behind a shared NAT and are easy to bypass.

---

### S18. "The team wants to split a monolith into microservices. How do you lead that?"

1. **Push back appropriately:** ask what problem we're solving. If it's "deployments are slow" or "the codebase is messy," start with a **modular monolith** — module boundaries, no cross-module DB access, separate schemas, then split later. That's a senior answer.
2. **Identify bounded contexts** with event storming / domain analysis; find the seams where data ownership is clean.
3. **Strangler fig pattern:** put a facade/gateway in front, extract the highest-value/lowest-coupling context first (often something read-heavy or clearly bounded like Notifications or Search).
4. **Data separation is the hard part:** dual-write or CDC replication during transition, then cut over reads, then writes. Remove cross-service joins by replicating read models via events.
5. **Platform prerequisites before service #2:** CI/CD per service, centralized logging/tracing, service templates (an internal starter), contract testing, observability, on-call ownership.
6. **Measure:** deployment frequency, lead time, MTTR, change-failure rate. If they don't improve, the split isn't paying for itself.

---

### S19. "A bug only happens in production and you can't reproduce it locally."

- Turn on targeted debug logging at runtime via `POST /actuator/loggers/com.acme.order` `{"configuredLevel":"DEBUG"}` — no redeploy.
- Use the `traceId` from the failing response to pull the full distributed trace and correlated logs.
- Check environment deltas: profile-specific config, `/actuator/env` and `/configprops` to see the **effective** values and their source; data volume/shape; timezone/locale; DB version and plan; TLS/proxy; feature flags.
- Reproduce with production-like data using Testcontainers + an anonymised data subset, or shadow traffic/replay.
- Capture more: thread dump if it's a hang, heap dump if it's memory, JFR recording for CPU/allocation.
- Add the missing observability as part of the fix, and a regression test that would have caught it.

---

### S20. "How do you prevent a single bad request from taking down the service?"

Defence in depth: request size limits (`spring.servlet.multipart.max-file-size`, `server.max-http-request-header-size`), page-size caps, query timeouts (`statement_timeout`, `jakarta.persistence.query.timeout`), pagination mandatory on list endpoints, regex-DoS review (no catastrophic backtracking in validation regex), disallow unbounded `Sort` fields (whitelist), input validation before any DB work, and per-endpoint concurrency limits. Plus circuit breakers so a poison-triggered downstream failure doesn't cascade.

**What each guard protects against:** *Defence in depth* = several independent layers, so if one is missed another still stops the problem. Examples: a 5 GB upload exhausts disk/memory → size limits; `GET /orders?size=10000000` loads the whole table → page cap; a user-supplied `sort=` that isn't indexed (or contains SQL) → allow-list of sortable fields; a query without a timeout holds a DB connection forever → `statement_timeout`. **Regex DoS (ReDoS):** some regex patterns like `(a+)+$` take exponential time on crafted input (`"aaaa...!"`), pinning a CPU core per request — a handful of requests can freeze a service. Prefer simple patterns, limit input length *before* matching, and avoid nested quantifiers.

```yaml
server.tomcat.max-swallow-size: 2MB
spring.servlet.multipart.max-file-size: 10MB
spring.data.web.pageable.max-page-size: 100
```

```java
private static final Set<String> SORTABLE = Set.of("createdAt", "status", "total");
Sort safe(Sort requested) { /* reject or drop any property not in SORTABLE */ }
```

---

### S21. "Your Spring Boot app is deployed to Kubernetes and pods restart during rolling deploys, dropping requests."

- Add `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s`.
- Kubernetes sends SIGTERM and removes the pod from endpoints **asynchronously** → add a `preStop` sleep (5–10 s) so the load balancer stops sending traffic before the app stops accepting.
- Correct probes: readiness → `/actuator/health/readiness` (and flip it to "out of service" on shutdown, which Boot does automatically), liveness → `/actuator/health/liveness`, plus a `startupProbe` for slow-starting apps so liveness doesn't kill them mid-boot.
- `terminationGracePeriodSeconds` > your shutdown timeout.
- Clients should retry idempotent requests; keep-alive/connection draining configured at the ingress.

**Why requests drop and how the pieces fit:** on a rolling deploy Kubernetes does two things *at the same time*: it removes the old pod from the Service endpoints (so no new traffic) and sends the container **SIGTERM** (a polite "please stop" signal). But endpoint removal takes a moment to reach every load balancer/ingress, so for a few seconds traffic may still arrive at a pod that is already shutting down → connection refused / 502. Without graceful shutdown, Tomcat stops immediately and in-flight requests are cut.

**Sequence you want:**
1. `preStop: sleep 10` — pod keeps serving while the endpoint removal propagates.
2. SIGTERM → Boot flips readiness to `REFUSING_TRAFFIC` and, with `server.shutdown=graceful`, stops accepting *new* connections but lets in-flight requests finish (up to `timeout-per-shutdown-phase`).
3. App exits; if it hasn't by `terminationGracePeriodSeconds`, Kubernetes sends SIGKILL — so that value must exceed `preStop` + shutdown timeout.

```yaml
# deployment.yaml (excerpt)
lifecycle:
  preStop: { exec: { command: ["sh", "-c", "sleep 10"] } }
terminationGracePeriodSeconds: 60
readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 } }
livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 } }
startupProbe:   { httpGet: { path: /actuator/health/liveness,  port: 8080 }, failureThreshold: 30, periodSeconds: 5 }
```

A **startupProbe** disables liveness checks until the app has started once — otherwise a slow boot (migrations, cache warm-up) is mistaken for "dead" and the pod is restarted in a loop.

---

### S22. "You need to process a 2 GB CSV upload without OOM."

- **Stream, never buffer:** read with a `BufferedReader`/`InputStream` from the multipart part or (better) have the client upload to S3 and pass you a key, then stream from S3.
- Parse incrementally (univocity/Commons CSV/Jackson `MappingIterator`), process in chunks of N, `flush()`+`clear()` the persistence context per chunk, or bypass JPA with `JdbcTemplate.batchUpdate`.
- Make it **asynchronous**: return 202 + job id, process via a queue, report progress and a per-row error report.
- Idempotency + restartability: track the last processed offset/line so a crash resumes rather than restarts. **Spring Batch** gives you chunking, restart, skip/retry policies, and a job repository for free — mention it as the right tool if this is a recurring need.
- Guard rails: file size limit, virus scan, row-count limit, backpressure so one huge job doesn't starve online traffic.

---

### S23. "Two services need the same data. One owns it. How do you give the other access?"

Ranked options:
1. **Sync API call** — simplest, always fresh, but temporal coupling and latency; fine for low volume.
2. **Event-carried state transfer** — owner publishes changes, consumer keeps a local read model. Fast reads, no runtime coupling, eventually consistent. Best for read-heavy paths.
3. **CQRS read model / materialized view** built from events.
4. **Shared database** — ❌ never; it couples deployments and schemas and destroys ownership.

Choose based on freshness requirement (can it be 500 ms stale?), read volume, and availability requirement (can service B serve traffic when A is down? Only options 2/3 allow yes).

---

### S24. "Your `@Async` method silently does nothing / throws away exceptions."

Check in order: is `@EnableAsync` present; is the method **public** and called from **another bean** (not self-invoked); is the return type `void` (exceptions vanish → use `CompletableFuture` or register an `AsyncUncaughtExceptionHandler`); is a proper executor configured (else unbounded thread creation); did you expect the `SecurityContext`/`MDC`/transaction to propagate (they don't — use `TaskDecorator` / `DelegatingSecurityContextExecutor`); did the app shut down before the task ran (`setWaitForTasksToCompleteOnShutdown(true)`).

**Why each check matters:** `@Async` is implemented by a proxy that submits your method to an executor, so every reason a proxy might be bypassed makes it run **synchronously on the caller's thread** (it looks like "nothing async happens") — missing `@EnableAsync`, self-invocation, `private`/`final` method. For `void` methods the caller has returned before the task runs, so an exception has no one to be thrown to; Spring only logs it via the uncaught-exception handler. Fix example:

```java
@Async("ioExecutor")
public CompletableFuture<Void> notifyUser(long id) {
    try { mailer.send(id); return CompletableFuture.completedFuture(null); }
    catch (Exception e) { return CompletableFuture.failedFuture(e); }   // caller can .exceptionally(...)/.join()
}
```

Quick way to verify it's truly async: log `Thread.currentThread().getName()` inside the method — it should show your executor's prefix (`io-1`), not `http-nio-8080-exec-3`.

---

### S25. "How do you handle a poison message that keeps failing and blocking your Kafka partition?"

- With a blocking `DefaultErrorHandler`, retries block the partition → head-of-line blocking, consumer lag grows, eventually `max.poll.interval.ms` is exceeded and you rebalance in a loop.
- **Fix:** classify errors. Non-retryable (validation, deserialization, business rejection) → send straight to **DLT** and commit. Retryable (timeout, 5xx) → bounded retries with backoff, then DLT.
- Use `@RetryableTopic` / non-blocking retry topics so retries don't block the main partition.
- Handle deserialization failures with `ErrorHandlingDeserializer` (otherwise the consumer can't even move past the record).
- Operationalise the DLT: alert on depth, provide a **replay tool** after the bug is fixed, and never let it become a silent graveyard.

**Why one bad message blocks everything:** a Kafka partition is read strictly in order, one record after another. If record #57 throws and the handler retries it (with the blocking error handler) 10 times with 5-second waits, records #58, #59… behind it just wait — **head-of-line blocking**. Lag climbs, and if the retries take longer than `max.poll.interval.ms` the broker removes the consumer, triggering a rebalance, after which the *same* message is redelivered and the cycle repeats. A **poison pill** is a message that can never succeed (malformed JSON, a business rule that will always reject it), so retrying it is pointless.

```java
@RetryableTopic(attempts = "4", backoff = @Backoff(delay = 1000, multiplier = 2.0),
                exclude = { ValidationException.class })            // never retry → goes straight to DLT
@KafkaListener(topics = "orders")
void handle(OrderEvent e) { service.process(e); }

@DltHandler
void dead(OrderEvent e) { log.error("parked {}", e.eventId()); alerts.notifyDeadLetter(e); }
```

With `@RetryableTopic`, a failed record is republished to `orders-retry-0/1/2` (delays 1 s, 2 s, 4 s) and the main partition moves on; only after exhausting retries does it land in `orders-dlt`. **Trade-off:** retry topics break strict ordering for the retried record (it's processed later than its successors) — acceptable for independent events, not for ones that must be ordered per key. **Deserialization trap:** if the bytes can't even be parsed into `OrderEvent`, the consumer fails inside `poll()` before your handler runs; wrapping the deserializer in `ErrorHandlingDeserializer` converts that into a normal error that can be routed to the DLT.

---

### S26. "Interviewer: your service calls 3 downstreams; one is optional. Design the failure behaviour."

Define **criticality tiers** explicitly: for the mandatory calls, failure → 503/502 with retry guidance; for the optional one, failure → omit the section, log at WARN, increment a `degraded` counter, and return a `partial` indicator. Wrap the optional call in its own circuit breaker + tighter timeout + smaller bulkhead so it can never be the reason the endpoint is slow. State the principle: **"Optional dependencies must have optional latency budgets."** Then add a dashboard for degradation rate so silent permanent degradation gets noticed.

---

### S27. "How do you migrate a Spring Boot 2.7 app to 3.x?"

1. **Prerequisites first:** upgrade to the latest 2.7.x, get on Java 17, fix all deprecation warnings, upgrade third-party libs to Jakarta-compatible versions (check the Spring Boot Migration Advisor / OpenRewrite `UpgradeSpringBoot_3_0`).
2. **`javax` → `jakarta`** across persistence, servlet, validation, annotation.
3. **Config keys:** many `spring.*` and `management.*` properties moved (the `spring-boot-properties-migrator` dependency reports them at startup — add it temporarily).
4. **Security 6:** replace `WebSecurityConfigurerAdapter` with `SecurityFilterChain` beans, `antMatchers` → `requestMatchers`, `authorizeRequests` → `authorizeHttpRequests`.
5. **Hibernate 6:** review native/HQL queries, generated SQL, `@Type` usage, and paging behaviour; run your integration suite against a real DB.
6. **Sleuth → Micrometer Tracing** for observability.
7. **Strategy:** one service at a time, behind a canary, with strong integration tests and contract tests; keep the migration mechanical (no feature work in the same PR).

---

### S28. "A new feature needs strong consistency but your architecture is event-driven. What do you do?"

Don't force eventual consistency where the business can't accept it. Options: keep that invariant **inside a single service and a single database transaction** (redraw the service boundary so the aggregate that must be consistent is owned by one service); or use a **reservation/two-phase business flow** (reserve → confirm with a TTL, e.g. seat/inventory holds); or a saga with a **semantic lock** and a visible PENDING state. The senior move is recognising that *consistency requirements should drive service boundaries* — if two things must be atomic, they probably belong to the same aggregate.

**Explained with an example:** in an event-driven system, services update their own data and tell others via events, so for a short time different services disagree (*eventual consistency*). That's fine for "update the recommendation list", but not for "never sell the same last concert seat twice" or "an account balance must never go negative". An **aggregate** (DDD term) is a cluster of data that must always be consistent together (e.g. an `Account` and its ledger entries); keeping an aggregate inside one service and one DB transaction gives you real ACID guarantees (use `@Version`/row locks) without distributed coordination.

**Reservation flow (for cross-service cases):** step 1 `reserve(seat 12A, ttl=10min)` → status `HELD`; step 2 after payment succeeds `confirm(reservationId)` → `SOLD`; if payment fails or the TTL expires, a job releases the hold. Nothing is ever double-sold because the *reserve* step is atomic inside the owning service, and the visible `HELD` state is the "semantic lock" other users see.

```sql
-- inside the owning service, one atomic statement — no race even with many pods
UPDATE seat SET status = 'HELD', held_until = now() + interval '10 minutes'
WHERE id = :seat AND status = 'FREE';        -- affected rows = 1 → you got it; 0 → someone else did
```

---

### S29. "You inherit a service with 40% test coverage and frequent production bugs. What's your plan as tech lead?"

1. **Stop the bleeding:** add tests around every bug fix (regression-first), and add a coverage ratchet on **changed lines** rather than a global target.
2. **Characterisation tests** on critical paths before refactoring (approval/golden-master tests on API responses).
3. Prioritise by risk: money paths, auth, data mutation — not by coverage percentage.
4. Add **contract tests** at boundaries, Testcontainers for real DB behaviour, and a smoke suite that runs post-deploy.
5. Improve feedback loop: fast unit tests in PR, slower integration nightly; fix flaky tests immediately (a flaky suite is worse than no suite because people stop believing it).
6. Make quality visible: DORA metrics, error budget, a weekly review of production incidents with action items.
7. Coverage is a lagging indicator — I'd track **change-failure rate** and **escaped defects** instead.

---

### S30. "How do you handle a breaking change in an API used by 20 clients?"

Version it (`/v2`), run v1 and v2 in parallel from the same codebase (adapter layer mapping to a single internal model), announce with `Deprecation` and `Sunset` headers plus docs and changelog, instrument **per-client usage metrics** on v1 so you know exactly who's left, reach out to laggards, then remove after the agreed window. For non-breaking evolution, prefer additive changes + tolerant readers so you don't need a v2 at all. Never silently change semantics of an existing field — that's the one thing clients can't defend against.

---

### S31. "Requests intermittently fail with 401 after 15 minutes of user activity."

Almost certainly **access-token expiry** with no refresh handling: token TTL is 15 min, the SPA isn't refreshing, or the refresh call races across parallel requests. Fixes: implement refresh-token rotation with a single-flight refresh on the client (queue in-flight requests during refresh), return a distinguishable error code (`token_expired` vs `invalid_token`) so the client knows to refresh rather than logging the user out, check clock skew between issuer and validator (`allowedClockSkew`), and verify all pods share the same signing key/JWKS cache. If sessions (not JWT), check sticky sessions vs a shared session store (Spring Session + Redis) — a rolling deploy wipes in-memory sessions.

**Why "after 15 minutes" is the clue:** a number that stays constant (15 min) points to a **TTL/expiry** rather than a random bug. The access token's `exp` claim is a timestamp; once passed, the resource server rejects it with 401. The login worked because the token was fresh. **Single-flight refresh** means: when the first request gets 401, the client calls `/token/refresh` once, and any other requests that fail meanwhile wait for that *same* refresh result instead of each calling refresh — otherwise with refresh-token rotation, the second parallel refresh uses an already-rotated token, the server treats it as theft and revokes everything, and the user is logged out. **Clock skew:** if the auth server's clock is 2 minutes ahead of your API pod's, a token can look "not yet valid" (`nbf`/`iat` in the future) or expire early; allow a small leeway (e.g. 30–60 s). **JWKS cache:** services fetch the issuer's public keys and cache them; after a key rotation a pod with a stale cache can't verify new tokens — intermittent 401 depending on which pod you hit.

```java
// Spring Security resource server: tolerate small clock drift
var validator = new DelegatingOAuth2TokenValidator<>(
        new JwtTimestampValidator(Duration.ofSeconds(60)),
        new JwtIssuerValidator(issuer));
jwtDecoder.setJwtValidator(validator);
```

**Diagnostic order:** compare the failing request's `exp` with the time it was sent; check whether failures cluster per pod (key/JWKS issue) or per user session (expiry/refresh issue).

---

### S32. "Explain a production incident you handled." (Be ready with a real STAR story.)

**Template to fill with your own:**
- **Situation:** "Checkout error rate went from 0.1% to 12% at 19:00 on a Friday, ~4,000 failed orders/hour."
- **Task:** "I was on call and incident lead."
- **Action:** "Dashboards showed errors isolated to one endpoint; traces pointed at the inventory service; its p99 had gone to 20 s after a deploy that added an unindexed query. We rolled their deploy back within 12 minutes, which restored the error rate. Root cause: a query without an index on a table that had grown 10×; contributing cause: our call had no timeout and no circuit breaker, so their slowness became our outage."
- **Result:** "MTTR 14 minutes. Follow-ups: added the index, added timeouts + Resilience4j breakers around all downstream calls (we found 6 other unprotected clients), added a p99 alert on every dependency, and added query-plan review to the DB change checklist. No repeat in 12 months."

Always include: how you communicated during the incident, the blameless postmortem, and the **systemic** fix — not just the immediate one.

---

### S33. "Your service must support 10× traffic next quarter. What do you do?"

1. **Measure the current ceiling** — load test to find the actual bottleneck (don't guess): DB, pool, CPU, downstream, or lock contention.
2. **Scale out statelessly** — no in-memory state or sticky sessions; move sessions to Redis; HPA on RPS/latency, not just CPU.
3. **Database is usually the wall:** read replicas for read traffic (`@Transactional(readOnly=true)` + `AbstractRoutingDataSource`), connection pooling via PgBouncer, indexing, caching hot reads, then partitioning/sharding as the last resort.
4. **Cache aggressively** with clear invalidation; add a CDN for static/semi-static responses.
5. **Make writes async** where the user doesn't need the result immediately (queue + workers), which converts a latency problem into a throughput problem you can scale.
6. **Protect yourself:** rate limits, bulkheads, load shedding, autoscaling limits, and cost modelling.
7. **Verify:** repeat the load test at 10× with production-like data, run a game day/chaos test, and pre-warm before known peaks.

**Explained:** the golden rule is *measure before changing*: pushing 10× load through a **load test** (k6/Gatling/JMeter) shows which resource saturates first (CPU, DB connections, a lock, a downstream rate limit), and fixing the wrong thing wastes the quarter. **Stateless** means any pod can serve any request — no per-user data in memory, so adding pods adds capacity linearly (**HPA** = Kubernetes Horizontal Pod Autoscaler adds pods when a metric crosses a threshold; scaling on requests-per-second or latency reacts better than CPU for I/O-bound services). **Read replicas** are DB copies that serve reads while one primary takes writes; route `readOnly` transactions to them, but remember **replication lag** (a read right after a write may not see it yet). **Sharding** splits data across databases by a key (tenant, customer id) — powerful but makes cross-shard queries and transactions hard, hence "last resort".

```java
// route read-only transactions to a replica
class RoutingDataSource extends AbstractRoutingDataSource {
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "replica" : "primary";
    }
}
```

**Async writes example:** instead of "POST /orders computes invoice + sends email + updates stats synchronously", save the order, return 201, and push the rest to a queue — the request path stays short and the workers scale independently. **Game day** = a rehearsed failure drill (kill a pod, slow a dependency) to confirm your assumptions.

---

### S34. "How do you decide between adding a feature to an existing service vs creating a new one?"

Ask: does it belong to the same bounded context and own the same data? Does it have a different scaling profile, release cadence, availability requirement, or compliance boundary? Will it need a different team's on-call? If it's the same data and same lifecycle → same service (a new module). If it introduces a new aggregate with independent scaling/ownership → new service. Default bias: **don't create a service to hold logic that can't be deployed independently of another** — that's just distributed coupling.

---

### S35. "A developer's PR adds `@Transactional` to a controller method. What's your review comment?"

Transactions belong at the **service layer**, which is the business-use-case boundary. On a controller it (a) usually doesn't work as intended (the proxy is around the controller, and it widens the transaction to include serialization and any remote calls), (b) holds a DB connection during HTTP I/O, and (c) leaks a persistence concern into the web layer. Suggest moving it to the service, keeping the transaction as short as possible, marking read paths `readOnly = true`, and keeping remote calls outside it.

**Why this is a real problem and how to explain it to the author:** a controller's job is HTTP — parsing the request, calling a use-case, shaping the response. A transaction should wrap *one business use-case* (e.g. "place order"), which may involve several repositories. If the controller method is transactional, the DB connection stays checked out while the response body is serialized and while any `restClient` call runs, and if two controllers call each other the boundaries get confusing. Also, a controller often *composes* multiple service calls — if each service is itself transactional and the outer controller method also is, the inner methods silently join the outer one (`REQUIRED`), so you can't tell which unit of work rolled back.

```java
// ❌ review comment target
@RestController class OrderController {
    @Transactional @PostMapping("/orders")
    OrderDto create(@RequestBody CreateOrder req) { ... }
}

// ✔ suggested
@RestController class OrderController {
    @PostMapping("/orders") OrderDto create(@RequestBody @Valid CreateOrder req) { return service.place(req); }
}
@Service class OrderService {
    @Transactional public OrderDto place(CreateOrder req) { ... }   // boundary = use case
}
```

Frame it constructively: "the service layer is where the business invariant lives, so that's where atomicity should be declared; it also makes the service reusable from a Kafka listener or scheduler without duplicating transaction handling."

---

### S36. "You need audit history of every change to an entity (who, what, when)."

- **Spring Data JPA Auditing** for the simple case: `@EnableJpaAuditing`, `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy` + an `AuditorAware` bean reading the `SecurityContext`.
- **Hibernate Envers** (`@Audited`) for full revision history tables with query support — cheap to adopt, but it writes on the same transaction and can bloat.
- **Event sourcing / an append-only event log** if history *is* the source of truth.
- **DB-level:** triggers or temporal tables; or CDC (Debezium) into a data warehouse — good when you must capture changes made outside the app too.
- **Trade-offs to state:** Envers doubles write volume and complicates schema migrations; app-level auditing misses direct SQL changes; CDC is decoupled but eventually consistent. Also: audit records must survive rollback of the business transaction only if that's actually desired — usually you *want* them in the same transaction so they can't lie.

---

### S37. "Requests are being handled but you have no idea which user/tenant caused an error in the logs."

Add a **correlation filter** early in the chain: read or generate `X-Request-Id`, put `requestId`, `traceId`, `userId`, `tenantId` into **MDC**, clear in `finally`, include them in the log pattern and in error responses. Propagate them outbound (a `RestClient`/`WebClient` interceptor and Kafka headers) so the whole call graph is correlatable. Then ship structured JSON logs so you can query by field. Remember to propagate MDC into `@Async`/executor threads via a `TaskDecorator` — this is the step people forget.

**What each part does:** a *correlation ID* is a unique label attached to one request so every log line from it can be found together. The filter creates it once at the edge (or reuses the caller's `X-Request-Id` so a chain of services shares it), stores it in **MDC** (a per-thread map the logging framework reads) and removes it in `finally` — if not cleared, a pooled thread would carry user A's ID into user B's request. The log pattern then prints `%X{requestId}`, so you can search "requestId=ab12" and see the whole story.

```java
@Component
class CorrelationFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String id = Optional.ofNullable(req.getHeader("X-Request-Id")).orElse(UUID.randomUUID().toString());
        MDC.put("requestId", id);
        Optional.ofNullable(req.getUserPrincipal()).ifPresent(p -> MDC.put("userId", p.getName()));
        res.setHeader("X-Request-Id", id);                 // client/support can quote it
        try { chain.doFilter(req, res); } finally { MDC.clear(); }
    }
}
```

Note: if the filter runs *before* authentication, `userId` isn't known yet — add it again after the security filter (or register this filter after it). **Outbound propagation** = add the same header to every `RestClient`/Kafka message so downstream services reuse the ID. Never log PII (emails, tokens) — use opaque user IDs.

---

### S38. "Business asks for a 'soft delete'. How do you implement it without breaking everything?"

Add `deleted_at` (nullable timestamp, better than a boolean — it records when). Enforce filtering **globally**, not per query: Hibernate `@SQLRestriction("deleted_at is null")` (or `@Where` pre-Hibernate 6.3) or a `@FilterDef` you enable per session; expose an explicit repository method for including deleted rows. Watch out for: unique constraints (a soft-deleted row still occupies the unique key → use a partial index `WHERE deleted_at IS NULL`), foreign keys and cascades, count/aggregate queries, and the fact that `deleteById` now means update. Also plan **hard-delete for GDPR/right-to-erasure** — soft delete is not deletion, and you must be able to purge.

**Why soft delete exists and what breaks:** deleting rows loses history, breaks foreign keys (orders referencing a deleted customer) and can't be undone, so many systems just *mark* the row as deleted. The cost: every query must remember to exclude those rows — forget once and "deleted" data reappears in a report. Hence the **global filter** that Hibernate appends automatically.

```java
@Entity
@SQLDelete(sql = "UPDATE customer SET deleted_at = now() WHERE id = ?")   // repository.delete() becomes an UPDATE
@SQLRestriction("deleted_at IS NULL")                                      // every SELECT gets this appended
class Customer { @Id Long id; Instant deletedAt; String email; }
```

```sql
-- unique email only among live rows, so a deleted user can re-register
CREATE UNIQUE INDEX uq_customer_email_live ON customer(email) WHERE deleted_at IS NULL;
```

**Probes:** *"What about native queries?"* — `@SQLRestriction` isn't applied to native SQL, so add the condition manually. *"What about cascade?"* — cascading soft delete needs the same annotations on children. *"GDPR?"* — keep a purge job (or anonymise PII fields) to physically remove or scrub data after the retention period.

---

### S39. "Your team disagrees: WebFlux vs MVC + virtual threads for a new service."

Frame it as a decision with criteria rather than a preference: what's the data access (JPA/JDBC → MVC, since R2DBC has no JPA equivalent and loses mapping/caching), does the team know reactive (debuggability and hiring matter), do we need streaming/backpressure (WebFlux's genuine edge), what's the concurrency profile? On Java 21 + Boot 3.2, **MVC with virtual threads gets most of the scalability with far lower cognitive cost**, so my default would be MVC + virtual threads, reserving WebFlux for gateways and streaming endpoints. Then write it up as an **ADR** so the decision and its context are recorded — and say you'd revisit if load testing shows otherwise.

**Why this framing:** reactive (WebFlux) means the *whole* call path must be non-blocking; one blocking JDBC call on an event-loop thread freezes many requests. With JPA/JDBC (blocking) you can't get the benefit, and R2DBC is a lower-level driver without lazy-loading, dirty checking or a cache. Virtual threads (Java 21) deliver "scalable blocking code", so you keep JPA, familiar stack traces, ThreadLocal-based security/MDC context, and imperative code. **ADR** (Architecture Decision Record) is a short document (usually in the repo: `docs/adr/0007-mvc-vs-webflux.md`) with *Context → Decision → Consequences*, so a future engineer knows *why* a choice was made rather than guessing. Evidence beats opinion: propose a small spike or load test of the actual workload to settle the argument.

**Quick decision table to say aloud:** blocking DB + CRUD APIs → MVC (+ virtual threads); gateway/proxy doing many parallel remote calls, SSE/WebSocket streaming, fully reactive data store → WebFlux.

---

### S40. "How would you introduce a feature flag system, and what are the risks?"

Use a real provider (LaunchDarkly, Unleash, Flagsmith) or a simple config-backed flag with `@RefreshScope` / a `FeatureService` bean; expose flags via a strategy interface so business code reads `if (features.isEnabled("new-pricing", ctx))` rather than reading config directly. Use them for progressive rollout, canary, kill switches, and trunk-based development. **Risks:** flag debt (every flag is a branch — 2^n paths, untested combinations), flags that live forever and become implicit config, inconsistent flag state across pods, and business logic hiding in flags. Mitigate with an owner and expiry date per flag, tests for both branches of critical flags, and a cleanup ritual each sprint.

**What a feature flag is:** a runtime switch that turns code paths on or off without redeploying, so *deploying* code is decoupled from *releasing* it. Uses: **progressive rollout** (1% → 10% → 100% of users while watching metrics), **canary** (new logic for internal users first), **kill switch** (instantly disable a misbehaving feature in prod without a rollback), and **trunk-based development** (merge unfinished work to main behind an off flag instead of long-lived branches).

```java
@Service
class PricingService {
    private final FeatureFlags features;
    Price price(Cart c, User u) {
        return features.isEnabled("new-pricing", u)      // evaluation context allows per-user/percentage targeting
                ? newEngine.price(c) : legacyEngine.price(c);
    }
}
```

**Why "2^n paths":** with n independent flags there are 2ⁿ combinations of behaviour, and you realistically test only a few — so flags interact in untested ways. **Inconsistent state:** with a config-backed flag refreshed per pod, pod A may have it on while pod B still has it off for a few seconds, so design the flagged behaviour to tolerate both versions (the same N-1 compatibility rule as deployments). Keep flags out of the domain model — a flag decides *which implementation to call*, not business rules.

---

### S41. "You're asked to expose an internal endpoint quickly for a partner, in production, today."

Push back with a safe minimum rather than a flat no: authenticate it (API key/mTLS/OAuth client credentials — never unauthenticated), rate limit it, restrict it at the network layer (allowlist/private link), return only a DTO with the minimum fields (no internal IDs/PII beyond need), log and monitor it separately, and put a documented expiry/review date on it. Then create a ticket for the proper versioned public API. **The senior signal is offering a controlled fast path plus a follow-up, not just refusing or just complying.**

---

### S42. "How do you keep 15 Spring Boot services consistent across teams?"

- An internal **parent POM / BOM** pinning Spring Boot and library versions.
- A **shared internal starter** for cross-cutting concerns: correlation IDs, exception handling and `ProblemDetail` format, Actuator config, metrics tags, security defaults, Kafka serializers, client timeouts.
- **Service templates / scaffolding** (Backstage, cookiecutter) so new services start compliant.
- **Automated enforcement:** ArchUnit rules (no entity in controller signatures, no `@Transactional` on controllers, layer dependencies), Spotless/Checkstyle, Sonar quality gates, dependency scanning, and a CI pipeline template.
- **Golden paths + ADRs**, not mandates: document the default choice, allow deviation with a written reason.
- Watch the trap: a shared library becomes a coupling point — version it, keep it thin and optional, and never put business logic in it.

**Why consistency matters:** with 15 services each built differently, every on-call engineer must relearn how each logs, authenticates, and reports errors; fixing one cross-cutting issue (e.g. a security patch) means 15 different changes. Consistency lowers cognitive load and makes fleet-wide changes cheap. Terms: a **BOM** (Bill of Materials) is a POM that only declares dependency *versions*, so every service imports the same tested set (`<scope>import</scope>`) without copy-pasting versions; a **starter** auto-configures shared behaviour once for all services (see Q4); a **golden path** is the well-supported, documented default way to do something, which teams can leave only with a reason, so autonomy isn't lost.

```java
// ArchUnit rule that stops a recurring mistake across all services in CI
@ArchTest
static final ArchRule no_entities_in_controllers =
    noClasses().that().resideInAPackage("..web..")
        .should().dependOnClassesThat().areAnnotatedWith(jakarta.persistence.Entity.class);
```

**Trade-off to name:** too much standardisation slows teams and creates a platform bottleneck; too little yields fragmentation. A common compromise is *"standardise the boring, cross-cutting things (observability, security, build); leave domain design to teams."* **Release trap:** if the shared starter changes, 15 services must upgrade — use semantic versioning and avoid breaking changes without a deprecation window.

---

## 19. System Design Rounds

At 9 YOE you'll get one design question. Use this skeleton and speak it out loud:

1. **Clarify requirements** — functional, then **non-functional: RPS, read/write ratio, data volume, latency SLO, consistency needs, availability target, growth.** *Never skip this step; skipping it is the most common failure.*
2. **Estimate** — back-of-envelope QPS, storage/day, payload sizes, peak/average ratio.
3. **API contract** — a few key endpoints/events, with idempotency and pagination decided.
4. **Data model + storage choice** — SQL vs NoSQL and *why*; indexes; partitioning key.
5. **High-level components** — gateway, services, cache, queue, DB, read models.
6. **Deep dive** into the part they care about (usually the write path or the hot read path).
7. **Scale & failure** — caching, replicas, sharding, async, backpressure, circuit breakers, timeouts, retries, idempotency; what happens when each component dies.
8. **Observability & ops** — metrics, alerts, SLOs, rollout strategy, migrations.
9. **Trade-offs & what you'd do differently at 10× scale.**

**Practise these five:** URL shortener; a rate limiter; an order/checkout flow with payments; a notification service (multi-channel, retries, dedupe); a news feed / activity timeline. Each maps cleanly to Spring Boot components you already know (Gateway, Redis, Kafka, Postgres, outbox, ShedLock, Resilience4j).

**What the terms in the skeleton mean (with a mini example — URL shortener):**
- **Functional vs non-functional requirements:** *functional* = what it does ("create short URL, redirect"); *non-functional* = how well ("100M redirects/day, p99 < 50 ms, 99.99% available, links never lost"). These numbers drive every later choice, which is why skipping them is the top failure.
- **Back-of-envelope estimate:** 100M redirects/day ÷ 86,400 s ≈ 1,200 RPS average; peak ≈ 3–5× → ~5,000 RPS. Reads ≫ writes (say 100:1) → optimise the read path with a cache. Storage: 100M new URLs/year × ~500 bytes ≈ 50 GB/year — small enough for one primary DB with replicas, so no sharding needed yet. Showing this arithmetic proves your design is sized, not guessed.
- **Partitioning key:** the field used to spread data across shards/partitions; choose one with even distribution and that matches the main access pattern (short-code hash for the shortener, `customerId` for orders).
- **SQL vs NoSQL "and why":** pick by access pattern and consistency — relational for transactions/joins (orders, payments); key-value/wide-column for huge lookups by key with simple queries (short code → URL).
- **Failure walk-through:** for each box ask "what if it dies?" — cache down → fall back to DB with rate limiting; DB primary down → replica promotion; queue down → outbox buffers; downstream slow → timeout + circuit breaker.

**Sample opening:** *"Before designing, can I confirm scale — expected RPS, read/write ratio, latency target, and how important is strong consistency versus availability?"* — then summarise the assumptions back so the interviewer can correct you.

---

## 20. Rapid-Fire Recall Tables

### Annotation quick reference

| Annotation | One-line meaning |
|---|---|
| `@SpringBootApplication` | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@Bean` / `@Component` | Manual factory method / scanned class |
| `@Primary` / `@Qualifier` | Default choice / explicit choice among candidates |
| `@Lazy` | Create on first use (also breaks some dependency cycles) |
| `@Profile` | Register bean only for given profile(s) |
| `@ConfigurationProperties` | Type-safe binding of a property tree |
| `@ConditionalOnMissingBean` | Back off if the user defined their own |
| `@Transactional` | Proxy-based tx boundary; rollback on unchecked only by default |
| `@Async` | Run on a `TaskExecutor`; return `CompletableFuture` |
| `@Scheduled` | Cron/fixedDelay/fixedRate; single-threaded by default |
| `@Cacheable` / `@CachePut` / `@CacheEvict` | Read-through / write-through / invalidate |
| `@RestControllerAdvice` + `@ExceptionHandler` | Global error mapping |
| `@Valid` / `@Validated` | Bean Validation / Spring validation with groups + method validation |
| `@EntityGraph` | Declarative eager fetch for one query |
| `@Version` | Optimistic locking |
| `@PreAuthorize` | SpEL authorization before method execution |
| `@MockitoBean` (`@MockBean`) | Replace a bean with a mock in the test context |
| `@Testcontainers` + `@ServiceConnection` | Real infrastructure in tests, auto-wired |

### "Doesn't work" cheat sheet — proxy limitations

`@Transactional`, `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize` **all fail silently** when:
1. called via `this.` (self-invocation),
2. on a `private`/`final`/`static` method,
3. on a class not managed by Spring,
4. invoked from `@PostConstruct` (proxy not yet applied).

**Memory hook:** *"Proxies only intercept calls that come in through the front door."*

**Why each case fails:** Spring doesn't change your class; it gives other beans a **proxy** that adds behaviour (transaction, async, cache) and then forwards to your real object. (1) `this.foo()` is a call *inside* the real object, so it never touches the proxy. (2) CGLIB proxies work by subclassing and overriding, which is impossible for `private`/`final`/`static` methods. (3) `new MyService()` objects aren't wrapped — only container-managed beans are. (4) Proxies are created in `postProcessAfterInitialization`, i.e. *after* `@PostConstruct` runs. **One-line fix menu:** move the method to another bean, inject the proxy of yourself (`@Lazy`), or use the programmatic equivalent (`TransactionTemplate`, `CacheManager`, `TaskExecutor`).

### Defaults worth memorizing

| Setting | Default |
|---|---|
| Embedded Tomcat port | 8080 |
| Tomcat max threads | 200 (`server.tomcat.threads.max`) |
| Tomcat accept queue | 100 (`accept-count`) |
| Hikari max pool size | 10 |
| `@Transactional` isolation | `DEFAULT` (DB's own) |
| `@Transactional` propagation | `REQUIRED` |
| Rollback rule | Unchecked exceptions only |
| `spring.jpa.open-in-view` | **true** (turn it off) |
| `ddl-auto` | `create-drop` with embedded DB, `none` otherwise |
| Actuator exposed over HTTP | `health`, `info` only |
| `@ManyToOne` / `@OneToOne` fetch | **EAGER** (change it) |
| `@OneToMany` / `@ManyToMany` fetch | LAZY |
| Scheduler pool size | 1 |
| Jackson | Fails on unknown properties = **false** in Boot |

**Why these are worth knowing:** each default is a common production trap. *Tomcat 200 threads* = at most 200 requests handled at once; the rest queue (accept queue 100) then are refused — so a slow downstream with no timeout exhausts it. *Hikari 10 connections* = at most 10 concurrent DB statements per instance; more concurrent requests wait for a connection. *Open-in-view true* silently holds a connection for the whole request, and *EAGER `@ManyToOne`* silently causes extra queries — both are "works in dev, hurts in prod". *Scheduler pool size 1* means a slow `@Scheduled` job delays all the others. *`ddl-auto`* defaults to creating the schema only for embedded DBs (H2) — so tests may pass on an auto-created schema that production's Flyway-managed schema doesn't match. *Jackson tolerating unknown fields* helps backward-compatible APIs (clients can add fields) but also hides typos in request payloads.

### One-liners you can drop as summaries

- "Auto-configuration is just `@Conditional` beans that back off when you define your own."
- "Constructor injection, because a missing dependency should break the build, not the customer."
- "Entities never leave the service layer."
- "At-least-once delivery plus an idempotent consumer equals effectively exactly-once."
- "Timeouts, retries with jitter, circuit breakers, bulkheads — in that order of importance."
- "Locks reduce the probability of double-processing; idempotency removes the consequence."
- "A smaller connection pool is often faster than a bigger one."
- "Optional dependencies must have optional latency budgets."
- "Consistency requirements should drive service boundaries."

---

## 21. Behavioural / Lead-Level Questions

At 9 years, roughly a third of the interview is about influence, judgement, and ownership. Prepare **2–3 STAR stories** you can reshape to fit many questions.

| Question | What they're testing | What to include |
|---|---|---|
| Toughest technical decision you made | Judgement, trade-off reasoning | Options considered, criteria, what you gave up, how you validated, ADR |
| A production incident you owned | Calm under pressure, systems thinking | Timeline, MTTR, communication, blameless RCA, systemic fix |
| A time you disagreed with your manager/architect | Influence without authority | Data you brought, how you disagreed and committed |
| Mentoring a struggling developer | Leadership | Specific feedback, pairing, measurable improvement |
| A technical decision you got wrong | Humility, learning | Own it plainly, what signal you missed, what you changed in your process |
| How you handle tech debt vs features | Pragmatism | Debt register, 15–20% capacity, tie debt to business risk in their language |
| How you review code | Standards | Correctness → design → readability; ask questions not commands; automate style |
| Why are you leaving / what do you want next | Motivation fit | Forward-looking, no blaming; concrete about the scope you want |

**Two habits that read as senior:** quantify outcomes ("cut p99 by 60%", "reduced deploys from 2 weeks to daily"), and describe how you made a fix **systemic** (a lint rule, template, alert, or runbook) rather than a one-off.

---

## 22. Last-Minute Cheat Sheet

**If you only revise 20 things, revise these:**

1. Auto-configuration mechanism: `AutoConfiguration.imports` + `@Conditional*` + `@ConditionalOnMissingBean` back-off + the `--debug` condition report.
2. `@SpringBootApplication` decomposition and the component-scan root-package trap.
3. Property precedence: command line > system props > env vars > profile yml > yml > defaults.
4. `@ConfigurationProperties` + `@Validated` over `@Value`.
5. Constructor injection and the four reasons why.
6. Bean lifecycle order and *why proxies exist only after initialization*.
7. Proxy limitations (self-invocation / private) applying to `@Transactional`, `@Async`, `@Cacheable`.
8. Propagation levels, especially `REQUIRES_NEW` and its connection-pool danger.
9. Rollback: unchecked rolls back, checked commits, `rollbackFor`.
10. N+1: how to detect it and the four fixes (`JOIN FETCH`, `@EntityGraph`, `@BatchSize`, projections) — plus the `JOIN FETCH` + `Pageable` in-memory trap.
11. `open-in-view=false` and "entities never leave the service layer."
12. Optimistic (`@Version`) vs pessimistic locking, and atomic `UPDATE ... WHERE` as a third option.
13. HikariCP sizing and the causes of pool exhaustion.
14. Spring Security 6 `SecurityFilterChain`, filter-chain order, `hasRole` vs `hasAuthority`.
15. JWT design: short TTL + refresh rotation, asymmetric signing, revocation is the hard part.
16. Resilience4j: circuit breaker states, decorator order, bulkhead, and **timeouts first**.
17. Transactional outbox + idempotent consumers for DB↔broker consistency.
18. Actuator health probes mapped to K8s liveness/readiness + graceful shutdown.
19. Test slices (`@WebMvcTest`/`@DataJpaTest`/`@SpringBootTest`), Testcontainers + `@ServiceConnection`, context caching.
20. Boot 3 changes: Jakarta, Micrometer Tracing, `ProblemDetail`, `RestClient`, **virtual threads** and their caveats.

**Interview delivery tips**
- Answer in **layers**: one-sentence definition → why it exists → a production example → trade-off. Stop and let them steer.
- When you don't know: "I haven't used X in production; my understanding is Y, and I'd verify by Z." Never bluff specifics — a wrong confident detail costs more than an admitted gap.
- **Volunteer the failure mode.** Saying "and the trap here is…" is the single strongest senior signal.
- Have numbers ready from your own work: RPS, data volume, team size, latency, incident count, cost saved.
- Ask *them* good questions: how do they handle on-call, what's their deploy frequency, how is tech debt funded, what does the service topology look like.

---

*Good luck. Read the one-line hooks the morning of the interview and the scenarios the night before — the scenarios are what they'll remember you for.*
