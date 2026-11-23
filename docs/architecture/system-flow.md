# Overall System Flow

## Mermaid system flow

```mermaid
flowchart TD
    Browser[Browser] --> Entry{Application entry}

    Entry -->|GET /| Welcome[Welcome Inertia page]
    Entry -->|Fortify /login| Fortify[Fortify authentication]
    Entry -->|OAuth| Socialite[Socialite controller]
    Entry -->|Public /share| Share[PublicShareController]

    Fortify --> AuthUser[Fortify authenticate callback]
    AuthUser -->|valid + not banned| LoginResponse[LoginResponse]
    AuthUser -->|invalid/banned| LoginError[Generic login error]

    LoginResponse --> Verified{Email verified}
    Verified -->|no| Verify[Verification Inertia page]
    Verified -->|yes| AppLanding[/app WorkspaceLandingController]

    AppLanding --> TenantMode{WORKSPACES_ENABLED}
    TenantMode -->|false| BindDefault[BindDefaultWorkspace]
    TenantMode -->|true| ChooseWorkspace{Workspace count}
    ChooseWorkspace -->|one| DashboardRedirect[/w/{slug}/dashboard]
    ChooseWorkspace -->|multiple| Picker[Workspace Picker]
    ChooseWorkspace -->|create-own + none| Onboarding[Workspace onboarding]

    Picker --> DashboardRedirect
    BindDefault --> WorkspaceRoutes

    DashboardRedirect --> Resolve[ResolveWorkspace slug]
    Resolve -->|missing/inactive/not member| Denied[403/404 Inertia error]
    Resolve -->|authorized| Context[WorkspaceContext + Spatie team id]
    Context --> Middleware[web auth verified module/workspace settings middleware]

    Middleware --> RouteController{Route controller}
    RouteController -->|dashboard| Dashboard[DashboardController]
    RouteController -->|equipment CRUD| Equipment[EquipmentController]
    RouteController -->|equipment categories CRUD| Categories[EquipmentCategoryController]
    RouteController -->|files| Files[File controllers/services]
    RouteController -->|chat| Chat[ChatController]
    RouteController -->|notifications| Notifications[NotificationController]
    RouteController -->|central admin| Admin[SuperAdmin controllers]

    Dashboard --> DashboardModels[Workspace, FileItem, User]
    DashboardModels --> DashboardProps[Inertia props and widgets]

    Equipment --> EquipmentModels[Equipment, EquipmentCategory, FileItem]
    Categories --> CategoryModels[EquipmentCategory]
    Files --> FileModels[FileItem, CompanyFileLink, Media]

    EquipmentModels --> Broadcast[ResourceChanged event]
    CategoryModels --> Broadcast
    Broadcast --> Reverb[Private Echo/Reverb channel]

    Chat --> ChatModels[Conversation, Message, conversation_user]
    ChatModels --> ChatBroadcast[MessageSent event]
    ChatBroadcast --> Reverb

    DashboardProps --> Inertia[Inertia middleware]
    EquipmentModels --> Inertia
    CategoryModels --> Inertia
    FileModels --> Inertia
    ChatModels --> Inertia

    Inertia --> SharedProps[auth, app settings, workspace, modules, flash]
    SharedProps --> Frontend[Vue 3 SPA mount]
    Frontend --> PageResolver[import.meta.glob resources/js/Pages/**/*.vue]
    PageResolver --> Layout[Command/guest layout]

    Admin --> AuthChecks{SuperAdmin or capability}
    AuthChecks -->|authorized| CentralModels[Users, roles, workspaces, modules, settings]
    AuthChecks -->|denied| Denied

    CentralModels --> Inertia

    subgraph MainModules[Main implemented modules]
        UsersDomain[Users/auth/profile]
        AccessDomain[Roles/permissions]
        WorkspacesDomain[Workspace/membership]
        FilesDomain[Personal/company/entity files]
        EquipmentDomain[Demo equipment module]
        CategoryDomain[Demo categories module]
        ChatDomain[Chat]
        NotificationsDomain[Notifications/mail]
        OperationsDomain[Ops/health/backups]
        ModulesDomain[Feature registry]
    end

    RouteController -.-> MainModules
```

## Concise explanation

1. Laravel bootstraps routes and middleware in `bootstrap/app.php`.
2. Fortify owns login and headless auth actions; custom responses direct verified users to `/app`.
3. `/app` is the convergence point: it resolves a workspace, redirects to a unique workspace, or renders a picker/onboarding view.
4. Multi-tenant workspace routes run through `ResolveWorkspace`; it verifies membership/super-admin access and activates `WorkspaceContext` plus the Spatie team ID.
5. Controllers perform authorization and query models. Inertia renders named Vue pages from `resources/js/Pages`.
6. Shared props expose auth, active workspace, module features and app settings.
7. The generic resource-broadcast system and chat broadcasting provide live updates through Reverb/Echo private channels.

## Implementation status

### Implemented
- Fortify authentication and email verification.
- Ban enforcement, 2FA, password reset and OAuth entry points.
- Workspace tenancy/membership/picker and team-scoped roles.
- Inertia/Vue pages and shared props.
- Personal/company/entity files.
- Equipment and equipment-category demo CRUD.
- Chat, notifications, module registry and broadcasting.
- Super-admin operations and settings.

### Partially implemented
- One-company/three-branch assumptions have no dedicated implementation. Workspaces are generic tenants and must not be inferred as branches.
- Module/domain extensibility is generic and demonstrative; no property entities exist.
- Workspace member roles are implemented with Spatie teams, but no branch-specific organizational policy exists.

### Not implemented
- Property, unit/apartment, lease, tenant, rent, invoice/payment or maintenance-request domains.
- Explicit branch table and branch-scoped authorization.
- A demonstrated one-company, three-branch hierarchy.

## Source references
- `bootstrap/app.php`
- `routes/web.php`
- `routes/workspace.php`
- `routes/channels.php`
- `app/Domains/Auth/Providers/FortifyServiceProvider.php`
- `app/Domains/Workspaces/Http/Middleware/ResolveWorkspace.php`
- `app/Domains/Workspaces/Http/Controllers/WorkspaceLandingController.php`
- `app/Domains/Workspaces/Http/Controllers/DashboardController.php`
- `app/Http/Middleware/HandleInertiaRequests.php`
- `app/Domains/Equipment/Http/Controllers/EquipmentController.php`
- `app/Domains/Chat/Http/Controllers/ChatController.php`
- `app/Support/Events/ResourceChanged.php`
- `resources/js/app.ts`

## Final architectural findings and mismatches

- The repository is a capable Laravel starter kit with a generic workspace tenant pattern, not a property-management implementation.
- “Workspace” is the only tenant-like entity. It must not be called a branch without stronger source evidence.
- The central shared database plus `workspace_id` global scopes are implemented. No separate branch databases or schemas are evident.
- Equipment is a documented reference demo for future entities. It is not evidence of property/asset lease functionality.
- There is no migration or model for properties, units, leases, rent, maintenance or payments.
- Team roles (`Admin`, `Editor`, `User`) exist and are correctly scoped to workspace/team IDs. This is reusable authorization infrastructure for branches if a future Branch entity is added, but it is not presently linked to branches.
- Three branches cannot be inferred from the `workspaces` table or configuration; any one-company/multiple-workspace mapping is a business assumption, not an implemented relation.
- Chat participant and message tables exist, but no property, branch or lease relation exists in chat authorization data; shared-workspace membership is used instead.
- File ownership is polymorphic and can eventually support property/unit entities, but only implemented owner types include User, Workspace and Equipment.
