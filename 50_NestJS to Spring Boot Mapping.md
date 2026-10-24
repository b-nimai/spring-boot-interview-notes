# NestJS → Spring Boot Mapping

> **For:** NestJS developers moving to Spring Boot. Each section shows NestJS code next to the Spring Boot equivalent and links to the detailed note.
> NestJS was heavily inspired by Spring/Angular, so most concepts map one-to-one. The real differences are the **runtime model** (event loop vs thread-per-request) and the **language** (TypeScript vs Java).

---

## 0. The big picture
| Concept | NestJS | Spring Boot |
|---|---|---|
| Language / runtime | TypeScript on Node.js (V8) | Java (or Kotlin) on the JVM |
| Concurrency model | **Single-threaded event loop**, non-blocking I/O, `async/await` | **Thread-per-request** (Tomcat pool, ~200 threads), blocking I/O by default; virtual threads (Java 21) or WebFlux for reactive |
| DI container | Nest IoC (module-scoped) | Spring IoC / `ApplicationContext` (app-wide) |
| HTTP layer | Express or Fastify adapter | Servlet container (embedded Tomcat) + Spring MVC |
| Package manager / build | npm/yarn + `package.json` + tsc | Maven/Gradle + `pom.xml` + javac |
| Entry point | `main.ts` → `NestFactory.create(AppModule)` | `main()` → `SpringApplication.run(App.class)` |
| Config | `@nestjs/config`, `.env` | `application.properties/yml`, profiles |
| ORM | TypeORM / Prisma / Mongoose | JPA / Hibernate / Spring Data JPA |
| Testing | Jest + `@nestjs/testing` | JUnit 5 + Mockito + Spring Test |

> **Interview line:** "Nest blocks nothing because of the event loop. Spring MVC blocks a pool thread per request, so throughput depends on the pool size and on keeping I/O fast. That's why connection pools, timeouts, and `@Async` matter more in Spring."

---

## 1. Bootstrapping
```ts
// NestJS — main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
  await app.listen(3000);
}
bootstrap();
```
```java
// Spring Boot — Application.java
@SpringBootApplication            // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);   // starts context + embedded Tomcat on 8080
    }
}
// port: server.port=3000 in application.properties
```
→ Notes [02](02_First%20Spring%20Boot%20Application.md), [15](15_Spring%20Boot%20Auto-Configuration.md)

## 2. Modules vs component scanning
```ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService],              // must export to share with other modules
})
export class UserModule {}
```
```java
// No module declarations. Any @Component/@Service/@Repository/@Controller
// under the main class's package is discovered and registered automatically.
package com.app.user;
@Service public class UserService { }
```
| NestJS | Spring |
|---|---|
| Providers are **module-scoped**; must be `exports`ed/`imports`ed | Beans are **global** to the `ApplicationContext` |
| `@Module` wiring by hand | `@ComponentScan` (via `@SpringBootApplication`) + `@Configuration` classes |
| `forRoot()/forFeature()` dynamic modules | Auto-configuration + `@ConfigurationProperties` + starters |
| `@Global()` module | Every bean is already "global" |
Organize Spring code by **package** (feature packages `user/`, `order/`) instead of modules.
→ Notes [06](06_Spring%20IoC%20Container%20and%20ApplicationContext.md), [15](15_Spring%20Boot%20Auto-Configuration.md)

## 3. Providers / DI
```ts
@Injectable()
export class UserService {
  constructor(private readonly userRepo: UserRepository,
              @Inject('PAYMENT') private readonly payment: PaymentGateway) {}
}
// custom providers
{ provide: 'PAYMENT', useClass: StripeGateway }
{ provide: 'CONFIG', useValue: { retries: 3 } }
{ provide: 'CLIENT', useFactory: (cfg: ConfigService) => new Client(cfg.get('URL')), inject: [ConfigService] }
```
```java
@Service
public class UserService {
    private final UserRepository userRepo;
    private final PaymentGateway payment;
    public UserService(UserRepository userRepo, @Qualifier("stripe") PaymentGateway payment) {  // ctor injection
        this.userRepo = userRepo; this.payment = payment;
    }
}
// custom providers → @Bean methods
@Configuration
public class AppConfig {
    @Bean("stripe") PaymentGateway stripe() { return new StripeGateway(); }        // useClass
    @Bean RetryConfig retryConfig() { return new RetryConfig(3); }                  // useValue
    @Bean Client client(@Value("${client.url}") String url) { return new Client(url); }  // useFactory
}
```
| NestJS | Spring |
|---|---|
| `@Injectable()` | `@Component` / `@Service` / `@Repository` |
| Injection token `'PAYMENT'` + `@Inject()` | Bean name + `@Qualifier("…")`, or `@Primary` for the default |
| `useClass`/`useValue`/`useFactory` | `@Bean` method |
| `@Optional()` | `Optional<T>`, `ObjectProvider<T>`, `@Autowired(required=false)` |
| `forwardRef()` (circular deps) | `@Lazy` (but refactor!) |
| Scope `DEFAULT` (singleton) / `REQUEST` / `TRANSIENT` | `singleton` (default) / `request` / `prototype` |
| `OnModuleInit` / `OnModuleDestroy` | `@PostConstruct` / `@PreDestroy` |
| `OnApplicationBootstrap` | `CommandLineRunner` / `ApplicationRunner` / `ApplicationReadyEvent` |
→ Notes [05](05_Inversion%20of%20Control%20and%20Dependency%20Injection.md), [07](07_Types%20of%20Dependency%20Injection%20in%20Spring.md), [08](08_Resolving%20Multiple%20Beans%20-%20Primary%20and%20Qualifier.md), [09](09_Component%20vs%20Bean.md), [10](10_Circular%20Dependency.md), [11](11_Bean%20Scopes.md), [13](13_Bean%20Lifecycle.md), [17](17_CommandLineRunner%20and%20ApplicationRunner.md)

## 4. Controllers and routing
```ts
@Controller('users')
export class UserController {
  constructor(private readonly users: UserService) {}

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) { return this.users.findOne(id); }

  @Get()
  findAll(@Query('page') page = 0) { return this.users.findAll(page); }

  @Post() @HttpCode(201)
  create(@Body() dto: CreateUserDto) { return this.users.create(dto); }

  @Delete(':id') @HttpCode(204)
  remove(@Param('id') id: string, @Headers('x-request-id') rid: string) { return this.users.remove(+id); }
}
```
```java
@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService users;
    public UserController(UserService users) { this.users = users; }

    @GetMapping("/{id}")
    public UserResponse findOne(@PathVariable Long id) { return users.findOne(id); }   // type conversion built in

    @GetMapping
    public List<UserResponse> findAll(@RequestParam(defaultValue = "0") int page) { return users.findAll(page); }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse create(@Valid @RequestBody CreateUserRequest dto) { return users.create(dto); }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> remove(@PathVariable Long id, @RequestHeader("x-request-id") String rid) {
        users.remove(id);
        return ResponseEntity.noContent().build();
    }
}
```
| NestJS | Spring |
|---|---|
| `@Controller('users')` | `@RestController` + `@RequestMapping("/users")` |
| `@Get/@Post/@Put/@Patch/@Delete` | `@GetMapping/@PostMapping/…` |
| `@Param`, `@Query`, `@Body`, `@Headers` | `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader` |
| `@Req()/@Res()` | `HttpServletRequest` / `HttpServletResponse` params |
| `@HttpCode(201)` | `@ResponseStatus(HttpStatus.CREATED)` or `ResponseEntity` |
| Returned object → JSON (class-transformer) | Returned object → JSON (**Jackson**) |
| `app.setGlobalPrefix('api')` | `server.servlet.context-path=/api` |
→ Notes [18](18_Layered%20Architecture%20and%20REST%20CRUD%20API.md), [22](22_Spring%20MVC%20Architecture%20and%20DispatcherServlet.md)

## 5. DTOs + validation (Pipes)
```ts
export class CreateUserDto {
  @IsString() @IsNotEmpty() @Length(2, 50) name: string;
  @IsEmail() email: string;
  @IsInt() @Min(18) age: number;
}
// ValidationPipe → 400 with error messages
```
```java
public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 50) String name,
    @NotBlank @Email String email,
    @NotNull @Min(18) Integer age) {}
// @Valid on the parameter → MethodArgumentNotValidException → 400 (format it in @RestControllerAdvice)
```
| NestJS | Spring |
|---|---|
| class-validator decorators | Jakarta Bean Validation (`@NotBlank`, `@Email`, `@Min`, `@Size`, `@Pattern`) |
| `ValidationPipe` | `@Valid` / `@Validated` (+ `spring-boot-starter-validation`) |
| `whitelist: true` (strip unknown props) | Jackson ignores unknown fields by default (`spring.jackson.deserialization.fail-on-unknown-properties` to reject); **DTOs** prevent mass assignment |
| `ParseIntPipe` | Automatic type conversion (bad input → `MethodArgumentTypeMismatchException` → 400) |
| Custom pipe (transform) | `Converter<S,T>` / `@InitBinder` / custom validator `ConstraintValidator` |
| class-transformer `@Exclude()` | Separate response DTO, or `@JsonIgnore` |
→ Notes [24](24_DTOs%20and%20Bean%20Validation.md)

## 6. Exception handling (Exception Filters)
```ts
throw new NotFoundException(`User ${id} not found`);   // built-in HttpExceptions

@Catch(HttpException)
export class HttpErrorFilter implements ExceptionFilter {
  catch(ex: HttpException, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse();
    res.status(ex.getStatus()).json({ statusCode: ex.getStatus(), message: ex.message });
  }
}
app.useGlobalFilters(new HttpErrorFilter());
```
```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String msg) { super(msg); }
}
// throw new ResourceNotFoundException("User " + id + " not found");
// (or throw new ResponseStatusException(HttpStatus.NOT_FOUND, "...") for a quick built-in)

@RestControllerAdvice                                   // global exception filter
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> notFound(ResourceNotFoundException ex, HttpServletRequest req) {
        return ResponseEntity.status(404).body(new ErrorResponse(404, ex.getMessage(), req.getRequestURI()));
    }
    @ExceptionHandler(MethodArgumentNotValidException.class)   // validation errors
    public ResponseEntity<ErrorResponse> invalid(MethodArgumentNotValidException ex) { ... }
}
```
NestJS's built-in `NotFoundException`, `BadRequestException`, and so on have no exact Spring equivalent. You define custom exceptions plus handlers, use `ResponseStatusException`, or use RFC 7807 `ProblemDetail`.
→ Notes [25](25_Exception%20Handling%20and%20Global%20Exception%20Handler.md)

## 7. Middleware → Filters
```ts
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();
    res.on('finish', () => console.log(`${req.method} ${req.url} ${res.statusCode} ${Date.now() - start}ms`));
    next();
  }
}
consumer.apply(LoggerMiddleware).forRoutes('users');
```
```java
@Component
public class LoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        long start = System.currentTimeMillis();
        try { chain.doFilter(req, res); }                    // = next()
        finally { log.info("{} {} {} {}ms", req.getMethod(), req.getRequestURI(), res.getStatus(),
                           System.currentTimeMillis() - start); }
    }
    @Override protected boolean shouldNotFilter(HttpServletRequest req) {   // = forRoutes()
        return !req.getRequestURI().startsWith("/users");
    }
}
```
`next()` = `chain.doFilter()`. Not calling it = blocking the request. Route scoping uses `FilterRegistrationBean.addUrlPatterns()` or `shouldNotFilter()`.
→ Notes [27](27_Servlet%20Filters%20-%20Basics%2C%20FilterChain%20and%20Order.md), [28](28_Advanced%20Filters%20-%20Wrappers%2C%20FilterRegistrationBean%20and%20OncePerRequestFilter.md)

## 8. Guards → Spring Security (and HandlerInterceptor)
```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(ctx: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', ctx.getHandler());
    const user = ctx.switchToHttp().getRequest().user;
    return !roles || roles.some(r => user.roles.includes(r));
  }
}
@UseGuards(AuthGuard('jwt'), RolesGuard) @Roles('admin')
@Delete(':id') remove() {}
```
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
    @Bean
    SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http.csrf(c -> c.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(a -> a
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers(HttpMethod.DELETE, "/users/**").hasRole("ADMIN")   // route-level "guard"
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()));        // = AuthGuard('jwt')
        return http.build();
    }
}

@PreAuthorize("hasRole('ADMIN')")              // = @Roles('admin') + RolesGuard
@DeleteMapping("/{id}") public void remove(@PathVariable Long id) { }

@GetMapping("/me")
public String me(@AuthenticationPrincipal Jwt jwt) { return jwt.getSubject(); }   // = @Req() req.user
```
| NestJS | Spring |
|---|---|
| `CanActivate` guard | `SecurityFilterChain` rules, `@PreAuthorize`, or a `HandlerInterceptor.preHandle` returning false |
| `AuthGuard('jwt')` (Passport) | `oauth2ResourceServer().jwt()` / custom JWT filter |
| `@nestjs/jwt` `JwtService.sign()` | `JwtEncoder` |
| Passport `LocalStrategy.validate()` | `UserDetailsService` + `DaoAuthenticationProvider` |
| `req.user` | `SecurityContextHolder` / `@AuthenticationPrincipal` / `Authentication` param |
| `Reflector` + custom metadata decorator | Custom annotation + AOP/interceptor reading `HandlerMethod.getMethodAnnotation()` |
| bcrypt package | `BCryptPasswordEncoder` |
| Passport Google strategy | `oauth2Login()` |
→ Notes [29](29_Spring%20MVC%20Interceptors.md), [44](44_Spring%20Security%20Fundamentals.md)–[48](48_OAuth2%20and%20OpenID%20Connect%20-%20Google%20Login.md)

## 9. Interceptors → AOP / HandlerInterceptor / ResponseBodyAdvice
```ts
@Injectable()
export class TimingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    return next.handle().pipe(
      map(data => ({ data, timestamp: new Date() })),        // transform response
      tap(() => console.log(`took ${Date.now() - start}ms`)));
  }
}
```
```java
// timing / logging around any bean method → AOP
@Aspect @Component
public class TimingAspect {
    @Around("@annotation(com.app.TrackTime)")
    public Object time(ProceedingJoinPoint jp) throws Throwable {
        long start = System.nanoTime();
        try { return jp.proceed(); }                           // = next.handle()
        finally { log.info("{} took {}ms", jp.getSignature().getName(), (System.nanoTime() - start) / 1_000_000); }
    }
}
// wrap every controller response → ResponseBodyAdvice
@RestControllerAdvice
public class EnvelopeAdvice implements ResponseBodyAdvice<Object> {
    public boolean supports(MethodParameter m, Class<? extends HttpMessageConverter<?>> c) { return true; }
    public Object beforeBodyWrite(Object body, MethodParameter m, MediaType t,
                                  Class<? extends HttpMessageConverter<?>> c, ServerHttpRequest req, ServerHttpResponse res) {
        return Map.of("data", body, "timestamp", Instant.now());
    }
}
```
| Nest interceptor use | Spring tool |
|---|---|
| Before/after the handler, knowing which handler | `HandlerInterceptor` (preHandle/afterCompletion) |
| Wrap or transform the response body | `ResponseBodyAdvice` |
| Timing, logging, caching, retry around **any** service method | AOP `@Around` (also `@Cacheable`, `@Retryable`) |
| `ClassSerializerInterceptor` | Jackson annotations / response DTOs |
→ Notes [29](29_Spring%20MVC%20Interceptors.md), [30](30_Introduction%20to%20AOP%20and%20Cross-Cutting%20Concerns.md)–[34](34_Custom%20Annotations%20and%20AOP%20Proxy%20Internals.md)

### Request pipeline order
```
NestJS:  Middleware → Guards → Interceptors (before) → Pipes → Handler → Interceptors (after) → Exception filters
Spring:  Filters (incl. Security) → DispatcherServlet → HandlerInterceptor.preHandle → arg resolution + @Valid
         → Controller (→ AOP proxies on services) → ResponseBodyAdvice → postHandle/afterCompletion
         → exceptions → @RestControllerAdvice
```

## 10. Configuration
```ts
ConfigModule.forRoot({ isGlobal: true, envFilePath: `.env.${process.env.NODE_ENV}` });
constructor(private config: ConfigService) { this.url = config.get<string>('DB_URL'); }
```
```java
// application.properties / application-dev.properties / application-prod.properties
// spring.datasource.url=${DB_URL}        ← env var substitution
@Value("${payment.retry-count:3}") int retries;           // single value with default

@ConfigurationProperties(prefix = "payment")              // typed config object (like a config namespace)
public record PaymentProps(String provider, int retryCount) {}
// select environment: SPRING_PROFILES_ACTIVE=prod  (like NODE_ENV)
```
| NestJS | Spring |
|---|---|
| `.env` + `ConfigService.get()` | `application.properties/yml` + `@Value` |
| `registerAs('payment', …)` namespaces | `@ConfigurationProperties(prefix="payment")` |
| Joi validation schema | `@Validated` on `@ConfigurationProperties` |
| `NODE_ENV`-based env files | Profiles: `application-{profile}.properties` + `spring.profiles.active` |
→ Notes [16](16_Externalized%20Configuration%20-%20application.properties%2C%20Value%20and%20ConfigurationProperties.md), [26](26_YAML%20Configuration%20and%20Spring%20Profiles.md)

## 11. Database: TypeORM → JPA / Spring Data
```ts
@Entity()
export class User {
  @PrimaryGeneratedColumn() id: number;
  @Column({ unique: true }) email: string;
  @ManyToOne(() => Department, d => d.users) department: Department;
  @OneToMany(() => Post, p => p.author, { cascade: true }) posts: Post[];
}
// repository
constructor(@InjectRepository(User) private repo: Repository<User>) {}
this.repo.findOne({ where: { email }, relations: ['department'] });
this.repo.find({ skip: 20, take: 10, order: { name: 'ASC' } });
```
```java
@Entity @Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(unique = true) private String email;
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "department_id") private Department department;
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL) private List<Post> posts = new ArrayList<>();
}

public interface UserRepository extends JpaRepository<User, Long> {     // no implementation needed
    @EntityGraph(attributePaths = "department")                         // = relations: ['department']
    Optional<User> findByEmail(String email);                           // derived query from the method name
}
userRepository.findAll(PageRequest.of(2, 10, Sort.by("name")));         // = skip/take/order
```
| TypeORM | JPA / Spring Data |
|---|---|
| `@PrimaryGeneratedColumn` | `@Id @GeneratedValue` |
| `@Column({ nullable, unique })` | `@Column(nullable, unique)` |
| `@ManyToOne(() => X, x => x.ys)` / `@JoinColumn` | `@ManyToOne` + `@JoinColumn` (owning side) |
| inverse side callback | `mappedBy = "field"` |
| `{ cascade: true }` | `cascade = CascadeType.ALL/PERSIST` |
| `eager: true` / `relations: [...]` | `fetch = EAGER` / `@EntityGraph`, `JOIN FETCH` |
| `Repository<User>` | `JpaRepository<User, Long>` |
| `QueryBuilder` | JPQL `@Query`, Criteria API, Specifications |
| `synchronize: true` | `spring.jpa.hibernate.ddl-auto=update` (dev only) |
| TypeORM migrations | Flyway / Liquibase |
| `dataSource.transaction(async m => …)` / `QueryRunner` | `@Transactional` on a service method |
| `repo.save(entity)` to update | Usually **not needed**: managed entities auto-update (dirty checking) inside `@Transactional` |

> ⚠️ Biggest mental shift from TypeORM: **persistence context + dirty checking**. In Spring, changing a loaded entity inside a `@Transactional` method saves it automatically, and changing it **outside** a transaction does **nothing**. This is the source of several lecture bugs (notes 41, 48).
→ Notes [19](19_Spring%20Data%20JPA%20-%20Entity%2C%20Repository%20and%20Database%20Config.md), [37](37_Hibernate%20and%20JPA%20Entity%20Mapping.md)–[43](43_Transaction%20Propagation%20and%20Isolation.md)

## 12. Async model: `async/await` vs threads
```ts
async getDashboard(id: number) {
  const [user, orders] = await Promise.all([this.users.find(id), this.orders.findByUser(id)]);
  return { user, orders };
}
```
```java
// Spring MVC: code is normally synchronous/blocking — each request has its own thread
public Dashboard getDashboard(Long id) {
    return new Dashboard(users.find(id), orders.findByUser(id));
}
// parallel calls → CompletableFuture (like Promise.all)
CompletableFuture<User> u = CompletableFuture.supplyAsync(() -> users.find(id), executor);
CompletableFuture<List<Order>> o = CompletableFuture.supplyAsync(() -> orders.findByUser(id), executor);
CompletableFuture.allOf(u, o).join();
return new Dashboard(u.join(), o.join());
// fire-and-forget → @Async (+ @EnableAsync); cron → @Scheduled (= @nestjs/schedule @Cron)
```
| Node/Nest | Java/Spring |
|---|---|
| `Promise` | `CompletableFuture` |
| `Promise.all` | `CompletableFuture.allOf` |
| `await` | `.join()` / `.get()` (blocks the thread) or just synchronous code |
| Event loop, no shared-memory threads | **Real threads**: shared singletons must be **thread-safe** (no mutable fields in `@Service`!) |
| `@Cron()` (`@nestjs/schedule`) | `@Scheduled(cron = "…")` |
| Bull queues | `@Async`, Kafka/RabbitMQ, Spring Batch |
| RxJS / Observables | Project Reactor (`Mono`/`Flux`) in WebFlux |
→ Notes [11](11_Bean%20Scopes.md) (thread-safety of singletons), [21](21_Servlets%2C%20Tomcat%20and%20Servlet%20Lifecycle.md) (thread-per-request)

## 13. Testing
```ts
const module = await Test.createTestingModule({
  providers: [UserService, { provide: getRepositoryToken(User), useValue: { findOne: jest.fn() } }],
}).compile();
const service = module.get(UserService);
repo.findOne.mockResolvedValue(user);
expect(await service.findOne(1)).toEqual(user);
expect(repo.findOne).toHaveBeenCalledWith({ where: { id: 1 } });
// e2e: supertest(app.getHttpServer()).get('/users/1').expect(200)
```
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;                     // = jest.fn() mock provider
    @InjectMocks UserService service;              // = module.get(UserService)
    @Test void findsUser() {
        when(repo.findById(1L)).thenReturn(Optional.of(user));       // = mockResolvedValue
        assertEquals(user, service.findOne(1L));                     // = expect().toEqual()
        verify(repo).findById(1L);                                   // = toHaveBeenCalledWith
    }
}
// controller/e2e-ish: @WebMvcTest + MockMvc (= supertest), @MockitoBean for services
mockMvc.perform(get("/users/1")).andExpect(status().isOk()).andExpect(jsonPath("$.email").value("a@b.com"));
```
| Jest / Nest | JUnit / Spring |
|---|---|
| `describe` / `it` | test class / `@Test` |
| `beforeEach` | `@BeforeEach` |
| `jest.fn()`, `mockResolvedValue` | `@Mock`, `when(..).thenReturn(..)` |
| `toHaveBeenCalledWith` / `.not.toHaveBeenCalled` | `verify(mock).m(..)` / `verify(mock, never())` |
| `Test.createTestingModule` with overrides | `@WebMvcTest` / `@SpringBootTest` + `@MockitoBean` |
| supertest | MockMvc (or `TestRestTemplate`/`WebTestClient`) |
| `expect(fn).toThrow()` | `assertThrows(...)` |
→ Notes [49](49_Testing%20with%20JUnit%2C%20Mockito%20and%20MockMvc.md)

## 14. Other quick mappings
| NestJS | Spring Boot |
|---|---|
| `@nestjs/swagger` | springdoc-openapi (`springdoc-openapi-starter-webmvc-ui`) |
| `@nestjs/terminus` health checks | Spring Boot **Actuator** (`/actuator/health`) |
| `@nestjs/cache-manager` | `@EnableCaching` + `@Cacheable` (+ Redis) |
| `@nestjs/event-emitter` | `ApplicationEventPublisher` + `@EventListener` |
| `@nestjs/throttler` | Bucket4j / Resilience4j RateLimiter / API gateway |
| `HttpModule` (axios) | `RestClient` / `WebClient` / OpenFeign |
| `@nestjs/microservices` | Spring Cloud (Gateway, OpenFeign, Config), Spring Kafka/AMQP |
| Logger (`new Logger(Ctx.name)`) | SLF4J `LoggerFactory.getLogger(X.class)` / Lombok `@Slf4j` |
| CLI `nest g resource` | Spring Initializr / IDE |
| `enableCors()` | `CorsConfigurationSource` bean / `@CrossOrigin` |

---

## 15. Interview-ready comparison questions
**Q1. How would you explain Spring DI to someone who knows NestJS?**
The same idea: constructor injection of providers resolved by a container. Differences: Spring beans are app-global (no module exports), discovered by component scanning, and configured with `@Bean`/auto-configuration instead of `@Module` providers.

**Q2. What's the biggest runtime difference between NestJS and Spring MVC?**
Event loop vs thread-per-request. In Spring, blocking I/O is normal but costs threads, so pool sizes, timeouts, and thread safety of singletons matter. For high concurrency, use virtual threads (Java 21) or WebFlux.

**Q3. Map Nest's request lifecycle to Spring.**
Middleware → Filters, Guards → Spring Security / `preHandle`, Pipes → `@Valid` + argument resolvers, Interceptors → AOP / `HandlerInterceptor` / `ResponseBodyAdvice`, Exception filters → `@RestControllerAdvice`.

**Q4. Why don't you need `repository.save()` after updating an entity in Spring?**
Hibernate dirty checking: managed entities in a transactional persistence context are flushed automatically at commit.

**Q5. How is `@Transactional` different from TypeORM's `dataSource.transaction()`?**
It's declarative (an AOP proxy) instead of a callback. Watch out for self-invocation, rollback only on unchecked exceptions by default, and propagation rules.

**Q6. Why must Spring services be stateless?**
They're singletons called concurrently by many request threads. Node is single-threaded, so shared mutable state there doesn't race in the same way.

## Quick revision
- Module/provider → package + `@Component`/`@Bean` (global context).
- Controller decorators → `@RestController` + `@*Mapping` + `@PathVariable/@RequestParam/@RequestBody`.
- Pipes → `@Valid` Bean Validation; filters (exception) → `@RestControllerAdvice`.
- Middleware → Filters; Guards → Spring Security/`@PreAuthorize`; Interceptors → AOP/HandlerInterceptor/ResponseBodyAdvice.
- TypeORM → JPA/Spring Data; **dirty checking + `@Transactional`** is the key mindset shift.
- `async/await` → threads + `CompletableFuture`; **singletons must be thread-safe**.
