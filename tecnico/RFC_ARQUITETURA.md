# RFC: Tainá Backend Architecture

**Status:** Draft  
**Author:** Technical Architecture Team  
**Date:** 2025-08-31  
**Version:** 1.0

## Executive Summary

This RFC defines the technical architecture for Tainá, a community-owned digital infrastructure platform. Tainá serves as the central backend monolith supporting multiple client applications across the ecosystem, enabling communities to self-host their digital services with complete data sovereignty.

The architecture prioritizes rapid development for MVP delivery while maintaining extensibility for future distributed deployment. Built as a modular monolith in Elixir/Phoenix, Tainá provides backend services for eight client applications through a unified API layer with real-time communication capabilities.

## System Overview

### Core Philosophy

Tainá operates on the principle of "Functional Core, Imperative Shell" with clear separation between pure business logic and external integrations. The system emphasizes community-scale deployment (5-50 users initially) with hardware constraints matching community budgets (Raspberry Pi to custom servers).

### Client Ecosystem Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Applications                       │
├─────────────┬─────────────┬─────────────┬─────────────────┤
│    Jaci     │    Guará    │   Araci     │   Others...     │
│  (Photos)   │ (Messaging) │ (Streaming) │   (6 more)      │
│             │             │             │                 │
│   Mobile    │    Web      │   Smart TV  │   IoT/CLI       │
│    Apps     │    Apps     │    Apps     │    Apps         │
└─────────────┴─────────────┴─────────────┴─────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  GraphQL + WS API │
                    └─────────┬─────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│                  TAINÁ BACKEND                            │
│                 (This Project)                            │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │           Modular Monolith                         │  │
│  │                                                     │  │
│  │  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ │  │
│  │  │ Auth  │ │ Media │ │ Chat  │ │Files  │ │ ...   │ │  │
│  │  │Context│ │Context│ │Context│ │Context│ │       │ │  │
│  │  └───────┘ └───────┘ └───────┘ └───────┘ └───────┘ │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Shared Infrastructure                  │  │
│  │  • Event Bus (NATS)  • Database (PostgreSQL)      │  │
│  │  • File Storage      • Cache Layer                 │  │
│  │  • Monitoring        • Security                    │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

## Architectural Principles

### Modular Monolith Structure

The system organizes as a single deployable application with clear context boundaries. Each functional area maintains independence while sharing common infrastructure.

**Context Organization:**
- Each client application maps to one primary context (Jaci → Media Context)
- Contexts communicate through well-defined interfaces
- Shared concerns (Auth, Monitoring) exist as cross-cutting contexts
- Database schemas align with context boundaries

### Event-Driven Communication

**Internal Events:** Message passing between contexts using OTP processes  
**External Events:** NATS-based pub/sub for client applications and future federation  
**Real-time:** Phoenix Channels for immediate client updates

### Storage Strategy

**Structured Data:** PostgreSQL with schema-per-context approach  
**File Storage:** Local filesystem with Minio-compatible API for future object storage migration  
**Cache Layer:** ETS/Redis for session and frequently accessed data

### API Design

**GraphQL:** Primary client API for flexible data fetching  
**WebSocket:** Real-time communication via Phoenix Channels  
**REST:** Administrative endpoints and webhook integrations

## Service Context Definitions

### Core Contexts

#### Authentication Context
**Purpose:** Centralized identity and authorization management  
**Responsibilities:**
- User registration and authentication (email/password, OTP)
- Role-based access control (Admin, Member, Guest)
- Session management and JWT token handling
- Community membership management

**Database Schema:**
```
users
├── id (uuid)
├── email (unique)
├── encrypted_password
├── role (enum: admin, member, guest)
├── community_id (foreign key)
├── confirmed_at
└── timestamps

communities
├── id (uuid)
├── name (unique)
├── settings (jsonb)
├── storage_quota
└── timestamps

user_sessions
├── id (uuid)
├── user_id (foreign key)
├── token_hash
├── expires_at
└── timestamps
```

#### Media Context (Jaci - Photos)
**Purpose:** Photo and video storage with intelligent organization  
**Responsibilities:**
- File upload and processing
- Automatic categorization and tagging
- Face recognition and grouping
- Timeline and album management
- Mobile backup synchronization

**Key Events:**
- `media.file.uploaded`
- `media.album.created`
- `media.face.detected`
- `media.backup.completed`

#### Communication Context (Guará - Messaging)
**Purpose:** Real-time messaging and group communication  
**Responsibilities:**
- Direct and group messaging
- Message encryption and security
- File sharing integration
- Online presence management
- Message history and search

**Key Events:**
- `chat.message.sent`
- `chat.user.online`
- `chat.group.created`
- `chat.file.shared`

#### Storage Context (Ybira - Files)
**Purpose:** General file storage and sharing  
**Responsibilities:**
- File and folder management
- Version control and history
- Sharing and permissions
- Sync across devices
- Trash and recovery

#### Streaming Context (Araci - Media Library)
**Purpose:** Video and audio streaming services  
**Responsibilities:**
- Media library management
- Transcoding and optimization
- Streaming protocols
- Watch history and recommendations
- Subtitle and metadata handling

#### Monitoring Context (Karai - Security)
**Purpose:** System security and monitoring  
**Responsibilities:**
- Intrusion detection
- System health monitoring
- Backup management
- Firewall integration
- Security event logging

#### Dashboard Context (Nhaman - Control Panel)
**Purpose:** Administrative interface and system management  
**Responsibilities:**
- System status overview
- User and community management
- Configuration management
- Analytics and reporting
- System maintenance tools

#### Automation Context (Ka'a - Smart Home)
**Purpose:** IoT device management and automation  
**Responsibilities:**
- Device discovery and pairing
- Automation rules and triggers
- Sensor data collection
- Integration with external services
- Energy monitoring

### Cross-Cutting Concerns

#### Event Bus
**Technology:** NATS with JetStream for durability  
**Pattern:** Event Sourcing for critical operations  
**Guarantees:** At-least-once delivery for external events

#### Database Design

**Primary Key Strategy:** NanoID for internal IDs, slugs for public URLs  
**Multi-tenant Strategy:** Schema-per-context with shared user/community tables  
**Migration Strategy:** Ecto migrations with context-aware organization

```
Database: taina_production

Schemas:
├── public (shared tables: users, communities, sessions)
├── media (photos, albums, faces, metadata)
├── chat (messages, groups, participants, reactions)
├── files (documents, folders, versions, shares)
├── streaming (movies, series, episodes, watch_history)
├── monitoring (events, metrics, alerts, backups)
├── dashboard (widgets, reports, configurations)
└── automation (devices, rules, triggers, sensors)
```

## Event-Driven Architecture

### Event Categories

**Command Events:** Trigger actions across contexts  
**Domain Events:** Represent business state changes  
**Integration Events:** External system communication  
**System Events:** Infrastructure and monitoring

### Event Schema Design

```elixir
%{
  type: "photo.uploaded",           # Simple dot notation
  user_id: "user_nanoid",
  data: %{                         # Flexible payload
    photo_id: "photo_nanoid",
    filename: "sunset.jpg"
  },
  timestamp: ~U[2025-08-31 10:30:00Z]
}
```

**Day 1 Event Types:**
- Authentication: `user.signed_in`, `user.signed_out`
- Media: `photo.uploaded`, `album.created`  
- Chat: `message.sent`, `user.online`, `user.offline`

### Event Flow Patterns

**User Action Flow:**
```
Client Request → Context Handler → Domain Logic → Event Emission → 
Event Bus → Interested Contexts → Side Effects → Client Notification
```

**Inter-Context Communication:**
```
Context A → Event Bus → Context B Handler → Business Logic → 
Response Event → Context A → Client Update
```

## API Layer Design

### GraphQL Operations (Day 1)

**Queries (Read Actions):**
- `me` - Current user info
- `photos` - List user photos  
- `albums` - List user albums
- `conversations` - List user conversations
- `messages` - Get conversation messages

**Mutations (Write Actions):**
- `signIn` / `signOut` - Authentication
- `uploadPhoto` - Add new photo
- `createAlbum` - Create photo album
- `sendMessage` - Send chat message
- `createConversation` - Start new chat

**Subscriptions (Live Updates):**
- `messageReceived` - New messages in conversation
- `photoUploaded` - Photo processing complete  
- `userPresence` - Online/offline status

### Real-time Communication

**Phoenix Channels Topics:**
- `user:{user_id}` - Personal notifications
- `community:{community_id}` - Community-wide updates  
- `conversation:{conversation_id}` - Chat messages
- `system:alerts` - System notifications

## Security Architecture

### Authentication Flow

```
1. Client → POST /api/auth/signin
2. Server validates credentials
3. JWT token issued with claims (user_id, community_id, role)
4. Client includes token in Authorization header
5. GraphQL resolver validates token and extracts context
```

### Authorization Strategy

**Role-Based Access Control (RBAC):**
- Admin: Full community access
- Member: Standard user permissions  
- Guest: Limited read access

**Resource-Level Security:**
- Media files: Owner + community members
- Messages: Conversation participants only
- Files: Explicit sharing permissions

### Data Protection

**Encryption at Rest:** Database and file system encryption  
**Encryption in Transit:** TLS 1.3 for all communications  
**Privacy:** Optional client-side encryption for sensitive data

## Database Schema Design

### Core Tables

```sql
-- Shared Schema (public)
CREATE TABLE communities (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  name VARCHAR(255) UNIQUE NOT NULL,
  subdomain VARCHAR(50) UNIQUE,
  settings JSONB DEFAULT '{}',
  storage_quota_gb INTEGER DEFAULT 100,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE users (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  community_id VARCHAR(21) NOT NULL REFERENCES communities(id) ON DELETE CASCADE,
  email VARCHAR(255) NOT NULL,
  encrypted_password VARCHAR(255) NOT NULL,
  role VARCHAR(20) DEFAULT 'member' CHECK (role IN ('admin', 'member', 'guest')),
  confirmed_at TIMESTAMP WITH TIME ZONE,
  last_sign_in_at TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(community_id, email)
);

-- Media Schema
CREATE SCHEMA media;

CREATE TABLE media.files (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  user_id VARCHAR(21) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  community_id VARCHAR(21) NOT NULL REFERENCES communities(id) ON DELETE CASCADE,
  filename VARCHAR(255) NOT NULL,
  original_filename VARCHAR(255) NOT NULL,
  file_path VARCHAR(500) NOT NULL,
  mime_type VARCHAR(100) NOT NULL,
  file_size_bytes BIGINT NOT NULL,
  metadata JSONB DEFAULT '{}',
  processing_status VARCHAR(20) DEFAULT 'pending',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE media.albums (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  user_id VARCHAR(21) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  community_id VARCHAR(21) NOT NULL REFERENCES communities(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(255) UNIQUE,  -- URL-friendly: /albums/vacation-2025
  description TEXT,
  cover_file_id VARCHAR(21) REFERENCES media.files(id),
  is_public BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Chat Schema  
CREATE SCHEMA chat;

CREATE TABLE chat.conversations (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  community_id VARCHAR(21) NOT NULL REFERENCES communities(id) ON DELETE CASCADE,
  name VARCHAR(255),
  conversation_type VARCHAR(20) DEFAULT 'direct' CHECK (conversation_type IN ('direct', 'group')),
  created_by VARCHAR(21) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE chat.messages (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  conversation_id VARCHAR(21) NOT NULL REFERENCES chat.conversations(id) ON DELETE CASCADE,
  user_id VARCHAR(21) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  message_type VARCHAR(20) DEFAULT 'text' CHECK (message_type IN ('text', 'file', 'image', 'system')),
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### Indexing Strategy

**Performance Indexes:**
- Users: `(community_id, email)`, `(community_id, role)`
- Media: `(user_id, created_at DESC)`, `(community_id, mime_type)`
- Messages: `(conversation_id, created_at DESC)`, `(user_id)`

**Search Indexes:**
- Full-text search on messages content
- GIN indexes on JSONB metadata fields

## Deployment and Scalability

### Hardware Requirements

**Phase 1 (MVP - 5 users):**
- Raspberry Pi 5 (8GB RAM) or Mini PC N100
- 1TB NVMe storage  
- 4-8W power consumption
- Single instance deployment

**Phase 2 (Growth - 20-30 users):**
- Custom NAS or dedicated server
- 16-32GB RAM, quad-core CPU
- 4-8TB storage in RAID configuration
- 30-50W power consumption

**Phase 3 (Scale - 50+ users):**
- High-end server (Ryzen 7, 64GB RAM)
- 20TB+ storage with redundancy
- Optional GPU for AI features
- 80-150W power consumption

### Deployment Strategy

**Single Node:** All services on one machine with Docker Compose  
**Process Supervision:** OTP supervision trees for fault tolerance  
**Database:** PostgreSQL with automated backups  
**Storage:** Local filesystem with planned Minio migration  
**Monitoring:** Built-in health checks and metrics

### Scaling Considerations

**Vertical Scaling:** Increase hardware specs within community budget  
**Storage Scaling:** Add drives to existing system  
**Network Scaling:** Distributed deployment for multi-location communities  
**Federation:** Future inter-community communication

## Testing Strategy

### Test Pyramid Structure

**Unit Tests:** Pure function testing with ExUnit  
**Integration Tests:** Context interaction testing  
**End-to-End Tests:** Full API flow testing

### Testing Approaches

**Property-Based Testing:** StreamData for data generation  
**Contract Testing:** GraphQL schema validation  
**Performance Testing:** Load testing with realistic community size  
**Security Testing:** Authentication and authorization validation

### Test Organization

```
test/
├── unit/
│   ├── contexts/
│   │   ├── auth/
│   │   ├── media/
│   │   └── chat/
│   └── support/
├── integration/
│   ├── api/
│   └── events/
└── e2e/
    ├── user_flows/
    └── admin_flows/
```

## Migration Path and Evolution

### MVP Implementation Order

1. **Foundation:** Authentication, basic user management
2. **Core Contexts:** Media (Jaci) and Communication (Guará) 
3. **API Layer:** GraphQL schema and Phoenix Channels
4. **Client Integration:** Mobile and web client APIs
5. **Monitoring:** Basic health checks and logging

### Future Architecture Evolution

**Multi-tenancy:** Support for multiple communities per instance  
**Federation:** Inter-community communication protocols  
**Edge Computing:** CDN integration for media delivery  
**AI Integration:** Local LLM for content processing

## Risk Mitigation

### Technical Risks

**Single Point of Failure:** OTP supervision and restart strategies  
**Data Loss:** Automated backups with restoration procedures  
**Performance Degradation:** Monitoring with alerting thresholds  
**Security Breaches:** Regular security audits and updates

### Community Risks

**Hardware Failure:** Redundant hardware recommendations  
**Internet Outages:** Offline-capable features where possible  
**Community Management:** Role separation and governance tools  
**Data Migration:** Export tools for community transition

## Conclusion

This RFC establishes a foundation for rapid MVP development while maintaining architectural integrity for future growth. The modular monolith approach balances simplicity with scalability, enabling community deployment on modest hardware while preserving expansion capabilities.

The event-driven architecture supports real-time features essential for modern user experiences, while the schema-per-context database design enables clear ownership boundaries and future service extraction.

Implementation should prioritize the Authentication and Media contexts for initial community validation, followed by Communication features for daily user engagement. The monitoring and security contexts provide operational stability required for community trust.

Success metrics include sub-100ms API response times, 99.9% uptime for community instances, and seamless client application integration across the ecosystem.
