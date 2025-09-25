## Proyecto 3: Sistema de Monitoreo de Dispositivos IoT con WebClient y Procesamiento Asíncrono
**⏱️ Duración estimada:** 2 horas
**🎯 Temas principales:** WebClient, Procesamiento Asíncrono, Eventos, Email, Configuración Avanzada, Testing

### Descripción del Sistema
Desarrollar una API REST para monitorear dispositivos IoT que consume datos de múltiples APIs externas de forma asíncrona, procesa alertas automáticamente y envía notificaciones por email cuando se detectan anomalías.

### Entidades Requeridas

#### Entidad `Device`
```java
@Entity
public class Device {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    @NotBlank @Size(min = 3, max = 50)
    private String deviceId;

    @NotBlank @Size(min = 2, max = 100)
    private String name;

    @Enumerated(EnumType.STRING)
    private DeviceType type; // TEMPERATURE, HUMIDITY, PRESSURE, MOTION

    @NotBlank
    private String location;

    @NotBlank
    private String apiEndpoint; // URL para obtener datos del dispositivo

    private Double minThreshold;
    private Double maxThreshold;

    @NotBlank
    private String alertEmail;

    private boolean active = true;

    @CreationTimestamp
    private LocalDateTime createdAt;

    // constructors, getters, setters
}
```

#### Entidad `DeviceReading`
```java
@Entity
public class DeviceReading {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "device_id", nullable = false)
    private Device device;

    @NotNull
    private Double value;

    private String unit;

    @CreationTimestamp
    private LocalDateTime timestamp;

    @Enumerated(EnumType.STRING)
    private ReadingStatus status; // NORMAL, WARNING, CRITICAL

    // constructors, getters, setters
}
```

#### Entidad `Alert`
```java
@Entity
public class Alert {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "device_id", nullable = false)
    private Device device;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reading_id", nullable = false)
    private DeviceReading reading;

    @NotBlank
    private String message;

    @Enumerated(EnumType.STRING)
    private AlertSeverity severity; // LOW, MEDIUM, HIGH, CRITICAL

    @CreationTimestamp
    private LocalDateTime createdAt;

    private LocalDateTime resolvedAt;

    private boolean emailSent = false;

    // constructors, getters, setters
}
```

### DTOs Requeridos

#### Device DTOs
```java
public class CreateDeviceRequest {
    @NotBlank @Size(min = 3, max = 50)
    private String deviceId;

    @NotBlank @Size(min = 2, max = 100)
    private String name;

    @NotNull
    private DeviceType type;

    @NotBlank
    private String location;

    @NotBlank @URL
    private String apiEndpoint;

    private Double minThreshold;
    private Double maxThreshold;

    @Email @NotBlank
    private String alertEmail;

    // getters, setters
}

public class DeviceResponse {
    private Long id;
    private String deviceId;
    private String name;
    private DeviceType type;
    private String location;
    private String apiEndpoint;
    private Double minThreshold;
    private Double maxThreshold;
    private String alertEmail;
    private boolean active;
    private LocalDateTime createdAt;
    private LocalDateTime lastReading;
    private Double lastValue;

    // constructors, getters, setters
}

public class DeviceReadingResponse {
    private Long id;
    private String deviceId;
    private String deviceName;
    private Double value;
    private String unit;
    private LocalDateTime timestamp;
    private ReadingStatus status;

    // constructors, getters, setters
}

public class AlertResponse {
    private Long id;
    private String deviceId;
    private String deviceName;
    private String message;
    private AlertSeverity severity;
    private LocalDateTime createdAt;
    private LocalDateTime resolvedAt;
    private boolean emailSent;

    // constructors, getters, setters
}
```

#### External API DTOs
```java
// DTO para recibir datos de APIs externas de dispositivos
public class ExternalDeviceData {
    private String deviceId;
    private Double temperature;
    private Double humidity;
    private Double pressure;
    private String unit;
    private Long timestamp;
    private String status;

    // getters, setters
}
```

### Configuración de WebClient

#### WebClientConfig
```java
@Configuration
public class WebClientConfig {

    @Bean
    @Primary
    public WebClient defaultWebClient() {
        return WebClient.builder()
            .codecs(configurer -> configurer.defaultCodecs().maxInMemorySize(2 * 1024 * 1024))
            .build();
    }

    @Bean("deviceApiWebClient")
    public WebClient deviceApiWebClient() {
        return WebClient.builder()
            .defaultHeader(HttpHeaders.USER_AGENT, "IoT-Monitor/1.0")
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .filter(ExchangeFilterFunction.ofRequestProcessor(clientRequest -> {
                // Log de requests
                System.out.println("Request: " + clientRequest.method() + " " + clientRequest.url());
                return Mono.just(clientRequest);
            }))
            .filter(ExchangeFilterFunction.ofResponseProcessor(clientResponse -> {
                // Log de responses y manejo de errores
                if (clientResponse.statusCode().isError()) {
                    System.err.println("Error response: " + clientResponse.statusCode());
                }
                return Mono.just(clientResponse);
            }))
            .build();
    }

    @Bean("weatherApiWebClient")
    public WebClient weatherApiWebClient(@Value("${external.weather.api.key}") String apiKey) {
        return WebClient.builder()
            .baseUrl("https://api.openweathermap.org/data/2.5")
            .defaultHeader("X-API-Key", apiKey)
            .build();
    }
}
```

### Endpoints a Implementar

#### DeviceController
- `POST /api/devices` - Registrar nuevo dispositivo
  - **Request Body:** `CreateDeviceRequest`
  - **Validaciones:** `@Valid`, deviceId único, URL válida
  - **Response:** `201 CREATED` con `DeviceResponse`
  - **Error:** `409 CONFLICT` si deviceId ya existe

- `GET /api/devices` - Listar todos los dispositivos
  - **Query Parameters:** `?active=true&type=TEMPERATURE`
  - **Response:** `200 OK` con lista de `DeviceResponse`

- `GET /api/devices/{id}` - Obtener dispositivo por ID
  - **Response:** `200 OK` con `DeviceResponse`
  - **Error:** `404 NOT FOUND` si no existe

- `PUT /api/devices/{id}/toggle` - Activar/desactivar dispositivo
  - **Response:** `200 OK` con `DeviceResponse`
  - **Error:** `404 NOT FOUND` si no existe

- `POST /api/devices/{id}/collect-data` - Recopilar datos manualmente
  - **Flujo:**
    1. Llamar a la API externa del dispositivo usando WebClient
    2. Procesar la respuesta de forma asíncrona
    3. Retornar `202 ACCEPTED` inmediatamente
    4. **Publicar evento `DataCollectionRequestedEvent`**
  - **Response:** `202 ACCEPTED` con mensaje de confirmación
  - **Error:** `404 NOT FOUND` si dispositivo no existe o está inactivo

#### ReadingController
- `GET /api/readings` - Obtener lecturas con filtros
  - **Query Parameters:** `?deviceId=sensor1&from=2025-01-01T00:00:00&to=2025-01-31T23:59:59&status=CRITICAL`
  - **Response:** `200 OK` con lista paginada de `DeviceReadingResponse`

- `GET /api/readings/latest` - Obtener últimas lecturas de todos los dispositivos
  - **Response:** `200 OK` con lista de `DeviceReadingResponse`

#### AlertController
- `GET /api/alerts` - Obtener alertas con filtros
  - **Query Parameters:** `?severity=HIGH&resolved=false&deviceId=sensor1`
  - **Response:** `200 OK` con lista paginada de `AlertResponse`

- `PUT /api/alerts/{id}/resolve` - Marcar alerta como resuelta
  - **Response:** `200 OK` con `AlertResponse`
  - **Error:** `404 NOT FOUND` si no existe

#### MonitoringController
- `POST /api/monitoring/start-collection` - Iniciar recopilación automática
  - **Flujo:** Iniciar tarea programada para recopilar datos de todos los dispositivos activos
  - **Response:** `200 OK` con mensaje de confirmación

- `POST /api/monitoring/stop-collection` - Detener recopilación automática
  - **Response:** `200 OK` con mensaje de confirmación

- `GET /api/monitoring/status` - Obtener estado del monitoreo
  - **Response:** `200 OK` con estadísticas (devices activos, lecturas hoy, alertas pendientes)

### Servicios de Integración Externa

#### DeviceDataCollectionService
```java
@Service
public class DeviceDataCollectionService {

    @Autowired
    @Qualifier("deviceApiWebClient")
    private WebClient deviceApiWebClient;

    @Autowired
    @Qualifier("weatherApiWebClient")
    private WebClient weatherApiWebClient;

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Async("dataCollectionExecutor")
    public CompletableFuture<Void> collectDataFromDevice(Device device) {
        return deviceApiWebClient
            .get()
            .uri(device.getApiEndpoint())
            .retrieve()
            .onStatus(HttpStatus::isError, clientResponse -> {
                // Manejar errores HTTP
                return Mono.error(new ExternalApiException("Error calling device API: " + clientResponse.statusCode()));
            })
            .bodyToMono(ExternalDeviceData.class)
            .timeout(Duration.ofSeconds(10))
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(2)))
            .doOnSuccess(data -> {
                // Publicar evento con los datos recibidos
                eventPublisher.publishEvent(new DeviceDataReceivedEvent(device.getId(), data));
            })
            .doOnError(error -> {
                // Log error y publicar evento de error
                System.err.println("Error collecting data from device " + device.getDeviceId() + ": " + error.getMessage());
                eventPublisher.publishEvent(new DeviceDataCollectionErrorEvent(device.getId(), error.getMessage()));
            })
            .then()
            .toFuture();
    }

    @Async("dataCollectionExecutor")
    public CompletableFuture<Void> collectWeatherData(String location) {
        return weatherApiWebClient
            .get()
            .uri("/weather?q={location}&units=metric", location)
            .retrieve()
            .bodyToMono(JsonNode.class)
            .timeout(Duration.ofSeconds(15))
            .doOnSuccess(weatherData -> {
                // Procesar datos meteorológicos
                // Publicar evento si es necesario
            })
            .then()
            .toFuture();
    }
}
```

#### DataProcessingService
```java
@Service
public class DataProcessingService {

    @Autowired
    private DeviceReadingRepository readingRepository;

    @Autowired
    private AlertService alertService;

    @EventListener
    @Async("dataProcessingExecutor")
    public void handleDeviceDataReceived(DeviceDataReceivedEvent event) {
        try {
            // 1. Simular procesamiento de 1-2 segundos
            Thread.sleep(1500);

            // 2. Crear DeviceReading
            DeviceReading reading = createReadingFromExternalData(event.getDeviceId(), event.getData());

            // 3. Determinar status (NORMAL, WARNING, CRITICAL)
            ReadingStatus status = determineReadingStatus(reading);
            reading.setStatus(status);

            // 4. Guardar lectura
            reading = readingRepository.save(reading);

            // 5. Si status es WARNING o CRITICAL, crear alerta
            if (status == ReadingStatus.WARNING || status == ReadingStatus.CRITICAL) {
                alertService.createAlert(reading);
            }

            System.out.println("Processed reading for device " + event.getDeviceId() +
                             " - Value: " + reading.getValue() + " - Status: " + status);

        } catch (Exception e) {
            System.err.println("Error processing device data: " + e.getMessage());
        }
    }

    private DeviceReading createReadingFromExternalData(Long deviceId, ExternalDeviceData data) {
        // Implementar conversión de ExternalDeviceData a DeviceReading
        // Determinar el valor según el tipo de dispositivo
    }

    private ReadingStatus determineReadingStatus(DeviceReading reading) {
        Device device = reading.getDevice();
        Double value = reading.getValue();

        if (device.getMinThreshold() != null && value < device.getMinThreshold()) {
            return ReadingStatus.CRITICAL;
        }
        if (device.getMaxThreshold() != null && value > device.getMaxThreshold()) {
            return ReadingStatus.CRITICAL;
        }

        // Implementar lógica para WARNING basada en proximidad a thresholds
        return ReadingStatus.NORMAL;
    }
}
```

### Eventos de Sistema

#### Eventos Personalizados
```java
public class DeviceDataReceivedEvent {
    private final Long deviceId;
    private final ExternalDeviceData data;
    private final LocalDateTime timestamp;

    public DeviceDataReceivedEvent(Long deviceId, ExternalDeviceData data) {
        this.deviceId = deviceId;
        this.data = data;
        this.timestamp = LocalDateTime.now();
    }

    // getters
}

public class AlertCreatedEvent {
    private final Long alertId;
    private final Long deviceId;
    private final String deviceName;
    private final AlertSeverity severity;
    private final String alertEmail;
    private final String message;

    // constructor, getters
}

public class DataCollectionRequestedEvent {
    private final Long deviceId;
    private final String requestedBy;
    private final LocalDateTime timestamp;

    // constructor, getters
}

public class DeviceDataCollectionErrorEvent {
    private final Long deviceId;
    private final String errorMessage;
    private final LocalDateTime timestamp;

    // constructor, getters
}
```

### Configuración de Procesamiento Asíncrono

#### AsyncConfig
```java
@Configuration
@EnableAsync
@EnableScheduling
public class AsyncConfig {

    @Bean(name = "dataCollectionExecutor")
    public TaskExecutor dataCollectionExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("data-collection-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Bean(name = "dataProcessingExecutor")
    public TaskExecutor dataProcessingExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(15);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("data-processing-");
        executor.initialize();
        return executor;
    }

    @Bean(name = "emailExecutor")
    public TaskExecutor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("email-");
        executor.initialize();
        return executor;
    }
}
```

### Tareas Programadas

#### ScheduledDataCollectionService
```java
@Service
public class ScheduledDataCollectionService {

    @Autowired
    private DeviceRepository deviceRepository;

    @Autowired
    private DeviceDataCollectionService dataCollectionService;

    private boolean collectionEnabled = false;

    @Scheduled(fixedRate = 300000) // Cada 5 minutos
    public void collectDataFromAllDevices() {
        if (!collectionEnabled) {
            return;
        }

        System.out.println("Starting scheduled data collection...");

        List<Device> activeDevices = deviceRepository.findByActiveTrue();

        for (Device device : activeDevices) {
            dataCollectionService.collectDataFromDevice(device)
                .whenComplete((result, throwable) -> {
                    if (throwable != null) {
                        System.err.println("Failed to collect data from device: " + device.getDeviceId());
                    }
                });
        }

        System.out.println("Scheduled data collection initiated for " + activeDevices.size() + " devices");
    }

    public void enableCollection() {
        this.collectionEnabled = true;
    }

    public void disableCollection() {
        this.collectionEnabled = false;
    }

    public boolean isCollectionEnabled() {
        return collectionEnabled;
    }
}
```

### Servicio de Alertas y Email

#### AlertService
```java
@Service
public class AlertService {

    @Autowired
    private AlertRepository alertRepository;

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    public Alert createAlert(DeviceReading reading) {
        Device device = reading.getDevice();

        // Determinar severidad basada en el valor y thresholds
        AlertSeverity severity = determineAlertSeverity(reading);

        String message = generateAlertMessage(device, reading, severity);

        Alert alert = new Alert();
        alert.setDevice(device);
        alert.setReading(reading);
        alert.setSeverity(severity);
        alert.setMessage(message);

        alert = alertRepository.save(alert);

        // Publicar evento para envío de email
        eventPublisher.publishEvent(new AlertCreatedEvent(
            alert.getId(),
            device.getId(),
            device.getName(),
            severity,
            device.getAlertEmail(),
            message
        ));

        return alert;
    }

    private AlertSeverity determineAlertSeverity(DeviceReading reading) {
        Device device = reading.getDevice();
        Double value = reading.getValue();

        // Implementar lógica de severidad
        if (device.getMaxThreshold() != null && value > device.getMaxThreshold() * 1.5) {
            return AlertSeverity.CRITICAL;
        }
        if (device.getMinThreshold() != null && value < device.getMinThreshold() * 0.5) {
            return AlertSeverity.CRITICAL;
        }
        // Más lógica para HIGH, MEDIUM, LOW...

        return AlertSeverity.MEDIUM;
    }

    private String generateAlertMessage(Device device, DeviceReading reading, AlertSeverity severity) {
        return String.format("Alert %s for device %s (%s): Value %s %s at %s. Location: %s",
            severity, device.getName(), device.getDeviceId(),
            reading.getValue(), reading.getUnit(), reading.getTimestamp(),
            device.getLocation());
    }
}
```

#### EmailAlertService
```java
@Service
public class EmailAlertService {

    @Autowired
    private JavaMailSender mailSender;

    @Autowired
    private AlertRepository alertRepository;

    @Value("${spring.mail.username}")
    private String fromEmail;

    @EventListener
    @Async("emailExecutor")
    public void handleAlertCreated(AlertCreatedEvent event) {
        try {
            // Simular delay de procesamiento
            Thread.sleep(1000);

            sendAlertEmail(
                event.getAlertEmail(),
                event.getDeviceName(),
                event.getMessage(),
                event.getSeverity()
            );

            // Marcar email como enviado
            alertRepository.findById(event.getAlertId()).ifPresent(alert -> {
                alert.setEmailSent(true);
                alertRepository.save(alert);
            });

            System.out.println("Alert email sent for device: " + event.getDeviceName());

        } catch (Exception e) {
            System.err.println("Failed to send alert email: " + e.getMessage());
        }
    }

    private void sendAlertEmail(String to, String deviceName, String message, AlertSeverity severity) {
        try {
            MimeMessage mimeMessage = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);

            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(String.format("IoT Alert [%s] - %s", severity, deviceName));

            String htmlContent = buildEmailContent(deviceName, message, severity);
            helper.setText(htmlContent, true);

            mailSender.send(mimeMessage);

        } catch (Exception e) {
            throw new EmailSendException("Failed to send alert email", e);
        }
    }

    private String buildEmailContent(String deviceName, String message, AlertSeverity severity) {
        // Implementar template HTML para el email
        return String.format("""
            <html>
            <body>
                <h2 style="color: %s;">IoT Device Alert</h2>
                <p><strong>Device:</strong> %s</p>
                <p><strong>Severity:</strong> %s</p>
                <p><strong>Message:</strong> %s</p>
                <p><strong>Timestamp:</strong> %s</p>
            </body>
            </html>
            """,
            getSeverityColor(severity), deviceName, severity, message, LocalDateTime.now());
    }

    private String getSeverityColor(AlertSeverity severity) {
        return switch (severity) {
            case CRITICAL -> "#FF0000";
            case HIGH -> "#FF6600";
            case MEDIUM -> "#FFAA00";
            case LOW -> "#00AA00";
        };
    }
}
```

### Configuración de Aplicación

#### application.properties
```properties
# Database H2
spring.datasource.url=jdbc:h2:mem:iotmonitor
spring.datasource.driverClassName=org.h2.Driver
spring.h2.console.enabled=true
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=false

# Email Configuration
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=${MAIL_USERNAME:iot-monitor@example.com}
spring.mail.password=${MAIL_PASSWORD:password}
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

# External APIs
external.weather.api.key=${WEATHER_API_KEY:your-api-key}
external.device.api.timeout=10000
external.device.api.retries=3

# Async Configuration
spring.task.execution.pool.core-size=10
spring.task.execution.pool.max-size=20
spring.task.execution.pool.queue-capacity=500

# Logging
logging.level.com.example.iotmonitor=INFO
logging.level.reactor.netty.http.client=DEBUG
logging.level.org.springframework.web.reactive=DEBUG

# Management (Actuator)
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=always
```

#### application-test.properties
```properties
# Test Database
spring.datasource.url=jdbc:h2:mem:testiot
spring.jpa.hibernate.ddl-auto=create-drop

# Disable emails in tests
spring.mail.host=localhost
spring.mail.port=25

# Disable scheduling in tests
spring.task.scheduling.enabled=false

# External API Mock
external.device.mock.enabled=true
external.weather.api.key=test-key
```

### Testing Requerido

#### Integration Tests
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(locations = "classpath:application-test.properties")
class DeviceMonitoringIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @MockBean
    private DeviceDataCollectionService dataCollectionService;

    @Test
    void shouldCreateDeviceAndTriggerDataCollection() {
        // 1. Crear dispositivo
        CreateDeviceRequest request = new CreateDeviceRequest();
        // ... configurar request

        ResponseEntity<DeviceResponse> response = restTemplate.postForEntity(
            "/api/devices", request, DeviceResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        // 2. Solicitar recopilación de datos
        Long deviceId = response.getBody().getId();
        ResponseEntity<String> collectResponse = restTemplate.postForEntity(
            "/api/devices/" + deviceId + "/collect-data", null, String.class);

        assertThat(collectResponse.getStatusCode()).isEqualTo(HttpStatus.ACCEPTED);

        // 3. Verificar que se llamó al servicio de recopilación
        verify(dataCollectionService).collectDataFromDevice(any(Device.class));
    }
}
```

#### WebClient Tests
```java
@ExtendWith(MockitoExtension.class)
class DeviceDataCollectionServiceTest {

    private DeviceDataCollectionService service;
    private MockWebServer mockWebServer;

    @BeforeEach
    void setUp() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();

        WebClient webClient = WebClient.builder()
            .baseUrl(mockWebServer.url("/").toString())
            .build();

        service = new DeviceDataCollectionService();
        // Inyectar webClient usando reflection o setter
    }

    @AfterEach
    void tearDown() throws IOException {
        mockWebServer.shutdown();
    }

    @Test
    void shouldCollectDataSuccessfully() {
        // Configurar mock response
        mockWebServer.enqueue(new MockResponse()
            .setResponseCode(200)
            .setHeader("Content-Type", "application/json")
            .setBody("{\"deviceId\": \"temp01\", \"temperature\": 23.5, \"unit\": \"C\"}"));

        Device device = new Device();
        device.setDeviceId("temp01");
        device.setApiEndpoint("/api/device-data");

        // Ejecutar
        CompletableFuture<Void> future = service.collectDataFromDevice(device);

        // Verificar
        assertThat(future).succeedsWithin(Duration.ofSeconds(5));

        RecordedRequest request = mockWebServer.takeRequest();
        assertThat(request.getPath()).isEqualTo("/api/device-data");
        assertThat(request.getMethod()).isEqualTo("GET");
    }

    @Test
    void shouldHandleApiError() {
        // Configurar mock error response
        mockWebServer.enqueue(new MockResponse()
            .setResponseCode(500)
            .setBody("Internal Server Error"));

        Device device = new Device();
        device.setApiEndpoint("/api/device-data");

        // Verificar que maneja el error apropiadamente
        CompletableFuture<Void> future = service.collectDataFromDevice(device);

        // El CompletableFuture debe completarse sin excepción (error manejado)
        assertThat(future).succeedsWithin(Duration.ofSeconds(10));
    }
}
```

#### Async Processing Tests
```java
@SpringBootTest
@TestPropertySource(locations = "classpath:application-test.properties")
class AsyncProcessingTest {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Autowired
    private DeviceReadingRepository readingRepository;

    @MockBean
    private AlertService alertService;

    @Test
    void shouldProcessDeviceDataAsynchronously() throws InterruptedException {
        // Crear datos de prueba
        ExternalDeviceData data = new ExternalDeviceData();
        data.setDeviceId("test01");
        data.setTemperature(25.0);

        // Publicar evento
        eventPublisher.publishEvent(new DeviceDataReceivedEvent(1L, data));

        // Esperar procesamiento asíncrono
        Thread.sleep(3000);

        // Verificar que se guardó la lectura
        List<DeviceReading> readings = readingRepository.findAll();
        assertThat(readings).isNotEmpty();

        DeviceReading reading = readings.get(0);
        assertThat(reading.getValue()).isEqualTo(25.0);
    }
}
```

### Criterios de Evaluación
1. **WebClient configurado correctamente** con filtros y manejo de errores
2. **Procesamiento asíncrono** con múltiples thread pools
3. **Integración con APIs externas** con retry y timeout
4. **Sistema de eventos** funcionando correctamente
5. **Envío de emails** asíncrono basado en eventos
6. **Tareas programadas** para recopilación automática de datos
7. **Tests de integración** cubriendo WebClient y procesamiento async
8. **Manejo robusto de errores** en llamadas externas
9. **Configuración por profiles** (dev, test, prod)
10. **Logging apropiado** para debugging y monitoreo

### Funcionalidades Adicionales Opcionales (Bonus)
- **Métricas con Micrometer/Actuator** para monitoreo de performance
- **Rate limiting** para las llamadas a APIs externas
- **Circuit breaker** con Resilience4j para APIs externas
- **Cache** con Spring Cache para lecturas frecuentes
- **Websockets** para notificaciones en tiempo real
