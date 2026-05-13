# WordPress Dependency Analysis & Re-Implementation Guide

## Greenfield Implementation - CommonsBooking

---

## 1. Executive Summary

Das CommonsBooking WordPress Plugin hat **umfangreiche Abhängigkeiten** zu WordPress-Core, WordPress-Plugin-API (Hooks), CMB2 (Metabox-Library) und WordPress-Datenbank-Schema. Für eine Greenfield-Implementation in einem anderen Tech-Stack müssen diese Abhängigkeiten neu implementiert oder durch Alternativen ersetzt werden.

**Key Findings:**
- **72+ WordPress Core-Funktionen** werden direkt verwendet
- **20+ WordPress Hooks** für Plugin-Lifecycle, Admin und Frontend
- **CMB2 Library** komplett WordPress-abhängig (muss ersetzt werden)
- **WordPress Database Schema** als Basis für Post-Meta-Struktur

---

## 2. Dependency Classification

### 2.1 Category Overview

| Kategorie | Abhängigkeiten | Neu implementieren? | Lösung |
|-----------|---------------|---------------------|--------|
| **Authentication/Users** | WordPress User System | ❌ Nein | Keycloak (empfohlen) |
| **Custom Post Types** | WP CPT API | ✅ Ja | Eigene DB-Schema + ORM |
| **Database/ORM** | $wpdb, WP_Query | ✅ Ja | PostgreSQL + Query Builder |
| **Plugin API (Hooks)** | add_action, add_filter | ✅ Ja | Event Bus / Message Queue |
| **REST API** | WP_REST_Controller | ✅ Ja | Eigenes Framework (z.B. FastAPI, Express) |
| **Shortcodes** | add_shortcode | ✅ Ja | Template Engine |
| **Cron Jobs** | wp_schedule_event | ✅ Ja | Background Job Scheduler |
| **Frontend (Scripts)** | wp_enqueue_scripts | ✅ Ja | Build Tool (Vite, Webpack) |
| **Admin UI** | CMB2 Library | ✅ Ja | React/Vue Admin Panel |
| **Settings/Options** | CMB2 + Options API | ✅ Ja | Admin Panel + DB Storage |
| **i18n** | load_textdomain | ✅ Ja | gettext / i18n Library |

---

## 3. Detailed Dependency Analysis

### 3.1 Authentication & User Management

**WordPress Functions Used:**
- `wp_get_current_user()`
- `get_current_user_id()`
- `get_user_meta()`
- `get_user_by()`
- `get_role()`, `add_role()`
- `$role->add_cap()`, `$role->remove_cap()`
- `wp_login_url()`, `wp_logout_url()`, `wp_registration_url()`

**Current WordPress Behavior:**
- WordPress User Roles: `administrator`, `cb_manager`, `subscriber`
- Capabilities system für CPT-Berechtigungen
- User-Meta für plugin-spezifische Daten

**➡️ Recommendation: Replace with Keycloak**
- JWT-based authentication
- Role-based access control (RBAC)
- OAuth 2.0 / OIDC support
- User federation support

**See Section 6 for detailed Keycloak analysis.**

---

### 3.2 Custom Post Types (CPTs)

**WordPress CPTs Registered:**
| CPT | Slug | Features |
|-----|------|----------|
| Item | `cb_item` | Title, Content, Featured Image, Custom Meta |
| Location | `cb_location` | Title, Content, Featured Image, Geo Data, Custom Meta |
| Timeframe | `cb_timeframe` | Title, Date Range, Grid Config, Repetition |
| Booking | `cb_booking` | Title, Status, Date Range, User Relation |
| Restriction | `cb_restriction` | Title, Date Range, Type, State Selection |
| Map | `cb_map` | Title, Configuration JSON |

**WordPress Functions Used:**
- `register_post_type()`
- `register_taxonomy()`
- `register_post_status()`
- `get_post()`, `get_post_meta()`, `update_post_meta()`
- `wp_update_post()`, `wp_insert_post()`, `wp_delete_post()`
- `get_post_type()`, `get_post_status()`

**➡️ Re-Implementation:**
```
Database Schema (PostgreSQL):

items:
  - id: UUID PRIMARY KEY
  - name: VARCHAR(255)
  - description: TEXT
  - category_id: UUID FK
  - image_url: VARCHAR(500)
  - created_at: TIMESTAMP
  - updated_at: TIMESTAMP
  - author_id: UUID FK → users

locations:
  - id: UUID PRIMARY KEY
  - name: VARCHAR(255)
  - description: TEXT
  - address: VARCHAR(500)
  - latitude: DECIMAL(10,8)
  - longitude: DECIMAL(11,8)
  - pickup_instructions: TEXT
  - created_at: TIMESTAMP
  - updated_at: TIMESTAMP
  - author_id: UUID FK → users

timeframes:
  - id: UUID PRIMARY KEY
  - type: ENUM('bookable','holiday','official_holiday','repair','booking','booking_canceled')
  - start_date: DATE
  - end_date: DATE
  - start_time: TIME (nullable)
  - end_time: TIME (nullable)
  - grid_duration: INT (minutes)
  - mode: ENUM('full_day','time_based')
  - repeat: ENUM('none','weekly','monthly')
  - item_id: UUID FK → items
  - location_id: UUID FK → locations
  - meta: JSONB
  - created_at: TIMESTAMP

bookings:
  - id: UUID PRIMARY KEY
  - status: ENUM('unconfirmed','confirmed','canceled')
  - start_date: DATE
  - end_date: DATE
  - item_id: UUID FK → items
  - location_id: UUID FK → locations
  - user_id: UUID FK → users
  - booking_code: VARCHAR(20)
  - notes: TEXT
  - booked_at: TIMESTAMP
  - confirmed_at: TIMESTAMP (nullable)
  - created_at: TIMESTAMP
  - updated_at: TIMESTAMP

restrictions:
  - id: UUID PRIMARY KEY
  - name: VARCHAR(255)
  - type: ENUM('item','location','category')
  - start_date: DATE
  - end_date: DATE
  - state: VARCHAR(10) (German state code)
  - item_id: UUID FK → items (nullable)
  - location_id: UUID FK → locations (nullable)
  - category_id: UUID FK → categories (nullable)
  - created_at: TIMESTAMP

categories:
  - id: UUID PRIMARY KEY
  - name: VARCHAR(255)
  - slug: VARCHAR(100)
  - type: ENUM('item','location')

booking_codes (Custom Table):
  - date: DATE
  - timeframe_id: UUID FK → timeframes
  - location_id: UUID FK → locations
  - item_id: UUID FK → items
  - code: VARCHAR(100)
  - PRIMARY KEY (date, timeframe_id, location_id, item_id, code)
```

---

### 3.3 Plugin API (Hooks System)

**add_action() Hooks Used (90+ matches):**

| Hook | Purpose | Re-Implementation |
|------|---------|-------------------|
| `init` | Plugin initialization, CPT registration, rewrite rules | App startup event |
| `admin_init` | Admin settings initialization | Admin module init |
| `admin_menu` | Menu pages creation | Router-based menus |
| `plugins_loaded` | Plugin loading sequence | Dependency injection |
| `wp_enqueue_scripts` | Frontend scripts/styles | Build tool + asset pipeline |
| `admin_enqueue_scripts` | Admin scripts/styles | Admin SPA bundles |
| `rest_api_init` | REST route registration | Route decorators |
| `save_post` | Post meta save hooks | ORM callbacks |
| `pre_get_posts` | Admin list filtering | Query middleware |
| `admin_notices` | Admin notifications | Notification service |
| `widgets_init` | Widget registration | Component registration |
| `cron_schedules` | Custom cron intervals | Job scheduler config |

**Custom Hooks Created:**
```php
// Plugin triggers these custom hooks:
do_action('commonsbooking_mail_sent');
do_action('commonsbooking_clear_cache');
do_action('commonsbooking_unschedule');
```

**➡️ Re-Implementation with Event Bus:**

```typescript
// TypeScript Event Bus Implementation
class EventBus {
  private handlers: Map<string, Set<EventHandler>> = new Map();

  on(event: string, handler: EventHandler, priority: number = 10): void;
  off(event: string, handler: EventHandler): void;
  emit(event: string, data: any): void;
}

// Usage:
// eventBus.on('booking.created', async (booking) => { ... });
// eventBus.emit('mail.sent', { to: 'user@example.com' });
```

**Hook Mapping:**
| WP Hook | Greenfield Equivalent |
|---------|----------------------|
| `init` | `app.on('startup')` |
| `admin_init` | `admin.on('init')` |
| `save_post` | ORM `afterSave()` callback |
| `pre_get_posts` | Query middleware |
| `admin_notices` | NotificationService.show() |
| `wp_schedule_event` | BullMQ scheduled jobs |

---

### 3.4 REST API

**Current WP REST Routes:**
| Route | Methods | Description |
|-------|---------|-------------|
| `/commonsbooking/v1/items` | GET | List/Create items |
| `/commonsbooking/v1/locations` | GET | List locations (GeoJSON) |
| `/commonsbooking/v1/availability` | GET | Availability slots |
| `/commonsbooking/v1/projects` | GET | Organization info |
| `/commonsbooking/v1/station_status.json` | GET | GBFS station status |
| `/commonsbooking/v1/station_information.json` | GET | GBFS station info |
| `/commonsbooking/v1/system_information.json` | GET | GBFS system info |
| `/commonsbooking/v1/vehicle_status.json` | GET | GBFS vehicle status |
| `/commonsbooking/v1/vehicle_availability.json` | GET | GBFS vehicle availability |
| `/commonsbooking/v1/discovery.json` | GET | GBFS discovery |

**WordPress Classes Used:**
- `WP_REST_Controller` (base class for all routes)
- `WP_REST_Request`
- `WP_REST_Response`
- `register_rest_route()`

**➡️ Re-Implementation:**

```typescript
// Express.js or FastAPI Implementation
import express from 'express';
const app = express();

// Route decorators (like WP_REST_Controller)
@Controller('/items')
class ItemsController extends ApiController {
  @Get('/')
  async getItems(@Query() params: ItemsQuery) { ... }

  @Get('/{id}')
  async getItem(@Param('id') id: string) { ... }

  @Post('/')
  @Auth(Roles.Admin)
  async createItem(@Body() data: CreateItemDto) { ... }
}

// OpenAPI/Swagger for schema documentation
// JSON Schema validation (like Opis in WP version)
```

---

### 3.5 Database & ORM

**$wpdb Usage (88+ matches):**

| File | Usage Pattern |
|------|---------------|
| Timeframe.php | Custom JOIN queries for meta ordering |
| BookingCodes.php | Custom INSERT/SELECT for codes table |
| Restriction.php | Complex multi-filter queries |
| CB1.php | Legacy migration queries |

**Current Schema (WordPress-based):**
```
wp_posts (WP Core)
  └── post_type: cb_booking, cb_item, cb_location, etc.

wp_postmeta (WP Core)
  └── _cb_location_lat, _cb_timeframe_grid, etc.
```

**➡️ Re-Implementation:**

```typescript
// PostgreSQL with Prisma or Drizzle ORM
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// Example: Get bookable timeframes
const timeframes = await prisma.timeframe.findMany({
  where: {
    type: 'bookable',
    startDate: { gte: today },
    item: { id: itemId },
    location: { id: locationId }
  },
  include: {
    item: true,
    location: true
  }
});
```

---

### 3.6 Shortcodes

**Current Shortcodes:**
| Shortcode | Handler Class | Output |
|-----------|---------------|--------|
| `[cb_items]` | `View\Item` | Item list with booking link |
| `[cb_locations]` | `View\Location` | Location list with map |
| `[cb_bookings]` | `View\Booking` | User booking list |
| `[cb_items_table]` | `View\Calendar` | Calendar grid |
| `[cb_map]` | `MapShortcode` | Leaflet map |
| `[cb_search]` | `SearchShortcode` | Search component |

**➡️ Re-Implementation:**

```typescript
// Template Engine (like Twig or Handlebars)
// Shortcodes become React/Vue components or template includes

// Option 1: Template tags (PHP-like)
{% cb_items category="bikes" limit="10" %}

{% cb_locations show_map="true" %}

{% cb_bookings user_id=current_user %}

{% cb_items_table item_id="123" start_date="today" %}

// Option 2: React Components (for SPA)
<ItemList category="bikes" limit={10} />
<LocationMap showMap={true} />
<BookingCalendar itemId="123" />
```

---

### 3.7 Cron Jobs

**Current WP Cron Usage:**
| Hook | Schedule | Purpose |
|------|----------|---------|
| `commonsbooking_send_booking_reminder` | hourly | Send reminders |
| `commonsbooking_send_booking_codes` | twicedaily | Email booking codes |
| `commonsbooking_cron_export` | as-needed | CSV export |
| `commonsbooking_cache_clear` | as-needed | Cache invalidation |

**Functions Used:**
- `wp_schedule_event()`
- `wp_clear_scheduled_hook()`
- `wp_next_scheduled()`
- `wp_schedule_single_event()`

**➡️ Re-Implementation:**

```typescript
// BullMQ (Redis-backed job queue)
// or node-cron for simpler setups

import Bull from 'bull';

// Booking reminder job
const reminderQueue = new Bull('booking-reminder', {
  redis: { host: 'redis', port: 6379 }
});

reminderQueue.process(async (job) => {
  const { bookingId } = job.data;
  await sendBookingReminder(bookingId);
});

// Schedule: every hour
reminderQueue.add(
  {},
  { repeat: { cron: '0 * * * *' } }
);

// CSV Export job
const exportQueue = new Bull('timeframe-export', { redis });

exportQueue.process(async (job) => {
  const { exportPath, filters } = job.data;
  await generateCsvExport(exportPath, filters);
});
```

---

### 3.8 Frontend Assets

**Scripts Registered (17 handles):**
| Handle | Library | Purpose |
|--------|---------|---------|
| `cb-spin` | spin.js | Loading spinner |
| `cb-leaflet` | Leaflet.js | Maps |
| `cb-leaflet-markercluster` | Leaflet clustering | Map clustering |
| `cb-scripts-select2` | Select2 | Dropdown enhancement |
| `cb-scripts-moment` | Moment.js | Date handling |
| `cb-vue` | Vue.js | Reactive UI |
| `cb-commons-search` | Custom | Search component |

**Styles Registered (7 handles):**
| Handle | Library | Purpose |
|--------|---------|---------|
| `cb-leaflet` | Leaflet CSS | Map styling |
| `cb-styles-select2` | Select2 CSS | Form styling |

**➡️ Re-Implementation:**

```typescript
// Vite or Webpack build system

// package.json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "vue": "^3.4",
    "leaflet": "^1.9",
    "leaflet.markercluster": "^1.5",
    "moment": "^2.29",
    "@vueuse/core": "^10.0"
  }
}

// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  plugins: [vue()],
  build: {
    rollupOptions: {
      input: {
        frontend: './src/frontend/main.ts',
        admin: './src/admin/main.ts'
      }
    }
  }
});
```

---

### 3.9 Admin UI (CMB2)

**CMB2 Usage (159 matches):**
- All CPT classes use `new_cmb2_box()` for meta boxes
- Custom field renderers for Holiday selection, Booking codes
- Options pages via CMB2

**CMB2 Features Used:**
- Text fields, Date picker, Time picker
- Select2 AJAX search (custom field type)
- File/image upload
- Repeater groups
- Conditional fields

**➡️ Re-Implementation:**

```typescript
// React Admin Panel with react-hook-form + Zod

// Item Edit Form
const ItemForm: React.FC = () => {
  const { register, handleSubmit } = useForm<ItemData>({
    resolver: zodResolver(itemSchema)
  });

  return (
    <AdminLayout>
      <Form onSubmit={handleSubmit}>
        <Input label="Name" {...register('name')} />
        <Textarea label="Description" {...register('description')} />
        <CategorySelect {...register('categoryId')} />
        <ImageUpload label="Image" {...register('image')} />

        {/* Timeframes Section */}
        <TimeframeEditor />

        {/* Custom Meta Fields */}
        <MetaFieldArray />
      </Form>
    </AdminLayout>
  );
};

// Zod Schema
const itemSchema = z.object({
  name: z.string().min(1).max(255),
  description: z.string().optional(),
  categoryId: z.string().uuid(),
  image: z.string().url().optional()
});
```

---

### 3.10 Settings/Options

**Current Options Structure:**
| Option Group | Keys |
|--------------|------|
| `commonsbooking_options_general` | api-activated, apikey_not_required, etc. |
| `commonsbooking_options_bookingcodes` | advance_generation_days, email_enabled, etc. |
| `commonsbooking_options_timeframe_export` | export_path, etc. |
| `commonsbooking_options_cache` | cache_adapter, redis_dsn, etc. |
| `commonsbooking_options_map` | map_tiles_url, etc. |

**➡️ Re-Implementation:**

```typescript
// Database-backed settings with Admin UI

// settings table
interface Setting {
  key: string;
  value: JSON;
  type: 'string' | 'number' | 'boolean' | 'json';
  group: string;
  description: string;
  updatedAt: Timestamp;
}

// Admin API
@Patch('/admin/settings/:group')
@Auth(Roles.Admin)
async updateSettings(
  @Param('group') group: string,
  @Body() settings: Record<string, any>
) {
  await this.settingsService.updateGroup(group, settings);
  await this.cacheService.invalidate(`settings:${group}`);
}
```

---

## 4. Re-Implementation Priority Matrix

### Phase 1: Core Infrastructure (Must Have)

| Component | Effort | Complexity | Priority |
|-----------|--------|------------|----------|
| Database Schema | Medium | Low | P0 |
| ORM / Query Layer | Medium | Medium | P0 |
| Authentication (Keycloak) | Low | Medium | P0 |
| REST API Framework | Low | Medium | P0 |
| Configuration/Settings | Low | Low | P0 |

### Phase 2: Business Logic

| Component | Effort | Complexity | Priority |
|-----------|--------|------------|----------|
| Booking Service | High | High | P1 |
| Availability/Calendar Service | High | High | P1 |
| Timeframe Service | Medium | Medium | P1 |
| Booking Rules Engine | Medium | High | P1 |
| Message/Email Service | Medium | Medium | P1 |
| Cache Service | Low | Low | P2 |

### Phase 3: Frontend

| Component | Effort | Complexity | Priority |
|-----------|--------|------------|----------|
| Public Frontend (Booking Flow) | High | High | P1 |
| Admin Panel | High | High | P1 |
| Map Integration (Leaflet) | Medium | Medium | P2 |
| iCal Export | Low | Low | P2 |
| CSV Export | Low | Low | P2 |

### Phase 4: Integrations

| Component | Effort | Complexity | Priority |
|-----------|--------|------------|----------|
| GBFS Endpoints | Low | Medium | P2 |
| Push Notifications | Medium | Medium | P2 |
| Cron Jobs / Background Tasks | Medium | Medium | P2 |
| API Key Management | Low | Low | P3 |

---

## 5. Component Migration Map

| WordPress Component | Greenfield Alternative | Notes |
|--------------------|----------------------|-------|
| `register_post_type()` | PostgreSQL tables + Prisma | Full control over schema |
| `register_taxonomy()` | Categories table + Many-to-many | Simpler implementation |
| `register_post_status()` | Enum fields | 'unconfirmed', 'confirmed', 'canceled' |
| `$wpdb` | Prisma/Drizzle ORM | Type-safe queries |
| `WP_REST_Controller` | Express/FastAPI controllers | Better testing |
| `add_shortcode()` | Twig/Handlebars templates | Component-based |
| `wp_schedule_event()` | BullMQ | Redis-backed reliability |
| `CMB2` | React Admin + react-hook-form | Better UX |
| `get_option()` | Database + SettingsService | Cacheable |
| `transients` | Redis cache | Faster than DB transients |
| `wp_enqueue_scripts` | Vite build | Code splitting, HMR |

---

## 6. Keycloak Integration Analysis

### 6.1 Can Keycloak Replace WordPress User System?

**Ja.** Keycloak kann das WordPress User-System vollständig ersetzen.

### 6.2 Keycloak Features

| Feature | Status | CommonsBooking Use Case |
|---------|--------|------------------------|
| OAuth 2.0 | ✅ Full Support | API authentication |
| OpenID Connect (OIDC) | ✅ Full Support | User authentication |
| JWT Tokens | ✅ Full Support | Stateless API auth |
| User Registration | ✅ Built-in | Public sign-up |
| Password Reset | ✅ Built-in | Email-based flow |
| Role-Based Access Control | ✅ Full Support | `cb_manager`, `admin` roles |
| LDAP Federation | ✅ Supported | Existing directory integration |
| Social Login | ✅ Google, GitHub, etc. | Optional convenience |
| Multi-Factor Auth | ✅ TOTP, WebAuthn | Security option |
| Session Management | ✅ Built-in | SSO across apps |
| Admin Console | ✅ Web UI | User/Role management |

### 6.3 Keycloak Architecture for CommonsBooking

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile App / Web App                    │
│                    (JWT in Authorization header)            │
└────────────────────────────┬────────────────────────────────┘
                             │ Authorization: Bearer <JWT>
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      CommonsBooking API                     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  JWT Validation Middleware                           │   │
│  │  - Verify signature against Keycloak JWKS            │   │
│  │  - Check token expiration                            │   │
│  │  - Extract roles from token claims                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                             │                               │
│  ┌──────────────────────────┴──────────────────────────┐   │
│  │  Role Mapping                                        │   │
│  │  keycloak realm roles → CB roles (admin, manager)   │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                             │ Token introspection (optional)
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                     Keycloak Server                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │  Users  │  │  Roles  │  │ Clients │  │ Realms  │       │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
│                                                             │
│  Realm: commonsbooking                                      │
│  Clients: mobile-app, admin-panel, api-service              │
└─────────────────────────────────────────────────────────────┘
```

### 6.4 Keycloak Client Configuration

```yaml
# Keycloak Realm: commonsbooking

clients:
  - clientId: commonsbooking-api
    protocol: openid-connect
    publicClient: false
    secret: ${API_CLIENT_SECRET}
    directAccessGrantsEnabled: true
    serviceAccountsEnabled: true
    authorizationServicesEnabled: true

  - clientId: commonsbooking-mobile
    protocol: openid-connect
    publicClient: true
    standardFlowEnabled: true      # Browser flow
    implicitFlowEnabled: false
    directAccessGrantsEnabled: true
    pushtoken_refresh_token_lifespan: 2592000  # 30 days

  - clientId: commonsbooking-admin
    protocol: openid-connect
    publicClient: true
    standardFlowEnabled: true
    directAccessGrantsEnabled: false  # Admin uses browser flow

# Realm Roles
roles:
  - name: admin
    description: Full system access

  - name: manager
    description: Can manage items/locations/bookings

  - name: user
    description: Can book items

# Role Mappings (Keycloak → CommonsBooking)
role_mappings:
  admin: ['cb_admin', 'cb_manager', 'cb_user']
  manager: ['cb_manager', 'cb_user']
  user: ['cb_user']
```

### 6.5 API JWT Validation (Node.js Example)

```typescript
import { importJWK, jwtVerify } from 'jose';

class KeycloakAuth {
  private jwks: Map<string, CryptoKey> = new Map();
  private keycloakUrl = process.env.KEYCLOAK_URL;

  async validateToken(token: string): Promise<JWTPayload> {
    const { header, payload } = decodeJWT(token);

    // Get or cache JWKS
    let publicKey = this.jwks.get(header.kid);
    if (!publicKey) {
      publicKey = await this.fetchPublicKey(header.kid);
      this.jwks.set(header.kid, publicKey);
    }

    // Verify signature
    const { payload: verified } = await jwtVerify(token, publicKey, {
      issuer: `${this.keycloakUrl}/realms/commonsbooking`,
      audience: 'commonsbooking-api'
    });

    return verified;
  }

  extractRoles(payload: JWTPayload): string[] {
    const realmAccess = payload.realm_access as { roles: string[] };
    return realmAccess?.roles || [];
  }

  hasRole(payload: JWTPayload, role: string): boolean {
    return this.extractRoles(payload).includes(role);
  }
}

// Middleware
const requireRole = (role: string) => async (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'No token' });

  const auth = new KeycloakAuth();
  const payload = await auth.validateToken(token);

  if (!auth.hasRole(payload, role)) {
    return res.status(403).json({ error: 'Insufficient permissions' });
  }

  req.user = payload;
  next();
};

// Routes
app.get('/admin/bookings', requireRole('cb_manager'), getBookings);
app.post('/bookings', requireRole('cb_user'), createBooking);
```

### 6.6 Keycloak vs WordPress User Comparison

| Aspect | WordPress | Keycloak | Winner |
|--------|-----------|----------|--------|
| Setup Effort | Low (bundled with WP) | Medium (separate server) | WordPress |
| User Management | Good | Excellent (Admin Console) | Keycloak |
| OAuth/OIDC | Via plugins | Native | Keycloak |
| JWT Support | Via plugins | Native | Keycloak |
| Multi-Factor Auth | Via plugins | Native | Keycloak |
| LDAP Integration | Via plugins | Native | Keycloak |
| SSO | Limited | Full | Keycloak |
| User Registration Flow | Basic | Fully customizable | Keycloak |
| Password Policies | Basic | Granular | Keycloak |
| Audit Logging | Basic | Comprehensive | Keycloak |
| Federation | Limited | Full | Keycloak |

### 6.7 Keycloak Deployment Options

| Option | Use Case | Effort |
|--------|----------|--------|
| **Keycloak Cloud (Red Hat)** | Production, managed | Low |
| **Self-Hosted Docker** | Single tenant, full control | Medium |
| **Kubernetes** | Multi-tenant, auto-scaling | High |
| **Keycloakify (SSOasaas)** | Quick start, hosted | Low |

### 6.8 Migration Path: WordPress → Keycloak

```
Phase 1: Parallel Setup
├── Install Keycloak
├── Create realm "commonsbooking"
├── Import existing WordPress users (script)
└── Configure client "commonsbooking-api"

Phase 2: API Changes
├── Add JWT validation middleware
├── Support both WP sessions AND JWT
└── Add token refresh endpoint

Phase 3: User Migration
├── Export WordPress users
├── Import to Keycloak (bcrypt → argon2)
├── Notify users to set new password
└── Disable WordPress auth for API

Phase 4: Full Cutover
├── Remove WordPress user dependencies
├── Enable Keycloak-only auth
└── Decommission WordPress user system
```

---

## 7. External Libraries Summary

### 7.1 WordPress-Dependent Libraries

| Library | Used In | Replacement |
|---------|---------|-------------|
| **CMB2** | All CPT classes, Settings | React Admin + Form libraries |
| **PHPMailer** | Messages | Nodemailer (Node.js) |
| **eluceo/iCal** | iCalendar Service | ical.js (JavaScript) or python-dateutil |
| **Symfony Cache** | Cache Service | Redis/Memory cache |
| **Opis JSON Schema** | API validation | Ajv (JS) or Pydantic (Python) |
| **Geocoder-php/Nominatim** | GeoHelper | nominatim-api (JS) or geocoding-py |
| **scssphp** | View (SCSS compile) | Sass (Dart) or PostCSS |
| **mustardbees/cmb-field-select2** | Admin UI | React Select |
| **ed-itsolutions/cmb2-field-ajax-search** | Admin UI | React AsyncSelect |

### 7.2 WP-Independent Libraries (Direct Replacements)

| Original | Replacement | Notes |
|----------|-------------|-------|
| Leaflet.js | Leaflet.js | Same library, bundled via npm |
| Moment.js | date-fns or dayjs | Lighter, tree-shakeable |
| Vue.js | Vue 3 | Same framework |
| Select2 | React Select or Choices.js | Same concept, better integration |
| MarkerCluster | Leaflet.markercluster | Same library |

---

## 8. Summary: What Must Be Reimplemented

### Completely New (No WordPress Equivalent)

1. **Custom Post Type System**
   - PostgreSQL schema for 6 entities
   - ORM layer (Prisma/Drizzle)
   - CRUD API

2. **Plugin Hook System**
   - Event bus implementation
   - Database-migration compatible callbacks

3. **Shortcode System**
   - Template engine (Twig/Handlebars)
   - Component library

4. **Admin UI (CMB2 replacement)**
   - React Admin Panel
   - Form handling (react-hook-form + Zod)
   - Custom meta field types

5. **Settings/Options UI**
   - Admin panel settings pages
   - Database-backed configuration

6. **Cron System**
   - BullMQ or similar job queue
   - Scheduled job management

7. **Frontend Asset Pipeline**
   - Vite build system
   - Component library

### Replace with Standard Alternative

1. **User Authentication** → Keycloak (OAuth 2.0 / OIDC)
2. **Email Sending** → Nodemailer + SMTP/SendGrid
3. **iCal Generation** → ical.js
4. **Cache** → Redis (already planned)
5. **Geocoding** → OpenStreetMap Nominatim API

### Minor Adjustments Needed

1. **GBFS Endpoints** → Keep same spec, implement in new language
2. **Template System** → Adapt existing logic to new template engine
3. **Map Display** → Same Leaflet, different bundler

---

## 9. Recommended Tech Stack (Greenfield)

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Backend** | Node.js + Express oder Go + Fiber | Async support, GBFS easy |
| **Database** | PostgreSQL + Prisma | Type-safe ORM, JSONB support |
| **Cache** | Redis + ioredis | Fast, BullMQ dependency |
| **Job Queue** | BullMQ | Reliable, Redis-backed |
| **Auth** | Keycloak | Full OIDC, RBAC, JWT |
| **Frontend (Public)** | Vue 3 + Vite | Lightweight, SSR possible |
| **Frontend (Admin)** | React + Vite | Component ecosystem |
| **Maps** | Leaflet + OpenStreetMap | Free, no API key needed |
| **Validation** | Zod (TS) oder Pydantic (Python) | Schema-first |
| **iCal** | ical.js | Pure JS implementation |
| **Email** | Nodemailer + SMTP | Flexible delivery |
| **Geocoding** | Nominatim API | Free, no key for light use |

---

*Document Version: 1.0*
*Analysis Date: 2026-05-13*
*Based on: CommonsBooking WordPress Plugin v2.10.10*