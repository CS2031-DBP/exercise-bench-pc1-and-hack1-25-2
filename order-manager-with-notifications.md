## Proyecto 1: Sistema de Gestión de Pedidos con Notificaciones Asíncronas
**⏱️ Duración estimada:** 2 horas
**🎯 Temas principales:** Core Spring Boot, JPA, Procesamiento Asíncrono, Email, Testing

### Descripción del Sistema
Desarrollar una API REST para gestionar pedidos de una tienda online que procese notificaciones de forma asíncrona y envíe emails de confirmación.

### Entidades Requeridas

#### Entidad `Customer`
```java
@Entity
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank @Size(min = 2, max = 100)
    private String name;

    @Email @NotBlank
    private String email;

    @NotBlank @Size(min = 10, max = 15)
    private String phone;

    // constructors, getters, setters
}
```

#### Entidad `Order`
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @NotNull @DecimalMin("0.01")
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status; // PENDING, CONFIRMED, SHIPPED, DELIVERED

    @CreationTimestamp
    private LocalDateTime createdAt;

    // constructors, getters, setters
}
```

### DTOs Requeridos

#### `CreateOrderRequest`
```java
public class CreateOrderRequest {
    @NotNull
    private Long customerId;

    @NotNull @DecimalMin("0.01")
    private BigDecimal totalAmount;

    // getters, setters
}
```

#### `OrderResponse`
```java
public class OrderResponse {
    private Long id;
    private String customerName;
    private String customerEmail;
    private BigDecimal totalAmount;
    private OrderStatus status;
    private LocalDateTime createdAt;

    // constructors, getters, setters
}
```

### Endpoints a Implementar

#### CustomerController
- `POST /api/customers` - Crear cliente
  - **Request Body:** `Customer` (sin id)
  - **Validaciones:** `@Valid` en el body
  - **Response:** `201 CREATED` con el cliente creado
  - **Error:** `400 BAD REQUEST` si las validaciones fallan

- `GET /api/customers/{id}` - Obtener cliente por ID
  - **Response:** `200 OK` con el cliente
  - **Error:** `404 NOT FOUND` si no existe

#### OrderController
- `POST /api/orders` - Crear pedido
  - **Request Body:** `CreateOrderRequest`
  - **Validaciones:** `@Valid` en el body
  - **Flujo:**
    1. Validar que el customer existe
    2. Crear el pedido con status `PENDING`
    3. **Publicar evento `OrderCreatedEvent` de forma asíncrona**
    4. Retornar `201 CREATED` inmediatamente
  - **Response:** `201 CREATED` con `OrderResponse`
  - **Error:** `404 NOT FOUND` si el customer no existe

- `PUT /api/orders/{id}/status` - Actualizar status del pedido
  - **Request Body:** `{"status": "CONFIRMED"}`
  - **Validaciones:** Solo permitir transiciones válidas
  - **Flujo:**
    1. Actualizar status
    2. **Publicar evento `OrderStatusChangedEvent` de forma asíncrona**
  - **Response:** `200 OK` con `OrderResponse`
  - **Error:** `404 NOT FOUND` o `400 BAD REQUEST` para transiciones inválidas

- `GET /api/orders/{id}` - Obtener pedido por ID
  - **Response:** `200 OK` con `OrderResponse`
  - **Error:** `404 NOT FOUND` si no existe

### Configuración de Procesamiento Asíncrono

#### Configuración
```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "orderTaskExecutor")
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("order-async-");
        executor.initialize();
        return executor;
    }
}
```

#### Eventos a Implementar
```java
public class OrderCreatedEvent {
    private final Long orderId;
    private final String customerEmail;
    private final String customerName;
    private final BigDecimal totalAmount;

    // constructor, getters
}

public class OrderStatusChangedEvent {
    private final Long orderId;
    private final OrderStatus newStatus;
    private final String customerEmail;

    // constructor, getters
}
```

#### Event Listeners
```java
@Component
public class OrderEventListener {

    @Autowired
    private EmailService emailService;

    @EventListener
    @Async("orderTaskExecutor")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Enviar email de confirmación
        // Simular procesamiento de 2-3 segundos
        // Log del procesamiento
    }

    @EventListener
    @Async("orderTaskExecutor")
    public void handleOrderStatusChanged(OrderStatusChangedEvent event) {
        // Enviar email de actualización de status
        // Log del procesamiento
    }
}
```

### Configuración de Email

#### application.properties
```properties
# Database H2
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.h2.console.enabled=true
spring.jpa.hibernate.ddl-auto=create-drop

# Email Configuration
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=${MAIL_USERNAME:test@example.com}
spring.mail.password=${MAIL_PASSWORD:testpassword}
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

# Async Configuration
logging.level.org.springframework.scheduling=DEBUG
```

#### EmailService
```java
@Service
public class EmailService {

    @Autowired
    private JavaMailSender mailSender;

    @Value("${spring.mail.username}")
    private String fromEmail;

    @Async("orderTaskExecutor")
    public CompletableFuture<Void> sendOrderConfirmation(String to, String customerName, Long orderId, BigDecimal amount) {
        // Implementar envío de email
        // Simular delay de 1-2 segundos
        // Manejar excepciones y log
        return CompletableFuture.completedFuture(null);
    }

    @Async("orderTaskExecutor")
    public CompletableFuture<Void> sendOrderStatusUpdate(String to, Long orderId, OrderStatus newStatus) {
        // Implementar envío de email
        return CompletableFuture.completedFuture(null);
    }
}
```

### Manejo de Errores

#### GlobalExceptionHandler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationExceptions(MethodArgumentNotValidException ex) {
        // Retornar 400 BAD REQUEST con detalles de validación
    }

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleEntityNotFound(EntityNotFoundException ex) {
        // Retornar 404 NOT FOUND
    }

    @ExceptionHandler(IllegalStateException.class)
    public ResponseEntity<ErrorResponse> handleIllegalState(IllegalStateException ex) {
        // Retornar 400 BAD REQUEST para transiciones de status inválidas
    }

    // Formato estándar de ErrorResponse
    public static class ErrorResponse {
        private String error;
        private String message;
        private LocalDateTime timestamp;
        private int status;

        // constructors, getters, setters
    }
}
```

### Testing Requerido

#### Test del Controller
```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @MockBean
    private OrderService orderService;

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldCreateOrderSuccessfully() throws Exception {
        // Test POST /api/orders con datos válidos
        // Verificar status 201 y response correcto
    }

    @Test
    void shouldReturn404WhenCustomerNotFound() throws Exception {
        // Test POST /api/orders con customer inexistente
        // Verificar status 404
    }

    @Test
    void shouldReturn400WithValidationErrors() throws Exception {
        // Test POST /api/orders con datos inválidos
        // Verificar status 400 y mensaje de error
    }
}
```

#### Test del Service
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private CustomerRepository customerRepository;

    @Mock
    private ApplicationEventPublisher eventPublisher;

    @InjectMocks
    private OrderService orderService;

    @Test
    void shouldCreateOrderAndPublishEvent() {
        // Test creación de order
        // Verificar que se publique el evento
        // Usar verify() para confirmar las interacciones
    }

    @Test
    void shouldThrowExceptionWhenCustomerNotFound() {
        // Test con customer inexistente
        // Verificar que lance EntityNotFoundException
    }
}
```

### Criterios de Evaluación
1. **Implementación completa de endpoints** con validaciones correctas
2. **Procesamiento asíncrono** funcionando correctamente con eventos
3. **Manejo de errores** con códigos HTTP apropiados
4. **Tests unitarios** cubriendo casos principales
5. **Configuración** correcta de async, email y base de datos
6. **Arquitectura en capas** bien estructurada (Controller → Service → Repository)
