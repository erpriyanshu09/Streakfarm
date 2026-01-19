# StreakFarm Telegram Mini App - Architecture

This document provides a comprehensive overview of the StreakFarm Telegram Mini App architecture, including all system components, data flows, and integrations.

## System Overview

StreakFarm is a gamified Telegram Mini App built with a modern tech stack that combines web technologies with blockchain integration. The system enables users to build streaks, earn rewards, and connect their TON wallets.

## High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        TelegramApp[Telegram App]
        MiniApp[Telegram Mini App<br/>React + Vite + TypeScript<br/>Tailwind CSS]
    end
    
    subgraph "Backend Services"
        EdgeFunctions[Supabase Edge Functions]
        Auth[Authentication Service]
        API[REST API / GraphQL]
    end
    
    subgraph "Data Layer"
        PostgreSQL[(PostgreSQL Database)]
        RealtimeDB[Realtime Subscriptions]
    end
    
    subgraph "External Services"
        TelegramAPI[Telegram Bot API]
        TONBlockchain[TON Blockchain]
        TONConnect[TON Connect Protocol]
    end
    
    TelegramApp -->|Hosts| MiniApp
    MiniApp -->|Telegram WebApp API| TelegramAPI
    MiniApp -->|TON Connect SDK| TONConnect
    MiniApp -->|HTTPS Requests| EdgeFunctions
    MiniApp -->|HTTPS Requests| API
    MiniApp -->|WebSocket| RealtimeDB
    
    EdgeFunctions -->|Query/Update| PostgreSQL
    API -->|Query/Update| PostgreSQL
    Auth -->|Verify User| TelegramAPI
    Auth -->|Store Session| PostgreSQL
    
    TONConnect -->|Wallet Operations| TONBlockchain
    EdgeFunctions -->|Blockchain Queries| TONBlockchain
    
    RealtimeDB -.->|Real-time Updates| PostgreSQL
```

## Detailed Component Architecture

```mermaid
graph TB
    subgraph "Frontend - Telegram Mini App"
        UI[UI Components<br/>React + Tailwind CSS]
        StateManagement[State Management<br/>React Query/Context]
        TONWallet[TON Wallet Integration<br/>@tonconnect/ui-react]
        TelegramSDK[Telegram WebApp SDK<br/>@twa-dev/sdk]
        Router[React Router]
    end
    
    subgraph "Supabase Backend"
        subgraph "Edge Functions"
            CheckStreak[check-streak]
            ProcessPoints[process-points]
            UpdateLeaderboard[update-leaderboard]
            HandleReferral[handle-referral]
            VerifyTON[verify-ton-transaction]
            ClaimRewards[claim-rewards]
        end
        
        AuthService[Supabase Auth<br/>JWT Tokens]
        RealtimeService[Realtime Service<br/>WebSocket]
        StorageService[Storage Service<br/>Files/Assets]
        RLS[Row Level Security<br/>Policies]
    end
    
    subgraph "PostgreSQL Database"
        direction LR
        Tables[("
        • profiles
        • boxes
        • badges
        • tasks
        • leaderboards
        • referrals
        • points_ledger
        • streaks
        • wallet_connections
        • transactions
        ")]
    end
    
    subgraph "External Integrations"
        TelegramBotAPI[Telegram Bot API]
        TONChain[TON Blockchain]
        TONConnectBridge[TON Connect Bridge]
    end
    
    UI --> StateManagement
    UI --> Router
    StateManagement --> TONWallet
    StateManagement --> TelegramSDK
    
    StateManagement -->|API Calls| EdgeFunctions
    StateManagement -->|Auth| AuthService
    StateManagement -->|Subscribe| RealtimeService
    
    EdgeFunctions -->|Execute| RLS
    RLS -->|Access| Tables
    AuthService -->|Verify| TelegramBotAPI
    
    TONWallet -->|Connect/Sign| TONConnectBridge
    TONConnectBridge -->|Transactions| TONChain
    VerifyTON -->|Query| TONChain
    
    RealtimeService -.->|Push Updates| StateManagement
```

## Database Schema

```mermaid
erDiagram
    profiles ||--o{ boxes : owns
    profiles ||--o{ badges : earns
    profiles ||--o{ tasks : completes
    profiles ||--o{ points_ledger : has
    profiles ||--o{ streaks : maintains
    profiles ||--o{ wallet_connections : has
    profiles ||--o{ referrals : "refers/referred_by"
    
    profiles {
        uuid id PK
        bigint telegram_id UK
        string username
        string first_name
        string last_name
        string photo_url
        int total_points
        int current_streak
        int longest_streak
        timestamp last_check_in
        timestamp created_at
        timestamp updated_at
    }
    
    boxes {
        uuid id PK
        uuid profile_id FK
        string type
        string rarity
        jsonb contents
        boolean opened
        timestamp opened_at
        timestamp created_at
    }
    
    badges {
        uuid id PK
        uuid profile_id FK
        string badge_type
        string name
        string description
        string icon_url
        jsonb metadata
        timestamp earned_at
    }
    
    tasks {
        uuid id PK
        uuid profile_id FK
        string task_type
        string status
        int points_reward
        jsonb requirements
        timestamp completed_at
        timestamp created_at
    }
    
    leaderboards {
        uuid id PK
        uuid profile_id FK
        string leaderboard_type
        int rank
        int score
        timestamp period_start
        timestamp period_end
        timestamp updated_at
    }
    
    referrals {
        uuid id PK
        uuid referrer_id FK
        uuid referred_id FK
        int points_earned
        string status
        timestamp created_at
    }
    
    points_ledger {
        uuid id PK
        uuid profile_id FK
        int amount
        string transaction_type
        string source
        jsonb metadata
        timestamp created_at
    }
    
    streaks {
        uuid id PK
        uuid profile_id FK
        date streak_date
        boolean checked_in
        int streak_count
        timestamp check_in_time
    }
    
    wallet_connections {
        uuid id PK
        uuid profile_id FK
        string wallet_address
        string network
        boolean is_primary
        timestamp connected_at
        timestamp last_verified
    }
    
    transactions {
        uuid id PK
        uuid profile_id FK
        string transaction_hash
        string transaction_type
        decimal amount
        string status
        jsonb metadata
        timestamp created_at
    }
```

## Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant TelegramApp
    participant MiniApp
    participant SupabaseAuth
    participant TelegramAPI
    participant Database
    
    User->>TelegramApp: Open Mini App
    TelegramApp->>MiniApp: Launch with initData
    MiniApp->>MiniApp: Parse Telegram initData
    MiniApp->>SupabaseAuth: Send initData for verification
    SupabaseAuth->>TelegramAPI: Verify initData signature
    TelegramAPI-->>SupabaseAuth: Signature valid
    SupabaseAuth->>Database: Get or create profile
    Database-->>SupabaseAuth: Profile data
    SupabaseAuth->>SupabaseAuth: Generate JWT token
    SupabaseAuth-->>MiniApp: Return JWT + session
    MiniApp->>MiniApp: Store session
    MiniApp-->>User: Show authenticated UI
```

## TON Wallet Connection Flow

```mermaid
sequenceDiagram
    participant User
    participant MiniApp
    participant TONConnect
    participant Wallet
    participant TONBlockchain
    participant EdgeFunction
    participant Database
    
    User->>MiniApp: Click "Connect Wallet"
    MiniApp->>TONConnect: Request connection
    TONConnect->>Wallet: Open wallet app
    User->>Wallet: Approve connection
    Wallet-->>TONConnect: Return wallet info
    TONConnect-->>MiniApp: Wallet connected
    
    MiniApp->>EdgeFunction: Save wallet connection
    EdgeFunction->>Database: Store wallet_address
    Database-->>EdgeFunction: Success
    EdgeFunction-->>MiniApp: Confirmation
    
    Note over MiniApp,Database: For transactions
    User->>MiniApp: Initiate transaction
    MiniApp->>TONConnect: Request signature
    TONConnect->>Wallet: Request approval
    User->>Wallet: Approve transaction
    Wallet->>TONBlockchain: Submit transaction
    TONBlockchain-->>Wallet: Transaction hash
    Wallet-->>TONConnect: Tx confirmed
    TONConnect-->>MiniApp: Transaction result
    MiniApp->>EdgeFunction: Verify transaction
    EdgeFunction->>TONBlockchain: Query transaction
    TONBlockchain-->>EdgeFunction: Tx details
    EdgeFunction->>Database: Update transaction record
```

## Streak Check-In Flow

```mermaid
sequenceDiagram
    participant User
    participant MiniApp
    participant EdgeFunction
    participant Database
    participant RealtimeService
    
    User->>MiniApp: Click "Check In"
    MiniApp->>EdgeFunction: POST /check-streak
    
    EdgeFunction->>Database: Get user's last check-in
    Database-->>EdgeFunction: Last check-in data
    
    EdgeFunction->>EdgeFunction: Calculate streak status
    
    alt Streak continues
        EdgeFunction->>Database: Increment streak counter
        EdgeFunction->>Database: Award points
        EdgeFunction->>Database: Check for badges
    else Streak broken
        EdgeFunction->>Database: Reset streak to 1
        EdgeFunction->>Database: Award base points
    end
    
    EdgeFunction->>Database: Update leaderboard
    EdgeFunction->>Database: Create points_ledger entry
    
    Database-->>EdgeFunction: Success
    EdgeFunction-->>MiniApp: Return updated streak data
    
    Database->>RealtimeService: Trigger update
    RealtimeService-->>MiniApp: Push real-time updates
    
    MiniApp-->>User: Show streak animation & rewards
```

## Points and Rewards Flow

```mermaid
graph LR
    subgraph "Point Sources"
        CheckIn[Daily Check-in]
        TaskComplete[Task Completion]
        Referral[Referral Bonus]
        Achievement[Achievement Unlocked]
        TONTx[TON Transaction]
    end
    
    subgraph "Points Processing"
        EdgeFunc[process-points<br/>Edge Function]
        Validation[Validate Action]
        Calculation[Calculate Points]
    end
    
    subgraph "Database Updates"
        Ledger[(points_ledger)]
        Profile[(profiles.total_points)]
        Leaderboard[(leaderboards)]
    end
    
    subgraph "Rewards Distribution"
        BadgeCheck{Badge<br/>Earned?}
        BoxDrop{Mystery<br/>Box?}
        Badges[(badges)]
        Boxes[(boxes)]
    end
    
    CheckIn --> EdgeFunc
    TaskComplete --> EdgeFunc
    Referral --> EdgeFunc
    Achievement --> EdgeFunc
    TONTx --> EdgeFunc
    
    EdgeFunc --> Validation
    Validation --> Calculation
    
    Calculation --> Ledger
    Calculation --> Profile
    Calculation --> Leaderboard
    
    Calculation --> BadgeCheck
    Calculation --> BoxDrop
    
    BadgeCheck -->|Yes| Badges
    BoxDrop -->|Yes| Boxes
```

## Edge Functions Overview

```mermaid
graph TB
    subgraph "Edge Functions"
        direction TB
        
        CheckStreak[check-streak<br/>---<br/>• Validate check-in time<br/>• Calculate streak status<br/>• Award points<br/>• Update profile]
        
        ProcessPoints[process-points<br/>---<br/>• Validate point source<br/>• Calculate amount<br/>• Create ledger entry<br/>• Update total points]
        
        UpdateLeaderboard[update-leaderboard<br/>---<br/>• Calculate rankings<br/>• Update positions<br/>• Handle time periods<br/>• Notify changes]
        
        HandleReferral[handle-referral<br/>---<br/>• Validate referral code<br/>• Create referral link<br/>• Award bonuses<br/>• Track conversions]
        
        VerifyTON[verify-ton-transaction<br/>---<br/>• Query blockchain<br/>• Validate transaction<br/>• Update wallet status<br/>• Award TON points]
        
        ClaimRewards[claim-rewards<br/>---<br/>• Validate eligibility<br/>• Distribute rewards<br/>• Create boxes/badges<br/>• Update inventory]
        
        ScheduledTasks[scheduled-tasks<br/>---<br/>• Reset daily streaks<br/>• Calculate leaderboards<br/>• Send notifications<br/>• Cleanup old data]
    end
    
    API[Client API Calls] --> CheckStreak
    API --> ProcessPoints
    API --> HandleReferral
    API --> VerifyTON
    API --> ClaimRewards
    
    Cron[Cron Jobs] --> ScheduledTasks
    
    CheckStreak -.-> ProcessPoints
    CheckStreak -.-> UpdateLeaderboard
    VerifyTON -.-> ProcessPoints
    ProcessPoints -.-> ClaimRewards
    ClaimRewards -.-> UpdateLeaderboard
```

## Real-time Updates Architecture

```mermaid
graph LR
    subgraph "Client Subscriptions"
        ProfileSub[Profile Updates]
        LeaderboardSub[Leaderboard Changes]
        PointsSub[Points Changes]
        NotificationSub[Notifications]
    end
    
    subgraph "Supabase Realtime"
        RealtimeEngine[Realtime Engine<br/>WebSocket Server]
        ChangeCapture[Change Data Capture<br/>PostgreSQL]
    end
    
    subgraph "Database"
        Profiles[(profiles)]
        Leaderboards[(leaderboards)]
        PointsLedger[(points_ledger)]
        Notifications[(notifications)]
    end
    
    Profiles -->|Row changes| ChangeCapture
    Leaderboards -->|Row changes| ChangeCapture
    PointsLedger -->|Row changes| ChangeCapture
    Notifications -->|Row changes| ChangeCapture
    
    ChangeCapture --> RealtimeEngine
    
    RealtimeEngine -.->|Push| ProfileSub
    RealtimeEngine -.->|Push| LeaderboardSub
    RealtimeEngine -.->|Push| PointsSub
    RealtimeEngine -.->|Push| NotificationSub
```

## Technology Stack

### Frontend
- **Framework**: React 18+
- **Build Tool**: Vite
- **Language**: TypeScript
- **Styling**: Tailwind CSS
- **UI Components**: Custom components + Headless UI
- **State Management**: React Query + Context API
- **Routing**: React Router
- **Telegram Integration**: @twa-dev/sdk
- **TON Integration**: @tonconnect/ui-react
- **HTTP Client**: Axios / Fetch API
- **WebSocket**: Supabase Realtime Client

### Backend
- **Platform**: Lovable Cloud / Supabase
- **Runtime**: Deno (Edge Functions)
- **Database**: PostgreSQL 15+
- **Auth**: Supabase Auth with Telegram provider
- **Storage**: Supabase Storage
- **Realtime**: Supabase Realtime (WebSocket)
- **Security**: Row Level Security (RLS) policies

### Blockchain
- **Network**: TON (The Open Network)
- **Wallet Connection**: TON Connect 2.0
- **SDK**: @ton/ton, @ton/core
- **Smart Contracts**: FunC (for future NFT/token features)

### External APIs
- **Telegram Bot API**: User verification, bot interactions
- **TON API**: Blockchain queries, transaction verification
- **TON Connect Bridge**: Wallet communication

## Security Considerations

```mermaid
graph TB
    subgraph "Security Layers"
        direction TB
        
        TelegramAuth[Telegram Authentication<br/>---<br/>• InitData verification<br/>• Signature validation<br/>• User identity check]
        
        JWTAuth[JWT Token Auth<br/>---<br/>• Secure token generation<br/>• Token expiration<br/>• Refresh mechanism]
        
        RLSPolicies[Row Level Security<br/>---<br/>• User isolation<br/>• Role-based access<br/>• Query filtering]
        
        EdgeValidation[Edge Function Validation<br/>---<br/>• Input sanitization<br/>• Business logic checks<br/>• Rate limiting]
        
        TONVerification[TON Verification<br/>---<br/>• Wallet ownership<br/>• Transaction validation<br/>• Smart contract security]
    end
    
    Client[Client Requests] --> TelegramAuth
    TelegramAuth --> JWTAuth
    JWTAuth --> EdgeValidation
    EdgeValidation --> RLSPolicies
    RLSPolicies --> Database[(Database)]
    
    TONOperations[TON Operations] --> TONVerification
    TONVerification --> EdgeValidation
```

## Deployment Architecture

```mermaid
graph TB
    subgraph "Production Environment"
        direction LR
        
        subgraph "CDN Layer"
            CDN[Content Delivery Network<br/>Static Assets]
        end
        
        subgraph "Application Layer"
            MiniAppDeploy[Mini App<br/>Vite Build<br/>Static Hosting]
        end
        
        subgraph "Supabase Cloud"
            EdgeRuntime[Edge Functions<br/>Deno Runtime<br/>Global Distribution]
            DBCluster[(PostgreSQL<br/>Primary + Replicas)]
            RealtimeCluster[Realtime Service<br/>WebSocket Cluster]
            StorageCluster[Storage Service<br/>S3-compatible]
        end
        
        subgraph "Monitoring"
            Logs[Logging Service]
            Metrics[Metrics & Analytics]
            Alerts[Alert Manager]
        end
    end
    
    CDN --> MiniAppDeploy
    MiniAppDeploy --> EdgeRuntime
    EdgeRuntime --> DBCluster
    EdgeRuntime --> RealtimeCluster
    EdgeRuntime --> StorageCluster
    
    EdgeRuntime --> Logs
    DBCluster --> Metrics
    Metrics --> Alerts
```

## Data Flow Examples

### Example 1: New User Onboarding

```mermaid
sequenceDiagram
    participant User
    participant MiniApp
    participant Auth
    participant DB
    participant EdgeFunc
    
    User->>MiniApp: First-time launch
    MiniApp->>Auth: Telegram initData
    Auth->>DB: Check if profile exists
    DB-->>Auth: No profile found
    Auth->>DB: Create new profile
    Auth->>DB: Create welcome tasks
    Auth->>EdgeFunc: Trigger onboarding flow
    EdgeFunc->>DB: Grant starter points
    EdgeFunc->>DB: Create mystery box
    EdgeFunc->>DB: Add to leaderboard
    DB-->>EdgeFunc: Success
    EdgeFunc-->>MiniApp: Onboarding complete
    MiniApp-->>User: Show welcome screen
```

### Example 2: Referral Flow

```mermaid
sequenceDiagram
    participant Referrer
    participant NewUser
    participant MiniApp
    participant EdgeFunc
    participant DB
    
    Referrer->>MiniApp: Generate referral link
    MiniApp->>EdgeFunc: Create referral code
    EdgeFunc->>DB: Store referral code
    DB-->>EdgeFunc: Code created
    EdgeFunc-->>MiniApp: Return referral URL
    MiniApp-->>Referrer: Show shareable link
    
    NewUser->>MiniApp: Open via referral link
    MiniApp->>EdgeFunc: Register with referral code
    EdgeFunc->>DB: Create profile with referrer_id
    EdgeFunc->>DB: Create referral record
    EdgeFunc->>DB: Award points to referrer
    EdgeFunc->>DB: Award points to new user
    DB-->>EdgeFunc: Success
    EdgeFunc-->>MiniApp: Registration complete
    MiniApp-->>NewUser: Welcome with bonus
    MiniApp-->>Referrer: Notification of new referral
```

## Scalability Considerations

1. **Database Scaling**
   - Connection pooling via PgBouncer
   - Read replicas for query distribution
   - Partitioning for large tables (points_ledger, streaks)
   - Indexes on frequently queried columns

2. **Edge Functions**
   - Stateless design for horizontal scaling
   - Automatic scaling based on load
   - Caching layer for frequently accessed data
   - Rate limiting per user/IP

3. **Real-time Updates**
   - Channel-based subscriptions
   - Selective updates (only changed fields)
   - Batch notifications for bulk operations
   - Client-side debouncing

4. **Caching Strategy**
   - CDN for static assets
   - Redis for session data (future)
   - Application-level caching for leaderboards
   - Browser caching for user profiles

## Future Enhancements

- **NFT Integration**: Mint badges as NFTs on TON blockchain
- **Token Economy**: Native $STREAK token for rewards
- **P2P Challenges**: User-to-user streak competitions
- **Social Features**: Friends list, activity feed
- **Advanced Analytics**: User behavior tracking, A/B testing
- **Push Notifications**: Via Telegram Bot API
- **Multi-language Support**: i18n implementation
- **Offline Mode**: Progressive Web App capabilities

## Conclusion

This architecture provides a scalable, secure, and maintainable foundation for the StreakFarm Telegram Mini App. The separation of concerns between frontend, backend, and blockchain layers allows for independent scaling and updates while maintaining a cohesive user experience.
