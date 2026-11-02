# Comprehensive Interview Guide: Campus Hub Project
### 25 Curated Technical Interview Questions & Detailed In-Depth Answers

This guide contains technical interview questions derived directly from the architecture, design choices, concurrency patterns, security mechanisms, and integrations implemented in the **Campus Hub** codebase.

---

## Table of Contents
1. [System Architecture & Design Patterns](#1-system-architecture--design-patterns) (Q1–Q4)
2. [Spring Boot & Java 21 Deep Dive](#2-spring-boot--java-21-deep-dive) (Q5–Q8)
3. [Spring Security & JWT Authentication](#3-spring-security--jwt-authentication) (Q9–Q12)
4. [JPA, Hibernate & Database Transactions](#4-jpa-hibernate--database-transactions) (Q13–Q15)
5. [Asynchronous Processing & AI Integration](#5-asynchronous-processing--ai-integration) (Q16–Q18)
6. [Real-Time WebSockets & Messaging](#6-real-time-websockets--messaging) (Q19–Q20)
7. [Third-Party Integrations: Razorpay & Cloudinary](#7-third-party-integrations-razorpay--cloudinary) (Q21–Q22)
8. [Frontend Engineering (React 19 & Axios Interceptors)](#8-frontend-engineering-react-19--axios-interceptors) (Q23–Q24)
9. [Scalability, Concurrency & Production Readiness](#9-scalability-concurrency--production-readiness) (Q25)

---

## 1. System Architecture & Design Patterns

### Q1: Can you give an architectural overview of Campus Hub and explain why you chose a decoupled client-server architecture?
**Answer:**
Campus Hub is built using a decoupled client-server model consisting of:
- **Presentation Layer**: A Single Page Application (SPA) built with React 19, Vite, and Tailwind CSS.
- **API & Business Logic Layer**: A stateless REST API and WebSocket broker built on Spring Boot 4 and Java 21.
- **Persistence Layer**: Spring Data JPA with Hibernate interfacing with MySQL 8.
- **External Integration Services**: Cloudinary (media hosting), Google Gemini 1.5 Pro (natural language processing and price appraisal), Razorpay (payments), and Gmail SMTP (notifications).

**Why decoupled?**
1. **Separation of Concerns**: The backend serves purely as a domain-driven API gateway providing data and business rule validation, while the frontend handles rendering and user interaction.
2. **Independent Scalability & Deployment**: The backend can scale horizontally behind a reverse proxy (e.g., NGINX / Render containers) independently of the static frontend assets served via CDNs.
3. **Multi-Client Readiness**: A stateless REST API with token-based authentication (JWT) enables future clients (such as Android or iOS mobile apps) to consume the exact same backend endpoints without architectural refactoring.

---

### Q2: What design patterns did you employ in this codebase, and where are they used?
**Answer:**
Several core software engineering design patterns are implemented:
1. **Builder Pattern**: Used via Lombok's `@Builder` on domain entities ([Item.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/model/entities/Item.java), [Claim.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/model/entities/Claim.java), etc.) and DTOs. This avoids telescoping constructors and ensures safe, immutable object instantiation.
2. **Repository Pattern**: Implemented via Spring Data JPA interfaces ([ItemRepository.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/repositories/ItemRepository.java)). It separates business logic from SQL/persistence mechanics.
3. **Interceptor / Chain of Responsibility Pattern**: 
   - Backend: [JwtAuthenticationFilter.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/config/JwtAuthenticationFilter.java) forms a link in Spring Security's `SecurityFilterChain`, intercepting requests before reaching controllers.
   - Frontend: Axios interceptors in [api.js](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/campus-hub-frontend/src/services/api.js) dynamically inject Bearer tokens on outgoing requests and globally handle `401 Unauthorized` responses.
4. **Observer / Publish-Subscribe Pattern**: Implemented in real-time messaging via Spring WebSocket STOMP broker ([WebSocketConfig.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/config/WebSocketConfig.java)). Connected clients subscribe to `/topic/public`, and messages published via `SimpMessagingTemplate` are automatically broadcast to all subscribers.
5. **Adapter / Facade Pattern**: [CloudinaryService.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/service/services/CloudinaryService.java) and [PaymentService.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/service/services/PaymentService.java) encapsulate third-party vendor SDK complexities behind simplified domain methods.

---

### Q3: How does the application handle Data Transfer Objects (DTOs), and why not expose JPA Entities directly to the REST controllers?
**Answer:**
The project isolates domain entities from the API layer using dedicated request and response DTOs (e.g., [ItemRequestDto.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/model/dto/ItemRequestDto.java), [ItemResponseDto.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/model/dto/ItemResponseDto.java)).

**Reasons for this separation:**
1. **Preventing Information Overexposure**: The [User.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/model/entities/User.java) entity contains `passwordHash` and `verificationToken`. Exposing entities directly risks accidental serialization of sensitive data into API responses.
2. **Preventing Infinite JSON Recursion**: Bidirectional JPA relationships (e.g., `Item` -> `List<ItemImage>` -> `Item`) cause circular serialization loops with Jackson unless annotated with `@JsonIgnore` or DTO-mapped.
3. **Decoupling Database Schema from API Contracts**: Database schema migrations (e.g., renaming a column) should not break client-facing contracts. DTOs insulate the API consumers from internal database refactoring.
4. **Targeted Validation**: Input DTOs allow strict Bean Validation annotations (`@NotBlank`, `@NotNull`, `@Min`) without polluting entity definitions.

---

### Q4: Explain the multi-part request handling for item creation where both JSON metadata and binary images are uploaded simultaneously.
**Answer:**
In [LostFoundController.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/controller/LostFoundController.java#L26-L43), the `/api/items` endpoint receives a `multipart/form-data` payload containing:
1. `@RequestParam("itemData") String itemDataJson`: A JSON string containing the title, description, category, and location.
2. `@RequestPart(value = "images", required = false) List<MultipartFile> images`: An array of binary image files.

**How it works under the hood:**
Standard `application/json` cannot transport binary image data without inefficient base64 encoding (which increases payload size by ~33%). Using `multipart/form-data` allows streaming raw file binaries alongside textual metadata.
The controller uses an injected `ObjectMapper` to deserialize `itemDataJson` into `ItemRequestDto`:
```java
ItemRequestDto itemRequestDto = objectMapper.readValue(itemDataJson, ItemRequestDto.class);
lostFoundService.reportItem(itemRequestDto, images, userDetails.getUsername());
```
This pattern provides strong validation, separates file handling from JSON parsing, and lets files stream directly to Cloudinary without accumulating in server memory.

---

## 2. Spring Boot & Java 21 Deep Dive

### Q5: What features of Java 21 and modern Spring Boot 4 are leveraged in this project?
**Answer:**
1. **Record & Pattern Matching Support**: Java 21 compiler optimizations, enhanced switch expressions, and modern syntax for immutability and DTO representations.
2. **Stateless Security Filter API**: Spring Security uses modern lambda-based DSL configuration (`http.csrf(AbstractHttpConfigurer::disable)...`) replacing deprecated `WebSecurityConfigurerAdapter`.
3. **Optimized Threading & Execution**: Java 21's runtime improvements for asynchronous background tasks when processing `@Async` methods (such as external Gemini AI calls).
4. **Spring WebMvc & Validation Starters**: Clean integration of Hibernate Validator and declarative exception handling via `@RestControllerAdvice`.

---

### Q6: How is global exception handling structured in the backend?
**Answer:**
The project uses `@RestControllerAdvice` in [GlobalExceptionHandler.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/exceptions/GlobalExceptionHandler.java) to catch exceptions across all controllers:
1. **ResourceNotFoundException**: Returns `404 NOT_FOUND` with a clean error message when a requested item or user ID does not exist.
2. **MethodArgumentNotValidException**: Catches Spring validation errors (e.g., blank fields, invalid emails) and extracts field-level validation messages into a clean structured JSON map with `400 BAD_REQUEST`.
3. **Security / Access Denied Exceptions**: Traps unauthorized operations (e.g., a student trying to approve claims) and converts them into standard `403 FORBIDDEN` responses.
4. **Fallback Exception Handler**: Catches uncaught runtime exceptions, logs stack traces server-side, and returns a sanitized `500 INTERNAL_SERVER_ERROR` without leaking internal stack traces to the client.

---

### Q7: What is the lifecycle of a Spring Bean, and how does Dependency Injection operate in this application?
**Answer:**
Spring Beans follow this lifecycle:
1. **Instantiation**: The Spring IoC container creates bean instances using reflection.
2. **Populate Properties**: Dependencies are resolved and injected. In this project, constructor injection is enforced via Lombok's `@RequiredArgsConstructor` (creating a constructor for all `final` fields).
3. **Initialization Callbacks**: Annotations like `@PostConstruct` or `InitializingBean` execute.
4. **Ready for Use**: The bean is placed in the `ApplicationContext` (default scope is **Singleton**).
5. **Destruction**: On application shutdown, `@PreDestroy` callbacks execute.

**Why constructor injection is preferred over `@Autowired` field injection:**
- Guarantees immutability (`final` references).
- Prevents `NullPointerException` during standalone unit testing (dependencies can be passed directly via mocks).
- Eliminates circular dependency issues at compile/startup time.

---

### Q8: How does `@Transactional` work in Spring, and what happens if an exception occurs during image upload or AI matching?
**Answer:**
Spring's `@Transactional` uses Spring AOP (Aspect-Oriented Programming) to create a dynamic proxy around the target service method (e.g., `LostFoundServiceImpl.reportItem`).
- When the method starts, the proxy opens a database transaction or joins an existing one.
- If the method completes normally, the transaction is committed.
- If a `RuntimeException` or `Error` is thrown, the transaction is rolled back.

**Behavior during image upload & AI matching:**
In `reportItem`:
1. The `Item` entity is saved first.
2. If Cloudinary upload fails with an `IOException`, it is wrapped into a `RuntimeException`, triggering a full database rollback; the `Item` record is not retained in MySQL.
3. The AI matching method (`geminiAiService.autoMatchLostAndFound`) is annotated with `@Async`. Because it runs on a separate worker thread, it executes outside the calling method's transaction boundary. Even if the AI API fails or times out, the user's item creation transaction has already committed successfully and is not rolled back.

---

## 3. Spring Security & JWT Authentication

### Q9: Walk me through the complete JWT Authentication flow from client request to endpoint execution.
**Answer:**
1. **Client Request**: The frontend attaches `Authorization: Bearer <token>` in the HTTP header via Axios interceptors.
2. **CORS Filter**: The request hits `CorsConfigurationSource` to ensure the origin is permitted and preflight `OPTIONS` requests pass through.
3. **JwtAuthenticationFilter**:
   - The filter extracts the `Bearer ` string from the `Authorization` header.
   - It invokes [JwtUtils.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/config/JwtUtils.java) to parse the cryptographic signature using the secret key (`HMAC-SHA256`).
   - It extracts the username (email) and checks if the token has expired.
4. **SecurityContext Population**:
   - If valid, it loads the user from `UserDetailsService`.
   - It verifies that `user.isAccountNonLocked()` and `user.isEnabled()` (email is verified) are true.
   - It builds a `UsernamePasswordAuthenticationToken` containing user principals and authorities (`ROLE_STUDENT` or `ROLE_ADMIN`).
   - It stores this token in `SecurityContextHolder.getContext().setAuthentication(...)`.
5. **Authorization Filter**: Spring Security checks route authorization (e.g., `.requestMatchers("/api/admin/**").hasRole("ADMIN")`).
6. **Controller Dispatch**: The request reaches the `@RestController`, where `@AuthenticationPrincipal` can inject the authenticated user directly into method parameters.

---

### Q10: How does the application enforce email verification before allowing login or claims?
**Answer:**
1. **User Entity**: [User.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/model/entities/User.java) implements Spring Security's `UserDetails` interface and overrides `isEnabled()`:
   ```java
   @Override
   public boolean isEnabled() {
       return isVerified;
   }
   ```
2. **Registration**: When a student signs up, `isVerified` defaults to `false`, a random UUID `verificationToken` is generated and saved, and an activation link is dispatched via `EmailService`.
3. **Login Interception**: During authentication via `authenticationManager.authenticate(...)`, Spring Security's `DaoAuthenticationProvider` checks `userDetails.isEnabled()`. If `false`, it immediately throws a `DisabledException`, preventing token issuance until the student clicks the email link.

---

### Q11: How do you prevent Cross-Origin Resource Sharing (CORS) errors while maintaining credentials support?
**Answer:**
In [SpringSecurityConfiguration.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/config/SpringSecurityConfiguration.java#L53-L67):
- When `allowCredentials` is set to `true`, browsers strictly forbid wildcard origin `setAllowedOrigins(List.of("*"))`.
- To solve this without hardcoding client ports, the configuration uses `setAllowedOriginPatterns(List.of("*"))`:
```java
configuration.setAllowedOriginPatterns(List.of("*"));
configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
configuration.setAllowedHeaders(Arrays.asList("Authorization", "Content-Type", "Accept", "Origin"));
configuration.setAllowCredentials(true);
```
This dynamically echoes the requesting origin while safely supporting Authorization headers and credentials across local development (`localhost:5173`) and cloud production URLs.

---

### Q12: How are passwords stored and compared securely in this system?
**Answer:**
- Passwords are never stored in plaintext. The project uses `BCryptPasswordEncoder` as a managed `@Bean`.
- **Salting & Hashing**: BCrypt automatically generates a unique 16-byte random salt for each password and concatenates it before running an adaptive key derivation function (cost factor default: 10 rounds).
- **Immunity to Rainbow Table Attacks**: Because the salt is unique per record, two identical passwords produce completely different hash strings.
- **Verification**: During login, `passwordEncoder.matches(rawPassword, encodedPassword)` re-hashes the raw password with the salt extracted from the stored hash and compares them in constant time to prevent timing attacks.

---

## 4. JPA, Hibernate & Database Transactions

### Q13: What is the N+1 Query Problem in JPA, and how does your project mitigate it?
**Answer:**
**The Problem**:
When fetching an entity with lazy relationships (e.g., fetching 10 `Item` entities that each have `@OneToMany List<ItemImage> images`), Hibernate executes 1 query to fetch the items, and then executes N additional queries (one for each item) as soon as `item.getImages()` is accessed. For 100 items, this results in 101 database round-trips.

**Mitigation in Campus Hub:**
1. **Explicit DTO Querying**: Instead of relying on automatic traversal of lazy entity collections across transaction boundaries, [LostFoundServiceImpl.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/service/services/LostFoundServiceImpl.java#L115-L122) explicitly queries images by ID:
   ```java
   List<String> imageUrls = itemImageRepository.findByItem_ItemId(item.getItemId())
           .stream()
           .map(ItemImage::getImageUrl)
           .collect(Collectors.toList());
   ```
2. **JOIN FETCH (Alternative for high throughput)**: For bulk queries, custom JPQL queries using `JOIN FETCH i.images` or `@EntityGraph(attributePaths = {"images"})` can load the parent item and its child images in a single SQL `JOIN` query.

---

### Q14: Explain the 3-Strike Fraud Penalty logic and how it ensures database consistency.
**Answer:**
The fraud penalty mechanism is handled in [AdminServiceImpl.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/service/services/AdminServiceImpl.java#L70-L103):
1. **Status Update**: Admin reviews a claim and marks it as `ClaimStatus.FRAUDULENT`.
2. **Fraud Aggregation**: The system executes `claimRepository.countByClaimer_EmailAndStatus(claimer.getEmail(), ClaimStatus.FRAUDULENT)`.
3. **Threshold Check (>= 3)**:
   - A `Penalty` record is created with an amount of ₹100.0, marked `PenaltyStatus.UNPAID`, and linked to the fraudulent claim.
   - The user's accumulated `unpaidFines` balance is incremented:
     ```java
     claimer.setUnpaidFines(claimer.getUnpaidFines() + penaltyAmount);
     userRepository.save(claimer);
     ```
   - An email notification is dispatched detailing the suspension of claiming privileges.
4. **Transaction Consistency**: The entire method is `@Transactional`. If saving the penalty or updating the user's fine balance fails, the claim status rollback prevents an inconsistent state where a user is marked fraudulent without the appropriate penalty being registered.

---

### Q15: Why are `CascadeType.ALL` and `orphanRemoval = true` used on the `Item` -> `ItemImage` relationship?
**Answer:**
In [Item.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/model/entities/Item.java#L35):
```java
@OneToMany(mappedBy = "item", cascade = CascadeType.ALL, orphanRemoval = true)
private List<ItemImage> images;
```
- **`CascadeType.ALL`**: Propagates all EntityManager operations (PERSIST, MERGE, REMOVE, REFRESH, DETACH) from the parent `Item` to its child `ItemImage` entities. Saving or deleting an item automatically saves or deletes its images.
- **`orphanRemoval = true`**: If an image is removed from the `images` list collection (e.g., `item.getImages().remove(0)`), Hibernate automatically generates a `DELETE` SQL statement for that detached child row in `item_images`. This prevents "zombie" orphaned image rows in the database.

---

## 5. Asynchronous Processing & AI Integration

### Q16: How does the Google Gemini AI auto-matching mechanism work without blocking the user's HTTP request?
**Answer:**
1. **Asynchronous Execution via `@Async`**: In [GeminiAiService.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/service/ai/GeminiAiService.java#L60), the method `autoMatchLostAndFound(Item newItem)` is annotated with `@Async`. When invoked from `LostFoundServiceImpl`, Spring hands off the task to a background task executor thread.
2. **Immediate HTTP Return**: The controller returns a `200 OK` response to the user within ~100ms, while the background thread communicates with Google Gemini.
3. **Prompt Construction**:
   - The service fetches all open items of the opposite type (`LOST` vs. `FOUND`) from `ItemRepository`.
   - It constructs a structured prompt including the new item's title, description, and location alongside the candidate items.
   - It instructs Gemini to act as a matcher and return strictly a JSON array of matching item IDs (e.g., `[2, 7]`).
4. **Parsing & Email Alert**: The response is parsed via Jackson `ObjectMapper`. For each matched candidate ID, `EmailService` sends notification emails to both the finder and the seeker.

---

### Q17: How is the Gemini AI response sanitized and parsed safely?
**Answer:**
LLMs often wrap JSON outputs inside Markdown code fences (e.g., ````json { ... } ````). If passed directly into a JSON parser, this causes a syntax exception.
In [GeminiAiService.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/service/ai/GeminiAiService.java#L131-L133), output sanitization is implemented:
```java
String aiText = rootNode.path("candidates").get(0).path("content").path("parts").get(0).path("text").asText();
// Clean up Markdown formatting Gemini sometimes adds
return aiText.replace("```json", "").replace("```", "").trim();
```
The cleaned string is then parsed into a structured `JsonNode` tree using Jackson's `objectMapper.readTree(aiText)`, allowing robust extraction of `estimatedNewPrice` and `analysis` values.

---

### Q18: What are the risks of using `@Async` in production, and how should it be configured?
**Answer:**
**Risks of default `@Async`**:
By default, Spring uses a `SimpleAsyncTaskExecutor`, which does not reuse threads; it spawns a new thread for every task. Under heavy load, this can cause thread exhaustion and crash the JVM with an `OutOfMemoryError`.

**Production Solution**:
Define a custom `ThreadPoolTaskExecutor` bean:
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("CampusHub-Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```
This bounds thread growth, recycles worker threads, and provides queue backpressure.

---

## 6. Real-Time WebSockets & Messaging

### Q19: Explain the difference between raw WebSockets and STOMP over SockJS as implemented in your project.
**Answer:**
1. **Raw WebSocket**: A low-level transport protocol (TCP-based duplex channel). It only defines how to send frames (text or binary); it has no concept of message routing, topics, headers, or destination addresses.
2. **SockJS**: A fallback emulation library. If the client browser or a network proxy does not support native WebSockets, SockJS falls back transparently to HTTP long-polling or streaming without changing client application code.
3. **STOMP (Simple Text Oriented Messaging Protocol)**: A higher-level application protocol running on top of WebSockets (analogous to how HTTP runs on top of TCP).
   - Introduces frames: `CONNECT`, `SUBSCRIBE`, `SEND`, `MESSAGE`.
   - Enables publish-subscribe routing via destinations (`/topic/public`, `/app/chat`).
   - Allows controllers to use `@MessageMapping` and `@SendTo` annotations.

---

### Q20: How is security handled during real-time chat message delivery?
**Answer:**
In [ChatController.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/controller/ChatController.java#L23-L33):
```java
@MessageMapping("/chat")
public void processMessage(@Payload ChatMessageDto chatMessageDto, Authentication authentication) {
    // 1. Ensure the sender is actually the authenticated user (prevent spoofing)
    chatMessageDto.setSenderEmail(authentication.getName());

    // 2. Save the message to the database
    ChatMessageDto savedMessage = chatService.saveMessage(chatMessageDto);

    // 3. Broadcast the message
    messagingTemplate.convertAndSend("/topic/public", savedMessage);
}
```
**Why this matters:**
A malicious client could attempt to forge the `senderEmail` in the payload JSON to impersonate another student. The controller overrides any client-submitted sender field with `authentication.getName()`, extracted directly from the verified security context established during handshake authentication.

---

## 7. Third-Party Integrations: Razorpay & Cloudinary

### Q21: Explain the two-step payment process with Razorpay and how you prevent payment tampering.
**Answer:**
Payment processing uses a secure two-phase workflow:

**Step 1: Server-Side Order Creation**
The client requests an order via `POST /api/payments/create-order`.
The backend [PaymentService.java](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/src/main/java/com/sheoran/LostAndFound/service/services/PaymentService.java#L19-L30) communicates directly with Razorpay using the secret key to create an order:
- Amount is converted to smallest currency units (paise: ₹10 = 1000 paise).
- Razorpay returns an official `order_id`.

**Step 2: Client Checkout & Signature Verification**
The frontend opens the Razorpay checkout widget. Once payment completes, Razorpay returns:
- `razorpay_order_id`
- `razorpay_payment_id`
- `razorpay_signature`

**Preventing Tampering**:
The client sends these three tokens back to the server. The server verifies:
$$\text{Signature} = \text{HMAC-SHA256}(\text{order\_id} + "|" + \text{payment\_id},\, \text{key\_secret})$$
Because only Razorpay and the backend possess the `key_secret`, a user cannot falsify a successful payment. If the computed hash matches `razorpay_signature`, the transaction is marked verified.

---

### Q22: Why use Cloudinary for image storage instead of saving images directly in MySQL (BLOB) or the local filesystem?
**Answer:**
1. **Database Performance**: Storing multi-megabyte images in MySQL as `BLOB` bloats database table sizes, degrades buffer pool cache efficiency, and slows backups.
2. **Ephemeral Cloud Filesystems**: Modern cloud hosting (Docker, Render, AWS ECS) uses ephemeral filesystems. Any images saved to local disk would be permanently lost whenever a container restarts or deploys.
3. **Content Delivery Network (CDN) & Optimization**: Cloudinary automatically caches images on global edge CDNs, applies lossy compression, and provides dynamic on-the-fly resizing and format conversion (e.g., auto WebP/AVIF delivery).

---

## 8. Frontend Engineering (React 19 & Axios Interceptors)

### Q23: How do the Axios interceptors in `api.js` manage authentication state and token expiration?
**Answer:**
In [api.js](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/campus-hub-frontend/src/services/api.js):
1. **Request Interceptor**:
   ```javascript
   api.interceptors.request.use((config) => {
       const token = localStorage.getItem('token');
       if (token) {
           config.headers.Authorization = `Bearer ${token}`;
       }
       return config;
   });
   ```
   Ensures that every outgoing HTTP request automatically carries the current JWT without requiring individual components to manually attach headers.
2. **Response Interceptor**:
   ```javascript
   api.interceptors.response.use(
       (response) => response,
       (error) => {
           if (error.response && error.response.status === 401) {
               localStorage.removeItem('token');
               localStorage.removeItem('user');
               window.location.href = '/login';
           }
           return Promise.reject(error);
       }
   );
   ```
   If the token expires during a session, the backend responds with `401 Unauthorized`. The response interceptor catches this globally, purges invalid session data, and redirects the user to the login screen.

---

### Q24: How does `AuthContext.jsx` provide global authentication state across React components?
**Answer:**
[AuthContext.jsx](file:///C:/Users/88she/OneDrive/Desktop/LostAndFound/campus-hub-frontend/src/context/AuthContext.jsx) uses the React Context API to manage user state:
- Maintains state for `user`, `token`, and `isAuthenticated`.
- Initializes state synchronously from `localStorage` on initial page load, preventing visual flicker or unauthenticated redirects on browser refresh.
- Exposes `login(userData, jwtToken)` and `logout()` helper functions that update both React state and persistent browser storage in sync.
- Components consume this state cleanly using a custom hook: `const { user, logout } = useAuth();`.

---

## 9. Scalability, Concurrency & Production Readiness

### Q25: If Campus Hub experiences a 100x traffic spike (e.g., during campus festival or semester start), what would be the bottlenecks and how would you resolve them?
**Answer:**
**Identified Bottlenecks & Solutions:**

1. **Database Connection Pool Exhaustion**:
   - *Bottleneck*: HikariCP default pool (10 connections) will saturate under high concurrent queries.
   - *Solution*: Tune `spring.datasource.hikari.maximum-pool-size`, add read-replicas for read-heavy operations (`getItems`, `getAvailableItems`), and introduce Redis caching for feed data.

2. **Synchronous External API Limits (Gemini & Cloudinary)**:
   - *Bottleneck*: Rate limits on external APIs can cause queue backlog.
   - *Solution*: Implement message queue workers (RabbitMQ or Apache Kafka) with backoff retry policies and circuit breakers (Resilience4j) around `GeminiAiService`.

3. **WebSocket Single-Node Limitation**:
   - *Bottleneck*: In-memory STOMP broker cannot broadcast messages across multiple backend instances.
   - *Solution*: Switch from the in-memory broker to an external dedicated message broker (e.g., RabbitMQ with STOMP plugin) using `enableStompBrokerRelay()`. All instances connect to the shared broker cluster.

4. **Static Asset Bottleneck**:
   - *Solution*: Offload the Vite React build bundle to an object store (e.g., AWS S3) served through Cloudflare or AWS CloudFront CDN.
