# Maven

> **Source:** Lecture 03 (`Notes_Of_Aditya/Lecture 03/code/MavenDemo`)

## Why Maven exists
Real Java projects need dozens of external JARs. Managing them by hand raises questions: where do I download them, which version, what does each JAR itself depend on, and do my teammates have the same versions?
**Maven** is a **project management and build automation tool** that standardizes project structure, dependencies, builds, and packaging. It is independent of Spring.

---

## 1. From `.java` to JAR
```
Hello.java --javac--> Hello.class --JVM--> output
```
- **JAR (Java ARchive)** = a ZIP-like bundle of `.class` files, packages, resources, and metadata.
- **Library JAR**: code for other developers. Usually doesn't contain its dependencies; those are described in its pom.
- **Spring Boot executable ("fat") JAR**: your code **plus** all dependencies **plus** an embedded server. Runnable with `java -jar`.
- **Classpath** = the places where Java searches for classes (your classes plus external JARs).

## 2. What Maven does
1. Standard project structure
2. Compiles code
3. Runs tests
4. Creates JAR/WAR
5. Downloads dependencies, including **transitive** ones
6. Manages versions
7. Uses **plugins** for build tasks

### Convention over Configuration
```
my-maven-project/
├── pom.xml
├── src/main/java        ← application code
├── src/main/resources   ← application.properties, configs, templates
├── src/test/java        ← tests
├── src/test/resources
└── target/              ← generated output (classes, JAR); safe to delete
```

## 3. `pom.xml` – Project Object Model (the heart of Maven)
```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <!-- Maven coordinates (GAV) uniquely identify a project/dependency -->
    <groupId>org.example</groupId>        <!-- organization / domain -->
    <artifactId>MavenDemo</artifactId>    <!-- project/module name -->
    <version>1.0-SNAPSHOT</version>       <!-- SNAPSHOT = still in development -->
    <packaging>jar</packaging>            <!-- jar (default) | war | pom -->

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>7.0.7</version>
        </dependency>
    </dependencies>
</project>
```
- **Dependency** = a library your **code** uses.
- **Plugin** = a tool **Maven** uses during the build (compiler, surefire for tests, `spring-boot-maven-plugin`).
- **Transitive dependency**: you add A, A needs B, and B needs C. Maven fetches all three.
- **Starter** (e.g. `spring-boot-starter-web`) = one dependency that brings in a whole group of related ones.

> In this course, `Lecture 05/code/SpringCoreDemo` adds `SpringCoreDemo2` (installed locally with `mvn install`) as a dependency. That's an example of using your own artifact from `~/.m2`.

## 4. Repositories
| Type | Where | Purpose |
|---|---|---|
| **Local** | `~/.m2/repository` | Cache of downloaded JARs + your `mvn install`ed artifacts |
| **Maven Central** | Public internet | Open-source libraries (Spring, Hibernate, Jackson…) |
| **Remote / Private** | Nexus, Artifactory, GitHub Packages | Company-internal artifacts |

**Lookup order:** read pom → check **local** repo → if missing, download from remote → store in local → use it. That's why the **first build is slow**.

- `settings.xml` (`~/.m2/settings.xml`) = user/machine-level config: credentials, proxies, mirrors.
  `pom.xml` = project-level config.
- If dependencies are corrupted (e.g. a half-downloaded JAR), delete that folder in `~/.m2/repository` and Maven re-downloads it.

## 5. Lifecycles and phases
Three lifecycles: **clean**, **default** (build), and **site** (docs/reports).

**Default lifecycle phases (in order):**
```
validate → compile → test → package → verify → install → deploy
```
| Phase | Does |
|---|---|
| validate | Checks the pom and project are valid |
| compile | `src/main/java` → `target/classes` |
| test | Runs tests from `src/test/java` |
| package | Creates `target/MavenDemo-1.0-SNAPSHOT.jar` |
| verify | Extra checks (integration tests, quality gates) |
| install | Copies the artifact to local `~/.m2/repository` |
| deploy | Uploads the artifact to a remote repo (Nexus etc.). This is **not** the same as pushing code to GitHub |

> **Golden rule:** running a phase runs **all earlier phases** first.
> `mvn package` = validate → compile → test → package.

### `mvn clean install`
`clean` (delete `target/`) **+** default lifecycle up to `install`. The result is a fresh build that is tested, packaged, and installed into `~/.m2`.

**Archetype** = a project template (Spring Initializr plays this role for Boot).

---

## Common commands
```bash
mvn clean            # delete target/
mvn compile
mvn test
mvn package          # build jar in target/
mvn clean install    # fresh build + install to ~/.m2
mvn dependency:tree  # see transitive dependencies
mvn spring-boot:run  # run a Boot app (via plugin)
```

## Interview Q&A
**Q1. What are Maven coordinates?**
`groupId:artifactId:version` (GAV). Together they uniquely identify an artifact.

**Q2. Dependency vs plugin?**
A dependency is used by your application code. A plugin is used by Maven to perform build tasks.

**Q3. What is a transitive dependency? How do you exclude one?**
It's a dependency of your dependency. Exclude it with `<exclusions><exclusion>…</exclusion></exclusions>` inside the dependency.

**Q4. What does `mvn install` do vs `mvn deploy`?**
`install` puts the artifact in the **local** `~/.m2` repo. `deploy` uploads it to a **remote** repository.

**Q5. If I run `mvn package`, does it run tests?**
Yes, because every earlier phase runs (validate, compile, test). Skip them with `-DskipTests`.

**Q6. What is SNAPSHOT?**
A development version that can change. Release versions (`1.0.0`) are immutable.

**Q7. `pom.xml` vs `settings.xml`?**
`pom.xml` is project-level config. `settings.xml` is user/machine-level config (credentials, mirrors, proxies).

**Q8. Fat JAR vs normal JAR?**
A fat JAR includes all dependencies (and the embedded server for Boot). A normal library JAR contains only its own classes.

## Quick revision
- `pom.xml` = heart. GAV = identity.
- Local `.m2` → Central → Remote/private.
- Default lifecycle: validate, compile, test, package, verify, install, deploy. Earlier phases always run.
- `target/` is generated output and safe to delete.
