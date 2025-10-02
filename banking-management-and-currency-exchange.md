
## 🏦 **Proyecto 5: API de Gestión Bancaria con Integración Externa**

### **Contexto**
Desarrollar una API REST para gestión de cuentas bancarias que se integra con una API externa de tipos de cambio. El sistema debe manejar múltiples monedas, transacciones, y autenticación con roles diferenciados.

### **Requisitos Técnicos**

#### **1. Configuración Inicial**
- Base de datos H2 en memoria
- Configurar `application.properties`:
  ```properties
  spring.datasource.url=jdbc:h2:mem:bankdb
  spring.jpa.show-sql=true
  api.exchange.url=${EXCHANGE_API_URL:https://api.exchangerate-api.com/v4/latest}
  api.exchange.timeout=5000
  jwt.secret=${JWT_SECRET:bankSecretKey987654321}
  jwt.expiration=7200000
  ```
- Profiles: `dev` (H2) y `prod` (configurar para PostgreSQL)

#### **2. Entidades a Implementar**

**Customer**
- `id` (Long)
- `firstName` (String, no nulo, máximo 100 caracteres)
- `lastName` (String, no nulo, máximo 100 caracteres)
- `email` (String, único, formato email)
- `password` (String, encriptado)
- `role` (Enum: CUSTOMER, ADMIN)
- `createdAt` (LocalDateTime)

**Account**
- `id` (Long)
- `accountNumber` (String, único, generado automáticamente: "ACC" + 10 dígitos)
- `currency` (Enum: USD, EUR, PEN)
- `balance` (BigDecimal, no negativo, precisión 2 decimales)
- `customerId` (Long, FK)
- `status` (Enum: ACTIVE, BLOCKED, CLOSED)
- `createdAt` (LocalDateTime)
- `updatedAt` (LocalDateTime)

**Transaction**
- `id` (Long)
- `accountId` (Long, FK)
- `type` (Enum: DEPOSIT, WITHDRAWAL, TRANSFER_OUT, TRANSFER_IN)
- `amount` (BigDecimal, positivo)
- `currency` (Enum: USD, EUR, PEN)
- `description` (String, máximo 500 caracteres)
- `balanceAfter` (BigDecimal)
- `createdAt` (LocalDateTime)
- `referenceTransactionId` (Long, nullable, para vincular transferencias)

#### **3. DTOs con Validaciones**

**RegisterCustomerRequest**
```java
- firstName (@NotBlank, @Size(max=100))
- lastName (@NotBlank, @Size(max=100))
- email (@NotBlank, @Email)
- password (@NotBlank, @Size(min=8))
```

**CreateAccountRequest**
```java
- currency (@NotNull)
- initialDeposit (@NotNull, @DecimalMin("0.0"), @Digits(integer=10, fraction=2))
```

**DepositRequest**
```java
- accountNumber (@NotBlank, @Pattern(regexp="ACC\\d{10}"))
- amount (@NotNull, @DecimalMin(value="0.01"))
- description (@Size(max=500))
```

**TransferRequest**
```java
- fromAccountNumber (@NotBlank, @Pattern(regexp="ACC\\d{10}"))
- toAccountNumber (@NotBlank, @Pattern(regexp="ACC\\d{10}"))
- amount (@NotNull, @DecimalMin(value="0.01"))
- description (@Size(max=500))
```

**ExchangeRequest**
```java
- accountNumber (@NotBlank)
- amount (@NotNull, @DecimalMin(value="0.01"))
- fromCurrency (@NotNull)
- toCurrency (@NotNull)
```

#### **4. Endpoints a Implementar**

**AuthController** (`/api/auth`)
- `POST /register` - Registrar cliente
  - Request: RegisterCustomerRequest
  - Response: 201 CREATED con customer ID

- `POST /login` - Login
  - Response: JWT con roles incluidos

**AccountController** (`/api/accounts`) - Autenticado
- `POST /create` - Crear cuenta bancaria
  - Request: CreateAccountRequest
  - Response: 201 CREATED con AccountResponse (id, accountNumber, currency, balance)
  - El customer debe estar autenticado
  - Generar accountNumber único automáticamente

- `GET /my-accounts` - Listar cuentas del cliente autenticado
  - Response: 200 OK con lista de AccountResponse

- `GET /{accountNumber}` - Obtener detalle de cuenta
  - Response: 200 OK con AccountResponse
  - Error 404 si no existe
  - Error 403 si no pertenece al usuario autenticado (excepto ADMIN)

- `GET /{accountNumber}/transactions` - Historial de transacciones
  - Query params: `?page=0&size=20`
  - Response: 200 OK con lista paginada de TransactionResponse
  - Ordenar por createdAt descendente

**TransactionController** (`/api/transactions`) - Autenticado
- `POST /deposit` - Realizar depósito
  - Request: DepositRequest
  - Response: 201 CREATED con TransactionResponse
  - Validar que la cuenta exista y pertenezca al usuario
  - Actualizar balance atómicamente

- `POST /withdrawal` - Realizar retiro
  - Request: DepositRequest (reutilizar)
  - Response: 201 CREATED con TransactionResponse
  - Validar fondos suficientes
  - Error 400 si balance insuficiente

- `POST /transfer` - Transferencia entre cuentas
  - Request: TransferRequest
  - Response: 201 CREATED con ambas transacciones (OUT e IN)
  - Validar que ambas cuentas existan
  - Validar que origen pertenezca al usuario autenticado
  - Debe ser transaccional (`@Transactional`)
  - Crear dos registros de Transaction vinculados por referenceTransactionId
  - Error 400 si misma cuenta origen/destino o fondos insuficientes

**ExchangeController** (`/api/exchange`) - Autenticado
- `POST /convert` - Convertir saldo entre monedas
  - Request: ExchangeRequest
  - Response: 200 OK con ExchangeResponse (originalAmount, convertedAmount, rate, newBalance)
  - Llamar a API externa para obtener tipo de cambio
  - Actualizar balance de la cuenta
  - Crear transacción tipo WITHDRAWAL y DEPOSIT
  - Error 503 si API externa no responde
  - Error 400 si fromCurrency == toCurrency

**AdminController** (`/api/admin`) - Solo ADMIN
- `GET /customers` - Listar todos los clientes
  - Response: 200 OK con lista completa

- `PUT /accounts/{accountNumber}/status` - Cambiar estado de cuenta
  - Request: `{"status": "BLOCKED"}`
  - Response: 200 OK con AccountResponse actualizado
  - Requiere `@PreAuthorize("hasRole('ADMIN')")`

#### **5. Integración con API Externa**

Implementar `ExchangeRateService`:
- Usar `RestTemplate` o `WebClient`
- Método `getExchangeRate(String fromCurrency, String toCurrency)`
- URL: `${api.exchange.url}/{fromCurrency}`
- Parsear JSON response:
  ```json
  {
    "base": "USD",
    "rates": {
      "EUR": 0.85,
      "PEN": 3.75
    }
  }
  ```
- Timeout de 5 segundos
- Cachear resultados por 10 minutos usando `@Cacheable` (opcional pero recomendado)
- Lanzar `ExchangeApiException` si falla

#### **6. Spring Security**

Configurar:
- Endpoints públicos: `/api/auth/**`
- Endpoints protegidos: `/api/accounts/**`, `/api/transactions/**`, `/api/exchange/**`
- Endpoints admin: `/api/admin/**` con `hasRole('ADMIN')`
- JWT debe incluir: username, role, customerId
- `JwtAuthenticationFilter` con extracción de customerId para contexto

#### **7. Manejo de Errores**

`GlobalExceptionHandler`:
```java
- InsufficientFundsException → 400
  {"error": "Insufficient funds", "availableBalance": 100.50, "timestamp": "..."}

- AccountNotFoundException → 404
  {"error": "Account not found", "timestamp": "..."}

- SameAccountTransferException → 400
  {"error": "Cannot transfer to the same account", "timestamp": "..."}

- ExchangeApiException → 503
  {"error": "Exchange rate service unavailable", "timestamp": "..."}

- AccessDeniedException → 403
  {"error": "You don't have permission to access this account", "timestamp": "..."}
```

#### **8. Testing**

**AccountServiceTest**
```java
- testCreateAccount_Success()
- testCreateAccount_InvalidInitialDeposit_ThrowsException()
- testGetAccountsByCustomer_ReturnsCorrectList()
- testGetAccountByNumber_NotOwned_ThrowsAccessDeniedException()
```

**TransactionServiceTest**
```java
- testDeposit_Success_UpdatesBalance()
- testWithdrawal_InsufficientFunds_ThrowsException()
- testTransfer_Success_CreatesLinkedTransactions()
- testTransfer_SameAccount_ThrowsException()
```

**ExchangeRateServiceTest**
```java
- testGetExchangeRate_Success_ReturnsCorrectRate()
- testGetExchangeRate_ApiTimeout_ThrowsException()
- testGetExchangeRate_InvalidCurrency_ThrowsException()
```

**TransactionControllerTest** (con `@WebMvcTest`)
```java
- testDeposit_ValidRequest_Returns201()
- testWithdrawal_InsufficientFunds_Returns400()
- testTransfer_Unauthenticated_Returns401()
```

#### **9. Configuración Adicional**

- Implementar `@Transactional` en métodos de transferencia
- Usar `@Lock(LockModeType.PESSIMISTIC_WRITE)` en Account para evitar race conditions
- Configurar `RestTemplate` con timeout:
  ```java
  @Bean
  public RestTemplate restTemplate() {
      SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
      factory.setConnectTimeout(5000);
      factory.setReadTimeout(5000);
      return new RestTemplate(factory);
  }
  ```

#### **10. Colección Postman**

Variables de entorno:
- `{{baseUrl}}`
- `{{token}}`
- `{{accountNumber}}` (guardar después de crear cuenta)

Flujo completo:
1. Register → Login → Save token
2. Create Account → Save accountNumber
3. Deposit → Withdrawal
4. Create second account
5. Transfer entre cuentas
6. Exchange currency
7. Get transactions history
8. (Como ADMIN) Block account
