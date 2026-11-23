# Sequence Diagrams

## 1. Login and authentication

### Actual route and middleware
- Fortify `/login` handles the GET and POST routes registered by the package.
- Fortify login view: `FortifyServiceProvider::boot()` renders `resources/js/Pages/Auth/Login.vue`.
- `FortifyServiceProvider::authenticateUsing()` loads `User` by email, checks password and ban state, updates `last_login_at`, and returns the user.
- `LoginResponse::toResponse()` verifies invitations, checks email verification, then redirects to `/app`.
- `/app` is handled by `WorkspaceLandingController::__invoke()` and is behind `auth`, `verified`.

```mermaid
sequenceDiagram
    actor Browser
    participant LoginVue as Auth/Login.vue
    participant Fortify as Laravel Fortify
    participant Provider as FortifyServiceProvider
    participant User as User model
    participant LoginResponse
    participant Landing as WorkspaceLandingController

    Browser->>LoginVue: GET /login
    LoginVue-->>Browser: Inertia Login page
    Browser->>Fortify: POST /login
    Fortify->>Provider: authenticateUsing(request)
    Provider->>User: User::where(email)->first()
    Provider->>User: Hash::check(password)
    alt valid, not banned
        Provider->>User: forceFill(last_login_at)->save()
        Provider-->>Fortify: authenticated User
        Fortify->>LoginResponse: toResponse(request)
        alt email not verified
            LoginResponse-->>Browser: redirect verification.notice
        else verified
            LoginResponse-->>Browser: redirect intended /app
            Browser->>Landing: GET /app
            Landing-->>Browser: workspace dashboard redirect or picker
        end
    else invalid or banned
        Provider-->>Fortify: null
        Fortify-->>Browser: generic invalid-credentials response
    end
```

Evidence:
- `app/Domains/Auth/Providers/FortifyServiceProvider.php:34-81`
- `app/Domains/Auth/Http/Responses/LoginResponse.php:10-26`
- `routes/web.php:134-145`
- `app/Domains/Workspaces/Http/Controllers/WorkspaceLandingController.php:25-101`
- `resources/js/Pages/Auth/Login.vue`

## 2. Workspace resolution and access control

### Actual route shape
- Multi-tenant: `GET /w/{workspace}/dashboard` from `routes/workspace.php`, mounted in `bootstrap/app.php`.
- Middleware: `web`, `auth`, `verified`, `ResolveWorkspace`.
- Single-tenant: workspace routes are mounted at the root and use `BindDefaultWorkspace` instead.

```mermaid
sequenceDiagram
    actor Browser
    participant Router as routes/workspace.php
    participant Resolve as ResolveWorkspace
    participant Workspace as Workspace
    participant Context as WorkspaceContext
    participant Spatie as PermissionRegistrar
    participant Controller as DashboardController

    Browser->>Router: GET /w/{workspace}/dashboard
    Router->>Resolve: auth + verified + ResolveWorkspace
    Resolve->>Workspace: where slug = parameter first()
    alt inactive or missing
        Resolve-->>Browser: 404/403
    else active
        Resolve->>Workspace: compare membership/status
        Resolve->>Resolve: user is super-admin OR belongsToWorkspace
        alt not member and not super-admin
            Resolve-->>Browser: 403
        else authorized
            Resolve->>Context: runFor(workspace, next)
            Context->>Spatie: set team id = workspace.id
            Context->>Controller: index(request)
            Controller-->>Browser: Inertia Dashboard
            Context->>Spatie: restore prior team id
        end
    end
```

Evidence:
- `bootstrap/app.php:36-52`
- `routes/workspace.php:39`
- `app/Domains/Workspaces/Http/Middleware/ResolveWorkspace.php:28-77`
- `app/Domains/Workspaces/Support/WorkspaceContext.php`
- `app/Domains/Workspaces/Models/Concerns/BelongsToWorkspace.php:29-43`

**Uncertainty:** `ResolveWorkspace` bypasses membership for `isSuperAdmin()` and allows any active workspace. It proves global support users, not branch semantics.

## 3. Dashboard loading

```mermaid
sequenceDiagram
    actor Browser
    participant DashboardVue as Dashboard.vue
    participant Route as /w/{workspace}/dashboard
    participant Resolve as ResolveWorkspace
    participant Controller as DashboardController
    participant Workspace as Workspace
    participant FileItem
    participant Registry as DashboardWidgetRegistry

    Browser->>Route: GET dashboard
    Route->>Resolve: resolve tenant and membership
    Resolve->>Controller: index(request)
    Controller->>Workspace: users()->count()
    alt files_feature_enabled
        Controller->>FileItem: workspace + user file count/sum
    end
    Controller->>Controller: unreadMessagesCount()
    Controller->>Registry: forUser(workspace,user,hidden)
    Registry-->>Controller: enabled widget data
    Controller-->>Browser: Inertia::render('Dashboard', props)
    Browser->>DashboardVue: resolve/resources/js/Pages/Dashboard.vue
    DashboardVue-->>Browser: render member/file/chat/widget UI
```

Evidence:
- `routes/workspace.php:39`
- `app/Domains/Workspaces/Http/Controllers/DashboardController.php:18-65`
- `app/Domains/Modules/Services/DashboardWidgetRegistry.php`
- `resources/js/app.ts:18-24`
- `resources/js/Pages/Dashboard.vue`

## 4. Representative CRUD: equipment create

This is the strongest implemented domain-like CRUD but remains a demo starter module.

```mermaid
sequenceDiagram
    actor Browser
    participant EquipmentVue as Equipment/Index.vue
    participant Route as POST /w/{workspace}/equipment
    participant Resolve as ResolveWorkspace
    participant Controller as EquipmentController
    participant Workspace
    participant Equipment
    participant Category as EquipmentCategory
    participant Broadcast as ResourceChanged

    Browser->>Route: POST equipment data
    Route->>Resolve: auth/verified/workspace/module
    Resolve->>Controller: store(request)
    Controller->>Workspace: currentTenant(request)
    Controller->>Workspace: canManageEquipment(user)
    alt permission denied
        Controller-->>Browser: 403
    else allowed
        Controller->>Controller: validate name/category/serial/notes
        Controller->>Category: exists rule scoped to workspace
        Controller->>Equipment: create(workspace_id,data)
        Controller->>Broadcast: broadcastResourceChanged(equipment, created, id, workspaceId)
        Broadcast-->>Browser: private workspace resource ping
        Controller-->>Browser: redirect to equipment.show + success flash
    end
    Browser->>EquipmentVue: follows redirect Inertia response
```

Evidence:
- `routes/workspace.php:163-180`
- `app/Domains/Equipment/Http/Controllers/EquipmentController.php:120-133,621-653`
- `app/Domains/Workspaces/Models/Workspace.php:118-129`
- `app/Support/Concerns/BroadcastsResourceChanges.php:19-34`
- `app/Support/Events/ResourceChanged.php:32-69`
- `routes/channels.php:39-51`

## 5. Real-time notification/broadcast flow

### Chat message

```mermaid
sequenceDiagram
    actor Sender
    participant ChatVue as Chat.vue
    participant Route as POST /chat/conversations/{conversation}/messages
    participant Controller as ChatController
    participant Conversation
    participant Message
    participant Broadcast as MessageSent
    participant Echo as Laravel Echo/Reverb
    actor Recipient

    Sender->>ChatVue: submit text/attachments
    ChatVue->>Route: POST message
    Route->>Controller: sendMessage(request, conversation)
    Controller->>Conversation: isParticipant(sender)
    alt not participant
        Controller-->>Sender: 403
    else participant
        Controller->>Controller: validate text/attachments
        Controller->>Conversation: messages()->create(user_id/body)
        Controller->>Message: addMedia attachments
        Controller->>Conversation: update last_message_at + sender pivot
        Controller->>Broadcast: broadcast(new MessageSent(...)->toOthers())
        Broadcast->>Echo: PrivateChannel chat.conversation.{id}
        Echo-->>Recipient: message.sent payload
        Controller->>Conversation: recipients where not sender
        Controller->>Recipient: NewChatMessageNotification
        Controller-->>Sender: 201 JSON formatted message
    end
```

Evidence:
- `routes/web.php:118-127`
- `app/Domains/Chat/Http/Controllers/ChatController.php:311-415`
- `app/Domains/Chat/Events/MessageSent.php:13-77`
- `routes/channels.php:13-15`
- `resources/js/Pages/Chat.vue`

### Workspace resource live updates

```mermaid
sequenceDiagram
    participant Mutator as CRUD Controller
    participant Trait as BroadcastsResourceChanges
    participant Event as ResourceChanged
    participant Channel as PrivateChannel workspace.{id}.resources
    participant Echo as Laravel Echo
    actor OtherClient

    Mutator->>Trait: broadcastResourceChanged(resource,action,id,workspace)
    Trait->>Event: create and dontBroadcastToCurrentUser()
    Event->>Channel: ShouldBroadcast
    Channel->>Channel: authorize workspace member/super-admin
    Channel-->>Echo: ResourceChanged payload
    Echo-->>OtherClient: ping + partial Inertia reload/live-row fetch
```

Evidence:
- `app/Support/Concerns/BroadcastsResourceChanges.php:19-34`
- `app/Support/Events/ResourceChanged.php:32-69`
- `routes/channels.php:39-51`
- Controllers such as `EquipmentController::store/update/destroy` call the trait.

## Frontend rendering
- `resources/js/app.ts` creates the Inertia/Vue application and resolves `resources/js/Pages/**/*.vue` via `import.meta.glob`, so an Inertia response maps to the named Vue page.
- `app/Http/Middleware/HandleInertiaRequests.php` shares auth, workspace, module, settings and other props.
- Inertia JSON/redirect responses drive normal Vue navigation; chat and live resource features also use Laravel Echo.

## Source references
- `routes/web.php`
- `routes/workspace.php`
- `bootstrap/app.php`
- `app/Domains/Auth/Providers/FortifyServiceProvider.php`
- `app/Domains/Auth/Http/Responses/LoginResponse.php`
- `app/Domains/Workspaces/Http/Middleware/ResolveWorkspace.php`
- `app/Domains/Workspaces/Http/Controllers/WorkspaceLandingController.php`
- `app/Domains/Workspaces/Http/Controllers/DashboardController.php`
- `app/Domains/Equipment/Http/Controllers/EquipmentController.php`
- `app/Domains/Chat/Http/Controllers/ChatController.php`
- `app/Support/Events/ResourceChanged.php`
- `routes/channels.php`
- `resources/js/app.ts`
