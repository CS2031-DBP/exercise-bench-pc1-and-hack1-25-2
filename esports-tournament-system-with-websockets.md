# 🎮 **Proyecto 4: Sistema de Torneos de eSports con WebSockets y Cache**

## **Contexto**
Desarrollar una plataforma de gestión de torneos de videojuegos que incluye registro de equipos, matchmaking automático, actualizaciones en tiempo real de partidas vía WebSockets, sistema de rankings con caché distribuido, y generación de reportes estadísticos procesados de forma asíncrona.

## **Requisitos Técnicos**

### **1. Configuración Inicial**
- Base de datos H2 en memoria para desarrollo
- Configurar `application.properties`:
  ```properties
  spring.datasource.url=jdbc:h2:mem:esportsdb
  spring.jpa.hibernate.ddl-auto=create-drop
  spring.cache.type=caffeine
  spring.cache.caffeine.spec=maximumSize=500,expireAfterWrite=300s
  jwt.secret=${JWT_SECRET:esportsSecretKey123}
  jwt.expiration=7200000
  tournament.matchmaking.algorithm=${MATCHMAKING_ALGO:ELO_BASED}
  tournament.max.teams.per.tournament=32
  websocket.allowed.origins=http://localhost:3000,http://localhost:4200
  ```
- Habilitar `@EnableAsync`, `@EnableCaching`, `@EnableScheduling`
- Configurar WebSocket con STOMP

### **2. Entidades a Implementar**

**Player**
- `id` (Long)
- `username` (String, único, 3-20 caracteres alfanuméricos)
- `email` (String, único, formato válido)
- `password` (String, encriptado)
- `role` (Enum: PLAYER, TOURNAMENT_ORGANIZER, ADMIN)
- `eloRating` (Integer, default 1200)
- `totalMatches` (Integer, default 0)
- `wins` (Integer, default 0)
- `losses` (Integer, default 0)
- `currentTeamId` (Long, FK nullable)
- `country` (String, código ISO 3166-1 alpha-2)
- `createdAt` (LocalDateTime)
- `lastActiveAt` (LocalDateTime)

**Team**
- `id` (Long)
- `name` (String, único, 3-50 caracteres)
- `tag` (String, único, 2-5 caracteres uppercase, ej: "TSM")
- `captainId` (Long, FK a Player)
- `averageElo` (Integer, calculado)
- `logo` (String, URL)
- `status` (Enum: RECRUITING, FULL, DISBANDED)
- `maxMembers` (Integer, default 5)
- `currentMembers` (Integer, default 1)
- `createdAt` (LocalDateTime)

**Tournament**
- `id` (Long)
- `name` (String, no nulo, máximo 100 caracteres)
- `game` (Enum: LOL, DOTA2, CSGO, VALORANT, FORTNITE)
- `format` (Enum: SINGLE_ELIMINATION, DOUBLE_ELIMINATION, ROUND_ROBIN, SWISS)
- `status` (Enum: REGISTRATION, MATCHMAKING, IN_PROGRESS, COMPLETED, CANCELLED)
- `maxTeams` (Integer, múltiplo de 2, máximo 32)
- `registeredTeams` (Integer, default 0)
- `prizePool` (BigDecimal)
- `organizerId` (Long, FK a Player)
- `registrationStartDate` (LocalDateTime)
- `registrationEndDate` (LocalDateTime)
- `tournamentStartDate` (LocalDateTime)
- `minEloRequirement` (Integer, nullable)
- `maxEloRequirement` (Integer, nullable)
- `createdAt` (LocalDateTime)

**TournamentRegistration**
- `id` (Long)
- `tournamentId` (Long, FK)
- `teamId` (Long, FK)
- `registrationDate` (LocalDateTime)
- `status` (Enum: PENDING, APPROVED, REJECTED, WITHDRAWN)
- `seed` (Integer, nullable, asignado en matchmaking)

**Match**
- `id` (Long)
- `tournamentId` (Long, FK)
- `round` (Integer, ej: 1 = cuartos, 2 = semis, 3 = final)
- `matchNumber` (Integer)
- `team1Id` (Long, FK nullable)
- `team2Id` (Long, FK nullable)
- `team1Score` (Integer, default 0)
- `team2Score` (Integer, default 0)
- `winnerId` (Long, FK nullable)
- `status` (Enum: SCHEDULED, LIVE, COMPLETED, WALKOVER)
- `scheduledTime` (LocalDateTime)
- `startedAt` (LocalDateTime, nullable)
- `completedAt` (LocalDateTime, nullable)
- `nextMatchId` (Long, FK nullable, para bracket progression)
- `streamUrl` (String, nullable)

**MatchEvent** (para tracking en tiempo real)
- `id` (Long)
- `matchId` (Long, FK)
- `eventType` (Enum: KILL, DEATH, OBJECTIVE, ROUND_WIN, PAUSE, RESUME)
- `teamId` (Long, FK)
- `playerId` (Long, FK nullable)
- `description` (String, máximo 200 caracteres)
- `timestamp` (LocalDateTime)

**Statistic** (estadísticas agregadas por jugador-torneo)
- `id` (Long)
- `playerId` (Long, FK)
- `tournamentId` (Long, FK)
- `kills` (Integer, default 0)
- `deaths` (Integer, default 0)
- `assists` (Integer, default 0)
- `kdRatio` (Double, calculado: kills/deaths)
- `matchesPlayed` (Integer)
- `mvpCount` (Integer, default 0)
- `lastUpdated` (LocalDateTime)

### **3. DTOs con Validaciones**

**RegisterPlayerRequest**
```java
- username (@NotBlank, @Size(min=3, max=20), @Pattern(regexp="^[a-zA-Z0-9_]+$"))
- email (@NotBlank, @Email)
- password (@NotBlank, @Size(min=8, max=100), @Pattern para incluir mayúscula, minúscula, número)
- country (@NotBlank, @Size(min=2, max=2))
```

**CreateTeamRequest**
```java
- name (@NotBlank, @Size(min=3, max=50))
- tag (@NotBlank, @Size(min=2, max=5), @Pattern(regexp="^[A-Z0-9]+$"))
- logo (@URL)
```

**CreateTournamentRequest** (solo TOURNAMENT_ORGANIZER/ADMIN)
```java
- name (@NotBlank, @Size(max=100))
- game (@NotNull)
- format (@NotNull)
- maxTeams (@NotNull, @Min(4), @Max(32), validar múltiplo de 2)
- prizePool (@NotNull, @DecimalMin("0.0"))
- registrationStartDate (@NotNull, @Future)
- registrationEndDate (@NotNull, @Future)
- tournamentStartDate (@NotNull, @Future)
- minEloRequirement (@Min(0), @Max(3000))
- maxEloRequirement (@Min(0), @Max(3000))
```

**RegisterTeamToTournamentRequest**
```java
- teamId (@NotNull)
- tournamentId (@NotNull)
```

**UpdateMatchScoreRequest**
```java
- matchId (@NotNull)
- team1Score (@NotNull, @Min(0))
- team2Score (@NotNull, @Min(0))
```

**SubmitMatchEventRequest**
```java
- matchId (@NotNull)
- eventType (@NotNull)
- teamId (@NotNull)
- playerId (nullable)
- description (@Size(max=200))
```

### **4. Endpoints a Implementar**

**AuthController** (`/api/auth`)
- `POST /register` - Registro de jugador
  - Request: RegisterPlayerRequest
  - Response: 201 CREATED con PlayerResponse
  - ELO inicial = 1200

- `POST /login` - Login
  - Response: JWT con roles

**TeamController** (`/api/teams`) - Autenticado
- `POST /create` - Crear equipo
  - Request: CreateTeamRequest
  - Response: 201 CREATED con TeamResponse
  - El creador es automáticamente el capitán
  - Validar que el jugador no esté en otro equipo activo
  - `currentMembers` = 1, `status` = RECRUITING

- `POST /{teamId}/invite` - Invitar jugador al equipo (solo capitán)
  - Request: `{"playerId": 123}`
  - Response: 202 ACCEPTED
  - Validar que el equipo no esté lleno
  - Publicar evento `TeamInvitationSentEvent`
  - Enviar notificación al jugador invitado

- `POST /invitations/{invitationId}/accept` - Aceptar invitación
  - Response: 200 OK con TeamResponse actualizado
  - Incrementar `currentMembers`
  - Actualizar `averageElo` del equipo
  - Si equipo llega a `maxMembers`, cambiar status a FULL

- `GET /{teamId}` - Detalle del equipo (público)
  - Response: 200 OK con TeamResponse (incluye lista de miembros con sus stats)

- `GET /search` - Buscar equipos
  - Query params: `?name=&minElo=&maxElo=&status=`
  - Response: 200 OK con lista paginada

- `DELETE /{teamId}/leave` - Abandonar equipo
  - Response: 204 NO_CONTENT
  - Si es capitán, transferir capitanía o disolver equipo
  - Recalcular `averageElo`

**TournamentController** (`/api/tournaments`)
- `POST /create` - Crear torneo (TOURNAMENT_ORGANIZER/ADMIN)
  - Request: CreateTournamentRequest
  - Response: 201 CREATED con TournamentResponse
  - Validar fechas: registration_end < tournament_start
  - Validar maxTeams es potencia de 2
  - Status inicial = REGISTRATION

- `GET /active` - Listar torneos activos (público)
  - Query params: `?game=&status=&minPrizePool=`
  - Response: 200 OK con lista de TournamentResponse
  - Incluir cantidad de equipos registrados

- `GET /{tournamentId}` - Detalle del torneo (público)
  - Response: 200 OK con TournamentResponse completo
  - Incluir bracket si el torneo está IN_PROGRESS o COMPLETED

- `POST /{tournamentId}/register` - Registrar equipo (capitán)
  - Request: RegisterTeamToTournamentRequest
  - Response: 201 CREATED con TournamentRegistrationResponse
  - Validaciones:
    - Usuario debe ser capitán del equipo
    - Torneo debe estar en status REGISTRATION
    - Fecha actual entre registrationStartDate y registrationEndDate
    - Equipo debe cumplir requisitos de ELO (minElo <= averageElo <= maxElo)
    - Equipo debe estar FULL (todos los miembros)
    - No puede estar ya registrado
  - Incrementar `registeredTeams`

- `POST /{tournamentId}/start-matchmaking` - Iniciar matchmaking (organizador)
  - Response: 202 ACCEPTED con mensaje "Matchmaking started"
  - Validar que el torneo tenga equipos registrados (mínimo 4)
  - Cambiar status a MATCHMAKING
  - Publicar evento `TournamentMatchmakingStartedEvent`
  - Proceso asíncrono generará el bracket

- `GET /{tournamentId}/bracket` - Ver bracket del torneo (público)
  - Response: 200 OK con estructura de bracket completa
  - Incluir todos los matches con sus estados

- `GET /{tournamentId}/leaderboard` - Tabla de posiciones (público, cached)
  - Response: 200 OK con ranking de equipos
  - Ordenar por: partidas ganadas, diferencia de puntos, ELO promedio
  - Cache de 5 minutos con `@Cacheable`

**MatchController** (`/api/matches`)
- `GET /{matchId}` - Detalle del match (público)
  - Response: 200 OK con MatchResponse
  - Incluir información de equipos, jugadores, y eventos

- `POST /{matchId}/start` - Iniciar match (organizador o admin)
  - Response: 200 OK con MatchResponse
  - Cambiar status a LIVE
  - Registrar `startedAt`
  - Publicar evento `MatchStartedEvent` (dispara WebSocket broadcast)

- `PUT /{matchId}/score` - Actualizar score (organizador durante match LIVE)
  - Request: UpdateMatchScoreRequest
  - Response: 200 OK con MatchResponse
  - Publicar evento `MatchScoreUpdatedEvent` (WebSocket)
  - Invalidar cache del leaderboard: `@CacheEvict`

- `POST /{matchId}/complete` - Completar match (organizador)
  - Request: `{"winnerId": 123}`
  - Response: 200 OK con MatchResponse
  - Cambiar status a COMPLETED
  - Registrar `completedAt`
  - Actualizar stats de jugadores (wins, losses, totalMatches)
  - Actualizar ELO de jugadores (algoritmo ELO estándar)
  - Si hay `nextMatchId`, asignar al ganador
  - Publicar evento `MatchCompletedEvent`
  - Invalidar múltiples caches

- `POST /{matchId}/events` - Agregar evento al match (organizador)
  - Request: SubmitMatchEventRequest
  - Response: 201 CREATED con MatchEventResponse
  - Solo si match está LIVE
  - Publicar evento `MatchEventCreatedEvent` (WebSocket broadcast)

- `GET /{matchId}/events` - Listar eventos del match (público)
  - Response: 200 OK con lista ordenada cronológicamente
  - Stream en tiempo real si match está LIVE

**StatisticsController** (`/api/statistics`)
- `GET /player/{playerId}` - Estadísticas del jugador (público, cached)
  - Query param: `?tournamentId=` (opcional)
  - Response: 200 OK con PlayerStatisticsResponse
  - Incluir: ELO history, K/D ratio, torneos jugados, MVPs
  - Cache de 10 minutos

- `GET /tournament/{tournamentId}/top-players` - Top jugadores del torneo (público, cached)
  - Query param: `?metric=kills|deaths|assists|kd_ratio&limit=10`
  - Response: 200 OK con lista rankeada
  - Cache de 5 minutos

- `POST /tournament/{tournamentId}/generate-report` - Generar reporte completo (organizador)
  - Response: 202 ACCEPTED con `{"reportId": "uuid", "status": "processing"}`
  - Proceso asíncrono genera PDF con todas las estadísticas
  - Publicar evento `ReportGenerationRequestedEvent`

- `GET /reports/{reportId}` - Obtener reporte generado
  - Response: 200 OK con `{"url": "https://...", "status": "completed"}` o 202 si aún procesando
  - Error 404 si no existe

**RankingController** (`/api/rankings`)
- `GET /global` - Ranking global de jugadores (público, cached)
  - Query params: `?game=&country=&page=0&size=50`
  - Response: 200 OK con lista paginada ordenada por ELO descendente
  - Cache de 15 minutos

- `GET /teams` - Ranking de equipos (público, cached)
  - Response: 200 OK con lista ordenada por averageElo
  - Cache de 15 minutos

### **5. WebSocket - Actualizaciones en Tiempo Real**

**Configuración WebSocket:**
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    // Configurar STOMP endpoint: /ws
    // Message broker: /topic
    // Application destination prefix: /app
}
```

**Endpoints WebSocket:**

**Cliente se suscribe a:**
- `/topic/match/{matchId}` - Recibe actualizaciones del match
- `/topic/tournament/{tournamentId}` - Recibe actualizaciones del torneo
- `/topic/leaderboard/{tournamentId}` - Recibe actualizaciones del ranking

**Servidor envía:**
```java
// Al actualizar score
{
  "type": "SCORE_UPDATE",
  "matchId": 123,
  "team1Score": 10,
  "team2Score": 8,
  "timestamp": "2025-10-01T14:30:00"
}

// Al agregar evento
{
  "type": "MATCH_EVENT",
  "matchId": 123,
  "eventType": "KILL",
  "teamId": 45,
  "playerUsername": "ProGamer",
  "description": "Double kill!",
  "timestamp": "2025-10-01T14:31:15"
}

// Al completar match
{
  "type": "MATCH_COMPLETED",
  "matchId": 123,
  "winnerId": 45,
  "winnerName": "Team Alpha",
  "timestamp": "2025-10-01T15:00:00"
}
```

**WebSocketService:**
```java
@Service
public class WebSocketService {
    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    public void broadcastMatchUpdate(Long matchId, MatchUpdateDTO update) {
        messagingTemplate.convertAndSend("/topic/match/" + matchId, update);
    }

    public void broadcastTournamentUpdate(Long tournamentId, TournamentUpdateDTO update) {
        messagingTemplate.convertAndSend("/topic/tournament/" + tournamentId, update);
    }
}
```

### **6. Procesamiento Asíncrono y Eventos**

**Eventos a implementar:**

**TournamentMatchmakingStartedEvent**
```java
- tournamentId (Long)
- format (TournamentFormat)
- registeredTeams (List<TournamentRegistration>)
```

**MatchStartedEvent**
```java
- matchId (Long)
- tournamentId (Long)
```

**MatchCompletedEvent**
```java
- matchId (Long)
- winnerId (Long)
- loserId (Long)
- team1Score (Integer)
- team2Score (Integer)
```

**ReportGenerationRequestedEvent**
```java
- reportId (String)
- tournamentId (Long)
- requestedBy (Long)
```

**Listeners:**

**MatchmakingListener** (con `@Async` y `@EventListener`)
- Al recibir `TournamentMatchmakingStartedEvent`:
  - Implementar algoritmo de seeding basado en ELO
  - Generar bracket según formato (single/double elimination)
  - Crear todos los matches de la primera ronda
  - Asignar `seed` a cada TournamentRegistration
  - Calcular y asignar `scheduledTime` para cada match
  - Cambiar status del torneo a IN_PROGRESS
  - Simular delay de 5-10 segundos para emular procesamiento complejo

**EloUpdateListener** (con `@Async`)
- Al recibir `MatchCompletedEvent`:
  - Calcular nuevo ELO para cada jugador usando fórmula:
    ```
    New ELO = Old ELO + K * (Actual - Expected)
    K = 32 (constante)
    Expected = 1 / (1 + 10^((OpponentELO - PlayerELO)/400))
    Actual = 1 si ganó, 0 si perdió
    ```
  - Actualizar ELO de todos los jugadores de ambos equipos
  - Actualizar `averageElo` de los equipos
  - Invalidar cache de rankings: `@CacheEvict(value = {"globalRanking", "teamRanking"})`

**ReportGenerationListener** (con `@Async`)
- Al recibir `ReportGenerationRequestedEvent`:
  - Simular generación de reporte (sleep 15-20 segundos)
  - Recopilar todas las estadísticas del torneo
  - Generar PDF con:
    - Resumen general del torneo
    - Bracket completo
    - Top 10 jugadores por diferentes métricas
    - Estadísticas por equipo
    - Match history completo
  - Guardar URL del reporte generado
  - (Simplificado: solo generar JSON con los datos y guardar la "URL")

**StatisticsUpdateListener** (con `@Async`)
- Al recibir `MatchCompletedEvent`:
  - Actualizar tabla Statistic para cada jugador participante
  - Recalcular K/D ratio
  - Determinar MVP del match (mayor kills - deaths)
  - Incrementar `mvpCount` del jugador MVP

**WebSocketNotificationListener** (síncrono, inmediato)
- Al recibir `MatchStartedEvent`, `MatchScoreUpdatedEvent`, `MatchCompletedEvent`, `MatchEventCreatedEvent`:
  - Broadcast inmediato vía WebSocket a subscribers

### **7. Sistema de Cache**

**Configuración de Cache:**
```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager(
            "globalRanking", "teamRanking", "tournamentLeaderboard",
            "playerStatistics", "topPlayers"
        );
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(500)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .recordStats());
        return cacheManager;
    }
}
```

**Uso de Cache:**
- `@Cacheable(value = "globalRanking", key = "#game + '_' + #country")`
- `@Cacheable(value = "playerStatistics", key = "#playerId + '_' + #tournamentId")`
- `@CacheEvict(value = "tournamentLeaderboard", key = "#tournamentId")`
- `@CacheEvict(value = {"globalRanking", "teamRanking"}, allEntries = true)`

### **8. Scheduled Tasks**

**TournamentScheduler:**
```java
@Scheduled(fixedRate = 60000) // Cada minuto
public void checkTournamentRegistrationDeadlines() {
    // Buscar torneos con registrationEndDate pasada y status = REGISTRATION
    // Cambiar status a MATCHMAKING
    // Publicar evento TournamentMatchmakingStartedEvent
}

@Scheduled(fixedRate = 300000) // Cada 5 minutos
public void checkScheduledMatches() {
    // Buscar matches con status SCHEDULED y scheduledTime <= now + 10 minutes
    // Enviar notificación a capitanes de equipos
}

@Scheduled(cron = "0 0 3 * * *") // 3 AM diario
public void cleanupOldEvents() {
    // Eliminar MatchEvents de matches completados hace más de 30 días
    // Archivar estadísticas antiguas
}
```

### **9. Spring Security + JWT**

- Roles: PLAYER, TOURNAMENT_ORGANIZER, ADMIN
- Endpoints públicos: `/api/auth/**`, `/api/tournaments/active`, `/api/tournaments/{id}`, `/api/teams/{id}`, `/api/matches/{id}`, `/api/rankings/**`, `/ws/**`
- Solo capitanes pueden invitar a su equipo
- Solo organizadores pueden crear torneos y manejar matches
- `@PreAuthorize("hasRole('ADMIN') or @tournamentSecurityService.isOrganizer(#tournamentId, authentication.principal.id)")`

**TournamentSecurityService:**
```java
@Service
public class TournamentSecurityService {
    public boolean isOrganizer(Long tournamentId, Long playerId) {
        // Verificar si el jugador es el organizador del torneo
    }

    public boolean isTeamCaptain(Long teamId, Long playerId) {
        // Verificar si el jugador es capitán del equipo
    }
}
```

### **10. Manejo de Errores**

```java
GlobalExceptionHandler:

- TeamFullException → 400
  {"error": "Team is full", "maxMembers": 5, "currentMembers": 5}

- EloRequirementNotMetException → 403
  {"error": "Team does not meet ELO requirements", "requiredElo": 1500, "teamElo": 1200}

- TournamentRegistrationClosedException → 400
  {"error": "Tournament registration has closed", "closedAt": "2025-09-30T23:59:59"}

- MatchNotLiveException → 400
  {"error": "Cannot submit events for a match that is not live"}

- BracketGenerationException → 500
  {"error": "Failed to generate tournament bracket", "reason": "..."}

- InvalidTeamSizeException → 400
  {"error": "Team must be full to register for tournament", "required": 5, "current": 3}

- PlayerAlreadyInTeamException → 409
  {"error": "Player is already in an active team", "currentTeam": "Team Alpha"}

- WebSocketConnectionException → 503
  {"error": "WebSocket service unavailable"}
```

### **11. Testing**

**MatchmakingServiceTest**
```java
- testGenerateBracket_SingleElimination_8Teams()
- testGenerateBracket_PowerOfTwoRequired_ThrowsException()
- testSeedTeams_ByElo_CorrectOrder()
- testCreateMatches_FirstRound_CorrectPairings()
```

**EloCalculationServiceTest**
```java
- testCalculateElo_HigherRankedWins_SmallGain()
- testCalculateElo_LowerRankedWins_LargeGain()
- testCalculateElo_EqualRanked_MediumGain()
```

**MatchServiceTest**
```java
- testStartMatch_ValidRequest_BroadcastsWebSocket()
- testCompleteMatch_UpdatesNextMatch()
- testUpdateScore_InvalidatesCache()
- testAddEvent_MatchNotLive_ThrowsException()
```

**WebSocketIntegrationTest**
```java
- testMatchScoreUpdate_SubscribersReceiveMessage()
- testMatchEvent_BroadcastToCorrectTopic()
- testMultipleSubscribers_AllReceiveUpdates()
```

**CacheTest**
```java
- testGlobalRanking_CachedFor15Minutes()
- testLeaderboard_InvalidatedOnScoreUpdate()
- testPlayerStats_DifferentTournaments_SeparateCache()
```

### **12. Colección Postman**

Variables de entorno:
- `{{baseUrl}}`
- `{{token}}`
- `{{teamId}}`
- `{{tournamentId}}`
- `{{matchId}}`

Flujo completo:
1. **Setup**
   - Register 10 players → Login → Save tokens
   - Create 4 teams → Invite members → Accept invitations

2. **Tournament Lifecycle**
   - Create tournament (as organizer)
   - Register 4 teams
   - Start matchmaking (wait for async processing)
   - Get bracket

3. **Match Flow**
   - Start first match
   - Submit multiple match events
   - Update scores
   - Complete match
   - Verify next match updated

4. **Real-time Testing**
   - Connect to WebSocket
   - Subscribe to match topic
   - Update score from Postman
   - Verify WebSocket message received

5. **Statistics & Reports**
   - Get player statistics
   - Get tournament leaderboard
   - Get top players
   - Request report generation
   - Poll report status

6. **Cache Testing**
   - Request global ranking (cache miss)
   - Request again (cache hit)
   - Complete a match
   - Request ranking (cache invalidated, new data)

### **13. Extra: API Externa (Opcional)**

Integrar con API de estadísticas de juegos reales:
- Riot Games API para League of Legends
- Steam API para CSGO/Dota 2
- Validar que los usernames de los jugadores existen en la plataforma
- Importar estadísticas reales para cálculo de ELO inicial más preciso

```java
@Service
public class GamePlatformIntegrationService {
    @Async
    public CompletableFuture<PlayerStats> fetchPlayerStats(String username, Game game) {
        // Llamada a API externa con RestTemplate
        // Timeout de 10 segundos
        // Cachear resultados por 1 hora
    }
}
```
