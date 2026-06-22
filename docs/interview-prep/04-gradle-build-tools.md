# Gradle & Build Tools — Interview Questions

> Phase added: Phase 0
> Real project reference: PulseBoard uses Gradle 8.8 multi-module build

---

## Q1: What is Gradle and how is it different from Maven?

Both are build automation tools for Java projects — they compile code, manage dependencies, 
run tests, and package artifacts.

| Feature | Gradle | Maven |
|---|---|---|
| Config language | Groovy/Kotlin DSL | XML |
| Performance | Incremental builds, build cache, faster | Slower, full rebuild |
| Flexibility | Highly customizable | Convention over configuration |
| Multi-module | Excellent support | Good but more verbose |
| Industry trend | Preferred for new projects | Legacy projects |

In PulseBoard I chose Gradle because of its multi-module support — one root `build.gradle` 
manages shared dependencies for all 10 services.

---

## Q2: What is a multi-module Gradle project?

A multi-module project has one root project containing multiple subprojects (modules). 
Each module is a separate Spring Boot service but shares common configuration from the root.

**PulseBoard structure:**
pulseboard/ ← root project
├── build.gradle ← shared config for ALL services
├── settings.gradle ← declares all submodules
└── services/
├── auth-service/
│ └── build.gradle ← service-specific dependencies only
├── team-service/
│ └── build.gradle
└── ...

The root `build.gradle` applies `spring-boot-starter-actuator`, Lombok, and test dependencies 
to all services. Each service `build.gradle` only adds its specific dependencies like 
`spring-cloud-starter-netflix-eureka-client`.

---

## Q3: What is the Gradle wrapper and why is it important?

The Gradle wrapper (`gradlew` / `gradlew.bat`) is a script that downloads and uses a specific 
version of Gradle defined in `gradle/wrapper/gradle-wrapper.properties`. 

**Why it matters:**
- Everyone on the team uses the same Gradle version — no "works on my machine" issues
- CI/CD doesn't need Gradle pre-installed
- Version is committed to source control

In PulseBoard I use Gradle 8.8 via the wrapper. I discovered this firsthand — Gradle 9.5 was 
incompatible with the Spring Boot dependency management plugin, so I locked the wrapper to 8.8.

---

## Q4: What is dependency management in Gradle?

Dependency management controls which version of a library is used when multiple dependencies 
pull in different versions of the same library (version conflicts).

In PulseBoard I use Spring Boot's BOM (Bill of Materials):
```groovy
dependencyManagement {
    imports {
        mavenBom "org.springframework.boot:spring-boot-dependencies:3.2.5"
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.1"
    }
}
A BOM is a special POM that declares compatible versions of all related libraries. By importing
it, I don't need to specify versions for Spring libraries — the BOM ensures they're all
compatible with each other.
Q5: What is the difference between implementation, compileOnly, and runtimeOnly?
These are Gradle dependency configurations:

CONFIGURATION
AVAILABLE AT COMPILE
AVAILABLE AT RUNTIME
IN JAR
implementation
✅
✅
✅
compileOnly
✅
❌
❌
runtimeOnly
❌
✅
✅
testImplementation
✅ (test only)
✅ (test only)
❌


In PulseBoard:

implementation 'org.springframework.boot:spring-boot-starter-web' — needed at compile and runtime
compileOnly 'org.projectlombok:lombok' — Lombok only needed during compilation for code generation
runtimeOnly 'com.h2database:h2' — H2 driver only needed at runtime, not referenced in code
annotationProcessor 'org.projectlombok:lombok' — runs Lombok annotation processor during compilation

