# Docker for Spring Boot

> **Part of:** Spring gap topics (63–71) · 🟦 Same concepts as Dockerizing a NestJS app, plus JVM-specific memory tuning.

## Why this topic matters
Product companies ship services as containers on Kubernetes. Expect "how do you containerize a Spring Boot app?", "why is my container OOM-killed?", and "how do you make images small and fast?".

---

## 1. Basics
- **Image** = an immutable template (layers). **Container** = a running instance of an image.
- **Layers** are cached. Order Dockerfile steps from least to most frequently changing.
- Containers are ephemeral: config comes via **env vars**, state goes to external DBs/volumes.

## 2. Multi-stage Dockerfile
```dockerfile
# ---- build stage ----
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw -q dependency:go-offline          # cached unless pom.xml changes
COPY src src
RUN ./mvnw -q package -DskipTests
RUN java -Djarmode=tools -jar target/*.jar extract --layers --launcher --destination extracted   # Boot 3.3+ layered extract

# ---- runtime stage ----
FROM eclipse-temurin:21-jre                    # JRE only → smaller attack surface
WORKDIR /app
RUN useradd --system --uid 1001 appuser
COPY --from=build /app/extracted/dependencies/ ./
COPY --from=build /app/extracted/spring-boot-loader/ ./
COPY --from=build /app/extracted/snapshot-dependencies/ ./
COPY --from=build /app/extracted/application/ ./
USER appuser                                   # don't run as root
EXPOSE 8080
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError"
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```
Why **layered jars**: dependencies (rarely change) and app classes (change every build) go in separate layers, so a code change rebuilds and pushes only a few KB instead of the whole 80 MB fat jar.
(Simpler but less cache-friendly: `COPY target/app.jar app.jar` + `ENTRYPOINT ["java","-jar","app.jar"]`.)

### Without a Dockerfile: Buildpacks
```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myorg/order-service:1.0
```
Cloud Native Buildpacks produce an optimized, layered, non-root image with JVM memory auto-configured.

## 3. JVM in containers (common interview question)
- Modern JVMs (10+, 8u191+) are **container-aware**: they read the cgroup CPU/memory limits.
- By default the heap max is only **25% of container memory**. Set `-XX:MaxRAMPercentage=70–75` so the remainder stays free for **non-heap** memory (metaspace, thread stacks, code cache, direct buffers, GC structures).
- `OOMKilled` (exit code 137) = the **container** exceeded its memory limit (heap + non-heap), killed by the kernel. That's different from a Java `OutOfMemoryError`. Fix by lowering the heap percentage, reducing threads, or raising the limit.
- Set `-XX:+ExitOnOutOfMemoryError` so Kubernetes restarts a broken pod instead of leaving it limping.
- Too few CPUs (e.g. a 0.5 CPU limit) means slow startup/JIT and a reduced default GC/thread count. Give Java services at least 1–2 vCPU.

## 4. docker-compose for local dev
```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/student_db     # service name as host
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
    depends_on:
      db: { condition: service_healthy }
  db:
    image: mysql:8.4
    environment: { MYSQL_ROOT_PASSWORD: "${DB_PASSWORD}", MYSQL_DATABASE: student_db }
    ports: ["3306:3306"]
    volumes: [db-data:/var/lib/mysql]
    healthcheck: { test: ["CMD", "mysqladmin", "ping", "-h", "localhost"], interval: 5s, retries: 10 }
  redis:
    image: redis:7
volumes: { db-data: {} }
```
- **Relaxed binding**: the env var `SPRING_DATASOURCE_URL` maps to `spring.datasource.url` ([16](16_Externalized%20Configuration%20-%20application.properties%2C%20Value%20and%20ConfigurationProperties.md)).
- Inside the compose network, use the **service name** (`db`), not `localhost`.
- Boot 3.1+ **Docker Compose support** (`spring-boot-docker-compose`) starts `compose.yaml` services automatically on `bootRun` and wires the connection details.

## 5. Kubernetes essentials (awareness level)
```yaml
resources:
  requests: { cpu: "500m", memory: "768Mi" }
  limits:   { memory: "1Gi" }
livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 }, initialDelaySeconds: 30 }
readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 } }
env:
  - name: SPRING_PROFILES_ACTIVE
    value: prod
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom: { secretKeyRef: { name: db-secret, key: password } }
```
Deployment (replicas, rolling updates), Service (stable DNS + load balancing), ConfigMap/Secret (config), HPA (autoscaling on CPU/custom metrics). Graceful shutdown + readiness probes give zero-downtime rollouts ([66](66_Actuator%2C%20Logging%20and%20Observability.md)).

## 6. Best practices checklist
- Multi-stage builds, JRE/slim base images, layered jars, pinned versions (not `latest`).
- Non-root user; no secrets in the image (use env/secret managers); `.dockerignore` (`target/`, `.git`).
- One process per container; logs to **stdout** (JSON); config via env.
- Health probes, resource requests/limits, `MaxRAMPercentage`.
- Startup speed: CDS/AppCDS, Spring AOT, or a GraalVM native image (`./mvnw -Pnative spring-boot:build-image`) for fast cold starts (serverless).

---

## Interview Q&A
**Q1. How do you containerize a Spring Boot app efficiently?**
Multi-stage build (JDK to build, JRE to run), layered jar extraction for caching, non-root user, JVM container flags. Or `spring-boot:build-image` buildpacks.

**Q2. Why multi-stage builds?**
The final image excludes build tools and sources: smaller, faster, more secure.

**Q3. Why did my pod get OOMKilled when the heap wasn't full?**
Container memory = heap + metaspace + thread stacks + direct buffers + more. The total exceeded the limit. Lower `MaxRAMPercentage`, reduce threads, or raise the limit.

**Q4. How does a Spring Boot container get its configuration?**
Env vars (relaxed binding: `SPRING_DATASOURCE_URL`), ConfigMaps/Secrets, profiles via `SPRING_PROFILES_ACTIVE`, or a config server.

**Q5. Why can't the app connect to `localhost:3306` in docker-compose?**
`localhost` is the app container itself. Use the DB service name (`db:3306`).

**Q6. What are layered jars?**
The jar split into dependency / loader / snapshot / application layers so Docker caches the unchanged dependencies between builds.

## Quick revision
- Multi-stage + JRE base + layered jars + non-root + `.dockerignore`.
- JVM in containers: `MaxRAMPercentage≈75`, leave room for non-heap; OOMKilled (137) ≠ Java OOM.
- Compose: service names as hosts, env-var relaxed binding, healthchecks.
- Kubernetes: requests/limits, liveness/readiness → Actuator probes, secrets via env.
