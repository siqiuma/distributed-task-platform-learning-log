
## Section 1 – Environment Setup & Architecture Alignment

### What I worked on

- Checked machine architecture to confirm Apple Silicon (ARM64)
```bash
uname -m
```

- Investigated JDK mismatch errors in IntelliJ (x86_64 vs ARM64)

- Identified that an Oracle JDK (x86_64) was being used incorrectly

- Installed Temurin (Eclipse Adoptium) OpenJDK 21 (ARM64)

- Verified installed JDKs:
```bash
/usr/libexec/java_home -V
```

- Updated shell configuration to point to the correct JDK:
```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home
export PATH="$JAVA_HOME/bin:$PATH"
```

- Reloaded shell configuration:
```bash
source ~/.zshrc
```

- Verified Java version and architecture:
```bash
java -version
java -XshowSettings:properties -version 2>&1 | grep os.arch
```
### What I learned

- Apple Silicon requires ARM64-compatible JDKs; mixing x86 Java causes subtle and hard-to-debug issues

- IDE warnings about architecture mismatches are important and should not be ignored

- Verifying tooling via CLI is more reliable than relying on IDE state

### Decision(s) made

- Standardize on Java 21 (Temurin, ARM64) for the entire project

- Fix architecture alignment before writing any business logic

### Problems / blockers

- .zshrc could not be edited due to permission issues

- Resolved by editing with elevated permissions:
```bash
sudo nano ~/.zshrc
```
### Open questions / next steps


- Ensure build tools (Maven) also use the same Java runtime


- Verify IntelliJ Project SDK and build JVM alignment

## Section 2 – Maven Modernization & Build Stabilization
### What I worked on

- Checked Maven version and runtime:
```bash
mvn -v
```

- Discovered system Maven (3.8.1) was running with Java 13 (x86_64)

- Introduced Maven Wrapper to the project:
```bash
mvn -N wrapper:wrapper -Dmaven=3.9.9
```

- Verified Maven Wrapper configuration:
```bash
./mvnw -v
```

- Confirmed Maven Wrapper runs with:

* Maven 3.9.9

* Java 21

* ARM64 architecture

- Adopted ./mvnw as the standard build command

- Ran full project build to validate setup:
```bash
./mvnw clean test
```
### What I learned

- Maven Wrapper is the correct way to manage Maven versions in professional Java projects

- Global Maven installations can silently use outdated Java versions

- Tooling reproducibility is critical for CI and team environments

### Decision(s) made

- Use Maven Wrapper exclusively (./mvnw)

- Do not rely on system-installed Maven going forward

### Problems / blockers

- Initial confusion about why Maven was using a different Java version than the terminal

- Root cause was the system Maven installation, not project configuration

### Open questions / next steps

- Review pom.xml to ensure Java 21 compatibility

- Add missing dependencies required for upcoming features

## Section 3 – Spring Boot Initialization & Validation Setup
### What I worked on

- Created Spring Boot project using Maven with:

* Java 21

* Spring Boot 4.0.0

- Verified project structure and main application entry point

- Added missing dependency for request validation:
```bash
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

- Rebuilt project after dependency update:
```bash
./mvnw clean test
```

- Confirmed build success and application context loading

- Implemented a simple /health endpoint as a sanity check

- Differentiated between:

/health (custom application endpoint)

/actuator/health (Spring Boot Actuator endpoint)

### What I learned

- Validation annotations (@Valid, @NotBlank) require explicit dependencies

- Actuator endpoints and application endpoints serve different purposes

- A small sanity endpoint helps confirm correct wiring early

### Decision(s) made

- Keep Spring Boot Actuator enabled for observability

- Maintain a custom /health endpoint for application-level checks

- Delay business logic until tooling and validation are fully correct

### Problems / blockers

- Confusion when seeing JSON health output instead of "OK"

- Resolved by realizing the wrong endpoint (/actuator/health) was being called

### Open questions / next steps

- Introduce persistence layer (JPA + H2)

- Model core domain (Task, TaskStatus)

- Implement first real REST APIs
