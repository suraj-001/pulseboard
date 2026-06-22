# Spring Boot Core — Interview Questions

> Phase added: Phase 0
> Real project reference: All 10 PulseBoard services built with Spring Boot 3.2.5

---

## Q1: What is Spring Boot and how is it different from Spring Framework?

Spring Framework is the core dependency injection and application framework — it gives you IoC 
container, AOP, data access, and MVC. But configuring it requires significant XML or Java 
config boilerplate. Spring Boot is an opinionated layer on top of Spring that provides:

- **Auto-configuration** — detects what's on your classpath and configures it automatically
- **Starter dependencies** — `spring-boot-starter-web` pulls in Tomcat, Spring MVC, Jackson
- **Embedded server** — no need to deploy to external Tomcat, the server runs inside your JAR
- **Production-ready features** — actuator, health checks, metrics out of the box

In PulseBoard, every service is a Spring Boot app. Adding `spring-boot-starter-web` to 
`auth-service` gives me a running REST server with zero XML configuration.

---

## Q2: What does `@SpringBootApplication` do?

It is a convenience annotation that combines three annotations:

- `@Configuration` — marks the class as a source of bean definitions
- `@EnableAutoConfiguration` — tells Spring Boot to auto-configure based on classpath
- `@ComponentScan` — scans the package and sub-packages for `@Component`, `@Service`, etc.

In PulseBoard every service has exactly one class annotated with `@SpringBootApplication` — 
for example `AuthServiceApplication`, `TeamServiceApplication`. This is the entry point.

---

## Q3: What is Dependency Injection and how does Spring implement it?

Dependency Injection is a design pattern where an object receives its dependencies from outside 
rather than creating them itself. This makes code loosely coupled and testable.

Spring implements DI through its IoC (Inversion of Control) container:
1. On startup, Spring scans for `@Component`, `@Service`, `@Repository`, `@Controller` classes
2. It creates instances (beans) and stores them in the ApplicationContext
3. Wherever `@Autowired` or constructor injection is used, Spring injects the right bean

**Three types of injection:**
- Constructor injection (recommended — explicit, testable, supports final fields)
- Setter injection (optional dependencies)
- Field injection (avoid — hides dependencies, hard to test)

In PulseBoard I use constructor injection everywhere.

---

## Q4: What is the difference between `@Component`, `@Service`, `@Repository`, `@Controller`?

All are specializations of `@Component` — all register a class as a Spring bean.

| Annotation | Layer | Extra Behavior |
|---|---|---|
| `@Component` | Any | None — generic bean |
| `@Service` | Business logic | None — semantic only |
| `@Repository` | Data access | Translates persistence exceptions |
| `@Controller` | Web layer | Handles HTTP requests |
| `@RestController` | Web layer | `@Controller` + `@ResponseBody` |

In PulseBoard: business logic classes use `@Service`, REST endpoints use `@RestController`.

---

## Q5: What is Spring Boot Auto-configuration? How does it work?

Auto-configuration is Spring Boot's ability to automatically configure your application based 
on the JARs present on the classpath and properties you've defined.

How it works:
1. `@EnableAutoConfiguration` triggers the process
2. Spring Boot reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
3. Each auto-configuration class is conditionally applied using `@ConditionalOnClass`, 
   `@ConditionalOnMissingBean`, `@ConditionalOnProperty` etc.

Example: If `spring-boot-starter-data-jpa` is on the classpath and you have a DataSource 
configured, Spring Boot automatically creates a `LocalContainerEntityManagerFactoryBean`, 
`JpaTransactionManager`, and `PlatformTransactionManager` for you.

In PulseBoard, adding H2 to the classpath + JPA starter automatically creates an in-memory 
datasource and Hibernate session factory — zero configuration needed.

---

## Q6: What is Spring Actuator and what endpoints does it provide?

Spring Actuator provides production-ready features to monitor and manage your application. 
It exposes HTTP endpoints for health checks, metrics, and more.

Key endpoints:
- `/actuator/health` — returns UP/DOWN status of the application
- `/actuator/info` — custom application info
- `/actuator/metrics` — JVM metrics, HTTP request stats
- `/actuator/env` — current environment properties
- `/actuator/beans` — all Spring beans registered

In PulseBoard, every service exposes `health` and `info` via actuator. The API Gateway's 
health is verified at `http://localhost:8080/actuator/health`. Eureka uses the health endpoint 
to determine if a service is UP or DOWN.

---

## Q7: What is the difference between `@Bean` and `@Component`?

`@Component` is used on a class — Spring detects and registers it automatically via component 
scanning. `@Bean` is used on a method inside a `@Configuration` class — it explicitly declares 
a bean and gives you full control over its creation, useful when you need to configure 
third-party classes you can't annotate.

```java
// @Component — on your own class
@Service
public class MyService { }

// @Bean — for third-party or complex creation logic
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}

## Q8: What is application.yml vs application.properties?
Both configure Spring Boot application properties. YAML (.yml) is preferred for:

Hierarchical structure — cleaner to read
Multi-document support
Less repetition for nested properties
In PulseBoard every service uses application.yml. Example from auth-service:

yaml

server:
  port: 8081
spring:
  application:
    name: auth-service
  datasource:
    url: jdbc:h2:mem:authdb
Equivalent in .properties:

properties

server.port=8081
spring.application.name=auth-service
spring.datasource.url=jdbc:h2:mem:authdb

Q9: What are Spring Profiles and how do you use them?
Profiles allow different configurations for different environments (dev, test, prod). You
activate a profile with spring.profiles.active=prod and create profile-specific files like
application-prod.yml.

In PulseBoard, config-server uses the native profile:
spring:
  profiles:
    active: native

This tells the config server to read configuration from local classpath instead of a Git
repository — appropriate for local development.


Q10: What is the Bean lifecycle in Spring?
Instantiation — Spring creates the bean instance
Dependency Injection — dependencies are injected
@PostConstruct — custom initialization logic runs
Bean is ready — used by the application
@PreDestroy — cleanup logic runs before destruction
Destruction — bean is removed from context
You can hook into the lifecycle using @PostConstruct and @PreDestroy annotations or by
implementing InitializingBean and DisposableBean interfaces.
