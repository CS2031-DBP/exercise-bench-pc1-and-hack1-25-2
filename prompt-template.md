Quiero que me ayudes a **generar un set 3 mini proyectos** que abarquen los siguientes temas de Spring Boot.

Requisitos de los mini proyectos
* **Duración estimada:** deben poder resolverse en máximo **2 horas**, de manera **individual**.
* **Nivel:** dificultad **medio/alto** (ni demasiado básico, ni tan complejo como para necesitar días).
* **Recursos permitidos:** se asume acceso a **documentación oficial, inteligencia artificial, y buscadores**.
* **Formato del enunciado:** el enunciado debe ser claro, auto contenido y debe
* **especificar explícitamente lo que se pide** que el participante implemente. Ejemplos:
   * endpoints con sus rutas y verbos HTTP.
   * entidades con sus atributos principales.
   * validaciones requeridas con anotaciones (`@NotNull`, `@Size`, etc.).
   * flujos de autenticación/autorización con Spring Security.
   * manejo de errores con códigos HTTP específicos.
   * uso de procesamiento asíncrono con `@Async` y eventos.

Temas clave a cubrir en los enunciados
1. **Core Spring Boot**
   * Configuración con `application.properties` y variables de entorno.
   * Inyección de dependencias y arquitectura en capas (Controller → Service → Repository).
   * DTOs y validación con Bean Validation.
2. **Spring Security + JWT**
   * Configuración de `SecurityFilterChain`.
   * Generación y validación de tokens JWT.
   * Autorización basada en roles (`@PreAuthorize`).
   * Filtros de autenticación personalizados.
3. **Spring Data JPA**
   * Configuración de entidades y relaciones.
   * Uso de repositorios JPA y queries personalizadas.
   * Configuración de base de datos (H2 en memoria o PostgreSQL).
4. **Procesamiento Asíncrono**
   * Uso de `@EnableAsync` y `@Async`.
   * Publicación de eventos con `ApplicationEventPublisher`.
   * Consumo con `@EventListener`.
   * Flujo típico: Controller → Evento → Procesamiento async → Response inmediato.
5. **Integración con APIs Externas**
   * Uso de `RestTemplate` o `WebClient`.
   * Configuración de headers y autenticación.
   * Manejo de errores en llamadas externas.
6. **Envío de Emails**
   * Configuración de `JavaMailSender` y propiedades SMTP.
   * Envío asíncrono de correos.
   * Manejo de errores en envío.
7. **Manejo de Errores**
   * Uso de `@RestControllerAdvice` y `@ExceptionHandler`.
   * Respuestas con códigos HTTP correctos (201, 202, 400, 401, 403, 404, 503).
   * Formato estándar de respuesta de error.
8. **Testing**
   * Tests unitarios con `MockitoExtension`.
   * Mocking de dependencias con `@Mock` y `@InjectMocks`.
   * Testing de controllers con `@WebMvcTest`.
9. **Configuración y Deployment**
   * Uso de variables de entorno y secrets.
   * Profiles de Spring (`dev`, `prod`).
   * Configuración de CORS si aplica.
10. **Postman / Testing de APIs**
* Creación de colecciones.
* Uso de variables de entorno.
* Testing de un flujo completo (auth → CRUD → async).
*
👉 En resumen: **quiero que los enunciados de proyectos estén diseñados para ejercitar estos temas en escenarios prácticos, con instrucciones claras y detalladas de lo que debe implementarse.**
