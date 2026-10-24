# Testing with JUnit, Mockito and MockMvc

> **Source:** Lecture 40 (`Notes_Of_Aditya/Lecture 40/code/springTestingDemo`, tests in `src/test/java`)
> **Related:** [07_Types of DI](07_Types%20of%20Dependency%20Injection%20in%20Spring.md) (constructor injection makes testing easy) · [44_Security](44_Spring%20Security%20Fundamentals.md)

## Why this topic matters
Every interview loop asks how you test a Spring Boot app: unit vs integration, `@Mock` vs `@MockitoBean`, `@WebMvcTest` vs `@SpringBootTest`, and what to verify.

---

## 1. Why and what
Manual testing (start the app, call Postman, eyeball the result) is slow and unrepeatable. An automated test is: **known input → run code → compare to expected → pass/fail**.

**Arrange – Act – Assert:**
```java
@Test
void shouldApplyDiscountToPrice() {
    double price = 1000, discount = 20;                               // Arrange
    double actual = priceCalculator.calculatePrice(price, discount);  // Act
    assertEquals(800, actual);                                         // Assert
}
```
Test **behaviours/requirements** (valid discount, 0, 100, negative, >100, negative price), not lines of code.
Skip: plain getters/setters, framework code (Spring Data's `save()` works), code without decisions.

## 2. Test types and the pyramid
| Type | What's real | Speed | Tool |
|---|---|---|---|
| **Unit** | One class | Fastest | JUnit (+ Mockito) |
| **Slice** | One Spring layer | Fast | `@WebMvcTest`, `@DataJpaTest` |
| **Integration** | Several real components | Slower | `@SpringBootTest` (+ Testcontainers) |
| **E2E** | Whole system | Slowest | Browser/API suites |
Pyramid: **many** unit tests → fewer slice/integration tests → very few E2E.
**Rule:** use the **smallest environment that can prove the behaviour**.

## 3. JUnit 5 (Jupiter) basics
`spring-boot-starter-test` (scope `test`) brings JUnit, Mockito, AssertJ, Hamcrest, Spring Test, and JsonPath. Boot 4 splits it into tech-specific starters (`spring-boot-starter-webmvc-test`, `-data-jpa-test`, `-security-test`). Tests live in `src/test/java`, mirroring the main package.
| Annotation / API | Use |
|---|---|
| `@Test` | A test method |
| `@BeforeEach` / `@AfterEach` | Run before/after **each** test (fresh state) |
| `@BeforeAll` / `@AfterAll` | Once per class (static) |
| `@DisplayName`, `@ParameterizedTest` + `@ValueSource`/`@CsvSource` | Readable names, data-driven tests |
| `assertEquals`, `assertTrue/False`, `assertNull/NotNull` | Assertions |
| `assertThrows(Ex.class, () -> …)` | Expected exception (returns it to inspect the message) |
Name tests by behaviour: `shouldRejectDiscountAboveHundred`, not `test1`. Run with `mvn test` or the IDE.

### Lecture: plain unit test (no Spring)
```java
public class PriceCalculatorTest {
    private PriceCalculator priceCalculator;

    @BeforeEach
    void setup() { priceCalculator = new PriceCalculator(); }        // just `new`: no Spring context

    @Test
    void shouldApplyDiscountToPrice() {
        assertEquals(800, priceCalculator.calculatePrice(1000, 20));
    }

    @Test
    void shouldRejectDiscountAboveHundred() {
        IllegalArgumentException ex = assertThrows(IllegalArgumentException.class,
                () -> priceCalculator.calculatePrice(1000, 120));
        assertEquals("Discount should be within 0 to 100", ex.getMessage());
    }
}
```

## 4. Service tests with Mockito
`ProductService` depends on `ProductRepository`. A real repository drags in Hibernate, a DataSource, and a DB, which isn't a unit test anymore. Replace it with a **test double**:
| Double | Behaviour |
|---|---|
| Dummy | Passed but unused |
| Stub | Returns canned answers |
| Fake | Simplified working impl (in-memory map repo) |
| **Mock** | Stubbable **and** records calls for verification |
| Spy | Wraps a real object, partially stubbed |

```java
@ExtendWith(MockitoExtension.class)                  // activates @Mock / @InjectMocks
public class ProductServiceTest {
    @Mock        private ProductRepository productRepository;   // Mockito mock (not a Spring bean)
    @InjectMocks private ProductService productService;         // new ProductService(mock) — not Spring DI

    @Test
    void shouldReturnProductWhenProductExist() {
        Product product = new Product(1L, "Laptop", 50000, 10);
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));   // STUB: control the dependency

        Product actual = productService.getProductById(1L);                      // ACT

        assertEquals("Laptop", actual.getName());                                // ASSERT: check the result
        verify(productRepository).findById(1L);                                  // VERIFY: check the interaction
    }

    @Test
    void shouldThrowExceptionWhenProductDoesNotExist() {
        when(productRepository.findById(99L)).thenReturn(Optional.empty());
        RuntimeException ex = assertThrows(RuntimeException.class, () -> productService.getProductById(99L));
        assertEquals("Product not found: 99", ex.getMessage());
    }

    @Test
    void shouldRejectProductWhenNameAlreadyExists() {
        Product request = new Product(null, "Keyboard", 200, 5);
        when(productRepository.existsByName("Keyboard")).thenReturn(true);
        assertThrows(IllegalArgumentException.class, () -> productService.createProduct(request));
        verify(productRepository, never()).save(any(Product.class));             // "must NOT save" is the rule
    }
}
```
**`when` = control the dependency · `assert` = check the result · `verify` = check the interaction.** Verify only interactions that are part of the behaviour.
Matchers: `any()`, `any(Product.class)`, `anyLong()`, `eq(x)`. If one argument uses a matcher, **all** must (`eq("Laptop"), eq(true)`). Also useful: `thenThrow`, `ArgumentCaptor`, `verify(mock, times(2))`, `verifyNoMoreInteractions`.
Constructor injection is what makes `@InjectMocks` (or a plain `new Service(mock)`) trivial.

## 5. Controller tests: `@WebMvcTest` + `MockMvc`
Calling `controller.getProduct(1L)` directly skips mappings, path-variable conversion, JSON, validation, advice, security, and status codes. Test **through Spring MVC** instead:
```java
@WebMvcTest                              // web slice: controllers, Jackson, validation, advice, security filters
public class ProductControllerTest {     // NOT loaded: services, repositories, Hibernate, DB
    @Autowired private MockMvc mockMvc;               // fake HTTP, no real server/port
    @MockitoBean private ProductService productService;   // mock placed INTO the Spring test context

    @Test
    void shouldReturnProductIfProductExist() throws Exception {
        when(productService.getProductById(1L)).thenReturn(new Product(1L, "Laptop", 50000, 10));

        mockMvc.perform(get("/api/product/1").accept(MediaType.APPLICATION_JSON))
               .andExpect(status().isOk())
               .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
               .andExpect(jsonPath("$.id").value(1))
               .andExpect(jsonPath("$.name").value("Laptop"))
               .andExpect(jsonPath("$.price").value(50000))
               .andExpect(jsonPath("$.stock").value(10));

        verify(productService).getProductById(1L);
    }
}
```
POST example (from the notes):
```java
mockMvc.perform(post("/api/products").with(csrf())                 // needed when CSRF is enabled
        .contentType(MediaType.APPLICATION_JSON)
        .content("""{ "name": "Keyboard", "price": 200, "stock": 5 }"""))
       .andExpect(status().isCreated())
       .andExpect(jsonPath("$.id").value(10));
```
| `@Mock` | `@MockitoBean` |
|---|---|
| Pure Mockito field | **Replaces a bean** in the Spring test context |
| Unit tests | `@WebMvcTest` / `@SpringBootTest` |
`@MockitoBean` (Spring Framework 6.2+) replaces the deprecated `@MockBean`.
With Spring Security on the classpath: `@WithMockUser(roles = "ADMIN")` for an authenticated principal, `.with(csrf())` for unsafe methods, `spring-boot-starter-security-test`. Without them you get 401/403.
Prefer `@WebMvcTest(ProductController.class)`: plain `@WebMvcTest` (as in the lecture) loads **all** controllers, so every controller's dependencies must be mocked.

## 6. Repository tests: `@DataJpaTest`
Mocking a repository proves nothing about the real query. `@DataJpaTest` loads entities, repositories, Hibernate, and an **embedded DB**. Each test is **transactional and rolled back**.
```java
@DataJpaTest
class ProductRepositoryTest {
    @Autowired ProductRepository repo;
    @Test
    void existsByNameShouldFindSavedProduct() {
        repo.save(new Product(null, "Laptop", 50000, 10));
        assertTrue(repo.existsByName("Laptop"));
        assertFalse(repo.existsByName("Mouse"));
    }
}
```
Needs `com.h2database:h2` (scope test), or **Testcontainers** with the real MySQL for production-like behaviour (`@AutoConfigureTestDatabase(replace = NONE)`).

## 7. Choosing the test
| Behaviour | Tool |
|---|---|
| Plain logic/calculation | JUnit |
| Service with dependencies | JUnit + Mockito (`@Mock`, `@InjectMocks`) |
| HTTP contract (URL, JSON, status, validation, security) | `@WebMvcTest` + MockMvc + `@MockitoBean` |
| Query/mapping | `@DataJpaTest` + H2/Testcontainers |
| Whole app wiring | `@SpringBootTest` (sparingly) |

## ⚠️ Issues in the lecture code
1. **`ProductRepositoryTest` is empty**, and the project has **no H2 dependency**. `@DataJpaTest` replaces the DataSource with an embedded DB by default, so with only MySQL on the classpath the context fails to start. Add H2 (test scope) or Testcontainers.
2. The generated `@SpringBootTest` `contextLoads()` test needs a running MySQL (the URL is in `application.properties`), so `mvn test` fails on a machine without the DB. Use a test profile or Testcontainers.
3. `ProductService` throws a generic `RuntimeException`. A custom `ProductNotFoundException` mapped to 404 makes tests and the API clearer.
4. `price` is a `double` (money should be `BigDecimal`), and `assertEquals(800, actualPrice)` on doubles should use a delta: `assertEquals(800, actual, 0.001)`.
5. **I couldn't run these tests here:** this machine has only JDK 25, and the project's Lombok annotation processing fails on it (`ExceptionInInitializerError: com.sun.tools.javac.tree.EndPosTable`). The project targets Java 21, so run with **JDK 21**. On JDK 23+ you also need Lombok in `annotationProcessorPaths`.

---

## Interview Q&A
**Q1. Unit vs integration test?**
A unit test checks one unit in isolation (dependencies mocked) and is fast. An integration test checks real components working together (Spring context, DB).

**Q2. `@Mock` vs `@MockitoBean` (`@MockBean`)?**
`@Mock` creates a plain Mockito mock with no Spring involvement. `@MockitoBean` puts the mock into the Spring application context, replacing the bean.

**Q3. `@InjectMocks` vs `@Autowired`?**
`@InjectMocks` is Mockito constructing the class under test with the mocks. `@Autowired` is Spring DI from a context.

**Q4. `@WebMvcTest` vs `@SpringBootTest`?**
Web-layer slice (controllers + MVC infrastructure, mocked services, fast) vs the full application context (slow, real beans).

**Q5. What does MockMvc do?**
Sends mock HTTP requests through the real DispatcherServlet/filters without a running server, and lets you assert status, headers, and JSON.

**Q6. Stubbing vs verification?**
`when(..).thenReturn(..)` defines a dependency's behaviour. `verify(..)` asserts that an interaction happened (or `never()`).

**Q7. Why not mock the repository to test a custom query?**
The mock returns whatever you tell it; it never runs the JPQL/SQL. Use `@DataJpaTest`.

**Q8. Why does my `@WebMvcTest` return 401/403?**
Spring Security is active. Add `@WithMockUser` and `.with(csrf())` (or import your security config).

**Q9. How do you test against a real database in CI?**
Testcontainers (a disposable MySQL/PostgreSQL Docker container), wired via `@ServiceConnection` in Boot 3.1+.

## Quick revision
- AAA; test behaviours and edge cases; many unit tests, few integration tests.
- JUnit: `@Test`, `@BeforeEach`, `assertEquals`, `assertThrows`.
- Mockito: `@ExtendWith(MockitoExtension)`, `@Mock`, `@InjectMocks`, `when/verify/never/any`.
- Web: `@WebMvcTest` + MockMvc + `@MockitoBean` (+ `@WithMockUser`, `csrf()`).
- Data: `@DataJpaTest` + H2/Testcontainers (rolled back per test).
