# Database Schema

## Scope and status

This repository uses one shared database/schema; workspaces are application-level tenants with `workspace_id` row filtering, not separate schemas or databases.

Evidence:
- `config/workspaces.php` explicitly says there is no schema/database-per-workspace and one database is shared.
- `app/Domains/Workspaces/Models/Concerns/BelongsToWorkspace.php:29-43` adds a global `workspace_id` scope and stamps creates with the active context.
- `app/Domains/Workspaces/Support/WorkspaceContext.php` activates a workspace for a request.

## Mermaid ER diagram

```mermaid
erDiagram
    users ||--o{ user_settings : has
    users ||--o{ workspace_user : belongs
    workspaces ||--o{ workspace_user : has
    users ||--o{ file_items : creates
    workspaces ||--o{ file_items : scopes
    file_items ||--o{ file_items : parent
    workspaces ||--o{ company_file_links : has
    file_items ||--o{ company_file_links : personal_file
    file_items ||--o{ company_file_links : company_parent
    users o|--o{ company_file_links : shared_by

    users ||--o{ conversations : creates
    users ||--o{ conversation_user : participates
    conversations ||--o{ conversation_user : has
    conversations ||--o{ messages : contains
    users o|--o{ messages : sends

    workspaces ||--o{ equipment_categories : has
    workspaces ||--o{ equipment : has
    equipment_categories o|--o{ equipment : categorizes
    file_items o|--o{ equipment : cover

    workspaces ||--o{ workspace_module_features : has
    modules ||--o{ workspace_module_features : overrides

    users {
        bigint id PK
        string first_name
        string last_name
        string email UK
        timestamp email_verified_at
        string password
        boolean is_super_admin
        timestamp banned_at
        uuid public_id UK
    }
    user_settings {
        bigint id PK
        bigint user_id FK
        json settings
    }
    workspaces {
        bigint id PK
        string slug UK
        string name
        string status
        json data
        boolean files_feature_enabled
        boolean company_files_enabled
        bigint storage_quota_bytes
    }
    workspace_user {
        bigint workspace_id PK_FK
        bigint user_id PK_FK
        timestamp created_at
        timestamp updated_at
    }
    equipment_categories {
        bigint id PK
        bigint workspace_id FK
        string name
        string color
        text description
        timestamp deleted_at
    }
    equipment {
        bigint id PK
        bigint workspace_id FK
        bigint equipment_category_id FK
        bigint cover_file_item_id FK
        string name
        string serial
        text notes
        timestamp deleted_at
    }
    file_items {
        bigint id PK
        uuid uuid UK
        bigint workspace_id FK
        bigint user_id FK
        bigint parent_id FK
        bigint owner_id
        string owner_type
        string type
        string scope
        string name
        bigint size
        timestamp deleted_at
    }
    company_file_links {
        bigint id PK
        bigint workspace_id FK
        bigint file_item_id FK
        bigint company_parent_id FK
        bigint shared_by_user_id FK
    }
    modules {
        bigint id PK
        string key UK
        string name
        boolean enabled
        string morph_alias
        json capabilities
        json features
    }
    workspace_module_features {
        bigint id PK
        bigint workspace_id FK
        bigint module_id FK
        json features
    }
    conversations {
        bigint id PK
        bigint created_by FK
        string title
        boolean is_group
        timestamp last_message_at
    }
    conversation_user {
        bigint id PK
        bigint conversation_id FK
        bigint user_id FK
        timestamp last_read_at
    }
    messages {
        bigint id PK
        bigint conversation_id FK
        bigint user_id FK
        text body
        boolean is_encrypted
        string type
    }
    permissions {
        bigint id PK
        string name
        string guard_name
    }
    roles {
        bigint id PK
        bigint team_id
        string name
        string guard_name
    }
    model_has_roles {
        bigint role_id FK
        bigint team_id
        bigint model_id
        string model_type
    }
    role_has_permissions {
        bigint permission_id FK
        bigint role_id FK
    }
```

## Table list

### Authentication and account tables
- `users`: names, unique email, password, email verification, ban fields, public UUID, super-admin flag, two-factor and profile columns. Eloquent: `app/Domains/Users/Models/User.php`.
- `password_reset_tokens`: email primary key and reset token.
- `sessions`: PHP session storage and optional user id.
- `personal_access_tokens`: Sanctum API tokens, polymorphic tokenable owner.
- `two_factor_*` columns: Fortify fields added to `users`.
- `user_settings`: JSON preferences, last-workspace state and dashboard settings.

Evidence: `database/migrations/0001_01_01_000000_create_users_table.php`, `2026_04_09_214216_add_two_factor_columns_to_users_table.php`, `2026_04_09_214216_create_personal_access_tokens_table.php`, `2026_04_18...` and `app/Domains/Users/Models/User.php`.

### Authorization and multi-tenant roles
- `roles`, `permissions`, `role_has_permissions`.
- `model_has_roles` and `model_has_permissions` support polymorphic assignments.
- Spatie teams are enabled; `roles.team_id` and assignment pivot `team_id` scope roles to a workspace. `team_id` is nullable on roles, allowing platform-level role design while the user is platform-super-admin via `users.is_super_admin`, not a team role.

Evidence: `config/permission.php:118-129`, `database/migrations/2026_04_07_173421_create_permission_tables.php`, `app/Domains/Users/Models/User.php:128-142`, `app/Domains/Access/Models/Role.php`.

### Workspace and membership tables
- `workspaces`: slug, name, status, JSON data, file flags and quotas.
- `workspace_user`: composite membership key between workspace and user.
- `workspace_invitations`: implemented invitation records; inspect/confirm exact columns before relying on it.

Evidence: `database/migrations/2019_09_15_000010_create_workspaces_table.php`, `2026_04_16_195151_create_workspace_user_table.php`, `app/Domains/Workspaces/Models/Workspace.php`, and `app/Domains/Workspaces/Models/WorkspaceInvitation.php`.

### Module tables
- `modules`: platform-global registry, enabled flag, morph alias and JSON features/capabilities.
- `workspace_module_features`: per-workspace feature overrides; unique `(workspace_id, module_id)`.

Evidence: migrations `2026_05_30_030000_create_modules_table.php` and `2026_05_30_050000_create_workspace_module_features_table.php`.

### Chat tables
- `conversations`: direct/group conversation metadata and last-message timestamp.
- `conversation_user`: participants plus `last_read_at`; unique per participant/conversation.
- `messages`: belongs to conversation and optional sender; encryption/type flags.

Evidence: migrations `2026_04_19_100000...` through `100002...` and `app/Domains/Chat/Models/Conversation.php`.

### File tables
- `file_items`: workspace, uploader, polymorphic owner, parent folder, scope, type, name, MIME and size. It has soft deletes and a UUID.
- `company_file_links`: workspace, personal file, target company folder and sharing user. Unique `(workspace_id, file_item_id)`.
- Spatie `media` table stores physical media metadata and conversions.

Evidence: `database/migrations/2026_04_20_160207_create_file_items_table.php`, `2026_04_24_100002_create_company_file_links_table.php`, `2026_04_15_171046_create_media_table.php`, `app/Domains/Files/Models/FileItem.php`.

### Implemented demo/business-domain tables
- `equipment_categories`: workspace-scoped lookup; name, colour, description and soft delete.
- `equipment`: workspace-scoped equipment record; category and cover-file foreign keys; name, serial and notes.
- These are clearly implemented as starter-kit demo modules, not property-management entities.

Evidence: `database/migrations/2026_05_27_115000_create_equipment_categories_table.php`, `2026_05_27_120000_create_equipment_table.php`, `app/Domains/Equipment/Models/Equipment.php`, `app/Domains/EquipmentCategory/Models/EquipmentCategory.php`.

### Other implemented tables
The migration set also includes app settings, notifications, mail settings/templates/layouts, activity log, pulse, cache/jobs, backups and file-share tables. These are infrastructure or starter domains rather than property-domain entities.

## Important indexes
- Unique `workspaces.slug`, `users.email`, `equipment_categories`/`equipment` and chat uniqueness.
- Composite workspace indexes on `file_items`, equipment and categories.
- Permission team and morph indexes/pivot primary keys.
- `conversation_user` unique `(conversation_id,user_id)` and lookup `(user_id,conversation_id)`.
- PostgreSQL trigram/performance migrations may add search indexes; they must be inspected for exact final schema, but no property-specific migrations were identified in the available migration names.

## Property-management relevance
- **Implemented:** generic users, workspaces/tenants, memberships, workspace roles, files, activity log, chat and demo equipment/category CRUD.
- **Partially implemented:** workspaces can function as a generic tenant boundary, but the repository does not evidence that they are branches. No branch table or branch foreign key exists in the inspected migrations.
- **Not implemented:** properties, units, buildings, tenants/leases, rents, payments, invoices, maintenance requests or occupancy tables. The `Equipment` tables are demo assets and should not be interpreted as properties.
- **Potential mismatch:** comments use `equipment`, `company`, `Workspace`, and “tenant”; none establish a one-company/three-branch business model. A property implementation would require explicit branch/site tables and branch-scoped foreign keys plus branch authorization, not merely assuming three workspaces.
- **Uncertain:** whether `workspaces.status`, `workspace_user` and membership roles are intended to model branches. The code only proves tenant/membership behavior.

## Source references
- `database/migrations/`
- `config/workspaces.php`
- `config/permission.php`
- `app/Domains/Workspaces/Models/Concerns/BelongsToWorkspace.php`
- `app/Domains/Workspaces/Models/Workspace.php`
- `app/Domains/Users/Models/User.php`
- `app/Domains/Files/Models/FileItem.php`
- `app/Domains/Equipment/Models/Equipment.php`
- `app/Domains/EquipmentCategory/Models/EquipmentCategory.php`
