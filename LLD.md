# Low-Level Design (LLD): Campus Hub (Lost & Found & Marketplace)

## 1. Document Overview
This Low-Level Design (LLD) provides an in-depth architectural and structural specification of the **Campus Hub** platform. It covers data models, entity relationships, class hierarchies, sequence diagrams for key user journeys, security filter chains, and integration with third-party providers (Google Gemini AI, Cloudinary, Razorpay, Gmail SMTP, and WebSockets).

---

## 2. High-Level Architectural Layers

The system follows a classic **Layered Clean Architecture** combined with an **Event/Async Pipeline** for external AI and notification tasks.

```mermaid
graph TD
    subgraph ClientLayer ["Client Layer (Browser / SPA)"]
        ReactApp["React 19 + Vite + Tailwind CSS"]
        Router["React Router v7"]
        Axios["Axios HTTP Client + Interceptor"]
        StompClient["SockJS + @stomp/stompjs Client"]
    end

    subgraph SecurityLayer ["Security & Filter Pipeline"]
        Cors["CorsConfigurationSource (CORS)"]
        JwtFilter["JwtAuthenticationFilter"]
        DaoAuth["DaoAuthenticationProvider"]
    end

    subgraph ControllerLayer ["API Controller Layer (REST + WebSocket)"]
        AuthController["AuthController (/api/auth)"]
        LostFoundCtrl["LostFoundController (/api/items)"]
        ClaimCtrl["ClaimController (/api/claims)"]
        MarketCtrl["MarketplaceController (/api/marketplace)"]
        PaymentCtrl["PaymentController (/api/payments)"]
        ChatCtrl["ChatController (/api/chat, /app/chat)"]
        AdminCtrl["AdminController (/api/admin)"]
    end

    subgraph ServiceLayer ["Business Logic Layer"]
        LFService["LostFoundService / Impl"]
        CService["ClaimService / Impl"]
        MService["MarketplaceService / Impl"]
        PService["PaymentService"]
        ChService["ChatService / Impl"]
        AdService["AdminService / Impl"]
        CloudService["CloudinaryService"]
        GeminiService["GeminiAiService (@Async)"]
        EmailSvc["EmailService"]
    end

    subgraph PersistenceLayer ["Persistence Layer (Spring Data JPA / Hibernate)"]
        UserRepo["UserRepository"]
        ItemRepo["ItemRepository"]
        ImageRepo["ItemImageRepository"]
        ClaimRepo["ClaimRepository"]
        MarketRepo["MarketplaceItemRepository"]
        ChatRepo["ChatMessageRepository"]
        PenaltyRepo["PenaltyRepository"]
    end

    subgraph Database ["Data Store"]
        MySQL[("MySQL 8.x (campus_hub)")]
    end

    subgraph ExternalServices ["External Cloud Services"]
        CloudinaryAPI["Cloudinary REST API"]
        GeminiAPI["Google Gemini 1.5 Pro API"]
        RazorpayAPI["Razorpay Payment Gateway API"]
        GmailSMTP["Gmail SMTP Server"]
    end

    ReactApp --> Router
    Router --> Axios
    Router --> StompClient

    Axios --> Cors
    Cors --> JwtFilter
    JwtFilter --> DaoAuth
    JwtFilter --> ControllerLayer

    StompClient --> ChatCtrl

    ControllerLayer --> ServiceLayer
    ServiceLayer --> PersistenceLayer
    PersistenceLayer --> MySQL

    ServiceLayer --> ExternalServices
```

---

## 3. Database Schema & Entity-Relationship (ER) Design

### 3.1 Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ ITEMS : "reports"
    USERS ||--o{ CLAIMS : "submits"
    USERS ||--o{ MARKETPLACE_ITEMS : "lists"
    USERS ||--o{ CHAT_MESSAGES : "sends"
    USERS ||--o{ PENALTIES : "incurs"

    ITEMS ||--o{ ITEM_IMAGES : "has"
    ITEMS ||--o{ CLAIMS : "receives"
    ITEMS ||--o{ CHAT_MESSAGES : "referenced in"

    MARKETPLACE_ITEMS ||--o{ ITEM_IMAGES : "has"
    MARKETPLACE_ITEMS ||--o{ CHAT_MESSAGES : "referenced in"

    CLAIMS ||--o| PENALTIES : "triggers (if fraudulent)"

    USERS {
        bigint userId PK
        string name
        string email UK
        string passwordHash
        string role
        double unpaidFines
        boolean isVerified
        string verificationToken
    }

    ITEMS {
        bigint itemId PK
        bigint reporter_id FK
        string type
        string title
        string description
        string location
        string contactPhone
        datetime dateReported
        string status
    }

    ITEM_IMAGES {
        bigint imageId PK
        bigint item_id FK
        bigint marketplace_id FK
        string imageUrl
        string publicId
    }

    CLAIMS {
        bigint claimId PK
        bigint item_id FK
        bigint claimer_id FK
        string proofDescription
        string status
        datetime submittedAt
    }

    MARKETPLACE_ITEMS {
        bigint marketId PK
        bigint seller_id FK
        string title
        string description
        double askingPrice
        int quantity
        double aiNewPriceEstimate
        string aiProfitAnalysis
        string status
    }

    CHAT_MESSAGES {
        bigint messageId PK
        bigint sender_id FK
        string content
        datetime timestamp
        bigint item_id FK
        bigint market_item_id FK
    }

    PENALTIES {
        bigint penaltyId PK
        bigint user_id FK
        bigint claim_id FK
        double amount
        string reason
        string status
        datetime issuedAt
    }
```

### 3.2 Enumerations

| Enum Name | Allowed Values | Description |
| :--- | :--- | :--- |
| **`Role`** | `STUDENT`, `ADMIN` | User privilege tier. |
| **`ItemType`** | `LOST`, `FOUND` | Classification for lost & found items. |
| **`ItemStatus`** | `OPEN`, `CLAIM_PENDING`, `RESOLVED` | Operational lifecycle of a lost/found item. |
| **`ClaimStatus`** | `PENDING`, `APPROVED`, `REJECTED`, `FRAUDULENT` | Verification state of a user's claim. |
| **`MarketStatus`** | `AVAILABLE`, `SOLD` | Availability state of a marketplace item. |
| **`PenaltyStatus`**| `UNPAID`, `PAID` | Settlement status of disciplinary fines. |

---

## 4. Class Design & Component Structure

### 4.1 Domain Entities

```mermaid
classDiagram
    class User {
        +Long userId
        +String name
        +String email
        +String passwordHash
        +Role role
        +Double unpaidFines
        +boolean isVerified
        +String verificationToken
        +getAuthorities() Collection
        +getPassword() String
        +getUsername() String
        +isEnabled() boolean
    }

    class Item {
        +Long itemId
        +User reporter
        +ItemType type
        +String title
        +String description
        +String location
        +String contactPhone
        +List~ItemImage~ images
        +LocalDateTime dateReported
        +ItemStatus status
    }

    class ItemImage {
        +Long imageId
        +Item item
        +MarketplaceItem marketplaceItem
        +String imageUrl
        +String publicId
    }

    class Claim {
        +Long claimId
        +Item item
        +User claimer
        +String proofDescription
        +ClaimStatus status
        +LocalDateTime submittedAt
    }

    class MarketplaceItem {
        +Long marketId
        +User seller
        +String title
        +String description
        +Double askingPrice
        +Integer quantity
        +Double aiNewPriceEstimate
        +String aiProfitAnalysis
        +MarketStatus status
    }

    class ChatMessage {
        +Long messageId
        +User sender
        +String content
        +LocalDateTime timestamp
        +Item relatedItem
        +MarketplaceItem relatedMarketItem
    }

    class Penalty {
        +Long penaltyId
        +User user
        +Claim relatedClaim
        +Double amount
        +String reason
        +PenaltyStatus status
        +LocalDateTime issuedAt
    }

    User "1" --> "*" Item : reports
    User "1" --> "*" Claim : files
    User "1" --> "*" MarketplaceItem : lists
    User "1" --> "*" ChatMessage : sends
    User "1" --> "*" Penalty : receives
    Item "1" --> "*" ItemImage : has
    Item "1" --> "*" Claim : has
    MarketplaceItem "1" --> "*" ItemImage : has
    Claim "1" --> "0..1" Penalty : causes
```

### 4.2 Repositories & Data Access Interfaces

```mermaid
classDiagram
    class UserRepository {
        <<interface>>
        +findByEmail(email: String) Optional~User~
        +findByVerificationToken(token: String) Optional~User~
        +existsByEmail(email: String) boolean
    }

    class ItemRepository {
        <<interface>>
        +findByTypeAndStatus(type: ItemType, status: ItemStatus, pageable: Pageable) Page~Item~
        +findByTypeAndStatusOrderByDateReportedDesc(type: ItemType, status: ItemStatus) List~Item~
        +searchItemsByKeyword(keyword: String, pageable: Pageable) Page~Item~
        +findByReporter(user: User) List~Item~
    }

    class ItemImageRepository {
        <<interface>>
        +findByItem_ItemId(itemId: Long) List~ItemImage~
        +findByMarketplaceItem_MarketId(marketId: Long) List~ItemImage~
    }

    class ClaimRepository {
        <<interface>>
        +findByItem_ItemId(itemId: Long) List~Claim~
        +findByClaimer_Email(email: String) List~Claim~
        +findByStatus(status: ClaimStatus) List~Claim~
        +countByClaimer_EmailAndStatus(email: String, status: ClaimStatus) long
    }

    class MarketplaceItemRepository {
        <<interface>>
        +findByStatus(status: MarketStatus, pageable: Pageable) Page~MarketplaceItem~
        +findByStatusOrderByMarketIdDesc(status: MarketStatus, pageable: Pageable) Page~MarketplaceItem~
        +searchMarketplaceItems(keyword: String, pageable: Pageable) Page~MarketplaceItem~
        +findBySeller(user: User) List~MarketplaceItem~
    }

    class ChatMessageRepository {
        <<interface>>
        +findTop200ByOrderByTimestampDesc() List~ChatMessage~
        +findByRelatedItem_ItemId(itemId: Long) List~ChatMessage~
    }

    class PenaltyRepository {
        <<interface>>
        +findByUser(user: User) List~Penalty~
        +findByUserAndStatus(user: User, status: PenaltyStatus) List~Penalty~
    }
```

---

## 5. Sequence Diagrams: Core Workflows

### 5.1 User Registration, Verification & Authentication

```mermaid
sequenceDiagram
    autonumber
    actor User as Student
    participant FE as React Frontend
    participant AuthCtrl as AuthController
    participant UserRepo as UserRepository
    participant EmailSvc as EmailService
    participant JwtUtils as JwtUtils
    participant Security as Spring Security

    %% Registration
    User->>FE: Fills registration form
    FE->>AuthCtrl: POST /api/auth/register {name, email, password}
    AuthCtrl->>UserRepo: existsByEmail(email)
    alt Email already registered
        AuthCtrl-->>FE: 400 Bad Request ("Email already registered")
    else Email available
        AuthCtrl->>AuthCtrl: Hash password via BCrypt
        AuthCtrl->>UserRepo: save(User with isVerified=false, verificationToken=UUID)
        AuthCtrl->>EmailSvc: sendVerificationEmail(email, token)
        EmailSvc-->>User: Verification link sent via Gmail SMTP
        AuthCtrl-->>FE: 201 Created ("Please check your email")
    end

    %% Email Verification
    User->>FE: Clicks verification link in email
    FE->>AuthCtrl: GET /api/auth/verify?token=...
    AuthCtrl->>UserRepo: findByVerificationToken(token)
    AuthCtrl->>UserRepo: save(user with isVerified=true, token=null)
    AuthCtrl-->>FE: 200 OK ("Account verified")

    %% Login
    User->>FE: Enters credentials
    FE->>AuthCtrl: POST /api/auth/login {email, password}
    AuthCtrl->>Security: authenticate(UsernamePasswordAuthenticationToken)
    alt Verification is false
        Security-->>AuthCtrl: DisabledException ("User is disabled")
        AuthCtrl-->>FE: 401 Unauthorized
    else Authentication successful
        AuthCtrl->>JwtUtils: generateToken(userDetails)
        JwtUtils-->>AuthCtrl: JWT Token String (24h expiry)
        AuthCtrl-->>FE: 200 OK {token, userDetails}
        FE->>FE: Store token in localStorage & update AuthContext
    end
```

---

### 5.2 Reporting Item + Cloudinary Upload + Gemini AI Auto-Matching

```mermaid
sequenceDiagram
    autonumber
    actor Reporter as Reporter (Student)
    participant FE as React Frontend
    participant LFController as LostFoundController
    participant LFService as LostFoundServiceImpl
    participant Cloudinary as CloudinaryService
    participant ItemRepo as ItemRepository
    participant Gemini as GeminiAiService (@Async)
    participant Email as EmailService

    Reporter->>FE: Fills Item Details & attaches photo files
    FE->>LFController: POST /api/items (Multipart: itemData JSON + files)
    LFController->>LFService: reportItem(itemRequestDto, files, userEmail)
    LFService->>ItemRepo: save(Item with status=OPEN)
    
    loop For each attached image
        LFService->>Cloudinary: uploadImage(MultipartFile)
        Cloudinary-->>LFService: {secure_url, public_id}
        LFService->>LFService: save ItemImage entity
    end

    LFService->>Gemini: autoMatchLostAndFound(savedItem) [Asynchronous Thread]
    LFService-->>FE: 200 OK ("Item reported successfully")

    Note over Gemini,Email: Async Thread executes in background
    Gemini->>ItemRepo: findByTypeAndStatusOrderByDateReportedDesc(oppositeType, OPEN)
    Gemini->>Gemini: Build comparison prompt with candidate items
    Gemini->>Gemini: Call Gemini 1.5 Pro API via RestTemplate
    alt Potential matches found [id1, id2]
        loop For each matched candidate
            Gemini->>Email: sendNotificationEmail(newReporter, "Match Found", candidateDetails)
            Gemini->>Email: sendNotificationEmail(candidateReporter, "Match Found", newDetails)
        end
    else No match
        Gemini->>Gemini: Terminate task
    end
```

---

### 5.3 Claim Submission, Founder Approval & 3-Strike Fraud Penalty

```mermaid
sequenceDiagram
    autonumber
    actor Claimer as Claimer
    actor Founder as Item Finder / Admin
    participant FE as React Frontend
    participant ClaimCtrl as ClaimController
    participant ClaimSvc as ClaimServiceImpl / AdminServiceImpl
    participant ClaimRepo as ClaimRepository
    participant ItemRepo as ItemRepository
    participant PenaltyRepo as PenaltyRepository
    participant UserRepo as UserRepository
    participant Email as EmailService

    Claimer->>FE: Submits claim with proof description
    FE->>ClaimCtrl: POST /api/claims {itemId, proofDescription}
    ClaimCtrl->>ClaimSvc: submitClaim(request, claimerEmail)
    ClaimSvc->>UserRepo: Check claimer unpaidFines > 0
    alt User has unpaid penalties
        ClaimSvc-->>FE: 403 Forbidden ("Must clear pending fines first")
    else User eligible
        ClaimSvc->>ItemRepo: update status to CLAIM_PENDING
        ClaimSvc->>ClaimRepo: save(Claim with status=PENDING)
        ClaimSvc-->>FE: 201 Created (ClaimResponseDto)
    end

    alt Founder Approves Claim
        Founder->>FE: Clicks Approve
        FE->>ClaimCtrl: PATCH /api/claims/{id}/approve
        ClaimCtrl->>ClaimSvc: approveClaimByFounder(claimId, founderEmail)
        ClaimSvc->>ClaimRepo: set status = APPROVED
        ClaimSvc->>ItemRepo: set status = RESOLVED
        ClaimSvc->>ClaimRepo: set other claims for item = REJECTED
        ClaimSvc->>Email: sendNotificationEmail(claimer, "Claim Approved! Contact Founder")
        ClaimSvc-->>FE: 200 OK
    else Admin Reviews and Flags as FRAUDULENT
        Founder->>FE: Admin clicks Mark Fraudulent
        FE->>ClaimCtrl: PATCH /api/admin/claims/{id}/review?status=FRAUDULENT
        ClaimCtrl->>ClaimSvc: reviewClaim(claimId, FRAUDULENT, adminEmail)
        ClaimSvc->>ClaimRepo: set status = FRAUDULENT
        ClaimSvc->>ClaimRepo: countByClaimer_EmailAndStatus(claimerEmail, FRAUDULENT)
        alt Fraud Count >= 3
            ClaimSvc->>PenaltyRepo: save(Penalty {user, amount=100.0, status=UNPAID})
            ClaimSvc->>UserRepo: user.unpaidFines += 100.0
            ClaimSvc->>Email: sendNotificationEmail(claimer, "Penalty Notice: ₹100 Applied")
        else Fraud Count < 3
            ClaimSvc->>Email: sendNotificationEmail(claimer, "Warning: Claim Rejected as Fraudulent")
        end
        ClaimSvc-->>FE: 200 OK
    end
```

---

### 5.4 Marketplace Item Posting, Gemini Valuation & Razorpay Payment

```mermaid
sequenceDiagram
    autonumber
    actor Seller as Seller
    actor Buyer as Buyer
    participant FE as React Frontend
    participant MarketCtrl as MarketplaceController
    participant MarketSvc as MarketplaceServiceImpl
    participant GeminiSvc as GeminiAiService (@Async)
    participant PayCtrl as PaymentController
    participant PaySvc as PaymentService
    participant Razorpay as Razorpay Gateway

    %% Post Item
    Seller->>FE: Submits item details (title, asking price, images)
    FE->>MarketCtrl: POST /api/marketplace (itemData + images)
    MarketCtrl->>MarketSvc: postItemForSale(requestDto, images, sellerEmail)
    MarketSvc->>MarketSvc: Upload images to Cloudinary
    MarketSvc->>MarketSvc: Save MarketplaceItem with status=AVAILABLE
    MarketSvc->>GeminiSvc: analyzeAndSaveItemPricing(marketId, title, desc, price)
    MarketSvc-->>FE: 201 Created

    Note over GeminiSvc: Async Background Task
    GeminiSvc->>GeminiSvc: Prompt Gemini with title, desc, askingPrice
    GeminiSvc->>GeminiSvc: Parse JSON response {estimatedNewPrice, analysis}
    GeminiSvc->>MarketSvc: Update MarketplaceItem in DB with AI estimates

    %% Purchase Listing Fee or Item Payment
    Buyer->>FE: Clicks Buy or Pay Fee
    FE->>PayCtrl: POST /api/payments/create-order
    PayCtrl->>PaySvc: createOrder(userEmail)
    PaySvc->>Razorpay: orders.create({amount: 1000, currency: "INR", receipt: ...})
    Razorpay-->>PaySvc: Order details {id, amount, currency}
    PaySvc-->>FE: Return Razorpay Order JSON
    FE->>Razorpay: Open Razorpay Checkout modal
    Buyer->>Razorpay: Completes payment
    Razorpay-->>FE: Returns {razorpay_order_id, payment_id, signature}
    FE->>PaySvc: Verify signature (HMAC SHA-256)
    PaySvc-->>FE: Verification Successful
```

---

### 5.5 Real-Time Chat System via WebSocket & STOMP

```mermaid
sequenceDiagram
    autonumber
    actor UserA as Student A
    actor UserB as Student B
    participant FEA as Frontend A
    participant FEB as Frontend B
    participant Broker as Spring STOMP Broker (/ws-chat)
    participant ChatCtrl as ChatController
    participant ChatSvc as ChatServiceImpl
    participant ChatRepo as ChatMessageRepository

    %% Connection Setup
    FEA->>Broker: Connect SockJS /ws-chat
    Broker-->>FEA: Connection Established
    FEB->>Broker: Connect SockJS /ws-chat
    Broker-->>FEB: Connection Established

    %% Subscriptions
    FEA->>Broker: SUBSCRIBE /topic/public
    FEB->>Broker: SUBSCRIBE /topic/public

    %% Message Send
    UserA->>FEA: Types message & hits Send
    FEA->>Broker: SEND /app/chat {content, senderEmail, itemId}
    Broker->>ChatCtrl: processMessage(chatMessageDto, authentication)
    ChatCtrl->>ChatCtrl: Set senderEmail from authenticated token
    ChatCtrl->>ChatSvc: saveMessage(chatMessageDto)
    ChatSvc->>ChatRepo: save(ChatMessage)
    ChatSvc-->>ChatCtrl: Return persisted ChatMessageDto
    ChatCtrl->>Broker: convertAndSend("/topic/public", savedMessage)
    Broker-->>FEA: Broadcast savedMessage
    Broker-->>FEB: Broadcast savedMessage
    FEB->>UserB: Render message in chat window
```

---

## 6. Security Architecture & Filter Chain

```mermaid
flowchart TD
    Req[Incoming HTTP Request] --> CORS[CorsFilter / UrlBasedCorsConfigurationSource]
    CORS -->|Preflight OPTIONS| Allow200[200 OK Preflight]
    CORS -->|Actual Request| JWT[JwtAuthenticationFilter]
    
    JWT --> CheckHeader{Has Bearer Header?}
    CheckHeader -->|No| AnonCheck{Is Public Route?}
    CheckHeader -->|Yes| ParseJWT[Extract & Parse Token via JwtUtils]
    
    ParseJWT --> ValidateToken{Token Valid & Not Expired?}
    ValidateToken -->|No| Reject401[401 Unauthorized]
    ValidateToken -->|Yes| LoadUser[UserDetailsService: loadUserByUsername]
    LoadUser --> IsVerified{user.isVerified == true?}
    IsVerified -->|No| Reject403[403 Account Not Verified]
    IsVerified -->|Yes| SetContext[SecurityContextHolder.setAuthentication]
    
    SetContext --> RouteCheck{Check Route Authority}
    AnonCheck -->|Yes: /api/auth/**, /api/public/**, /ws-chat/**| PassAnon[Execute Controller]
    AnonCheck -->|No| RejectAnon[401 Unauthorized]

    RouteCheck -->|/api/admin/** and Has ROLE_ADMIN| PassAdmin[Execute AdminController]
    RouteCheck -->|/api/admin/** and Not Admin| RejectForbidden[403 Forbidden]
    RouteCheck -->|Authenticated Request| PassAuth[Execute Business Controller]
```

---

## 7. Design Patterns Applied

| Pattern | Implementation in Codebase | Purpose |
| :--- | :--- | :--- |
| **Builder Pattern** | Lombok `@Builder` on `User`, `Item`, `Claim`, `MarketplaceItem`, `ChatMessage`, `Penalty`. | Fluent, immutable instantiation of complex entities and DTOs. |
| **Repository Pattern** | Spring Data JPA interfaces (`ItemRepository`, `UserRepository`, etc.). | Decouples business layer from data persistence and SQL generation. |
| **Filter / Interceptor Pattern** | `JwtAuthenticationFilter` (backend) & Axios request/response interceptors (frontend). | Centralized, reusable handling of JWT authentication, header injection, and session expiration. |
| **Publish-Subscribe (Pub/Sub)** | Spring WebSocket STOMP Broker (`/topic/public`). | Enables scalable broadcasting of real-time messages to connected room participants. |
| **Adapter / Facade Pattern** | `CloudinaryService`, `PaymentService` (Razorpay), and `GeminiAiService`. | Wraps complex third-party vendor SDKs and APIs into clean domain methods. |
| **Global Exception Handler** | `@RestControllerAdvice` in `GlobalExceptionHandler.java`. | Centralized interception and standardized formatting of runtime errors and validation failures. |

---

## 8. Summary of API Contracts

| Method | Endpoint | Access Tier | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/auth/register` | Public | Register new user account. |
| `POST` | `/api/auth/login` | Public | Authenticate user & return JWT. |
| `GET` | `/api/auth/verify` | Public | Verify user email token. |
| `GET` | `/api/items` | Authenticated | Paginated feed of lost/found items with type & status filters. |
| `POST` | `/api/items` | Authenticated | Report new item with multipart photos; triggers Gemini matching. |
| `GET` | `/api/items/{id}` | Authenticated | Fetch specific item details and associated photos. |
| `DELETE`| `/api/items/{id}` | Reporter Only| Remove item listing. |
| `POST` | `/api/claims` | Authenticated | Submit claim with proof description (blocked if unpaid fines exist). |
| `PATCH`| `/api/claims/{id}/approve` | Item Founder | Founder approves claim and marks item resolved. |
| `PATCH`| `/api/claims/{id}/reject` | Item Founder | Founder rejects insufficient claim. |
| `GET` | `/api/marketplace` | Authenticated | Paginated listings of available items for sale. |
| `POST` | `/api/marketplace` | Authenticated | List item for sale; triggers Gemini deal appraisal. |
| `POST` | `/api/payments/create-order`| Authenticated | Initialize Razorpay order. |
| `GET` | `/api/chat/history` | Authenticated | Fetch last 200 messages for global room. |
| `WS` | `/app/chat` -> `/topic/public` | Connected | Send & broadcast real-time messages. |
| `GET` | `/api/admin/claims/pending` | Admin Only | View all unresolved claims. |
| `PATCH`| `/api/admin/claims/{id}/review` | Admin Only | Review claim, approve, or mark fraudulent (penalizes on 3rd strike). |
| `PATCH`| `/api/admin/penalties/{id}/waive`| Admin Only | Waive individual penalty fine. |
