# Real Problems Solved — Interview Stories

> These are real problems hit while building PulseBoard and how they were solved.
> "Tell me about a challenge you faced" — use these stories.

---

## Problem 1: Gradle 9.5 Incompatibility with Spring Boot

**Situation:** I installed Gradle 9.5 (latest) and tried to build the first Spring Boot 
service. Got a cryptic `getArtifacts` error — `java.util.Set getArtifacts(Spec)` not found.

**Investigation:** Read the full stack trace, identified the error was in the Spring Boot 
dependency management plugin, not in my code. Searched the Spring Cloud GitHub issues and 
found that the `io.spring.dependency-management` plugin uses an internal Gradle API that was 
removed in Gradle 9.

**Solution:** Downgraded to Gradle 8.8 which is fully compatible with Spring Boot 3.2.5. 
Updated `gradle-wrapper.properties` to lock the version, ensuring everyone on the project 
uses the same version.

**Lesson:** Always check compatibility matrices before using the latest version of build tools. 
Lock your tool versions in the wrapper/lockfile.

---

## Problem 2: Corporate SSL Inspection Blocking Maven Downloads

**Situation:** After fixing the Gradle version, new dependency downloads were failing with 
`PKIX path building failed: unable to find valid certification path`. Maven Central was 
unreachable from the build tool.

**Investigation:** The browser could reach `repo1.maven.org` but Gradle couldn't. Recognized 
this as corporate SSL inspection — the company's proxy was intercepting HTTPS traffic and 
replacing certificates with its own, which Java's default trust store didn't recognize.

**Solution:** Temporarily disabled the SSL inspection for dependency downloads. Long-term 
solution would be to import the corporate certificate into Java's cacerts trust store.

**Lesson:** Corporate network security policies affect developer tools differently than browsers. 
Java has its own certificate trust store separate from the OS.

---

## Problem 3: Service Registration Order Dependency

**Situation:** Started config-server before discovery-server. Config-server logs were full of 
`Connection refused` errors trying to register with Eureka at `localhost:8761`. 

**Investigation:** Read the logs carefully — config-server was starting up and immediately 
trying to register with Eureka, but Eureka wasn't running yet. The services have a startup 
order dependency that isn't enforced automatically in local development.

**Solution:** Established a strict startup order — discovery-server first, config-server 
second, api-gateway third, then business services. Documented this in the README.

**Production solution:** Kubernetes readiness probes and init containers handle this 
automatically — a service doesn't start until its dependencies are healthy.

**Lesson:** In distributed systems, startup order matters. Design for resilience — services 
should retry registration with backoff rather than failing hard on startup.

---

## Problem 4: Git Push Failing with Permission Denied

**Situation:** `git push origin main` failed with `refusing to allow a Personal Access Token 
to create or update workflow files without workflow scope`.

**Investigation:** GitHub requires a specific `workflow` scope to push to `.github/workflows/` 
directory. My token only had `repo` scope.

**Solution:** Two options — add `workflow` scope to the token, or remove the workflows 
placeholder file from Git tracking. Chose the second option since actual GitHub Actions 
workflow files are a Phase 6 task anyway.

**Lesson:** GitHub token scopes are granular. The `repo` scope covers most operations but 
workflow files require explicit permission.

---

## Problem 5: `.gradle` Folder Causing Git Issues

**Situation:** `git add .` failed with `Permission denied` on `.gradle/8.8/checksums/checksums.lock` 
because Gradle daemon was holding the file open.

**Investigation:** The `.gradle` folder is Gradle's local cache — it should never be committed 
to Git. It was getting picked up by `git add .` because it wasn't in `.gitignore`.

**Solution:** Added `.gradle/`, `build/`, and `.idea/` to `.gitignore`. Used `git rm --cached` 
to remove them from tracking. Used explicit `git add [specific files]` instead of `git add .` 
when Gradle is running.

**Lesson:** Always set up `.gitignore` before the first commit. Generated folders (build 
output, IDE files, dependency caches) should never be in version control.

---

## Problem 6: Gradle Wrapper Not Working After Manual Setup

**Situation:** Created `gradlew.bat` by copying from the Gradle installation bin folder. 
Running it gave `Unable to access jarfile gradle-gradle-cli-main-9.5.0.jar`.

**Investigation:** The `gradle.bat` in the bin folder is the full Gradle launcher script — 
it points to the full Gradle installation. The `gradlew.bat` wrapper script is a different 
file that only knows how to download and run a specific Gradle distribution using 
`gradle-wrapper.jar`.

**Solution:** Created proper `gradlew.bat` with the standard wrapper script content, and 
downloaded `gradle-wrapper.jar` separately from the Gradle GitHub repository.

**Lesson:** The Gradle wrapper and the Gradle launcher are different things. The wrapper is 
self-contained and downloads the right version of Gradle automatically.

---

*New problems will be added as we encounter and solve them in later phases.*