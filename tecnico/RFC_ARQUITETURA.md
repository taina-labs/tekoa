# RFC: Tainá PWA Superapp Architecture

**Status:** Draft
**Author:** Technical Architecture Team
**Date:** 2025-08-31
**Version:** 2.0 (PWA Monolith)

## Executive Summary

This RFC defines the technical architecture for Tainá, a community-owned digital infrastructure platform delivered as a Progressive Web Application (PWA). Tainá is a superapp monolith that combines multiple services (messaging, photos, file storage) into a single, cohesive user experience, enabling communities to self-host their digital services with complete data sovereignty.

The architecture prioritizes rapid MVP development through a mobile-first PWA approach while maintaining clear service boundaries for future client extraction. Built as a modular monolith in Elixir/Phoenix with LiveView, Tainá delivers real-time, app-like experiences without the complexity of separate native applications or API layers.

## System Overview

### Core Philosophy

Tainá operates on the principle of "Functional Core, Imperative Shell" with clear separation between pure business logic and external integrations. The system emphasizes community-scale deployment (5-50 users initially) with hardware constraints matching community budgets (Raspberry Pi to custom servers).

### PWA Superapp Architecture

```
┌──────────────────────────────────────────────────────────┐
│               TAINÁ PWA SUPERAPP                         │
│            (Mobile-First Web Application)                │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │         LiveView Frontend Layer                   │  │
│  │                                                    │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐          │  │
│  │  │  Ybira   │ │   Jaci   │ │  Guará   │          │  │
│  │  │  Files   │ │  Photos  │ │Messaging │          │  │
│  │  │   UI     │ │    UI    │ │    UI    │          │  │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘          │  │
│  │       │            │            │                 │  │
│  │  ┌────┴────────────┴────────────┴─────────────┐   │  │
│  │  │     Shared Components & Navigation        │   │  │
│  │  │  (Service Switcher, Auth, Layouts)        │   │  │
│  │  └───────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────┘  │
│                          │                               │
│  ┌────────────────────────────────────────────────────┐  │
│  │         Service Layer (Business Logic)           │  │
│  │                                                    │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐          │  │
│  │  │  Ybira   │◄┤   Jaci   │ │  Guará   │          │  │
│  │  │  (Files) │ │ (Photos) │◄┤(Messaging)          │  │
│  │  └─────┬────┘ └──────────┘ └──────────┘          │  │
│  │        └──────► (Foundation Service)              │  │
│  │                                                    │  │
│  │  ┌──────────┐ ┌──────────────────────────────┐   │  │
│  │  │   Auth   │ │        Core                  │   │  │
│  │  │          │ │  (Communities, Shared Logic) │   │  │
│  │  └──────────┘ └──────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────┘  │
│                          │                               │
│  ┌────────────────────────────────────────────────────┐  │
│  │         Infrastructure Layer                      │  │
│  │  • PostgreSQL (Separate Schemas)                 │  │
│  │  • Phoenix PubSub • Phoenix Channels             │  │
│  │  • File Storage   • Session Management           │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
              │
    ┌─────────▼──────────┐
    │  Home Server +     │
    │  VPN Setup         │
    └────────────────────┘
```

**Note:** Arrows indicate service dependencies. Ybira (Files) is the foundation service used by both Jaci (Photos) and Guará (Messaging) for file operations.

## Architectural Principles

### Modular Monolith Structure

The system organizes as a single deployable Phoenix application with clear service boundaries. Each service maintains independence while sharing common infrastructure, enabling future extraction to standalone applications.

**Service Organization:**

```elixir
lib/taina/
├── auth/              # Authentication & authorization
├── core/              # Communities & shared business logic
├── ybira/             # File system service (foundation)
├── jaci/              # Photos service (depends on Ybira)
└── guara/             # Messaging service (depends on Ybira)

lib/taina_web/
├── live/
│   ├── ybira/         # File browser LiveViews
│   ├── jaci/          # Photo gallery LiveViews
│   └── guara/         # Chat LiveViews
├── components/        # Shared UI components
└── controllers/       # Session, future API endpoints
```

**Service Communication:**

- Services call each other directly through public APIs (e.g., `Taina.Ybira.upload/1`)
- Clear boundaries prevent tight coupling
- Shared infrastructure (Auth, Core) accessible to all services

### PWA Frontend Layer

**LiveView Architecture:** Server-rendered UI with real-time updates, minimal JavaScript
**Mobile-First Design:** Touch-optimized, responsive layouts
**Service Switcher:** Unified navigation between Ybira, Jaci, and Guará
**Progressive Enhancement:** Installable, offline-capable fallback page

**LiveView Benefits:**

- Real-time updates without client-side state management
- Direct access to backend contexts (no API serialization overhead)
- Simple upload handling with `Phoenix.LiveView.allow_upload`
- Built-in security (CSRF, authentication integration)

### Event-Driven Communication

**UI Updates:** Phoenix PubSub for LiveView subscriptions (e.g., new message notifications)
**Real-time Features:** Phoenix Channels for chat presence and typing indicators
**Inter-Service Events:** Direct function calls for MVP (async events deferred)
**Future Federation:** Event bus for inter-community communication (post-MVP)

**Simplified Approach:** Avoid over-engineering. Use PubSub only when needed for cross-user updates.

### Storage Strategy

**Structured Data:** PostgreSQL with separate schemas per service
**File Storage:** Local filesystem via Phoenix uploads
**Session Management:** Phoenix session-based authentication
**Cache Layer:** ETS for frequently accessed data (optional Redis for multi-instance)

### API Design (Future)

**MVP:** No external API layer. LiveView serves all UI needs.
**Post-MVP:** REST or GraphQL APIs added when extracting native clients
**Real-time:** Phoenix Channels for chat and presence (used internally by LiveView)

## Service Context Definitions

### Core Contexts

#### Authentication Context (`Taina.Auth`)

**Purpose:** Centralized identity and authorization management
**Responsibilities:**

- User registration and authentication (email/password)
- Role-based access control (Admin, Member, Guest)
- Session management (Phoenix session-based)
- Community membership management

**Database Schema (public schema):**

```
users
├── id (nanoid)
├── email (unique per community)
├── encrypted_password
├── role (enum: admin, member, guest)
├── community_id (foreign key)
├── confirmed_at
└── timestamps

communities
├── id (nanoid)
├── name (unique)
├── settings (jsonb)
├── storage_quota_gb
└── timestamps
```

**LiveView Integration:**

- Login/Register LiveViews with form validation
- Session-based authentication via `on_mount` hooks
- Plug pipeline for authorization checks

#### Storage Service (`Taina.Ybira` - Files) **[Foundation Service]**

**Purpose:** General-purpose file system service (used by Jaci and Guará)
**Responsibilities:**

- File and folder management (CRUD operations)
- iPad Files-like navigation (breadcrumbs, hierarchical browsing)
- File upload via `Phoenix.LiveView.allow_upload`
- Document preview (PDF minimum for MVP)
- Permissions and sharing (owner + community members)

**Database Schema (ybira schema):**

```
ybira.files
├── id (nanoid)
├── user_id (foreign key to public.users)
├── community_id (foreign key to public.communities)
├── folder_id (foreign key to ybira.folders, nullable for root)
├── filename
├── original_filename
├── file_path
├── mime_type
├── file_size_bytes
├── metadata (jsonb)
└── timestamps

ybira.folders
├── id (nanoid)
├── user_id
├── community_id
├── parent_id (self-referential, nullable for root)
├── name
└── timestamps
```

**Public API (used by other services):**

```elixir
Taina.Ybira.upload(user, attrs)           # Upload file
Taina.Ybira.get_file(id)                   # Retrieve file metadata
Taina.Ybira.delete_file(id, user)          # Delete with auth check
Taina.Ybira.list_files(folder_id, user)    # List files in folder
```

**LiveView Components:**

- File browser with folder navigation
- Upload component with progress tracking
- Document preview modal (PDF, images)
- File actions (rename, move, delete)

##### Ybira Internal Architecture

**File Storage Structure:**

Ybira stores files on the local filesystem with a deterministic path structure that ensures organization, prevents collisions, and supports future sharding:

```
/var/taina/storage/
└── communities/
    └── {community_id}/
        ├── files/
        │   └── {year}/
        │       └── {month}/
        │           └── {file_id}.{ext}    # Original file
        ├── thumbnails/
        │   └── {file_id}/
        │       ├── small.webp   (200px)
        │       ├── medium.webp  (800px)
        │       └── large.webp   (1600px)
        └── temp/
            └── {upload_id}/     # Temporary upload staging
```

**Design Rationale:**

- **Community isolation**: Top-level directory per community enables quota enforcement and backup granularity
- **Date-based sharding**: Year/month folders prevent filesystem issues with large directories (10K+ files)
- **Original file preservation**: Files stored with their original extension for direct serving
- **Separate thumbnails**: Dedicated thumbnail directory enables efficient cleanup and regeneration
- **Temp directory**: Isolated temporary storage for multi-part uploads and processing

**Upload Pipeline:**

```
1. LiveView Upload Start
   └─→ Phoenix.LiveView.allow_upload validates file type/size
       └─→ Temp file created in /var/taina/storage/communities/{id}/temp/{upload_id}/

2. Upload Progress
   └─→ LiveView streams chunks via WebSocket
       └─→ Progress updates sent to client

3. Upload Complete (consume_uploaded_entries)
   └─→ Taina.Ybira.upload/3 called with temp file path
       ├─→ Validate: mime type, file size, virus scan (future)
       ├─→ Generate: file_id (nanoid), final path
       ├─→ Move: temp file → permanent location
       ├─→ Extract: metadata (size, mime, hash for deduplication)
       ├─→ Create: database record in ybira.files
       └─→ Schedule: thumbnail generation (async)

4. Post-Upload
   └─→ Temp directory cleaned up
   └─→ PubSub broadcast: {:file_uploaded, file_id} (for UI updates)
```

**File Serving Strategy:**

For MVP, Ybira serves files directly through Phoenix:

```elixir
# TainaWeb.YbiraController
def download(conn, %{"id" => file_id}) do
  with {:ok, file} <- Ybira.get_file(file_id),
       :ok <- Ybira.authorize(:read, file, current_user(conn)) do

    conn
    |> put_resp_header("content-disposition", ~s(attachment; filename="#{file.original_filename}"))
    |> put_resp_header("cache-control", "private, max-age=3600")
    |> send_file(200, file.file_path)
  end
end
```

**Future optimization**: Use Nginx X-Accel-Redirect for zero-copy file serving:

```elixir
conn
|> put_resp_header("x-accel-redirect", "/internal/files/#{file.file_path}")
|> send_resp(200, "")
```

**Storage Quota Enforcement:**

Quota is tracked in real-time and enforced at upload:

```elixir
def upload(%User{community_id: cid} = user, file_params) do
  with {:ok, community} <- Core.get_community(cid),
       :ok <- check_quota(community, file_params.size) do
    # Proceed with upload
    # Update community.storage_used_bytes atomically
  end
end

defp check_quota(%Community{storage_quota_gb: quota, storage_used_bytes: used}, file_size) do
  quota_bytes = quota * 1_073_741_824  # GB to bytes
  if used + file_size <= quota_bytes do
    :ok
  else
    {:error, :quota_exceeded}
  end
end
```

**Quota calculation** is maintained via database trigger or periodic recalculation job.

**Thumbnail Generation:**

Thumbnails are generated asynchronously using Oban (background job processor):

```elixir
defmodule Taina.Ybira.Workers.ThumbnailWorker do
  use Oban.Worker, queue: :media_processing

  def perform(%{args: %{"file_id" => file_id}}) do
    with {:ok, file} <- Ybira.get_file(file_id),
         true <- image?(file.mime_type) do
      generate_thumbnails(file)
    end
  end

  defp generate_thumbnails(file) do
    # Use ImageMagick/Vix for image processing
    # Generate: small (200px), medium (800px), large (1600px)
    # Save to thumbnails/{file_id}/ directory
  end
end
```

**Supported formats for thumbnails:**

- Images: JPEG, PNG, HEIC, WebP, GIF
- Documents: PDF (first page thumbnail)
- Videos: Frame extraction at 0:00 (future)

**File Cleanup Strategy:**

Orphan file detection runs daily via Oban:

```elixir
# Find files in filesystem not in database
filesystem_files = list_all_files_on_disk(community_id)
database_files = Ybira.list_file_paths(community_id)

orphans = filesystem_files -- database_files
Enum.each(orphans, &File.rm/1)

# Find database records without files (integrity check)
missing = database_files -- filesystem_files
Enum.each(missing, fn path ->
  Logger.error("Missing file: #{path}")
  # Alert admin or mark for investigation
end)
```

**File deletion flow:**

1. User deletes file → `Ybira.delete_file(file_id, user)`
2. Database record marked as `deleted_at` (soft delete)
3. Background job permanently removes file and thumbnails after 30 days
4. Hard delete updates community storage quota

**Deduplication (Future):**

Files are hashed (SHA-256) on upload. If hash matches existing file:

- Reference existing file_path
- Create new database record with same file_path
- Storage quota not increased (shared file)
- Deletion only removes filesystem file when last reference deleted

#### Photos Service (`Taina.Jaci`) **[Depends on Ybira]**

**Purpose:** Photo and video gallery with organization
**Responsibilities:**

- Photo/video upload (delegates to Ybira)
- Timeline and grid gallery views
- Basic album management
- Photo/video metadata extraction

**Database Schema (jaci schema):**

```
jaci.photos
├── id (nanoid)
├── file_id (foreign key to ybira.files)
├── user_id
├── community_id
├── taken_at (extracted from EXIF)
├── metadata (jsonb: dimensions, location, camera info)
└── timestamps

jaci.albums
├── id (nanoid)
├── user_id
├── community_id
├── name
├── slug (URL-friendly)
├── cover_photo_id (foreign key to jaci.photos)
└── timestamps

jaci.album_photos
├── album_id
├── photo_id
└── position (integer for ordering)
```

**Service Integration:**

```elixir
# Jaci uses Ybira for file storage
{:ok, file} = Taina.Ybira.upload(user, file_params)
{:ok, photo} = Taina.Jaci.create_photo(user, %{file_id: file.id, metadata: ...})
```

**LiveView Components:**

- Photo upload with live progress
- Gallery grid with infinite scroll
- Album management UI
- Photo detail view with metadata

##### Jaci Internal Architecture

**Photo Processing Pipeline:**

Jaci leverages Ybira for file storage but adds photo-specific processing:

```
1. User Upload (LiveView)
   └─→ Taina.Ybira.upload(user, photo_file)
       └─→ Returns: {:ok, %File{id: file_id, ...}}

2. Photo Post-Processing (async Oban job)
   ├─→ EXIF Extraction
   │   ├─→ taken_at (DateTime from EXIF DateTimeOriginal)
   │   ├─→ camera_info (make, model, lens)
   │   ├─→ location (GPS coordinates if present)
   │   └─→ dimensions (width x height)
   ├─→ Orientation Fix
   │   └─→ Auto-rotate based on EXIF orientation flag
   ├─→ Thumbnail Generation (delegates to Ybira)
   │   ├─→ small: 200px square (gallery grid)
   │   ├─→ medium: 800px wide (detail view)
   │   └─→ large: 1600px wide (fullscreen)
   └─→ Create jaci.photos record
       └─→ Links to ybira.files via file_id

3. Gallery Display
   └─→ Query jaci.photos with thumbnails
       └─→ Lazy load images as user scrolls
```

**EXIF Extraction:**

Uses ExifTool or pure Elixir library (Exiftool.ex):

```elixir
defmodule Taina.Jaci.PhotoProcessor do
  def extract_metadata(file_path) do
    case System.cmd("exiftool", ["-json", file_path]) do
      {json, 0} ->
        data = Jason.decode!(json) |> List.first()

        %{
          taken_at: parse_datetime(data["DateTimeOriginal"]),
          camera: %{
            make: data["Make"],
            model: data["Model"],
            lens: data["LensModel"]
          },
          location: parse_gps(data),
          dimensions: %{
            width: data["ImageWidth"],
            height: data["ImageHeight"]
          },
          orientation: data["Orientation"]
        }

      _ -> {:error, :exif_extraction_failed}
    end
  end
end
```

**Photo Timeline Strategy:**

Gallery displays photos in reverse chronological order by `taken_at` (fallback to `created_at`):

```elixir
def list_photos_timeline(user, opts \\ []) do
  from(p in Photo,
    where: p.user_id == ^user.id or p.community_id == ^user.community_id,
    order_by: [desc: coalesce(p.taken_at, p.created_at)],
    preload: [:file],
    limit: ^opts[:limit] || 50,
    offset: ^opts[:offset] || 0
  )
  |> Repo.all()
end
```

**Infinite Scroll Implementation:**

LiveView uses `phx-hook` for scroll detection:

```javascript
// assets/js/hooks.js
Hooks.InfiniteScroll = {
  mounted() {
    this.observer = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting) {
        this.pushEvent("load-more", {});
      }
    });
    this.observer.observe(this.el);
  },
  destroyed() {
    this.observer.disconnect();
  },
};
```

```elixir
# In LiveView
def handle_event("load-more", _params, socket) do
  next_offset = socket.assigns.offset + 50
  more_photos = Jaci.list_photos_timeline(socket.assigns.current_user, offset: next_offset)

  {:noreply,
   socket
   |> update(:photos, &(&1 ++ more_photos))
   |> assign(:offset, next_offset)}
end
```

**Format Handling:**

HEIC (iOS photos) converted to JPEG for browser compatibility:

```elixir
defmodule Taina.Jaci.Converters.HeicConverter do
  def convert_if_needed(%File{mime_type: "image/heic"} = file) do
    output_path = String.replace(file.file_path, ".heic", ".jpg")

    System.cmd("heif-convert", [file.file_path, output_path])

    # Update file record with new path and mime_type
    Ybira.update_file(file, %{
      file_path: output_path,
      mime_type: "image/jpeg"
    })
  end

  def convert_if_needed(file), do: {:ok, file}
end
```

**Dependencies:** Requires `libheif` on server for HEIC support.

**Album Performance:**

Album photos use position-based ordering for custom arrangements:

```elixir
def list_album_photos(album_id) do
  from(ap in AlbumPhoto,
    where: ap.album_id == ^album_id,
    order_by: ap.position,
    preload: [photo: :file]
  )
  |> Repo.all()
end

def reorder_photos(album_id, photo_ids) do
  Enum.with_index(photo_ids, fn photo_id, index ->
    from(ap in AlbumPhoto,
      where: ap.album_id == ^album_id and ap.photo_id == ^photo_id
    )
    |> Repo.update_all(set: [position: index])
  end)
end
```

**Gallery View Modes:**

- **Grid View:** Thumbnails in responsive grid (2-6 columns based on screen size)
- **Timeline View:** Photos grouped by date (Today, Yesterday, Last Week, etc.)
- **Map View (Future):** Photos with GPS data displayed on map

**Memory Optimization:**

Large galleries use virtual scrolling to render only visible photos:

- Render 50 photos initially
- Load next 50 as user scrolls (intersection observer)
- Unload photos more than 200 items away from viewport
- Thumbnails cached with HTTP headers: `Cache-Control: public, max-age=31536000`

#### Messaging Service (`Taina.Guara`) **[Depends on Ybira]**

**Purpose:** Real-time messaging (DM and groups only, no status)
**Responsibilities:**

- Direct messages and group conversations
- Text, image, video, and audio attachments (via Ybira)
- Real-time presence via Phoenix Channels
- Message history and delivery
- Typing indicators

**Database Schema (guara schema):**

```
guara.conversations
├── id (nanoid)
├── community_id
├── name (for groups, null for DM)
├── conversation_type (enum: 'direct', 'group')
├── created_by
└── timestamps

guara.participants
├── conversation_id
├── user_id
├── joined_at
└── last_read_at

guara.messages
├── id (nanoid)
├── conversation_id
├── user_id
├── content (text)
├── message_type (enum: 'text', 'image', 'video', 'audio')
├── file_id (foreign key to ybira.files, nullable)
├── metadata (jsonb)
└── timestamps
```

**Service Integration:**

```elixir
# Guará uses Ybira for file attachments
{:ok, file} = Taina.Ybira.upload(user, attachment)
{:ok, message} = Taina.Guara.send_message(conversation_id, user, %{
  content: "Check this out!",
  message_type: :image,
  file_id: file.id
})
```

**Phoenix Channels Integration:**

- `conversation:{id}` topic for real-time messages
- Presence tracking for online users
- Typing indicators via presence metadata

**LiveView Components:**

- Conversation list with unread counts
- Message thread with infinite scroll
- Message composer with attachment support
- Real-time message updates via PubSub

##### Guará Internal Architecture

**Message Delivery Flow:**

Guará uses a hybrid approach: LiveView for UI, Phoenix Channels for real-time features:

```
1. User Sends Message (LiveView event)
   └─→ handle_event("send_message", %{"content" => text}, socket)
       └─→ Taina.Guara.send_message(conversation_id, user, %{content: text})
           ├─→ Validate: user is participant, content not empty
           ├─→ Insert: guara.messages record
           ├─→ Broadcast via PubSub: {:new_message, message}
           │   └─→ All LiveViews subscribed to conversation update UI
           └─→ Broadcast via Channel: "new_msg" event
               └─→ Online users see typing indicator stop

2. Message with Attachment
   ├─→ User uploads file via Ybira
   ├─→ Send message with file_id
   └─→ Recipients see inline attachment (image preview, video player, etc.)

3. Unread Count Update
   └─→ PubSub broadcast to user:{recipient_id}
       └─→ Conversation list LiveView updates badge count
```

**Phoenix Channels Integration:**

Channels handle presence and ephemeral features (typing indicators), while LiveView handles message persistence:

```elixir
# lib/taina_web/channels/conversation_channel.ex
defmodule TainaWeb.ConversationChannel do
  use Phoenix.Channel
  alias Taina.Guara.Presence

  def join("conversation:" <> conversation_id, _params, socket) do
    send(self(), :after_join)
    {:ok, assign(socket, :conversation_id, conversation_id)}
  end

  def handle_info(:after_join, socket) do
    # Track user presence
    {:ok, _} = Presence.track(socket, socket.assigns.user_id, %{
      online_at: DateTime.utc_now(),
      typing: false
    })

    push(socket, "presence_state", Presence.list(socket))
    {:noreply, socket}
  end

  def handle_in("typing:start", _payload, socket) do
    # Broadcast typing indicator to other participants
    Presence.update(socket, socket.assigns.user_id, %{typing: true})
    {:noreply, socket}
  end

  def handle_in("typing:stop", _payload, socket) do
    Presence.update(socket, socket.assigns.user_id, %{typing: false})
    {:noreply, socket}
  end
end
```

**LiveView + Channel Coordination:**

LiveView mounts channel for real-time features but handles message rendering:

```elixir
# In GuaraLive.ConversationShow
def mount(%{"id" => conversation_id}, _session, socket) do
  if connected?(socket) do
    # Subscribe to message broadcasts via PubSub
    Phoenix.PubSub.subscribe(Taina.PubSub, "conversation:#{conversation_id}")

    # Join Channel for presence (client-side via JS hook)
    # push_event sends instruction to JavaScript hook to join channel
  end

  messages = Guara.list_messages(conversation_id, limit: 50)

  {:ok,
   socket
   |> assign(:conversation_id, conversation_id)
   |> assign(:messages, messages)
   |> assign(:typing_users, [])}
end

def handle_info({:new_message, message}, socket) do
  # Real-time message arrival via PubSub
  {:noreply, update(socket, :messages, &[message | &1])}
end

def handle_info({:typing, %{user_id: user_id, typing: true}}, socket) do
  # Typing indicator from Channel presence
  {:noreply, update(socket, :typing_users, &[user_id | &1])}
end
```

**Why Both PubSub and Channels?**

- **PubSub**: Persistent message delivery, cross-LiveView updates, server-side only
- **Channels**: Ephemeral presence, typing indicators, bidirectional client events

**Message History Pagination:**

Conversation thread uses reverse infinite scroll (load older messages upward):

```elixir
def list_messages(conversation_id, opts \\ []) do
  from(m in Message,
    where: m.conversation_id == ^conversation_id,
    order_by: [desc: m.created_at],
    limit: ^opts[:limit] || 50,
    offset: ^opts[:offset] || 0,
    preload: [:user, :file]
  )
  |> Repo.all()
  |> Enum.reverse()  # Most recent at bottom
end
```

**Unread Count Calculation:**

Efficient unread tracking via `last_read_at`:

```elixir
def unread_count(conversation_id, user_id) do
  participant = get_participant(conversation_id, user_id)

  from(m in Message,
    where: m.conversation_id == ^conversation_id,
    where: m.created_at > ^participant.last_read_at,
    where: m.user_id != ^user_id,
    select: count(m.id)
  )
  |> Repo.one()
end

def mark_as_read(conversation_id, user_id) do
  from(p in Participant,
    where: p.conversation_id == ^conversation_id and p.user_id == ^user_id
  )
  |> Repo.update_all(set: [last_read_at: DateTime.utc_now()])

  # Broadcast unread count update
  Phoenix.PubSub.broadcast(
    Taina.PubSub,
    "user:#{user_id}",
    {:unread_count_updated, conversation_id, 0}
  )
end
```

**Attachment Rendering:**

Messages with attachments render inline based on file type:

```elixir
# In message component template
<%= if @message.file_id do %>
  <%= case @message.message_type do %>
    <% :image -> %>
      <img src={Routes.ybira_path(@socket, :download, @message.file_id)}
           class="max-w-md rounded-lg" />

    <% :video -> %>
      <video controls class="max-w-md">
        <source src={Routes.ybira_path(@socket, :download, @message.file_id)} />
      </video>

    <% :audio -> %>
      <audio controls>
        <source src={Routes.ybira_path(@socket, :download, @message.file_id)} />
      </audio>

    <% _ -> %>
      <a href={Routes.ybira_path(@socket, :download, @message.file_id)}
         class="flex items-center gap-2 p-2 bg-gray-100 rounded">
        <span class="icon-file"></span>
        <%= @message.file.original_filename %>
      </a>
  <% end %>
<% end %>
```

**Message Delivery Guarantees:**

For MVP, Guará provides at-least-once delivery semantics:

- Message saved to database before broadcast (durability)
- PubSub broadcast is best-effort (if LiveView disconnected, message appears on reconnect)
- No message queue or retry logic needed for community-scale deployment
- Future: Add delivery receipts and read receipts for group chats

**Presence Tracking:**

Phoenix.Presence automatically handles online/offline status:

```elixir
# In conversation LiveView
def handle_info(%{event: "presence_diff", payload: diff}, socket) do
  online_users =
    Presence.list("conversation:#{socket.assigns.conversation_id}")
    |> Map.keys()

  {:noreply, assign(socket, :online_users, online_users)}
end
```

**Performance Considerations:**

- Conversations limited to 100 participants (community scale)
- Message history loaded in 50-message chunks
- Presence tracked per-conversation (not global to avoid overhead)
- Typing indicators debounced client-side (300ms) to reduce broadcasts

**Direct Message (DM) Optimization:**

DM conversations are automatically created on first message:

```elixir
def get_or_create_dm(user_id, recipient_id) do
  case find_dm_conversation(user_id, recipient_id) do
    nil ->
      create_conversation(%{
        conversation_type: :direct,
        created_by: user_id,
        participant_ids: [user_id, recipient_id]
      })

    conversation -> {:ok, conversation}
  end
end

defp find_dm_conversation(user_id, recipient_id) do
  from(c in Conversation,
    join: p1 in Participant, on: p1.conversation_id == c.id and p1.user_id == ^user_id,
    join: p2 in Participant, on: p2.conversation_id == c.id and p2.user_id == ^recipient_id,
    where: c.conversation_type == :direct,
    having: count(p.user_id) == 2,
    select: c
  )
  |> Repo.one()
end
```

### Post-MVP Services (Deferred)

The following services are planned for future iterations but not included in the 1-year MVP:

- **Araci (Streaming):** Video/audio streaming library
- **Karai (Monitoring):** Security and system monitoring dashboard
- **Nhaman (Dashboard):** Administrative control panel
- **Ka'a (Smart Home):** IoT device management

**MVP Focus:** Ybira (Files), Jaci (Photos), Guará (Messaging) provide core community needs.

### Cross-Cutting Concerns

#### Database Design

**Primary Key Strategy:** NanoID for internal IDs, slugs for public URLs
**Multi-tenant Strategy:** Separate PostgreSQL schemas per service
**Migration Strategy:** Ecto migrations organized by service context

```
Database: taina_production

Schemas (MVP):
├── public (shared: users, communities)
├── ybira (files, folders)
├── jaci (photos, albums, album_photos)
└── guara (conversations, participants, messages)

Future Schemas (Post-MVP):
├── araci (streaming media)
├── karai (monitoring, security)
└── ... (other services)
```

**Schema Isolation Benefits:**

- Clean service boundaries for future extraction
- Separate permissions per schema if needed
- Clear ownership of tables
- Migration organization matches service organization

#### Real-time Communication

**Phoenix PubSub:** For LiveView updates across users (e.g., new messages)
**Phoenix Channels:** For chat presence and typing indicators
**No External Event Bus:** NATS deferred to federation phase (keep it simple for MVP)

## Real-time Architecture (Simplified for MVP)

### LiveView Updates

**PubSub Topics:**

- `user:{user_id}` - Personal notifications
- `conversation:{conversation_id}` - New messages in chat
- `album:{album_id}` - Photo additions to shared albums

**Usage Pattern:**

```elixir
# In LiveView
def mount(_params, _session, socket) do
  Phoenix.PubSub.subscribe(Taina.PubSub, "user:#{socket.assigns.current_user.id}")
  {:ok, socket}
end

def handle_info({:new_message, message}, socket) do
  # Update UI in real-time
  {:noreply, update(socket, :messages, &[message | &1])}
end

# In service context
Phoenix.PubSub.broadcast(Taina.PubSub, "user:#{user.id}", {:new_message, message})
```

### Phoenix Channels (Chat Only)

**Guará-specific channels:**

- `conversation:{id}` - Real-time messaging
- Presence tracking for online/offline status
- Typing indicators via presence metadata

**No Complex Event Sourcing:** Keep it simple. Direct function calls + PubSub for cross-user updates.

## LiveView Layer Design

### PWA Structure

```
lib/taina_web/
├── live/
│   ├── auth_live/
│   │   ├── login.ex              # Login form
│   │   └── register.ex           # Registration form
│   ├── ybira_live/
│   │   ├── file_browser.ex       # Main file browser
│   │   ├── upload.ex             # Upload modal
│   │   └── preview.ex            # File preview modal
│   ├── jaci_live/
│   │   ├── gallery.ex            # Photo grid/timeline
│   │   ├── upload.ex             # Photo upload
│   │   ├── album_index.ex        # Album list
│   │   └── album_show.ex         # Album detail
│   └── guara_live/
│       ├── conversation_index.ex # Conversation list
│       └── conversation_show.ex  # Message thread
├── components/
│   ├── core_components.ex        # Phoenix default components
│   ├── navigation.ex             # Service switcher, nav bar
│   ├── file_upload.ex            # Shared upload component
│   └── avatar.ex                 # User avatar component
├── controllers/
│   └── session_controller.ex     # Session create/destroy
└── router.ex                      # Routes and pipelines
```

### Route Structure

```elixir
scope "/", TainaWeb do
  pipe_through [:browser, :require_authenticated_user]

  # Service navigation
  live "/", DashboardLive.Index           # Landing/service switcher

  # Ybira (Files)
  live "/files", YbiraLive.FileBrowser
  live "/files/folders/:id", YbiraLive.FileBrowser

  # Jaci (Photos)
  live "/photos", JaciLive.Gallery
  live "/photos/albums", JaciLive.AlbumIndex
  live "/photos/albums/:slug", JaciLive.AlbumShow

  # Guará (Messaging)
  live "/messages", GuaraLive.ConversationIndex
  live "/messages/:id", GuaraLive.ConversationShow
end

scope "/auth", TainaWeb do
  pipe_through :browser

  live "/login", AuthLive.Login
  live "/register", AuthLive.Register
  post "/session", SessionController, :create
  delete "/session", SessionController, :delete
end
```

### LiveView Patterns

**Authentication Hook:**

```elixir
defmodule TainaWeb.UserAuth do
  def on_mount(:require_authenticated_user, _params, session, socket) do
    case get_user_from_session(session) do
      {:ok, user} -> {:cont, assign(socket, current_user: user)}
      {:error, _} -> {:halt, redirect(socket, to: "/auth/login")}
    end
  end
end
```

**File Upload Pattern:**

```elixir
def mount(_params, _session, socket) do
  {:ok,
   socket
   |> assign(:uploaded_files, [])
   |> allow_upload(:photos, accept: ~w(.jpg .jpeg .png .heic), max_entries: 10)}
end

def handle_event("save", _params, socket) do
  uploaded_files =
    consume_uploaded_entries(socket, :photos, fn %{path: path}, entry ->
      # Process upload via Ybira service
      Taina.Ybira.upload(socket.assigns.current_user, path, entry)
    end)

  {:noreply, update(socket, :uploaded_files, &(&1 ++ uploaded_files))}
end
```

### PWA Manifest & Service Worker

**Manifest.json:**

```json
{
  "name": "Tainá",
  "short_name": "Tainá",
  "description": "Community Digital Infrastructure",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#1F2937",
  "background_color": "#111827",
  "icons": [...]
}
```

**Service Worker:** Basic offline fallback for MVP (full offline sync deferred)

## Security Architecture

### Authentication Flow (Session-Based)

```
1. User visits /auth/login LiveView
2. Form submission → POST /auth/session
3. Server validates credentials via Taina.Auth
4. Phoenix session created with user_id
5. Redirect to dashboard
6. LiveView mount checks session via on_mount hook
7. Current user loaded and assigned to socket
```

**Session Storage:** Phoenix encrypted cookies (stateless) or ETS/Redis (stateful)

### Authorization Strategy

**Role-Based Access Control (RBAC):**

- Admin: Full community access + user management
- Member: Standard user permissions (own files, community messages)
- Guest: Limited read access (future feature)

**Resource-Level Security:**

```elixir
# In service contexts
def delete_file(file_id, %User{} = current_user) do
  with {:ok, file} <- get_file(file_id),
       :ok <- authorize(:delete, file, current_user) do
    # Perform deletion
  end
end

defp authorize(:delete, %File{user_id: user_id}, %User{id: user_id}), do: :ok
defp authorize(:delete, %File{community_id: cid}, %User{community_id: cid, role: :admin}), do: :ok
defp authorize(_, _, _), do: {:error, :unauthorized}
```

**LiveView Security:**

- CSRF protection built-in
- Session-based auth prevents token theft
- File uploads validated server-side (mime type, size)

### Data Protection

**Encryption at Rest:** Optional database/filesystem encryption at deployment level
**Encryption in Transit:** TLS 1.3 for all communications (handled by reverse proxy)
**Privacy:** Community data isolation via community_id checks

## Performance & Caching Architecture

### ETS Caching Strategy

ETS (Erlang Term Storage) provides in-memory caching for frequently accessed data:

**What to Cache:**

```elixir
# User sessions (avoid DB lookup on every request)
:ets.new(:user_sessions, [:set, :public, :named_table])

# Community settings (read-heavy, rarely change)
:ets.new(:community_settings, [:set, :public, :named_table])

# File metadata (avoid filesystem stat calls)
:ets.new(:file_metadata, [:set, :public, :named_table])
```

**Cache Invalidation:**

```elixir
defmodule Taina.Core.CommunityCache do
  def get(community_id) do
    case :ets.lookup(:community_settings, community_id) do
      [{^community_id, settings}] -> {:ok, settings}
      [] -> load_and_cache(community_id)
    end
  end

  defp load_and_cache(community_id) do
    case Core.get_community(community_id) do
      {:ok, community} ->
        :ets.insert(:community_settings, {community_id, community})
        {:ok, community}
      error -> error
    end
  end

  def invalidate(community_id) do
    :ets.delete(:community_settings, community_id)
    # Broadcast invalidation to all nodes (future clustering)
    Phoenix.PubSub.broadcast(Taina.PubSub, "cache:invalidate", {:community, community_id})
  end
end
```

**TTL Strategy (Future):**

For MVP, cache invalidation is manual. Future: Add TTL with periodic cleanup:

```elixir
# Store with timestamp
:ets.insert(:cache, {key, value, System.monotonic_time(:second)})

# Periodic cleanup via Oban
defmodule Taina.Workers.CacheCleanup do
  def perform(_job) do
    now = System.monotonic_time(:second)
    ttl = 3600  # 1 hour

    :ets.select_delete(:cache, [
      {{:_, :_, :"$1"}, [{:<, :"$1", now - ttl}], [true]}
    ])
  end
end
```

### Database Query Optimization

**Index Usage:**

All queries leverage indexes defined in schema design:

```elixir
# GOOD: Uses idx_files_user
def list_user_files(user_id) do
  from(f in File,
    where: f.user_id == ^user_id,
    order_by: [desc: f.created_at]
  )
end

# BAD: Full table scan
def list_files_by_original_filename(filename) do
  from(f in File,
    where: f.original_filename == ^filename
  )
end
# Fix: Add GIN index for trigram search if filename search needed
```

**N+1 Prevention:**

Always preload associations in context layer:

```elixir
# GOOD: Single query with preload
def list_photos(user_id) do
  from(p in Photo,
    where: p.user_id == ^user_id,
    preload: [:file, :user]
  )
  |> Repo.all()
end

# BAD: N+1 queries (1 for photos, N for files)
def list_photos_bad(user_id) do
  from(p in Photo, where: p.user_id == ^user_id)
  |> Repo.all()
  # File loaded lazily for each photo
end
```

**Query Timeouts:**

Set timeouts for all queries to prevent long-running operations:

```elixir
Repo.all(query, timeout: 5_000)  # 5 second timeout
```

### File Serving Performance

**Phoenix send_file Strategy:**

MVP serves files via Phoenix with caching headers:

```elixir
def download(conn, %{"id" => file_id}) do
  with {:ok, file} <- Ybira.get_file(file_id),
       :ok <- authorize(:read, file, current_user(conn)) do
    conn
    |> put_resp_header("cache-control", "private, max-age=31536000")
    |> put_resp_header("content-type", file.mime_type)
    |> put_resp_header("etag", file.etag)
    |> send_file(200, file.file_path)
  end
end
```

**Range Request Support:**

Enable resumable downloads and video streaming:

```elixir
def download(conn, %{"id" => file_id}) do
  # ... authorization ...

  case get_req_header(conn, "range") do
    ["bytes=" <> range] ->
      send_file_range(conn, file.file_path, range)
    _ ->
      send_file(conn, 200, file.file_path)
  end
end
```

**Nginx X-Accel-Redirect (Future Optimization):**

For high-traffic deployments, offload file serving to Nginx:

```nginx
# nginx.conf
location /internal/files/ {
  internal;  # Only accessible via X-Accel-Redirect
  alias /var/taina/storage/;
}
```

```elixir
def download(conn, %{"id" => file_id}) do
  # ... authorization ...

  conn
  |> put_resp_header("x-accel-redirect", "/internal/files/#{relative_path(file)}")
  |> put_resp_header("content-type", file.mime_type)
  |> send_resp(200, "")
end
```

**Benefits:** Zero-copy file serving, frees Phoenix processes immediately.

### Thumbnail Caching

**HTTP Cache Headers:**

Thumbnails are immutable (file_id never changes):

```elixir
def thumbnail(conn, %{"file_id" => file_id, "size" => size}) do
  thumbnail_path = "/var/taina/storage/.../thumbnails/#{file_id}/#{size}.webp"

  conn
  |> put_resp_header("cache-control", "public, max-age=31536000, immutable")
  |> put_resp_header("content-type", "image/webp")
  |> send_file(200, thumbnail_path)
end
```

**CDN-Ready:**

Thumbnails can be served via CDN (Cloudflare, BunnyCDN) for distributed communities:

- Set CORS headers for cross-origin access
- Use deterministic URLs: `/thumbnails/{file_id}/{size}.webp`
- CDN respects cache headers (1 year cache)

### LiveView Performance

**Temporary Assigns:**

Prevent large data from being stored in LiveView state:

```elixir
def mount(_params, _session, socket) do
  {:ok,
   socket
   |> assign(:photos, list_photos())
   |> assign(:current_user, user)
   |> temporary_assigns([photos: []])}  # Photos not serialized to client
end
```

**Stream Collections (Phoenix 1.7+):**

For large lists, use streams instead of assigns:

```elixir
def mount(_params, _session, socket) do
  {:ok,
   socket
   |> stream(:photos, list_photos())}
end

def handle_event("load-more", _params, socket) do
  more_photos = list_photos(offset: socket.assigns.offset + 50)

  {:noreply, stream(socket, :photos, more_photos)}
end
```

**Debouncing User Input:**

Prevent excessive server calls for search/filter inputs:

```javascript
// assets/js/app.js
let searchTimeout;
input.addEventListener("input", (e) => {
  clearTimeout(searchTimeout);
  searchTimeout = setTimeout(() => {
    liveSocket.push("search", { query: e.target.value });
  }, 300);
});
```

### Connection Pooling

**Ecto Connection Pool:**

Configure pool size based on hardware:

```elixir
# config/prod.exs
config :taina, Taina.Repo,
  pool_size: 10,  # 2x CPU cores recommended
  queue_target: 50,  # Queue wait time (ms)
  queue_interval: 1000  # Check interval (ms)
```

**Phoenix Endpoint Pool:**

```elixir
config :taina, TainaWeb.Endpoint,
  http: [
    port: 4000,
    transport_options: [num_acceptors: 10]
  ]
```

### Monitoring & Metrics

**Phoenix LiveDashboard:**

Built-in monitoring for MVP:

```elixir
# router.ex
import Phoenix.LiveDashboard.Router

scope "/" do
  pipe_through [:browser, :require_admin]

  live_dashboard "/admin/dashboard",
    metrics: TainaWeb.Telemetry,
    ecto_repos: [Taina.Repo]
end
```

**Custom Telemetry Metrics:**

```elixir
defmodule TainaWeb.Telemetry do
  def metrics do
    [
      # Phoenix Metrics
      summary("phoenix.router_dispatch.stop.duration"),
      summary("phoenix.endpoint.stop.duration"),

      # Ecto Metrics
      summary("taina.repo.query.total_time"),
      counter("taina.repo.query.count"),

      # Custom Metrics
      counter("taina.ybira.uploads.count"),
      summary("taina.ybira.uploads.duration"),
      last_value("taina.storage.used_bytes")
    ]
  end
end
```

### Performance Targets (MVP)

**Response Times:**

- File upload: <2s for 10MB file (depends on network)
- Photo gallery load: <500ms (50 thumbnails)
- Message send: <100ms (write + broadcast)
- File download start: <50ms (authorization + headers)

**Resource Limits:**

- PostgreSQL connections: 10-20 (community scale)
- LiveView processes: 100 concurrent users max (Raspberry Pi 5)
- File uploads: Max 100MB per file, 1GB total per hour per user
- Message attachments: Max 25MB per attachment

## Backup & Recovery Architecture

### Backup Strategy

**What to Backup:**

1. **PostgreSQL Database:** All schemas (public, ybira, jaci, guara)
2. **File Storage:** `/var/taina/storage/communities/{id}/files/`
3. **Thumbnails:** `/var/taina/storage/communities/{id}/thumbnails/` (optional, can regenerate)
4. **Configuration:** Docker Compose files, environment variables

**Backup Frequency:**

- **Database:** Daily full backup + continuous WAL archiving (future)
- **Files:** Daily incremental backup (rsync to external drive)
- **Configuration:** Backup on change (manual or via git)

**Automated Backup Script:**

```bash
#!/bin/bash
# /opt/taina/backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/mnt/backup/taina"
COMMUNITY_ID="your_community_id"

# 1. PostgreSQL Backup
docker exec taina-db pg_dump -U postgres taina_production \
  | gzip > "$BACKUP_DIR/db_$DATE.sql.gz"

# 2. File Storage Backup (incremental)
rsync -avz --delete \
  /var/taina/storage/communities/$COMMUNITY_ID/ \
  $BACKUP_DIR/files/

# 3. Retain last 30 days of backups
find $BACKUP_DIR -name "db_*.sql.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
```

**Scheduled via Cron:**

```cron
# Backup daily at 2 AM
0 2 * * * /opt/taina/backup.sh >> /var/log/taina-backup.log 2>&1
```

### Restore Procedures

**Database Restore:**

```bash
# 1. Stop Tainá application
docker compose down

# 2. Restore PostgreSQL dump
gunzip -c db_20250101_020000.sql.gz | \
  docker exec -i taina-db psql -U postgres taina_production

# 3. Restart application
docker compose up -d
```

**File Storage Restore:**

```bash
# Restore files from backup
rsync -avz --delete \
  /mnt/backup/taina/files/ \
  /var/taina/storage/communities/$COMMUNITY_ID/
```

**Full System Restore:**

1. Install fresh Tainá instance on new hardware
2. Restore configuration files and `.env`
3. Restore database dump
4. Restore file storage
5. Regenerate thumbnails (optional): `mix taina.thumbnails.regenerate`

### Disaster Recovery

**Recovery Time Objective (RTO):** 4 hours (time to restore service)
**Recovery Point Objective (RPO):** 24 hours (acceptable data loss)

**Disaster Scenarios:**

1. **Disk Failure:**
   - If RAID: Replace disk, rebuild array
   - If single disk: Restore from backup to new disk
   - RTO: 2-4 hours

2. **Database Corruption:**
   - Restore from last known good backup
   - Potential data loss: Last 24 hours
   - RTO: 1 hour

3. **Complete Hardware Failure:**
   - Deploy Tainá to new hardware
   - Restore from remote backup
   - RTO: 4-8 hours

4. **Accidental Data Deletion:**
   - Soft delete (30-day retention) allows recovery
   - Hard delete: Restore specific files from backup
   - RTO: 1-2 hours

**Backup Storage Recommendations:**

- **Primary:** External USB drive (encrypted)
- **Secondary:** NAS or cloud storage (Backblaze B2, rsync.net)
- **Geographic:** Keep off-site backup for physical disasters

### Data Integrity Checks

**Scheduled Integrity Verification:**

```elixir
defmodule Taina.Workers.IntegrityCheck do
  use Oban.Worker, queue: :maintenance

  def perform(_job) do
    # Check database-filesystem consistency
    check_file_integrity()
    check_orphan_files()
    check_missing_files()
  end

  defp check_file_integrity do
    Ybira.list_all_files()
    |> Enum.each(fn file ->
      unless File.exists?(file.file_path) do
        Logger.error("Missing file: #{file.id} - #{file.file_path}")
        Taina.Alerts.notify_admin(:missing_file, file)
      end
    end)
  end
end
```

## Database Schema Design (MVP)

### Shared Schema (public)

```sql
CREATE TABLE public.communities (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  name VARCHAR(255) UNIQUE NOT NULL,
  subdomain VARCHAR(50) UNIQUE,
  settings JSONB DEFAULT '{}',
  storage_quota_gb INTEGER DEFAULT 100,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE public.users (
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

CREATE INDEX idx_users_community ON public.users(community_id, role);
CREATE INDEX idx_users_email ON public.users(community_id, email);
```

### Ybira Schema (Files)

```sql
CREATE SCHEMA ybira;

CREATE TABLE ybira.folders (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  user_id VARCHAR(21) NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  community_id VARCHAR(21) NOT NULL REFERENCES public.communities(id) ON DELETE CASCADE,
  parent_id VARCHAR(21) REFERENCES ybira.folders(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE ybira.files (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  user_id VARCHAR(21) NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  community_id VARCHAR(21) NOT NULL REFERENCES public.communities(id) ON DELETE CASCADE,
  folder_id VARCHAR(21) REFERENCES ybira.folders(id) ON DELETE SET NULL,
  filename VARCHAR(255) NOT NULL,
  original_filename VARCHAR(255) NOT NULL,
  file_path VARCHAR(500) NOT NULL,
  mime_type VARCHAR(100) NOT NULL,
  file_size_bytes BIGINT NOT NULL,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_files_user ON ybira.files(user_id, created_at DESC);
CREATE INDEX idx_files_folder ON ybira.files(folder_id);
CREATE INDEX idx_folders_parent ON ybira.folders(parent_id);
```

### Jaci Schema (Photos)

```sql
CREATE SCHEMA jaci;

CREATE TABLE jaci.photos (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  file_id VARCHAR(21) NOT NULL REFERENCES ybira.files(id) ON DELETE CASCADE,
  user_id VARCHAR(21) NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  community_id VARCHAR(21) NOT NULL REFERENCES public.communities(id) ON DELETE CASCADE,
  taken_at TIMESTAMP WITH TIME ZONE,
  metadata JSONB DEFAULT '{}',  -- EXIF data, dimensions, etc.
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE jaci.albums (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  user_id VARCHAR(21) NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  community_id VARCHAR(21) NOT NULL REFERENCES public.communities(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(255) UNIQUE,
  cover_photo_id VARCHAR(21) REFERENCES jaci.photos(id) ON DELETE SET NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE jaci.album_photos (
  album_id VARCHAR(21) NOT NULL REFERENCES jaci.albums(id) ON DELETE CASCADE,
  photo_id VARCHAR(21) NOT NULL REFERENCES jaci.photos(id) ON DELETE CASCADE,
  position INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (album_id, photo_id)
);

CREATE INDEX idx_photos_user ON jaci.photos(user_id, taken_at DESC NULLS LAST);
CREATE INDEX idx_photos_file ON jaci.photos(file_id);
```

### Guará Schema (Messaging)

```sql
CREATE SCHEMA guara;

CREATE TABLE guara.conversations (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  community_id VARCHAR(21) NOT NULL REFERENCES public.communities(id) ON DELETE CASCADE,
  name VARCHAR(255),  -- For groups, null for DM
  conversation_type VARCHAR(20) DEFAULT 'direct' CHECK (conversation_type IN ('direct', 'group')),
  created_by VARCHAR(21) NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE guara.participants (
  conversation_id VARCHAR(21) NOT NULL REFERENCES guara.conversations(id) ON DELETE CASCADE,
  user_id VARCHAR(21) NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  last_read_at TIMESTAMP WITH TIME ZONE,
  PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE guara.messages (
  id VARCHAR(21) PRIMARY KEY DEFAULT nanoid(),
  conversation_id VARCHAR(21) NOT NULL REFERENCES guara.conversations(id) ON DELETE CASCADE,
  user_id VARCHAR(21) NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  message_type VARCHAR(20) DEFAULT 'text' CHECK (message_type IN ('text', 'image', 'video', 'audio')),
  file_id VARCHAR(21) REFERENCES ybira.files(id) ON DELETE SET NULL,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation ON guara.messages(conversation_id, created_at DESC);
CREATE INDEX idx_participants_user ON guara.participants(user_id);
```

### Indexing Strategy

**Performance Indexes:**

- Users: Community + email lookup, role-based queries
- Files: User timeline, folder navigation
- Photos: User timeline, album associations
- Messages: Conversation threads, user participation

**Search Indexes (Future):**

- Full-text search on message content (GIN index)
- File name search (trigram index)

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

**Plug & Play Deployment:**

- Docker Compose setup for one-command deployment
- Automated database initialization and migrations
- VPN configuration for secure remote access (handled by infrastructure team)
- Reverse proxy with automatic TLS certificates

**Single Node Architecture:**

```
┌─────────────────────────────────────┐
│      Home Server / Mini PC          │
├─────────────────────────────────────┤
│  Docker Compose:                    │
│  ├── taina (Phoenix app)            │
│  ├── postgres (with all schemas)    │
│  ├── nginx (reverse proxy)          │
│  └── vpn-service (WireGuard/etc)    │
└─────────────────────────────────────┘
```

**Process Supervision:** OTP supervision trees for fault tolerance
**Database:** PostgreSQL with automated backups to local storage
**Storage:** Local filesystem (configurable storage path)
**Monitoring:** Phoenix LiveDashboard for system metrics

### Scaling Considerations

**MVP Timeline (1 Year):**

- Installable instance setup on home servers
- VPN infrastructure for remote access
- PWA fully functional with all three services
- Document preview support (PDF minimum)

**Vertical Scaling:** Increase hardware specs within community budget
**Storage Scaling:** Mount additional drives, adjust storage path
**Future Extraction:** Services designed for future native app clients (APIs added when needed)

## Testing Strategy

### Test Pyramid Structure

**Unit Tests:** Pure business logic in service contexts
**Integration Tests:** Service interactions (Jaci→Ybira, Guará→Ybira)
**LiveView Tests:** User interactions and UI behavior
**End-to-End Tests:** Critical user flows (upload photo, send message)

### Testing Approaches

**LiveView Testing:**

```elixir
test "upload photo successfully", %{conn: conn} do
  {:ok, view, _html} = live(conn, "/photos")

  file = %{
    last_modified: 1_594_171_879_000,
    name: "photo.jpg",
    content: File.read!("test/fixtures/photo.jpg"),
    size: 1_396,
    type: "image/jpeg"
  }

  view
  |> file_input(:photo_upload, :photos, [file])
  |> render_upload(file)

  assert has_element?(view, "#photo-#{photo.id}")
end
```

**Context Testing:** Pure Elixir functions with ExUnit
**Property-Based Testing:** StreamData for edge cases (file names, message content)
**Performance Testing:** Load testing with realistic community size (5-50 users)
**Security Testing:** Authorization checks in context functions

### Test Organization

```
test/
├── taina/
│   ├── auth_test.exs
│   ├── ybira_test.exs
│   ├── jaci_test.exs
│   └── guara_test.exs
├── taina_web/
│   ├── live/
│   │   ├── ybira_live_test.exs
│   │   ├── jaci_live_test.exs
│   │   └── guara_live_test.exs
│   └── controllers/
│       └── session_controller_test.exs
├── integration/
│   └── service_integration_test.exs  # Jaci→Ybira, etc.
└── support/
    ├── fixtures.ex
    └── conn_case.ex
```

## MVP Implementation Order (1-Year Timeline)

### Phase 1: Foundation (Months 1-2)

1. **Phoenix Setup:** LiveView, Tailwind, asset pipeline
2. **Authentication:** User/community models, session-based auth
3. **Base UI:** Navigation shell, service switcher, responsive layout
4. **PWA Basics:** Manifest, service worker, offline fallback

### Phase 2: Ybira - File System (Months 3-4)

1. **File Service:** Upload, storage, metadata
2. **Folder Navigation:** Hierarchical browsing, breadcrumbs
3. **File Operations:** CRUD, move, rename
4. **Document Preview:** PDF viewer minimum, image preview

### Phase 3: Jaci - Photos (Months 5-6)

1. **Photo Service:** Integration with Ybira
2. **Upload UI:** LiveView with progress tracking
3. **Gallery Views:** Grid, timeline
4. **Albums:** Basic album management

### Phase 4: Guará - Messaging (Months 7-9)

1. **Chat Service:** Conversations, messages
2. **Real-time:** Phoenix Channels, presence
3. **Message UI:** Thread view, composer
4. **Attachments:** Text, images, videos, audio via Ybira

### Phase 5: Polish & Deploy (Months 10-12)

1. **Mobile Optimization:** Touch interactions, responsive design
2. **Installation Flow:** Docker Compose, VPN setup
3. **Testing:** End-to-end user flows
4. **Documentation:** User guides, admin docs

### Future Architecture Evolution (Post-MVP)

**Native Apps:** Extract Jaci/Guará as native mobile apps with REST/GraphQL APIs
**Electric SQL:** Client-side sync for offline-first messaging
**Federation:** Inter-community communication
**Additional Services:** Araci (streaming), Karai (monitoring), etc.
**AI Features:** Local LLM for photo organization, smart search

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

This RFC establishes a pragmatic architecture for rapid MVP delivery through a PWA superapp approach. By consolidating three core services (Ybira, Jaci, Guará) into a single LiveView application, we dramatically simplify development while maintaining clear service boundaries for future extraction.

**Key Architectural Decisions:**

- **LiveView-only frontend** eliminates API serialization overhead and client-side state complexity
- **Session-based auth** provides security without token management
- **Separate PostgreSQL schemas** enable clean service boundaries and future extraction
- **Direct service calls** keep complexity low while service APIs remain extraction-ready
- **Minimal real-time infrastructure** (PubSub + Channels only where needed)

**MVP Success Criteria (1-Year):**

- Installable Tainá instance on home servers with VPN
- Fully functional PWA with mobile-first design
- File system with iPad-like navigation + PDF preview
- Photo gallery with upload and basic albums
- Messaging with DM/groups, presence, and media attachments
- Sub-100ms response times for typical operations
- 99.5% uptime for community instances

**Post-MVP Evolution:**
The architecture supports extracting services as native apps by adding REST/GraphQL APIs without refactoring business logic. Service dependencies (Jaci→Ybira, Guará→Ybira) are explicitly designed for API extraction.

Implementation prioritizes Ybira (foundation) → Jaci (photos) → Guará (messaging), ensuring each service is fully functional before moving to the next. This incremental approach delivers value early and validates architectural decisions with real usage.
