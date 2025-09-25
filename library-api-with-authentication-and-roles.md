## Proyecto 2: API de Biblioteca con Autenticación JWT y Autorización por Roles
**⏱️ Duración estimada:** 2 horas
**🎯 Temas principales:** Spring Security, JWT, Autorización por Roles, JPA, API Externa

### Descripción del Sistema
Desarrollar una API REST para gestionar una biblioteca digital con autenticación JWT, autorización por roles, y integración con una API externa para validar información de libros.

### Entidades Requeridas

#### Entidad `User`
```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    @NotBlank @Size(min = 3, max = 50)
    private String username;

    @NotBlank @Size(min = 6)
    private String password;

    @Email @NotBlank
    private String email;

    @Enumerated(EnumType.STRING)
    private Role role; // ADMIN, LIBRARIAN, USER

    private boolean enabled = true;

    @CreationTimestamp
    private LocalDateTime createdAt;

    // constructors, getters, setters
}
```

#### Entidad `Book`
```java
@Entity
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank @Size(min = 1, max = 13)
    private String isbn;

    @NotBlank @Size(min = 1, max = 200)
    private String title;

    @NotBlank @Size(min = 1, max = 100)
    private String author;

    @NotNull @Min(1)
    private Integer totalCopies;

    @NotNull @Min(0)
    private Integer availableCopies;

    @CreationTimestamp
    private LocalDateTime createdAt;

    // constructors, getters, setters
}
```

#### Entidad `Loan`
```java
@Entity
public class Loan {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "book_id", nullable = false)
    private Book book;

    @CreationTimestamp
    private LocalDateTime loanDate;

    @NotNull
    private LocalDateTime dueDate;

    private LocalDateTime returnDate;

    @Enumerated(EnumType.STRING)
    private LoanStatus status; // ACTIVE, RETURNED, OVERDUE

    // constructors, getters, setters
}
```

### DTOs Requeridos

#### Autenticación
```java
public class LoginRequest {
    @NotBlank
    private String username;

    @NotBlank
    private String password;

    // getters, setters
}

public class RegisterRequest {
    @NotBlank @Size(min = 3, max = 50)
    private String username;

    @NotBlank @Size(min = 6)
    private String password;

    @Email @NotBlank
    private String email;

    // getters, setters (role se asigna como USER por defecto)
}

public class JwtResponse {
    private String token;
    private String type = "Bearer";
    private String username;
    private String email;
    private String role;

    // constructors, getters, setters
}
```

#### Libros
```java
public class CreateBookRequest {
    @NotBlank @Size(min = 1, max = 13)
    private String isbn;

    @NotBlank @Size(min = 1, max = 200)
    private String title;

    @NotBlank @Size(min = 1, max = 100)
    private String author;

    @NotNull @Min(1)
    private Integer totalCopies;

    // getters, setters
}

public class BookResponse {
    private Long id;
    private String isbn;
    private String title;
    private String author;
    private Integer totalCopies;
    private Integer availableCopies;
    private LocalDateTime createdAt;

    // constructors, getters, setters
}
```

### Configuración de Spring Security

#### SecurityConfig
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Autowired
    private JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;

    @Autowired
    private JwtRequestFilter jwtRequestFilter;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .exceptionHandling(ex -> ex.authenticationEntryPoint(jwtAuthenticationEntryPoint))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> {
                auth.requestMatchers("/api/auth/**").permitAll()
                    .requestMatchers("/h2-console/**").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/books/**").hasAnyRole("USER", "LIBRARIAN", "ADMIN")
                    .requestMatchers(HttpMethod.POST, "/api/books").hasAnyRole("LIBRARIAN", "ADMIN")
                    .requestMatchers(HttpMethod.PUT, "/api/books/**").hasAnyRole("LIBRARIAN", "ADMIN")
                    .requestMatchers(HttpMethod.DELETE, "/api/books/**").hasRole("ADMIN")
                    .requestMatchers("/api/loans/**").hasAnyRole("USER", "LIBRARIAN", "ADMIN")
                    .requestMatchers("/api/users/**").hasRole("ADMIN")
                    .anyRequest().authenticated();
            });

        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
        http.headers(headers -> headers.frameOptions().disable()); // Para H2 console

        return http.build();
    }
}
```

#### JWT Utilities
```java
@Component
public class JwtTokenUtil {

    private String secret = "mySecretKey";
    private int jwtExpiration = 86400; // 24 horas

    public String generateToken(String username, String role) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("role", role);
        return createToken(claims, username);
    }

    public String getUsernameFromToken(String token) {
        // Implementar extracción de username
    }

    public String getRoleFromToken(String token) {
        // Implementar extracción de role
    }

    public Boolean isTokenExpired(String token) {
        // Implementar validación de expiración
    }

    public Boolean validateToken(String token, String username) {
        // Implementar validación completa
    }

    private String createToken(Map<String, Object> claims, String subject) {
        // Implementar creación de token con Jwts.builder()
    }
}
```

#### Filtro JWT
```java
@Component
public class JwtRequestFilter extends OncePerRequestFilter {

    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain chain) throws ServletException, IOException {

        final String requestTokenHeader = request.getHeader("Authorization");

        // Implementar extracción y validación de token
        // Configurar SecurityContext si el token es válido

        chain.doFilter(request, response);
    }
}
```

### Endpoints a Implementar

#### AuthController
- `POST /api/auth/login` - Autenticar usuario
  - **Request Body:** `LoginRequest`
  - **Validaciones:** `@Valid` en el body
  - **Response:** `200 OK` con `JwtResponse`
  - **Error:** `401 UNAUTHORIZED` si credenciales incorrectas

- `POST /api/auth/register` - Registrar nuevo usuario
  - **Request Body:** `RegisterRequest`
  - **Validaciones:** `@Valid`, verificar username único
  - **Response:** `201 CREATED` con mensaje de confirmación
  - **Error:** `409 CONFLICT` si username ya existe

#### BookController
- `GET /api/books` - Listar todos los libros
  - **Autorización:** `@PreAuthorize("hasAnyRole('USER', 'LIBRARIAN', 'ADMIN')")`
  - **Response:** `200 OK` con lista de `BookResponse`

- `GET /api/books/{id}` - Obtener libro por ID
  - **Autorización:** `@PreAuthorize("hasAnyRole('USER', 'LIBRARIAN', 'ADMIN')")`
  - **Response:** `200 OK` con `BookResponse`
  - **Error:** `404 NOT FOUND` si no existe

- `POST /api/books` - Crear nuevo libro
  - **Autorización:** `@PreAuthorize("hasAnyRole('LIBRARIAN', 'ADMIN')")`
  - **Request Body:** `CreateBookRequest`
  - **Flujo:**
    1. Validar ISBN único
    2. **Validar libro con API externa (Open Library)**
    3. Crear libro con availableCopies = totalCopies
  - **Response:** `201 CREATED` con `BookResponse`
  - **Error:** `409 CONFLICT` si ISBN ya existe, `400 BAD REQUEST` si la API externa no confirma el libro

- `PUT /api/books/{id}` - Actualizar libro
  - **Autorización:** `@PreAuthorize("hasAnyRole('LIBRARIAN', 'ADMIN')")`
  - **Request Body:** `CreateBookRequest`
  - **Response:** `200 OK` con `BookResponse`
  - **Error:** `404 NOT FOUND` si no existe

- `DELETE /api/books/{id}` - Eliminar libro
  - **Autorización:** `@PreAuthorize("hasRole('ADMIN')")`
  - **Validación:** No permitir eliminar si hay préstamos activos
  - **Response:** `204 NO CONTENT`
  - **Error:** `404 NOT FOUND` o `409 CONFLICT` si hay préstamos activos

#### LoanController
- `POST /api/loans` - Crear préstamo
  - **Autorización:** `@PreAuthorize("hasAnyRole('USER', 'LIBRARIAN', 'ADMIN')")`
  - **Request Body:** `{"bookId": 1}` (userId se obtiene del JWT)
  - **Validaciones:**
    - Usuario no puede tener más de 3 préstamos activos
    - Libro debe tener copias disponibles
  - **Flujo:** Decrementar availableCopies, dueDate = +14 días
  - **Response:** `201 CREATED` con datos del préstamo
  - **Error:** `400 BAD REQUEST` si supera límite o no hay copias

- `PUT /api/loans/{id}/return` - Devolver libro
  - **Autorización:** `@PreAuthorize("hasAnyRole('LIBRARIAN', 'ADMIN') or @loanService.isLoanOwner(#id, authentication.name)")`
  - **Flujo:** Marcar como RETURNED, incrementar availableCopies
  - **Response:** `200 OK` con datos actualizados
  - **Error:** `404 NOT FOUND` o `409 CONFLICT` si ya está devuelto

- `GET /api/loans/my` - Obtener préstamos del usuario autenticado
  - **Autorización:** `@PreAuthorize("hasAnyRole('USER', 'LIBRARIAN', 'ADMIN')")`
  - **Response:** `200 OK` con lista de préstamos

### Integración con API Externa

#### BookValidationService
```java
@Service
public class BookValidationService {

    @Autowired
    private RestTemplate restTemplate;

    public boolean validateBookByISBN(String isbn) {
        try {
            String url = "https://openlibrary.org/api/books?bibkeys=ISBN:" + isbn + "&format=json";

            // Implementar llamada a Open Library API
            // Configurar headers si es necesario
            // Retornar true si el libro existe, false si no
            // Manejar timeouts y errores de conexión

        } catch (Exception e) {
            // Log error pero no fallar el proceso
            // Retornar true para no bloquear la creación
            return true;
        }
    }
}
```

#### Configuración RestTemplate
```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setRequestFactory(new HttpComponentsClientHttpRequestFactory());
        return restTemplate;
    }
}
```

### Configuración de Aplicación

#### application.properties
```properties
# Database H2
spring.datasource.url=jdbc:h2:mem:library
spring.datasource.driverClassName=org.h2.Driver
spring.h2.console.enabled=true
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true

# JWT Configuration
jwt.secret=${JWT_SECRET:mySecretKey}
jwt.expiration=${JWT_EXPIRATION:86400}

# External API Configuration
external.api.timeout=5000
external.api.retries=3

# Logging
logging.level.com.example.library=DEBUG
logging.level.org.springframework.security=DEBUG
```

### Testing Requerido

#### Security Tests
```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@TestPropertySource(locations = "classpath:test.properties")
class SecurityIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldAllowAccessToPublicEndpoints() {
        // Test acceso sin token a /api/auth/login
    }

    @Test
    void shouldDenyAccessToProtectedEndpointsWithoutToken() {
        // Test acceso sin token a /api/books
        // Verificar 401 UNAUTHORIZED
    }

    @Test
    void shouldAllowAccessWithValidToken() {
        // Test con token válido
        // Verificar acceso exitoso
    }

    @Test
    void shouldDenyAccessBasedOnRole() {
        // Test que USER no pueda crear libros
        // Verificar 403 FORBIDDEN
    }
}
```

#### Service Tests
```java
@ExtendWith(MockitoExtension.class)
class BookServiceTest {

    @Mock
    private BookRepository bookRepository;

    @Mock
    private BookValidationService bookValidationService;

    @InjectMocks
    private BookService bookService;

    @Test
    void shouldCreateBookWhenValidISBN() {
        // Test creación exitosa
        // Mock de validación externa
    }

    @Test
    void shouldThrowExceptionWhenISBNExists() {
        // Test ISBN duplicado
    }
}
```

### Criterios de Evaluación
1. **Autenticación JWT** funcionando correctamente
2. **Autorización por roles** implementada con `@PreAuthorize`
3. **Filtros de seguridad** configurados correctamente
4. **Integración con API externa** con manejo de errores
5. **Endpoints protegidos** según los roles especificados
6. **Tests de seguridad** cubriendo casos de autorización
