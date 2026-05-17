---
name: bn-java-tdd
description: Подход TDD и правила написания Java-тестов. ВЫЗЫВАЙ ВСЕГДА при написании, изменении или рефакторинге тестов, при красно-зелёно-рефактор цикле, при упоминании TDD, unit/integration/e2e тестов или /bn-java-tdd.
---

# TDD и написание Java-тестов

Скилл обязателен при любом написании или изменении тестов. Следуй ему полностью, не полагаясь на память.

## Подход TDD (красный — зелёный — рефакторинг)

1. **СНАЧАЛА** напиши красный тест. Для сложных интеграционных и end-to-end тестов и тестирования классов с большим количеством инжектируемых компонентов **ОБЯЗАТЕЛЬНО** используй Spring Boot тесты. Если сомневаешься, использовать Spring-тесты или нет — спрашивай у пользователя, не гадай!
2. Убедись, что тест **падает** (красный). Если тест, написанный для новой бизнес-логики, **не падает** — это **ПЛОХОЙ** тест, **ОБЯЗАТЕЛЬНО** переделай его!
3. Напиши **минимальный** код для прохождения (зелёный). Если тесты падают — **ИСПРАВЛЯЙ код, НЕ тесты!**
4. Рефакторинг бизнес-логики — **только после** зелёного теста.

Проверка цикла: после шага 1 запускай `mvn test` (или целевой модуль/класс) и убедись в падении; после шага 3 — в прохождении.

### Критерии "красного" теста:
- Тест должен быть минимально возможным (1 ассерт)
- Тест должен проверять **одно** бизнес-правило
- Тест не должен содержать бизнес-логики (только вызов метода и проверку)
- Если тест зеленый без реализации - он написан неправильно (проверяет уже существующее поведение)

## Дополнительные правила тестов

- **Не** вызывай через Reflection API приватные методы для их тестирования. Проверяй функциональность приватных методов через вызов связанных публичных методов.
- Избегай дублирования проверок — если сомневаешься, стоит ли покрывать кейс, спрашивай у пользователя, не гадай!
- Для каждого теста используй `@DisplayName` с описанием на **русском языке**; помещай аннотацию **после** `@Test`. Все методы с аннотацией `@Test` должны иметь public модификатор
- Используй `@ParameterizedTest` **когда**: одна и та же логика с разными входными данными; edge cases и граничные значения; тестирование бизнес-правил с разными комбинациями.
- **Не** тестируй конструкторы, Lombok-генерируемые методы, простые геттеры/сеттеры.


## Стек тестирования

- JUnit 5 + Mockito — unit-тесты по умолчанию.
- Spring Boot Test — интеграционные, e2e и классы с множеством `@Autowired` зависимостей.

## BDD стиль (Given-When-Then)

**Все тесты ДОЛЖНЫ следовать BDD структуре:**

```java
@Test
@DisplayName("Должен создать пользователя с валидными данными")
public void shouldCreateUserWithValidData() {
    // Given - подготовка данных
    UserCreateRequest request = UserCreateRequest.builder()
        .email("test@example.com")
        .name("John Doe")
        .build();

    User expectedUser = User.builder()
        .id(1L)
        .email("test@example.com")
        .build();
    
    when(userRepository.save(any(User.class))).thenReturn(expectedUser);
    
    // When - выполнение действия
    User result = userService.createUser(request);
    
    // Then - проверка результата
    assertThat(result.getId()).isEqualTo(1L);
    assertThat(result.getEmail()).isEqualTo("test@example.com");
    verify(userRepository).save(any(User.class));
}
```

### Разделение секций:
- ***Given (Дано)*** - подготовка контекста (моки, данные)
- ***When (Когда)*** - выполнение тестируемого действия
- ***Then (Тогда)*** - проверка результатов (assertions, verify)

***Запрещено*** смешивать Arrange/Act/Assert (AAA) - используй Given/When/Then с комментариями или методами-разделителями.

## Шаблоны тестов для разных слоёв

### 1. Unit тест для сервиса (без Spring)
```java
@ExtendWith(MockitoExtension.class)
public class UserServiceTest {
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailValidator emailValidator;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    @DisplayName("Должен выбросить исключение при создании пользователя с существующим email")
    public void shouldThrowExceptionWhenEmailExists() {
        // Given
        String existingEmail = "existing@example.com";
        when(userRepository.existsByEmail(existingEmail)).thenReturn(true);
        
        // When & Then
        assertThatThrownBy(() -> userService.createUser(existingEmail))
            .isInstanceOf(BusinessException.class)
            .hasMessage("Email already exists: " + existingEmail);
        
        verify(userRepository, never()).save(any());
    }
}
```

### 2. Интеграционный тест с Spring Boot
```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = Replace.ANY) // Используй H2 для тестов
@Transactional
public class UserServiceIntegrationTest {
    @Autowired
    private UserService userService;
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    @DisplayName("Должен сохранить пользователя в базу данных")
    public void shouldSaveUserToDatabase() {
        // Given
        User user = User.builder()
            .email("test@example.com")
            .name("Test User")
            .build();
        
        // When
        User saved = userService.save(user);
        
        // Then
        User found = entityManager.find(User.class, saved.getId());
        assertThat(found).isNotNull();
        assertThat(found.getEmail()).isEqualTo("test@example.com");
    }
}
```

### 3. Тест контроллера
```java
@WebMvcTest(UserController.class)
public class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    @DisplayName("Должен вернуть 200 OK при получении пользователя по ID")
    public void shouldReturnUserWhenIdExists() throws Exception {
        // Given
        Long userId = 1L;
        UserDto userDto = new UserDto(userId, "test@example.com");
        when(userService.findById(userId)).thenReturn(userDto);
        
        // When & Then
        mockMvc.perform(get("/api/users/{id}", userId))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(userId))
            .andExpect(jsonPath("$.email").value("test@example.com"));
    }
}
```

### Использование @BeforeEach для общей настройки:
```java
@BeforeEach
void setUp() {
    defaultUser = UserTestDataFactory.createDefaultUser();
    // Общая настройка моков
    lenient().when(emailValidator.isValid(any())).thenReturn(true);
}
```

## Mockito правила и лучшие практики

### Разрешено:
- `@Mock` для зависимостей
- `@InjectMocks` для тестируемого класса
- `when().thenReturn()` для стабов
- `verify()` для проверки вызовов
- `ArgumentCaptor` для захвата аргументов
- `any()`, `eq()`, `argThat()` для матчеров

### Запрещено:
- Мокать value objects (String, Integer, LocalDate) - используй реальные объекты
- Мокать коллекции (List, Map) - создавай реальные коллекции
- Использовать `doReturn().when()` без необходимости (предпочитай `when()`)
- Мокать `final` классы без `mockito-inline` (спроси у пользователя)
- Статические моки без явного разрешения пользователя

### Примеры правильного использования:

```java
// ✅ Хорошо
User user = User.builder()
    .id(1L)
    .build();
when(userRepository.findById(1L)).thenReturn(Optional.of(user));

// ❌ Плохо - мокаем value object
when(mockEmail.toString()).thenReturn("test@example.com");

// ✅ Аргумент матчеры
verify(userService).processUser(argThat(u -> u.getId() > 0));

// ✅ ArgumentCaptor
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(userRepository).save(captor.capture());
User savedUser = captor.getValue();
assertThat(savedUser.getEmail()).isEqualTo("test@example.com");
```

### Верификация:
- Используй verify(userRepository, times(1)).save(any()) для проверки количества вызовов
- Используй verifyNoMoreInteractions(userRepository) когда нужно убедиться, что больше вызовов не было
- Не используй verifyZeroInteractions() - устарел, используй verifyNoInteractions()

### Использование ресурсных файлов для больших данных:
- JSON: `src/test/resources/test-data/users.json`
- Используй `@JsonFileSource` из `@ParameterizedTest`

```java
@ParameterizedTest
@JsonFileSource(resources = "/test-data/users.json")
void shouldProcessUsersFromFile(User user, String expected) {
    // тест с данными из файла
}
```

## AssertJ для читаемых assertions

**Используй AssertJ вместо JUnit assertions:**

```java
// ✅ AssertJ (рекомендуется)
import static org.assertj.core.api.Assertions.*;

assertThat(user.getName()).isEqualTo("John");
assertThat(user.getEmail()).contains("@");
assertThat(list).hasSize(3).contains("item1", "item2");
assertThatThrownBy(() -> userService.validate(null))
    .isInstanceOf(ValidationException.class)
    .hasMessageContaining("User cannot be null");

// ❌ JUnit assertions (устаревший стиль)
assertEquals("John", user.getName());
assertTrue(list.contains("item1"));
```

### Цепочки assertions:

```java
assertThat(user)
    .extracting(User::getId, User::getEmail, User::isActive)
    .containsExactly(1L, "test@example.com", true);
```

## Анти-паттерны в тестах (избегать)

1. **Тест, который ничего не проверяет** (нет assertions)
   ```java
   // ❌ Плохо
   @Test
   void testMethod() {
       service.doSomething(); // нет проверки!
   }
   ```
2. **Тест с логикой внутри (условия, циклы)**
   ```java
    // ❌ Плохо
    @Test
    void testUsers() {
        for (User user : users) {
            if (user.isActive()) {
                assertThat(user.getEmail()).isNotNull();
            }
        }
    }
   ``` 
3. Тест, зависимый от порядка выполнения (@TestMethodOrder)
    ```java
    // ❌ Плохо - тесты не должны зависеть друг от друга
    @Test
    void testCreate() { ... } // создаёт ID=1

    @Test
    void testUpdate() { ... } // ожидает, что ID=1 существует
    ```
4. Тест с Thread.sleep() - используй Awaitility
    ```java
    // ❌ Плохо
    Thread.sleep(1000);

    // ✅ Хорошо
    await().atMost(5, SECONDS)
        .until(() -> asyncResult.isDone());
    ```
5. Тест, который модифицирует глобальное состояние
    ```java
    // ❌ Плохо
    System.setProperty("app.mode", "test");

    // ✅ Хорошо - используй @TestPropertySource
    @TestPropertySource(properties = "app.mode=test")
    ```