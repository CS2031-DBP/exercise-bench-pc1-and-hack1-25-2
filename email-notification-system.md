## 📦 **Proyecto 4: Sistema de Notificaciones con Procesamiento Asíncrono**

### **Contexto**
Desarrollar un microservicio que gestione notificaciones por email con procesamiento asíncrono. El sistema debe permitir el registro de usuarios, autenticación JWT, y el envío de notificaciones que se procesan en segundo plano.

### **Requisitos Técnicos**

#### **1. Configuración Inicial**
- Base de datos H2 en memoria para desarrollo
- Configurar `application.properties` con:
  ```properties
  spring.datasource.url=jdbc:h2:mem:notificationdb
  spring.jpa.hibernate.ddl-auto=update
  spring.mail.host=${MAIL_HOST:smtp.gmail.com}
  spring.mail.port=${MAIL_PORT:587}
  spring.mail.username=${MAIL_USERNAME}
  spring.mail.password=${MAIL_PASSWORD}
  jwt.secret=${JWT_SECRET:defaultSecretKey123456789}
  jwt.expiration=3600000
  ```
- Habilitar procesamiento asíncrono con `@EnableAsync`

#### **2. Entidades a Implementar**

**User**
- `id` (Long, auto-generado)
- `username` (String, único, no nulo)
- `email` (String, formato email válido, no nulo)
- `password` (String, mínimo 8 caracteres, encriptado con BCrypt)
- `role` (Enum: USER, ADMIN)
- `createdAt` (LocalDateTime)

**Notification**
- `id` (Long, auto-generado)
- `recipientEmail` (String, formato email válido)
- `subject` (String, máximo 200 caracteres, no nulo)
- `body` (String, máximo 2000 caracteres, no nulo)
- `status` (Enum: PENDING, SENT, FAILED)
- `userId` (Long, FK a User)
- `createdAt` (LocalDateTime)
- `sentAt` (LocalDateTime, nullable)

#### **3. DTOs con Validaciones**

**RegisterRequest**
```java
- username (@NotBlank, @Size(min=3, max=50))
- email (@NotBlank, @Email)
- password (@NotBlank, @Size(min=8, max=100))
```

**LoginRequest**
```java
- username (@NotBlank)
- password (@NotBlank)
```

**NotificationRequest**
```java
- recipientEmail (@NotBlank, @Email)
- subject (@NotBlank, @Size(max=200))
- body (@NotBlank, @Size(max=2000))
```

**NotificationResponse**
```java
- id
- recipientEmail
- subject
- status
- createdAt
- sentAt
```

#### **4. Endpoints a Implementar**

**AuthController** (`/api/auth`)
- `POST /register` - Registrar nuevo usuario
  - Request: RegisterRequest
  - Response: 201 CREATED con mensaje "User registered successfully"
  - Validar que username no exista

- `POST /login` - Autenticar usuario
  - Request: LoginRequest
  - Response: 200 OK con JWT token en formato `{"token": "eyJ..."}`
  - Error 401 si credenciales inválidas

**NotificationController** (`/api/notifications`) - Requiere autenticación JWT
- `POST /send` - Crear y enviar notificación
  - Request: NotificationRequest
  - Response: 202 ACCEPTED con mensaje "Notification queued for processing"
  - Debe publicar evento `NotificationCreatedEvent`
  - Validar que el usuario autenticado exista

- `GET /my-notifications` - Listar notificaciones del usuario autenticado
  - Response: 200 OK con lista de NotificationResponse
  - Ordenar por createdAt descendente

- `GET /{id}` - Obtener notificación por ID
  - Response: 200 OK con NotificationResponse
  - Error 404 si no existe
  - Error 403 si la notificación no pertenece al usuario autenticado

**AdminController** (`/api/admin/notifications`) - Solo rol ADMIN
- `GET /all` - Listar todas las notificaciones del sistema
  - Response: 200 OK con lista completa
  - Requiere `@PreAuthorize("hasRole('ADMIN')")`

#### **5. Spring Security + JWT**

Implementar:
- `SecurityFilterChain` con configuración de endpoints públicos y protegidos
- `JwtUtil` con métodos:
  - `generateToken(String username, String role)` - retorna String
  - `validateToken(String token)` - retorna boolean
  - `extractUsername(String token)` - retorna String
  - `extractRole(String token)` - retorna String
- `JwtAuthenticationFilter` que:
  - Extrae token del header `Authorization: Bearer {token}`
  - Valida el token
  - Configura el SecurityContext con `UsernamePasswordAuthenticationToken`
- Configurar CORS para permitir origen `http://localhost:3000`

#### **6. Procesamiento Asíncrono**

**Evento**
```java
@Getter
public class NotificationCreatedEvent {
    private final Long notificationId;
    // Constructor
}
```

**NotificationEventListener**
- Método anotado con `@Async` y `@EventListener`
- Escucha `NotificationCreatedEvent`
- Simula envío de email (puede usar `Thread.sleep(2000)` para simular latencia)
- Actualiza status de la notificación a SENT o FAILED
- Registra `sentAt` con timestamp actual
- Manejo de excepciones: si falla, status = FAILED

#### **7. Configuración de Email**

Implementar `EmailService` con:
- `@Async` en método `sendEmail(String to, String subject, String body)`
- Usar `JavaMailSender`
- Configurar `MimeMessage` con encoding UTF-8
- Lanzar `MessagingException` si falla

#### **8. Manejo de Errores**

Implementar `GlobalExceptionHandler` con `@RestControllerAdvice`:

```java
- UsernameAlreadyExistsException → 400 BAD_REQUEST
  {"error": "Username already exists", "timestamp": "..."}

- InvalidCredentialsException → 401 UNAUTHORIZED
  {"error": "Invalid username or password", "timestamp": "..."}

- NotificationNotFoundException → 404 NOT_FOUND
  {"error": "Notification not found", "timestamp": "..."}

- AccessDeniedException → 403 FORBIDDEN
  {"error": "Access denied", "timestamp": "..."}

- MethodArgumentNotValidException → 400 BAD_REQUEST
  {"error": "Validation failed", "details": [...], "timestamp": "..."}

- EmailSendingException → 503 SERVICE_UNAVAILABLE
  {"error": "Email service temporarily unavailable", "timestamp": "..."}
```

#### **9. Testing**

Implementar tests unitarios para:

**AuthServiceTest**
```java
- testRegisterUser_Success()
- testRegisterUser_UsernameExists_ThrowsException()
- testLogin_Success_ReturnsToken()
- testLogin_InvalidCredentials_ThrowsException()
```

**NotificationServiceTest**
```java
- testCreateNotification_Success()
- testGetUserNotifications_ReturnsCorrectList()
- testGetNotificationById_NotFound_ThrowsException()
- testGetNotificationById_AccessDenied_ThrowsException()
```

**NotificationControllerTest** (con `@WebMvcTest`)
```java
- testSendNotification_Authenticated_Returns202()
- testSendNotification_InvalidData_Returns400()
- testGetMyNotifications_Authenticated_Returns200()
```

#### **10. Colección Postman**

Crear colección con:
- Variable de entorno `{{baseUrl}}` = `http://localhost:8080`
- Variable `{{token}}` que se actualiza automáticamente después del login
- Carpetas:
  1. **Auth**: register, login
  2. **Notifications**: send, my-notifications, get by id
  3. **Admin**: all notifications
- Test script en login que guarda el token:
  ```javascript
  pm.environment.set("token", pm.response.json().token);
  ```
