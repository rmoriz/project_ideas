# CommonsBooking - Product Requirements Document

## Version: 1.0 (Greenfield Specification)
### Basierend auf CommonsBooking WordPress Plugin v2.10.10

---

## 1. Overview & Vision

### Project Name
**CommonsBooking** - Ein Buchungssystem für gemeinschaftlich nutzbare Gegenstände

### Core Value Proposition
Eine Plattform, die es Organisationen ermöglicht, Gegenstände (Fahrräder, Werkzeuge, Sportgeräte etc.) zur gemeinsamen Nutzung anzubieten. Nutzer können verfügbare Gegenstände an verschiedenen Standorten einsehen, buchen und ihre Buchungen verwalten.

### Zielgruppe
- **Organisationen**: Gemeinschaften, Vereine, Stadtteilinitiativen, die gemeinsam genutzte Ressourcen anbieten möchten
- **Endnutzer**: Personen, die Gegenstände ausleihen möchten
- **Administratoren**: Manager, die das System konfigurieren und Buchungen verwalten

---

## 2. Domain Model (Kernkonzepte)

### 2.1 Custom Post Types (CPTs)

Das System basiert auf 5 WordPress Custom Post Types:

| CPT | Slug | Beschreibung |
|-----|------|--------------|
| **Location** | `cb_location` | Physische Standorte, an denen Gegenstände abgeholt/zurückgegeben werden |
| **Item** | `cb_item` | Buchbare Gegenstände (Fahrräder, Werkzeuge, etc.) |
| **Timeframe** | `cb_timeframe` | Zeitfenster, die Verfügbarkeit definieren |
| **Booking** | `cb_booking` | Buchungsanfragen und -bestätigungen |
| **Restriction** | `cb_restriction` | Blockierungen (Reparaturen, Feiertage, Ferien) |

### 2.2 Entitäten und Attribute

#### Location (Standort)
```
- ID (UUID)
- Name
- Beschreibung
- Adresse (Strasse, PLZ, Stadt)
- Koordinaten (latitude, longitude) - Geocodiert via OpenStreetMap Nominatim
- Abhol-/Rückgabeanweisungen ( pickupInstructions)
- Öffnungszeiten (verwaltet via Timeframes)
- Ansprechpartner (Author/User-Relation)
- Bild(er)
- Karte (Map) - Konfiguration für Visualisierung
- API-Share Konfigurationen (für GBFS Push)
```

#### Item (Gegenstand)
```
- ID (UUID)
- Name
- Beschreibung
- Kategorie (Taxonomie)
- Zuständige Manager (User-Relation)
- Bilder (thumbnail, medium, large, full)
- Standort-Zuordnung (via Timeframes)
- Custom Map Konfiguration
```

#### Timeframe (Zeitfenster)
```
- ID (UUID)
- Typ (enum - IDs aus Timeframe.php):
  - 1: OPENING_HOURS (deprecated, nicht implementiert)
  - 2: BOOKABLE - Buchbares Zeitfenster
  - 3: HOLIDAYS - Ferien/Schließung
  - 4: OFF_HOLIDAYS - Gesetzliche Feiertage (disabled)
  - 5: REPAIR - Reparatur/Blockierung (nicht überbuchbar)
  - 6: BOOKING - Gebuchte Zeit
  - 7: BOOKING_CANCELED - Stornierte Buchung (disabled)
- Aktiv implementiert in getTypes(): BOOKABLE, HOLIDAYS, REPAIR, BOOKING
- Start-Datum/Zeit
- End-Datum/Zeit
- Grid (Zeitraster in Minuten, Standard: 60)
- Related Location
- Related Item
- Modus (full-day oder time-based)
- Repetetion (wiederholend):
  - Keine, Wöchentlich (Wochentage), Monatlich (Wochentag)
- Meta:
  - advanceBookingDays (Tage im Voraus buchbar, Standard: 31)
  - minimumBookingDays, maximumBookingDays
  - custom grid slots
  - holiday_year, holiday_state (für deutsche Feiertage)
```

#### Booking (Buchung)
```
- ID (UUID)
- Status (Custom Post Status):
  - "unconfirmed" - Initialstatus, wartet auf Bestätigung
  - "confirmed" - Bestätigte Buchung
  - "canceled" - Stornierte Buchung
- Zeitraum (startDate, endDate)
- Location (via Timeframe)
- Item (via Timeframe)
- User (Bucher)
- BookedAt (Timestamp der Buchung)
- ConfirmedAt (optional)
- Booking Code (generierter 8-stelliger Code)
- Typ (daily/time-based)
```

#### Restriction (Einschränkung)
```
- ID (UUID)
- Name
- Typ (aus getTypes()):
  - "repair" - Total breakdown (macht Item/Location unavailable)
  - "hint" - Notice (Information, blockiert nicht)
- Status (aus getStates()):
  - "none" - Not active
  - "active" - Active (wird angewendet)
  - "solved" - Problem solved
- Start-Datum
- End-Datum
- Related Item(s) oder Location(s)
- Hinweis: Deutsche Feiertage für Ferien werden NICHT hier konfiguriert,
  sondern im Timeframe (via Holiday-Service) für spezifische Bundesländer
```

---

## 3. Booking System Logic

### 3.1 Booking Workflow

```
[Nutzer sucht Item] → [Wählt Zeitraum] → [System prüft Verfügbarkeit]
       ↓
[Slot verfügbar?] → [Ja] → [Nutzer gibt Daten ein] → [Buchung erstellt (unconfirmed)]
       ↓ Nein
[Slot gesperrt] → [Keine Buchung möglich]

[Admin sieht unconfirmed] → [Admin bestätigt] → [Status: confirmed]
       ↓
[Nutzer erhält Bestätigung + Code per E-Mail]
       ↓
[Nutzer holt Item ab] → [Admin bestätigt Abholung]
       ↓
[Nutzer gibt Item zurück] → [Booking: canceled/closed]
```

### 3.2 Booking States

| Status | Beschreibung | Erlaubte Übergänge |
|--------|--------------|-------------------|
| `unconfirmed` | Neue Buchung, wartet auf Bestätigung | → `confirmed`, → `canceled` |
| `confirmed` | Bestätigte aktive Buchung | → `canceled` |
| `canceled` | Stornierte/abgeschlossene Buchung | (Endzustand) |

### 3.3 Booking Rules (Geschäftsregeln)

Das System unterstützt konfigurierbare Buchungsregeln:

```
Regel-Definition:
- Name
- Berechnung ( calculation):
  - MIN_DAYS_BEFORE_BOOKING: Mindestvorlaufzeit
  - MAX_DAYS_BEFORE_BOOKING: Maximalvorlaufzeit
  - MIN_BOOKING_LENGTH: Mindestdauer
  - MAX_BOOKING_LENGTH: Maximaldauer
  - ONLY_WEEKDAYS: Nur Werktage
  - NO_SPLIT_BOOKING: Keine Unterbrechung durch Ferien
- Operator (AND/OR)
- Parameter Values
- Applicable Categories
- Excluded User Roles
- Priority

Gültigkeitsprüfung:
- Item/Location-Admin: Immer erlaubt (bypass)
- Excluded Roles: Regeln werden ignoriert
- Categories: Regel gilt nur für Items in diesen Kategorien
```

### 3.4 Timeframe Grid System

Timeframes definieren ein Zeitraster (Grid):

```
Grid-Konfiguration:
- gridDuration: Minuten pro Slot (Standard: 60)
- slots: Array von start/end Timestamps
- mode: "full-day" | "time-based"

Verarbeitung:
- Day aggregiert Timeframe-Slots
- Week aggregiert Days
- Calendar kompiliert Weeks + Restrictions + Holidays
```

---

## 4. API Specification

### 4.1 REST API Endpoints

**Base URL:** `/commonsbooking/v1/`
**Authentication:** API Key (X-Api-Key header) oder anonymous wenn aktiviert
**Schema:** JSON Schema Validation via Opis

#### Items Endpoint
```
GET /items
GET /items/{id}
GET /items/schema

Response:
{
  "items": [
    {
      "id": "string",
      "name": "string",
      "url": "string",
      "description": "string",
      "ownerId": "string",
      "projectId": "string",
      "image": "string",
      "images": {
        "thumbnail": [],
        "medium": [],
        "large": [],
        "full": []
      }
    }
  ],
  "projects": {...},
  "locations": {...},
  "availability": [...]
}
```

#### Locations Endpoint (GeoJSON)
```
GET /locations
GET /locations/{id}

Response:
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": {
        "id": "string",
        "name": "string",
        "description": "string",
        "url": "string",
        "address": "string",
        "pickupInstructions": "string"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [longitude, latitude]
      }
    }
  ]
}
```

#### Availability Endpoint
```
GET /availability
GET /availability?itemId={id}
GET /availability?locationId={id}

Query Parameters:
- start: ISO-8601 date (default: today)
- end: ISO-8601 date (default: +2 weeks)

Response:
{
  "availability": [
    {
      "start": "ISO-8601",
      "end": "ISO-8601",
      "itemId": int,
      "locationId": int
    }
  ]
}
```

#### Projects Endpoint
```
GET /projects

Response:
{
  "projects": [
    {
      "id": "1",
      "name": "string (Site Title)",
      "url": "string",
      "description": "string (Tagline)",
      "language": "string"
    }
  ]
}
```

### 4.2 GBFS (General Bikeshare Feed Specification)

Implementiert nach GBFS Version 3.1-RC2 für Fahrrad-Sharing-Integration.

| Endpoint | Beschreibung |
|----------|--------------|
| `/station_status.json` | Verfügbare Fahrzeuge pro Station |
| `/station_information.json` | Stationsdetails mit GeoJSON |
| `/system_information.json` | Systemweite Informationen |
| `/vehicle_status.json` | Fahrzeugstatus |
| `/vehicle_availability.json` | Verfügbarkeitszeiten für Fahrzeuge |
| `/discovery.json` | Feed-Discovery (Auto-Discovery) |

**Response Format:**
```json
{
  "data": { ... },
  "last_updated": "ISO-8601",
  "ttl": 60,
  "version": "3.1-RC2"
}
```

### 4.3 Extended API (Mobile App Support)

Die bestehende API ist **read-only**. Für eine Mobile App (iOS/Android) müssen folgende Write-Endpoints ergänzt werden:

#### Authentication Endpoints

```
POST /auth/register
POST /auth/login
POST /auth/logout
POST /auth/refresh
GET  /auth/me

Request (Register):
{
  "email": "string",
  "password": "string",
  "name": "string"
}

Request (Login):
{
  "email": "string",
  "password": "string"
}

Response:
{
  "token": "JWT",
  "refreshToken": "string",
  "user": {
    "id": "int",
    "email": "string",
    "name": "string",
    "roles": ["subscriber"]
  }
}
```

#### Booking Endpoints (CRUD)

```
POST   /bookings              # Neue Buchung erstellen
GET    /bookings              # Eigene Buchungen (auth required)
GET    /bookings/{id}         # Einzelne Buchung
PUT    /bookings/{id}         # Buchung aktualisieren
DELETE /bookings/{id}         # Buchung stornieren

Request (Create):
{
  "itemId": int,
  "locationId": int,
  "startDate": "Y-m-d",
  "endDate": "Y-m-d",
  "userName": "string",
  "userEmail": "string",
  "userPhone": "string (optional)",
  "notes": "string (optional)"
}

Response (Create):
{
  "id": int,
  "status": "unconfirmed",
  "item": {...},
  "location": {...},
  "startDate": "Y-m-d",
  "endDate": "Y-m-d",
  "bookedAt": "ISO-8601",
  "message": "string"
}

Request (Update - Admin only):
{
  "status": "confirmed" | "canceled",
  "bookingCode": "string (optional)"
}
```

#### Booking Codes Endpoint

```
GET /booking-codes?timeframe={id}&item={id}&location={id}&date={Y-m-d}

Response:
{
  "codes": [
    {
      "date": "Y-m-d",
      "code": "ABC12345",
      "timeframeId": int,
      "itemId": int,
      "locationId": int
    }
  ]
}
```

#### User Profile Endpoints

```
GET  /users/me
PUT  /users/me
GET  /users/me/bookings

Request (Update):
{
  "name": "string",
  "phone": "string",
  "notificationEmail": true | false,
  "notificationSms": true | false
}
```

#### Availability Check (Enhanced)

```
POST /availability/check

Request:
{
  "itemId": int,
  "locationId": int,
  "startDate": "Y-m-d",
  "endDate": "Y-m-d"
}

Response:
{
  "available": true | false,
  "slots": [
    {
      "date": "Y-m-d",
      "available": true | false,
      "reason": "string (wenn nicht verfügbar)"
    }
  ],
  "bookingRules": {
    "minAdvanceDays": int,
    "maxAdvanceDays": int,
    "minDuration": int,
    "maxDuration": int,
    "allowedDays": ["Mon","Tue",...]
  }
}
```

#### Admin Endpoints (Role: cb_manager, administrator)

```
GET    /admin/bookings                    # Alle Buchungen
GET    /admin/bookings?status=unconfirmed # Nach Status filtern
PUT    /admin/bookings/{id}/confirm       # Buchung bestätigen
PUT    /admin/bookings/{id}/cancel        # Buchung stornieren
GET    /admin/timeframes                  # Timeframes verwalten
POST   /admin/timeframes                  # Timeframe erstellen
PUT    /admin/timeframes/{id}             # Timeframe bearbeiten
DELETE /admin/timeframes/{id}             # Timeframe löschen
GET    /admin/items                       # Items verwalten
POST   /admin/items                       # Item erstellen
PUT    /admin/items/{id}                  # Item bearbeiten
GET    /admin/locations                   # Locations verwalten
POST   /admin/locations                   # Location erstellen
PUT    /admin/locations/{id}              # Location bearbeiten
```

### 4.4 Mobile App Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile App (iOS/Android)                │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │  Auth   │  │ Browse  │  │  Book   │  │ Profile │       │
│  │ Screen  │  │  Items  │  │  Flow   │  │  View   │       │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘       │
│       │            │            │            │             │
│  ┌────┴────────────┴────────────┴────────────┴────┐       │
│  │              API Client / Repository            │       │
│  │         (JWT Token Management, Caching)         │       │
│  └──────────────────────┬──────────────────────────┘       │
└─────────────────────────┼───────────────────────────────────┘
                          │ HTTPS + JWT
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Backend API                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  REST API Server                                     │   │
│  │  - Node.js / Express oder Go / Fiber                 │   │
│  │  - JWT Authentication                                │   │
│  │  - Schema Validation (Opis JSON Schema)              │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────┴──────────────────────────────┐   │
│  │  Business Logic Layer                                │   │
│  │  - Booking Service (Rules, Validation)               │   │
│  │  - Availability Service (Calendar, Grid)             │   │
│  │  - Notification Service (Email, Push)                │   │
│  │  - Export Service (CSV, iCal)                        │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────┴──────────────────────────────┐   │
│  │  Data Layer                                          │   │
│  │  - PostgreSQL / MySQL                                │   │
│  │  - Redis (Cache, Sessions)                           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 4.5 Push Notifications (Mobile)

```
Endpoint: POST /push/register

Request:
{
  "userId": int,
  "token": "string (FCM/APNS)",
  "platform": "ios" | "android"
}

Trigger Events:
- Booking confirmed
- Booking reminder (24h before)
- Booking canceled by admin
- Item availability restored
```

---

## 5. Message System (E-Mail-Benachrichtigungen)

### 5.1 Message Types

| Type | Trigger | Empfänger |
|------|---------|-----------|
| **BookingMessage** | Buchung erstellt/bestätigt | Nutzer |
| **BookingReminderMessage** | X Tage vor Buchung | Nutzer |
| **LocationBookingReminderMessage** | Buchung anstehende Abholung | Location-Admin |
| **BookingCodesMessage** | Periodische Code-Übersicht | Nutzer |
| **RestrictionMessage** | Sperrung eingetreten | Admin |
| **AdminMessage** | Administrative Benachrichtigungen | Admin |

### 5.2 Message Template System

```
Template-Engine: commonsbooking_parse_template
- Variablen: {{var_name}} Syntax
- Verfügbare Variablen pro Message-Typ unterschiedlich

Standard-Variablen:
- {{user_name}}, {{user_email}}
- {{item_name}}, {{item_id}}
- {{location_name}}, {{location_address}}
- {{booking_start}}, {{booking_end}}
- {{booking_code}}
- {{site_name}}, {{site_url}}
```

---

## 6. Frontend-Funktionen

### 6.1 Shortcodes

| Shortcode | Beschreibung |
|-----------|--------------|
| `[cb_items]` | Zeigt buchbare Items |
| `[cb_locations]` | Zeigt Standorte |
| `[cb_bookings]` | Zeigt Nutzer-Buchungen (login required) |
| `[cb_items_table]` | Kalender-Tabelle mit Verfügbarkeit |
| `[commonsbooking_map]` | Karte mit Standorten |

### 6.2 Kalender-System (Litepicker-Integration)

**Frontend-JSON-Response:**
```json
{
  "minDate": "Y-m-d",
  "startDate": "Y-m-d",
  "endDate": "Y-m-d",
  "days": {
    "Y-m-d": {
      "date": "d.m.Y",
      "slots": [...],
      "locked": boolean,
      "bookedDay": boolean,
      "partiallyBookedDay": boolean,
      "holiday": boolean,
      "repair": boolean,
      "fullDay": boolean,
      "firstSlotBooked": boolean,
      "lastSlotBooked": boolean
    }
  },
  "bookedDays": ["Y-m-d"],
  "lockDays": ["Y-m-d"],
  "holidays": ["Y-m-d"],
  "advanceBookingDays": int
}
```

### 6.3 User Widget

WordPress Widget für eingeloggte Nutzer:
- Willkommensnachricht
- Link zu "Meine Buchungen"
- Link zu "Mein Profil"
- Logout-Button
- Login/Registrierung-Link (für nicht eingeloggte)

---

## 7. Export & Sync

### 7.1 Timeframe Export (CSV)

```
Exported Data:
- Timeframes (ID, Typ, Start, Ende, Modus)
- Locations (konfigurierbare Felder)
- Items (konfigurierbare Felder)
- User-Felder (konfigurierbar)

Format:
- Semicolon-delimited
- Backslash-Escaping
- Pagination: 100 Items pro Batch
- Intermediate storage via Transient
```

### 7.2 iCalendar Export

```
Bibliothek: eluceo/iCal
- URL-Rewrite: /commonsbooking_ical_download
- Gesicherte Links via wp_hash(user_id)
- Formate:
  - SingleDay/MultiDay für ganztägige Buchungen
  - TimeSpan für zeitspezifische Slots
- CANCELED Status für stornierte Buchungen
```

### 7.3 API Push (Share)

```
Konfiguration:
- Push URL
- API Key
- Enabled/Disabled Flag

Trigger:
- Automatisch nach Datenänderungen
- Via API Helper
```

---

## 8. Settings & Konfiguration

### 8.1 Option Categories (tatsächlich verwendet)

| Category | Key | Beschreibung |
|----------|-----|--------------|
| General | `commonsbooking_options_general` | Grundeinstellungen, CPT-Slugs, Lock-Days |
| Booking Codes | `commonsbooking_options_bookingcodes` | Code-Generierung, E-Mail-Settings |
| Export | `commonsbooking_options_export` | CSV-Export Pfad, Interval, Felder |
| Reminder | `commonsbooking_options_reminder` | Erinnerungs-E-Mails, Zeiten, Aktivierung |
| API | `commonsbooking_options_api` | API-Aktivierung, Key-Optionen, Shares |
| Templates | `commonsbooking_options_templates` | Farben, E-Mail-Templates, Header/Footer |
| Restrictions | `commonsbooking_options_restrictions` | Booking Rules, Restriction-Benachrichtigungen |
| Migration | `commonsbooking_options_migration` | CB1-Migration, User-Fields |

### 8.2 Cache Configuration

```
Adapter-Optionen:
- filesystem: Default, sys_get_temp_dir/symfony_cache
- redis: Via REDIS_DSN
- disabled: Für Development/Testing

Konfiguration:
- commonsbooking_disableCache filter
- Deaktiviert bei WP_DEBUG
```

```
Adapter-Optionen:
- filesystem: Default, sys_get_temp_dir/symfony_cache
- redis: Via REDIS_DSN
- disabled: Für Development/Testing

Konfiguration:
- commonsbooking_disableCache filter
- Deaktiviert bei WP_DEBUG
```

---

## 9. Datenbank-Schema

> **Hinweis für Greenfield:** Die neue Implementierung verwendet SQLite als Default-Datenbank (für einfache Installation ohne externe Abhängigkeiten) mit optionalem PostgreSQL-Support für höhere Last und Production-Umgebungen. Das Schema unten ist das Original-WordPress-Schema.

### 9.1 Booking Codes Tabelle

```sql
CREATE TABLE cb_bookingcodes (
  date DATE NOT NULL,
  timeframe BIGINT UNSIGNED NOT NULL,
  location BIGINT UNSIGNED NOT NULL,
  item BIGINT UNSIGNED NOT NULL,
  code VARCHAR(100) NOT NULL,
  PRIMARY KEY (date, timeframe, location, item, code)
);
```

### 9.2 Meta-Felder (Post Meta)

| CPT | Meta-Key | Typ | Beschreibung |
|-----|----------|-----|--------------|
| Location | `_cb_location_lat` | float | Breitengrad |
| Location | `_cb_location_lng` | float | Längengrad |
| Location | `_cb_location_address` | string | Vollständige Adresse |
| Location | `_cb_location_pickup_instructions` | string | Abholinfo |
| Item | `_cb_item_category` | int | Kategorie-ID |
| Item | `_cb_item_template` | string | Template-Name |
| Booking | `_cb_booking_code` | string | 8-stelliger Code |
| Booking | `_cb_booking_user_id` | int | Bucher-ID |
| Timeframe | `_cb_timeframe_grid` | string | Grid-JSON |
| Timeframe | `_cb_timeframe_advance` | int | Vorlauf-Tage |
| Restriction | `_cb_restriction_type` | string | Typ |
| Restriction | `_cb_restriction_repeat` | string | Wiederholung |

---

## 10. User Roles & Permissions (Greenfield)

### 10.1 Rollenmodell

**Wichtig: Ein User kann mehrere Rollen gleichzeitig haben.**

Rollen sind nicht exklusiv - ein User hat typischerweise eine Kombination:
- Ein Station-Manager hat automatisch auch `team_member` und `user` Rechte
- Ein Team-Member hat `user` Rechte zusätzlich
- Die effektiven Berechtigungen ergeben sich aus der **Summe aller Rollen**

**Minimal Role vs. Additional Roles:**

| User | Minimal Role | Additional Roles (inherited) |
|------|--------------|------------------------------|
| Admin | `admin` | - |
| Station Manager | `station_manager` | `team_member`, `user` |
| Team Member | `team_member` | `user` |
| User | `user` | - |
| Public | `public` | - |

### 10.2 Role Hierarchy (Minimal Role)

```
┌─────────────────────────────────────────────────────────────┐
│                          ADMIN                              │
│  - Systemweit alle Berechtigungen                          │
│  - Organisationseinstellungen verwalten                     │
│  - Alle Stationen & Items einsehen                         │
│  - Benutzer-Management (Rollen zuweisen)                   │
│  - Globale Statistiken & Reports                            │
│  - API-Keys verwalten                                      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      STATION MANAGER                        │
│  - Eine oder mehrere Stationen verwalten                   │
│  - Ausleihe & Rücknahme bestätigen                         │
│  - Buchungen seiner Station bestätigen/stornieren          │
│  - Zeitfenster für seine Stationen erstellen               │
│  - Sperrungen/Reparaturen melden                           │
│  - Statistiken seiner Station einsehen                      │
│  - Eigene Team-Members verwalten                           │
│  + Erbt: team_member, user                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       TEAM MEMBER                           │
│  - Vereinsmitglied / Helfer                                │
│  - Station-Manager unterstützten                           │
│  - Ausleihe & Rücknahme durchführen                        │
│  - Eigene Buchungen verwalten                              │
│  + Erbt: user                                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                           USER                              │
│  - Angemeldeter Nutzer mit Konto                           │
│  - Items & Stationen einsehen                              │
│  - Buchungen erstellen & verwalten                         │
│  - Eigene Historie einsehen                                │
│  - Profileinstellungen ändern                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                          PUBLIC                             │
│  - Kein Konto erforderlich                                 │
│  - Stationen & Verfügbarkeit einsehen                      │
│  - Keine Buchung möglich (optional)                        │
│  - Standort auf Karte sehen                                │
│  - Kontaktdaten sehen (optional)                           │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 Role Definitions

#### PUBLIC (Öffentlich)
| Attribut | Wert |
|----------|------|
| **ID** | `public` |
| **Beschreibung** | Nicht-angemeldete Besucher |
| **Authentifizierung** | Keine |

**Berechtigungen (Read-only):**

| Resource | Create | Read | Update | Delete |
|----------|--------|------|--------|--------|
| Stations | ❌ | ✅ (öffentliche) | ❌ | ❌ |
| Items | ❌ | ✅ (veröffentlichte) | ❌ | ❌ |
| Availability | ❌ | ✅ | ❌ | ❌ |
| Booking | ❌ | ❌ | ❌ | ❌ |
| Profile | ❌ | ❌ | ❌ | ❌ |

---

#### USER (Angemeldeter Nutzer)
| Attribut | Wert |
|----------|------|
| **ID** | `user` |
| **Beschreibung** | Registrierter Nutzer mit eigenem Konto |
| **Authentifizierung** | Email + Passwort oder SSO (Keycloak) |

**Berechtigungen:**

| Resource | Create | Read | Update | Delete |
|----------|--------|------|--------|--------|
| Stations | ❌ | ✅ (alle) | ❌ | ❌ |
| Items | ❌ | ✅ (alle) | ❌ | ❌ |
| Availability | ❌ | ✅ | ❌ | ❌ |
| Booking | ✅ (eigene) | ✅ (eigene) | ✅ (stornieren) | ❌ |
| Profile | ❌ | ✅ (eigenes) | ✅ (eigenes) | ✅ (eigenes Konto) |

**Einschränkungen:**
- Kann nur eigene Buchungen erstellen
- Kann nur eigene Buchungen stornieren
- Kein Zugriff auf Admin-Funktionen
- Zeitfenster: nur wenn Rolle erlaubt

---

#### TEAM MEMBER (Vereinsmitglied)
| Attribut | Wert |
|----------|------|
| **ID** | `team_member` |
| **Beschreibung** | Aktives Vereinsmitglied, das bei Ausleihe hilft |
| **Authentifizierung** | Keycloak + MFA empfohlen |
| **Typ** | Organization-assigned |

**Berechtigungen:**

| Resource | Create | Read | Update | Delete |
|----------|--------|------|--------|--------|
| Stations | ❌ | ✅ (zugewiesene) | ❌ | ❌ |
| Items | ❌ | ✅ (zugewiesene Stationen) | ❌ | ❌ |
| Availability | ❌ | ✅ | ❌ | ❌ |
| Booking | ❌ | ✅ (Station-Buchungen) | ✅ (Bestätigen/Rücknahme) | ❌ |
| Check-in/out | ❌ | ❌ | ✅ | ❌ |

**Erweiterte Rechte:**

```typescript
// Team Member kann:
// - Ausleihe bestätigen (Booking: confirmed)
// - Rücknahme bestätigen (Booking: canceled/returned)
// - Buchungen einsehen die seine Station betreffen
// - Keine Zeitfenster bearbeiten
// - Keine anderen Stationen sehen
```

---

#### STATION MANAGER (Verleih-Mitarbeiter)
| Attribut | Wert |
|----------|------|
| **ID** | `station_manager` |
| **Beschreibung** | Mitarbeiter einer Station (z.B. Fahrradladen, Vereins-Büro) |
| **Authentifizierung** | Keycloak + MFA |
| **Typ** | Station-assigned |
| **Scope** | Eine oder mehrere Stationen |

**Berechtigungen:**

| Resource | Create | Read | Update | Delete |
|----------|--------|------|--------|--------|
| Stations | ❌ | ✅ (zugewiesene) | ✅ (Meta, Öffnungszeiten) | ❌ |
| Items | ❌ | ✅ (Station-Items) | ✅ (Zuordnung ändern) | ❌ |
| Availability | ❌ | ✅ | ❌ | ❌ |
| Timeframes | ✅ | ✅ (eigene) | ✅ (eigene) | ✅ (eigene) |
| Bookings | ✅ | ✅ (Station-Buchungen) | ✅ (alle Station-Operationen) | ❌ |
| Team Members | ❌ | ✅ (eigene Station) | ✅ (zuweisen/entfernen) | ❌ |
| Restrictions | ✅ | ✅ | ✅ | ✅ |
| Reports | ❌ | ✅ (Station) | ❌ | ❌ |

**Erweiterte Rechte:**

```typescript
// Station Manager kann:
// - Zeitfenster für seine Station erstellen/bearbeiten
// - Sperrungen (Reparatur, Ferien) für seine Station setzen
// - Team Members seiner Station zuweisen
// - Buchungen bestätigen, stornieren
// - Ausleihe/Rücknahme durchführen
// - Station-spezifische Berichte einsehen
// - Eigene Station-Metadaten bearbeiten
```

**Einschränkungen:**
- Keine anderen Stationen
- Keine Organisationsweiten Einstellungen
- Keine anderen Station Manager verwalten

---

#### ADMIN (Systemadministrator)
| Attribut | Wert |
|----------|------|
| **ID** | `admin` |
| **Beschreibung** | Vollständiger Systemzugriff |
| **Authentifizierung** | Keycloak + MFA + Hardware Token |
| **Scope** | Systemweit |

**Berechtigungen:**

| Resource | Create | Read | Update | Delete |
|----------|--------|------|--------|--------|
| Organizations | ✅ | ✅ | ✅ | ✅ |
| Stations | ✅ | ✅ (alle) | ✅ | ✅ |
| Items | ✅ | ✅ (alle) | ✅ | ✅ |
| Categories | ✅ | ✅ | ✅ | ✅ |
| Timeframes | ✅ | ✅ | ✅ | ✅ |
| Bookings | ✅ | ✅ (alle) | ✅ | ✅ |
| Users | ✅ | ✅ (alle) | ✅ | ✅ |
| Station Managers | ✅ | ✅ | ✅ | ✅ |
| Team Members | ✅ | ✅ | ✅ | ✅ |
| Restrictions | ✅ | ✅ | ✅ | ✅ |
| API Keys | ✅ | ✅ | ✅ | ✅ |
| Settings | ❌ | ✅ | ✅ | ❌ (nur via Config) |
| Reports | ❌ | ✅ (alle) | ❌ | ❌ |
| Export | ❌ | ✅ | ❌ | ❌ |

**Admin-spezifische Funktionen:**

```typescript
// Admin kann:
// - Neue Stationen erstellen
// - Neue Items erstellen
// - Organization-Einstellungen ändern
// - Benutzer-Accounts erstellen/löschen
// - Rollen zuweisen
// - API-Keys erstellen
// - Globale Statistiken
// - Alle Daten exportieren
// - Systemkonfiguration (nicht über App)
```

---

### 10.3 Permission Matrix

**Hinweis:** Höhere Rollen erben alle Rechte der niedrigeren Rollen.

| Permission | PUBLIC | USER | TEAM_MEMBER | STATION_MANAGER | ADMIN |
|------------|--------|------|-------------|-----------------|-------|
| View public stations | ✅ | ✅ | ✅ | ✅ | ✅ |
| View all stations | ❌ | ✅ | ✅ (assigned) | ✅ (assigned) | ✅ |
| View items | ✅ | ✅ | ✅ | ✅ | ✅ |
| View availability | ✅ | ✅ | ✅ | ✅ | ✅ |
| Create booking | ❌ | ✅ | ✅ | ✅ | ✅ |
| Cancel own booking | ❌ | ✅ | ✅ | ✅ | ✅ |
| View own bookings | ❌ | ✅ | ✅ | ✅ | ✅ |
| View station bookings | ❌ | ❌ | ✅ | ✅ | ✅ |
| Confirm booking | ❌ | ❌ | ✅ | ✅ | ✅ |
| Check-in/out | ❌ | ❌ | ✅ | ✅ | ✅ |
| Create timeframes | ❌ | ❌ | ❌ | ✅ (own station) | ✅ |
| Edit timeframes | ❌ | ❌ | ❌ | ✅ (own station) | ✅ |
| Create restrictions | ❌ | ❌ | ❌ | ✅ (own station) | ✅ |
| Edit station meta | ❌ | ❌ | ❌ | ✅ (own station) | ✅ |
| Manage team members | ❌ | ❌ | ❌ | ✅ (own station) | ✅ |
| View station reports | ❌ | ❌ | ❌ | ✅ (own station) | ✅ |
| Create stations | ❌ | ❌ | ❌ | ❌ | ✅ |
| Manage all stations | ❌ | ❌ | ❌ | ❌ | ✅ |
| Manage users | ❌ | ❌ | ❌ | ❌ | ✅ |
| Manage roles | ❌ | ❌ | ❌ | ❌ | ✅ |
| API key management | ❌ | ❌ | ❌ | ❌ | ✅ |
| View all reports | ❌ | ❌ | ❌ | ❌ | ✅ |

**Implizite Vererbung (effektive Permissions):**

| User hat Rolle... | ...dann hat er automatisch auch |
|-------------------|--------------------------------|
| `station_manager` | `team_member`, `user` |
| `team_member` | `user` |
| `user` | - |
| `admin` | Alle Rollen |

**Beispiel: Effektive Permissions eines Users mit `[station_manager, team_member]`:**
- Alle `user` Permissions ✅
- Alle `team_member` Permissions ✅
- Alle `station_manager` Permissions ✅
- Keine `admin` Permissions ❌

---

### 10.4 Keycloak Role Mapping

```yaml
# Keycloak Realm: commonsbooking

# Realm Roles (Systemweit)
roles:
  - name: admin
    description: System administrator
    composites:
      - station_manager
      - team_member
      - user

  - name: station_manager
    description: Station manager - manages one or more stations
    composites:
      - team_member

  - name: team_member
    description: Organization team member
    composites:
      - user

  - name: user
    description: Registered user with booking capability

  - name: public
    description: Unauthenticated public access

# Client Scope: commonsbooking-mobile
# Role mappings to JWT claims
token_mappers:
  - name: realm_roles
    claim_name: roles
    claim_value: realm

  - name: station_assignment
    claim_name: stations
    claim_value: client_scope

# Example JWT Payload
{
  "sub": "user-uuid-123",
  "email": "user@example.org",
  "name": "Max Mustermann",
  "roles": ["user", "team_member"],
  "stations": ["station-uuid-abc", "station-uuid-def"],
  "organization_id": "org-uuid-xyz"
}
```

---

### 10.5 Implementation Example (TypeScript)

```typescript
// Rollen sind kumulativ - ein User kann mehrere Rollen haben

// Rollen als Array (typischerweise aus Keycloak JWT)
type Role = 'public' | 'user' | 'team_member' | 'station_manager' | 'admin';
type UserRoles = Role[];

// Berechtigungen
enum Permission {
  STATIONS_READ = 'stations:read',
  STATIONS_READ_ALL = 'stations:read:all',
  ITEMS_READ = 'items:read',
  BOOKINGS_READ_OWN = 'bookings:read:own',
  BOOKINGS_READ_STATION = 'bookings:read:station',
  BOOKING_CREATE = 'booking:create',
  BOOKING_CANCEL_OWN = 'booking:cancel:own',
  BOOKING_CONFIRM = 'booking:confirm',
  BOOKING_CHECKINOUT = 'booking:checkinout',
  STATIONS_CREATE = 'stations:create',
  STATIONS_UPDATE = 'stations:update',
  TIMEFRAMES_CREATE = 'timeframes:create',
  USERS_MANAGE = 'users:manage',
}

// Permissions pro Basis-Rolle (ohne Vererbung)
const baseRolePermissions: Record<Role, Permission[]> = {
  public: [
    Permission.STATIONS_READ,
    Permission.ITEMS_READ,
  ],
  user: [
    Permission.STATIONS_READ,
    Permission.ITEMS_READ,
    Permission.BOOKINGS_READ_OWN,
    Permission.BOOKING_CREATE,
    Permission.BOOKING_CANCEL_OWN,
  ],
  team_member: [
    Permission.BOOKINGS_READ_STATION,
    Permission.BOOKING_CONFIRM,
    Permission.BOOKING_CHECKINOUT,
  ],
  station_manager: [
    Permission.STATIONS_READ_ALL,
    Permission.STATIONS_UPDATE,
    Permission.TIMEFRAMES_CREATE,
  ],
  admin: Object.values(Permission),
};

// Hierarchie für implizite Vererbung
const roleHierarchy: Record<Role, Role[]> = {
  public: [],
  user: [],
  team_member: ['user'],
  station_manager: ['team_member', 'user'],
  admin: ['station_manager', 'team_member', 'user'],
};

// Berechnet alle effektiven Permissions aus Rollen-Array
function getEffectivePermissions(roles: UserRoles): Set<Permission> {
  const permissions = new Set<Permission>();

  for (const role of roles) {
    // Direct permissions
    for (const p of baseRolePermissions[role]) {
      permissions.add(p);
    }
    // Inherited permissions (Role Hierarchy)
    for (const inherited of roleHierarchy[role]) {
      for (const p of baseRolePermissions[inherited]) {
        permissions.add(p);
      }
    }
  }

  return permissions;
}

// User Context aus JWT
interface UserContext {
  id: string;
  email: string;
  roles: UserRoles;  // Array von Rollen!
  stations: string[];  // Station-IDs für station_manager
  organizationId: string;
}

// Permission check middleware
function requirePermission(permission: Permission) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const user = req.user as UserContext;

    // Hole effektive Permissions aus allen Rollen
    const effectivePermissions = getEffectivePermissions(user.roles);

    if (!effectivePermissions.has(permission)) {
      return res.status(403).json({
        error: 'FORBIDDEN',
        message: `Permission '${permission}' required`,
        userRoles: user.roles
      });
    }

    // Station-specific checks
    if (user.roles.includes('station_manager')) {
      const resourceStationId = req.params.stationId;
      if (resourceStationId && !user.stations.includes(resourceStationId)) {
        return res.status(403).json({
          error: 'FORBIDDEN',
          message: 'Access denied for this station',
          allowedStations: user.stations
        });
      }
    }

    next();
  };
}

// Helper: Check if user has specific role
function hasRole(user: UserContext, role: Role): boolean {
  return user.roles.includes(role);
}

// Helper: Check if user can manage station
function canManageStation(user: UserContext, stationId: string): boolean {
  return user.roles.includes('admin') ||
         (user.roles.includes('station_manager') && user.stations.includes(stationId));
}

// Usage Examples
const user1: UserContext = {
  id: 'user-1',
  email: 'team@verein.de',
  roles: ['team_member'],  // Nur Team Member
  stations: [],
  organizationId: 'org-1'
};
// Permissions: user + team_member = [STATIONS_READ, ITEMS_READ, BOOKINGS_READ_OWN, BOOKING_CREATE,
//                                    BOOKING_CANCEL_OWN, BOOKINGS_READ_STATION, BOOKING_CONFIRM, BOOKING_CHECKINOUT]

const user2: UserContext = {
  id: 'user-2',
  email: 'manager@fahrradladen.de',
  roles: ['station_manager', 'team_member'],  // Explicit multi-role
  stations: ['station-abc', 'station-def'],
  organizationId: 'org-1'
};
// Permissions: station_manager + team_member + user = ALLE station_manager Permissions!

const user3: UserContext = {
  id: 'user-3',
  email: 'admin@example.org',
  roles: ['admin'],  // Admin hat implizit alle anderen Rollen
  stations: [],
  organizationId: 'org-1'
};
// Permissions: admin = ALLE Permissions

// Route examples
app.post('/bookings', requirePermission(Permission.BOOKING_CREATE), bookingController.create);
app.post('/bookings/:id/confirm',
  requirePermission(Permission.BOOKING_CONFIRM),
  bookingController.confirm
);
app.post('/bookings/:id/checkin',
  requirePermission(Permission.BOOKING_CHECKINOUT),
  bookingController.checkin
);
app.post('/timeframes',
  requirePermission(Permission.TIMEFRAMES_CREATE),
  timeframeController.create
);
```

**Keycloak JWT Claim Beispiel mit mehreren Rollen:**

```json
{
  "sub": "user-uuid-123",
  "email": "station-manager@verein.de",
  "name": "Station Manager",
  "roles": ["station_manager", "team_member", "user"],
  "stations": ["station-uuid-abc", "station-uuid-def"],
  "organization_id": "org-uuid-xyz",
  "iat": 1622548800,
  "exp": 1622635200,
  "iss": "https://keycloak.example.org/realms/commonsbooking"
}
```

---

### 10.6 Migration: WordPress Roles → Keycloak Roles

| WordPress Rolle | Keycloak Rolle | Anmerkungen |
|-----------------|----------------|-------------|
| `administrator` | `admin` | 1:1 Mapping |
| `cb_manager` | `station_manager` | War station-spezifisch, jetzt breiter |
| `subscriber` | `user` | 1:1 Mapping |
| (keine) | `team_member` | Neue Rolle, keine WP-Entsprechung |
| (keine) | `public` | Keine Authentifizierung |

---

## 11. Upgrade & Migration

### 11.1 Versions-Upgrades

| Version | Aufgaben |
|---------|----------|
| 2.6.0 | Booking-Migration, Set advanceBookingDaysDefault |
| 2.8.0 | Unschedule old scheduler events |
| 2.8.2 | Reset color scheme, fix iCal titles |
| 2.8.5 | Remove breaking postmeta (AJAX) |
| 2.9.0 | Set multi-select timeframe default (AJAX) |
| 2.9.2 | Enable location booking notification |
| 2.10 | Migrate map settings |
| 2.10.5 | Migrate cache settings |

### 11.2 CB1 Migration

Unterstützung für Daten aus CommonsBooking 1.x (bis v0.9.4.18):
- Post-Type Mapping (CB1 → CB2 IDs)
- Buchungs-Migration
- Booking Codes Migration
- Meta-Field Mapping

---

## 12. External Integrations

### 12.1 Geocoding

```
Service: OpenStreetMap Nominatim
- URL: nominatim.openstreetmap.org
- User-Agent: CommonsBooking v{version} Contact: mail@commonsbooking.org
- Fallback: null bei Fehler
```

### 12.2 Mail Versand

```
Library: PHPMailer v7
- SMTP-Konfiguration via WordPress
- Templates via commonsbooking_parse_template
```

### 12.3 Cache

```
Library: Symfony Cache
- PSR-16 compliant
- Tag-based invalidation
```

---

## 13. Technical Constraints

### 13.1 Date Handling

```
Format: d.m.Y (German locale default)
Internal Storage: Unix Timestamp
Timezone: WordPress timezone setting

Verarbeitung:
- Helper::FormattedDate()
- Helper::FormattedTime()
- Helper::FormattedDateTime()
- UTC-Konversion für APIs
```

### 13.2 URL Rewrites

| Slug | Rewrite | Query Var |
|------|---------|-----------|
| `commonsbooking_ical_download` | iCal Download | - |
| `commonsbooking_get_vehicle` | GBFS Vehicle ID | `commonsbooking_vehicle_cloaked_id` |

### 13.3 Cron Jobs

| Hook | Schedule | Beschreibung |
|------|----------|--------------|
| `commonsbooking_send_booking_reminder` | hourly | Sendet Erinnerungen |
| `commonsbooking_send_booking_codes` | twicedaily | Code-E-Mails |
| `commonsbooking_cron_export` | as-needed | CSV Export |
| `commonsbooking_cache_clear` | as-needed | Cache invalidation |

---

## 14. User Flows

### 14.1 Buchungsablauf (End-to-End)

```
1. Nutzer besucht Seite mit [cb_items] oder [cb_items_table]
2. System zeigt verfügbare Items mit Kalender
3. Nutzer wählt Zeitraum im Litepicker
4. System prüft Verfügbarkeit (AJAX):
   - Timeframe.exists?(item, location, start, end)
   - !Restriction.exists?(item, location, start, end)
   - BookingRule.checkCompliance(user, item, start, end)
5. Bei verfügbar:
   - Nutzer gibt Name/E-Mail ein
   - System erstellt Booking (Status: unconfirmed)
   - Admin wird benachrichtigt (AdminMessage)
6. Admin bestätigt Buchung:
   - Status → confirmed
   - BookingCode wird generiert
   - Nutzer erhält BookingMessage + Code
7. Gebühr:
   - Scheduler sendet Reminder (X Tage vorher)
   - Nutzer holt Item ab
   - Admin bestätigt Return
   - Booking → canceled
```

### 14.2 Zeitrahmen-Konfiguration (Admin)

```
1. Admin öffnet Item/Location bearbeiten
2. Timeframe hinzufügen:
   - Typ wählen (Bookable, Holiday, Repair, etc.)
   - Start/Ende Datum setzen
   - Grid konfigurieren (full-day oder time-slots)
   - Wiederholung setzen (optional)
3. System erstellt Timeframe-Posts
4. Calendar View aktualisiert sich automatisch
```

---

## 15. Tech Stack Recommendations

### 15.1 Option A: TypeScript (Node.js / Deno / Bun)

**Primary Choice für Web-Backend mit schneller Iteration**

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Runtime** | Node.js 20+ LTS oder Bun | Production-ready, gute Ecosystem |
| **Framework** | Express.js oder Fastify | Flexibel, gute Middleware-Ökosystem |
| **ORM** | Prisma | Type-safe, excellent DX, SQLite default, PostgreSQL support |
| **Cache** | Redis (ioredis) + BullMQ | Job Queue + Cache aus einem |
| **Validation** | Zod | Type-safe Schema-Validation |
| **Auth** | Keycloak (OIDC) + jose | JWT validation, OIDC client |
| **API Docs** | OpenAPI / Swagger | Schema-first |
| **Frontend** | Vue 3 + Vite oder React + Vite | Beide funktionieren gut mit TS |
| **Maps** | Leaflet + TypeScript | OpenStreetMap Integration |
| **iCal** | ical.js | Pure JS iCal generation |
| **Email** | Nodemailer | SMTP + Templates |
| **Testing** | Vitest + Playwright | Schnell, gute TS-Unterstützung |

**Pro:**
- Schnellere Development-Zyklen
- Riesige NPM ecosystem (50k+ packages)
- TypeScript für type-safe code
- Keycloak SDK offiziell für TS/JS
- Einfacher für Frontend-Devs

**Contra:**
- Runtime GC overhead
- Weniger performant als Rust für CPU-intensive tasks

**Beispiel: Projektstruktur**

```
commonsbooking/
├── packages/
│   ├── api/                    # Node.js + Express/Fastify
│   │   ├── src/
│   │   │   ├── routes/         # REST endpoints
│   │   │   ├── services/       # Business logic
│   │   │   ├── middleware/     # Auth, validation
│   │   │   ├── models/         # Prisma client
│   │   │   └── jobs/           # BullMQ workers
│   │   ├── prisma/
│   │   │   └── schema.prisma   # Database schema
│   │   └── package.json
│   │
│   ├── frontend/               # Vue 3 oder React + Vite
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── views/
│   │   │   ├── composables/
│   │   │   └── stores/
│   │   └── package.json
│   │
│   ├── admin/                  # React Admin Panel
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── pages/
│   │   │   └── hooks/
│   │   └── package.json
│   │
│   └── shared/                 # Shared types/utils
│       ├── src/
│       │   ├── types/          # Zod schemas, TS interfaces
│       │   └── utils/
│       └── package.json
│
├── docker-compose.yml          # SQLite (default), Redis, Keycloak
│                              # Optional: PostgreSQL für Production
├── keycloak/                   # Keycloak realm exports
└── README.md
```

**Startpunkt Backend:**

```typescript
// packages/api/src/index.ts
import express from 'express';
import { PrismaClient } from '@prisma/client';
import { authenticate } from './middleware/auth';
import { validate } from './middleware/validation';
import { bookingRouter } from './routes/booking';
import { itemRouter } from './routes/item';
import { availabilityRouter } from './routes/availability';

const app = express();
const prisma = new PrismaClient();

// Middleware
app.use(express.json());
app.use(authenticate); // Keycloak JWT validation

// Routes
app.use('/bookings', bookingRouter);
app.use('/items', itemRouter);
app.use('/availability', availabilityRouter);
app.use('/gbfs', gbfsRouter); // GBFS endpoints

app.listen(3000, () => {
  console.log('CommonsBooking API running on :3000');
});
```

**Startpunkt Database Schema:**

```prisma
// packages/api/prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

// Datenbank: SQLite als Default (für Development/Small Production)
// Für Production mit höherer Last: PostgreSQL
// Wechsel via DATABASE_URL environment variable
datasource db {
  provider = "sqlite"  // Alternative: "postgresql"
  url      = env("DATABASE_URL")  // sqlite: "file:./commonsbooking.db"
                                 // postgres: "postgresql://user:pass@localhost:5432/commonsbooking"
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  keycloakId String  @unique
  roles     Role[]
  createdAt DateTime @default(now())

  items     Item[]
  locations Location[]
  bookings  Booking[]
}

enum Role {
  ADMIN
  MANAGER
  USER
}

model Item {
  id          String   @id @default(uuid())
  name        String
  description String?
  categoryId  String?
  authorId    String
  author      User     @relation(fields: [authorId], references: [id])
  imageUrl    String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  category    Category?     @relation(fields: [categoryId], references: [id])
  timeframes  Timeframe[]
  bookings    Booking[]
}

model Location {
  id                  String   @id @default(uuid())
  name                String
  description         String?
  address             String
  latitude            Float?
  longitude           Float?
  pickupInstructions  String?
  authorId            String
  author              User     @relation(fields: [authorId], references: [id])
  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  timeframes          Timeframe[]
  bookings            Booking[]
}

model Timeframe {
  id          String        @id @default(uuid())
  type        TimeframeType
  startDate   DateTime
  endDate     DateTime
  startTime   String?
  endTime     String?
  gridDuration Int          @default(60)
  mode        TimeframeMode @default(FULL_DAY)
  repeat      RepeatMode    @default(NONE)
  itemId      String
  item        Item          @relation(fields: [itemId], references: [id])
  locationId  String
  location    Location      @relation(fields: [locationId], references: [id])
  meta        Json?
  createdAt   DateTime      @default(now())

  @@index([itemId, locationId, startDate])
}

enum TimeframeType {
  BOOKABLE
  HOLIDAY
  OFFICIAL_HOLIDAY
  REPAIR
  BOOKING
  BOOKING_CANCELED
}

enum TimeframeMode {
  FULL_DAY
  TIME_BASED
}

enum RepeatMode {
  NONE
  WEEKLY
  MONTHLY
}

model Booking {
  id            String        @id @default(uuid())
  status        BookingStatus @default(UNCONFIRMED)
  startDate     DateTime
  endDate       DateTime
  itemId        String
  item          Item          @relation(fields: [itemId], references: [id])
  locationId    String
  location      Location      @relation(fields: [locationId], references: [id])
  userId        String
  user          User          @relation(fields: [userId], references: [id])
  bookingCode   String?
  notes         String?
  bookedAt      DateTime      @default(now())
  confirmedAt   DateTime?
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  @@unique([bookingCode])
  @@index([userId, status])
  @@index([itemId, startDate])
}

enum BookingStatus {
  UNCONFIRMED
  CONFIRMED
  CANCELED
}

model Category {
  id     String  @id @default(uuid())
  name   String
  slug   String  @unique
  type   CategoryType
  items  Item[]

  @@index([slug])
}

enum CategoryType {
  ITEM
  LOCATION
}

model Restriction {
  id          String   @id @default(uuid())
  name        String
  type        RestrictionType
  startDate   DateTime
  endDate     DateTime
  state       String?  // German state code: BW, BY, etc.
  itemId      String?
  locationId  String?
  categoryId  String?
  createdAt   DateTime @default(now())

  @@index([startDate, endDate])
}

enum RestrictionType {
  ITEM
  LOCATION
  CATEGORY
}
```

---

### 15.2 Option B: Rust (Axum / Actix)

**Primary Choice für maximale Performance**

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Framework** | Axum oder Actix-web | Async, fast, type-safe |
| **ORM** | Diesel oder SQLx | Type-safe SQL, async |
| **Cache** | Redis (deadpool-redis) + Queuer | async Redis client |
| **Validation** | validator oder schemars | Derive macros |
| **Auth** | Keycloak (OIDC) + jsonwebtoken | JWT validation |
| **API Docs** | utoipa (OpenAPI) | Rust-native |
| **Frontend** | Vue 3 + Vite oder Leptos (SSR) | TS für UI wenn Leptos |
| **Maps** | Leaflet + wasm-bindgen | Für Rust-side geocoding |
| **iCal** | icalendar crate | Rust iCal generation |
| **Email** | Lettre | SMTP client |
| **Testing** | Rust tests + Playwright | Unit + E2E |

**Pro:**
- Höchste Performance (10-100x vs Node.js)
- Memory safety ohne GC
- Thread-safe concurrency
- Compile-time type safety
- Niedrige Server-Kosten bei hoher Last

**Contra:**
- Längere Development-Zyklen
- Kleinere Crate-Ecosystem (aber wachsend)
- Steilere Lernkurve für Team
- Keycloak SDK nicht offiziell für Rust

**Beispiel: Projektstruktur**

```
commonsbooking/
├── Cargo.toml                   # Workspace
├── packages/
│   ├── api/                     # Rust (Axum)
│   │   ├── src/
│   │   │   ├── routes/          # REST endpoints
│   │   │   ├── services/        # Business logic
│   │   │   ├── middleware/      # Auth, validation
│   │   │   ├── models/          # SQLx models
│   │   │   └── jobs/            # Background workers
│   │   ├── migrations/          # SQL migrations
│   │   ├── Cargo.toml
│   │   └── Dockerfile
│   │
│   ├── frontend/                # Vue 3 + Vite (TypeScript)
│   │   ├── src/
│   │   └── package.json
│   │
│   ├── admin/                   # React + Vite
│   │   └── package.json
│   │
│   └── shared/                  # Rust crate für shared types
│       ├── src/
│       │   ├── types/           # Shared structs
│       │   └── schemas/         # JSON Schema
│       └── Cargo.toml
│
├── docker-compose.yml
└── keycloak/
```

**Startpunkt Backend:**

```rust
// packages/api/src/main.rs
use axum::{routing::get, Router};
use sqlx::PgPool;
use std::env;

mod routes;
mod services;
mod middleware;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let database_url = env::var("DATABASE_URL")?;
    let pool = PgPool::connect(&database_url).await?;

    let app = Router::new()
        .route("/health", get(|| async { "OK" }))
        .nest("/api/v1", routes::booking::router())
        .nest("/api/v1", routes::items::router())
        .nest("/api/v1", routes::availability::router())
        .nest("/gbfs", routes::gbfs::router())
        .layer(axum::middleware::from_fn(middleware::auth::validate_jwt))
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

**Startpunkt Database Schema:**

```sql
-- packages/api/migrations/001_initial_schema.sql

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255),
    keycloak_id VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TYPE user_role AS ENUM ('ADMIN', 'MANAGER', 'USER');

CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role user_role NOT NULL,
    PRIMARY KEY (user_id, role)
);

CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    type category_type NOT NULL
);

CREATE TYPE category_type AS ENUM ('ITEM', 'LOCATION');

CREATE TABLE items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id UUID REFERENCES categories(id),
    author_id UUID NOT NULL REFERENCES users(id),
    image_url VARCHAR(500),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    address VARCHAR(500) NOT NULL,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    pickup_instructions TEXT,
    author_id UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TYPE timeframe_type AS ENUM (
    'BOOKABLE', 'HOLIDAY', 'OFFICIAL_HOLIDAY',
    'REPAIR', 'BOOKING', 'BOOKING_CANCELED'
);

CREATE TYPE timeframe_mode AS ENUM ('FULL_DAY', 'TIME_BASED');
CREATE TYPE repeat_mode AS ENUM ('NONE', 'WEEKLY', 'MONTHLY');

CREATE TABLE timeframes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type timeframe_type NOT NULL,
    start_date TIMESTAMP WITH TIME ZONE NOT NULL,
    end_date TIMESTAMP WITH TIME ZONE NOT NULL,
    start_time TIME,
    end_time TIME,
    grid_duration INTEGER DEFAULT 60,
    mode timeframe_mode DEFAULT 'FULL_DAY',
    repeat repeat_mode DEFAULT 'NONE',
    item_id UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    meta JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_timeframes_item_location ON timeframes(item_id, location_id, start_date);

CREATE TYPE booking_status AS ENUM ('UNCONFIRMED', 'CONFIRMED', 'CANCELED');

CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    status booking_status DEFAULT 'UNCONFIRMED',
    start_date TIMESTAMP WITH TIME ZONE NOT NULL,
    end_date TIMESTAMP WITH TIME ZONE NOT NULL,
    item_id UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    booking_code VARCHAR(20) UNIQUE,
    notes TEXT,
    booked_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    confirmed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_bookings_user_status ON bookings(user_id, status);
CREATE INDEX idx_bookings_item_date ON bookings(item_id, start_date);

CREATE TYPE restriction_type AS ENUM ('ITEM', 'LOCATION', 'CATEGORY');

CREATE TABLE restrictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    type restriction_type NOT NULL,
    start_date TIMESTAMP WITH TIME ZONE NOT NULL,
    end_date TIMESTAMP WITH TIME ZONE NOT NULL,
    state VARCHAR(10),  -- German state: BW, BY, etc.
    item_id UUID REFERENCES items(id),
    location_id UUID REFERENCES locations(id),
    category_id UUID REFERENCES categories(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_restrictions_dates ON restrictions(start_date, end_date);

-- Booking codes table (custom table as in original)
CREATE TABLE booking_codes (
    date DATE NOT NULL,
    timeframe_id UUID NOT NULL REFERENCES timeframes(id) ON DELETE CASCADE,
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    item_id UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    code VARCHAR(100) NOT NULL,
    PRIMARY KEY (date, timeframe_id, location_id, item_id, code)
);
```

---

### 15.3 Comparison: TypeScript vs Rust

| Kriterium | TypeScript (Node.js) | Rust (Axum/Actix) |
|-----------|---------------------|--------------------|
| **Development Speed** | ⚡⚡⚡⚡⚡ Schnell | ⚡⚡ Langsam |
| **Performance** | ⚡⚡⚡ Gut | ⚡⚡⚡⚡⚡ Exzellent |
| **Type Safety** | ⚡⚡⚡⚡ Stark (mit TS) | ⚡⚡⚡⚡⚡ Total (compile-time) |
| **Ecosystem** | ⚡⚡⚡⚡⚡ Riesig (NPM) | ⚡⚡⚡ Gut (crates.io) |
| **Memory Usage** | ⚡⚡ Höher (GC) | ⚡⚡⚡⚡⚡ Minimal |
| **Concurrency** | ⚡⚡⚡ Event-loop | ⚡⚡⚡⚡⚡ Thread-safe async |
| **Learning Curve** | ⚡⚡⚡ Einfach | ⚡⚡ Steil |
| **Keycloak Integration** | ⚡⚡⚡⚡⚡ Offizieller SDK | ⚡⚡⚡ Kein offizieller SDK |
| **Server Costs** | ⚡⚡⚡ Mittel | ⚡⚡⚡⚡ Niedrig |
| **Team Experience** | ⚡⚡⚡⚡ Frontend-Devs | ⚡⚡ System-Devs |
| **Database Support** | ⚡⚡⚡⚡⚡ SQLite + PostgreSQL via Prisma | ⚡⚡⚡ SQLite + PostgreSQL via SQLx |

### 15.4 Decision Matrix

```
┌─────────────────────────────────────────────────────────────┐
│                     Tech Stack Decision                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Team Background:                                           │
│  ├── JavaScript/TypeScript heavy  ──→  TypeScript (Option A)│
│  ├── Systems programming/Rust       ──→  Rust (Option B)    │
│  └── Mixed team                     ──→  TypeScript empfohlen│
│                                                             │
│  Performance Requirements:                                  │
│  ├── < 1000 req/s                   ──→  TypeScript (Option A)│
│  ├── 1000-10000 req/s               ──→  Either             │
│  └── > 10000 req/s                  ──→  Rust (Option B)    │
│                                                             │
│  Time to Market:                                           │
│  ├── < 3 months                     ──→  TypeScript (Option A)│
│  ├── 3-6 months                     ──→  Either             │
│  └── > 6 months                     ──→  Rust (Option B)    │
│                                                             │
│  Scale Expectations:                                        │
│  ├── < 10000 active users           ──→  TypeScript (Option A)│
│  ├── 10000-100000 active users      ──→  Either             │
│  └── > 100000 active users          ──→  Rust (Option B)    │
│                                                             │
│  Database Choice:                                          │
│  ├── Development/Small Prod        ──→  SQLite (Default)    │
│  ├── Medium-Large Production       ──→  PostgreSQL          │
│  └── High Write Load               ──→  PostgreSQL          │
│                                                             │
│  Recommendation:                                           │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│                                                             │
│  For most teams and use cases:                              │
│  TypeScript (Option A) mit Node.js + Fastify               │
│  + SQLite (Default) oder PostgreSQL                        │
│  → Schnellere Entwicklung, weniger Risiko, Keycloak SDK    │
│                                                             │
│  For high-scale or performance-critical:                    │
│  Rust (Option B) mit Axum + PostgreSQL                     │
│  → Beste Performance, lowest operational costs             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 15.5 Shared Architecture Decisions (Both Options)

**Database: SQLite (Default) mit PostgreSQL Support**

```yaml
# Development / Small Production (SQLite - keine额外 Abhängigkeit)
DATABASE_URL="file:./commonsbooking.db"

# Production / High Load (PostgreSQL)
DATABASE_URL="postgresql://user:pass@localhost:5432/commonsbooking"

# Warum SQLite?
# ✅ Kein separater Database Server nötig
# ✅ Null-Konfiguration, Datei-basiert
# ✅ Perfekt für < 10.000 aktive Buchungen/Tag
# ✅ Einfaches Backup (Datei kopieren)
# ✅ Funktioniert auf Shared Hosting

# Wann PostgreSQL?
# ⚠️ > 10.000 aktive Buchungen/Tag
# ⚠️ > 100 gleichzeitige Nutzer
# ⚠️ Komplexe SQL-Joins mit grossen Datenmengen
# ⚠️ Full-Text Search über grössere Datasets
# ⚠️ Replikation / High Availability benötigt
```

**Keycloak Configuration:**

```yaml
# Keycloak Realm: commonsbooking
realm: commonsbooking

clients:
  - clientId: commonsbooking-api
    publicClient: false
    secret: ${API_CLIENT_SECRET}
    directAccessGrantsEnabled: true
    serviceAccountsEnabled: true
    audiences: commonsbooking-api

  - clientId: commonsbooking-mobile
    publicClient: true
    standardFlowEnabled: true
    directAccessGrantsEnabled: true

# Role mapping
# Keycloak roles: admin, manager, user
# Backend roles: ADMIN, MANAGER, USER
```

**Database Connection Pool:**

```yaml
# TypeScript: prisma.schema (SQLite als Default)
datasource db {
  provider = "sqlite"  # oder "postgresql" für Production
  url      = env("DATABASE_URL")
}

# Datenbank-URL Environment Variables:
# Development (SQLite):
#   DATABASE_URL="file:./commonsbooking.db"
# Production (PostgreSQL):
#   DATABASE_URL="postgresql://user:pass@localhost:5432/commonsbooking"

# Rust: config.rs
database_url = "postgres://user:pass@localhost/commonsbooking"  # oder SQLite Pfad
pool_size = 20  # For high concurrency (nur PostgreSQL)
```

**Redis Configuration:**

```yaml
# Both: docker-compose.yml
redis:
  image: redis:7-alpine
  command: redis-server --appendonly yes
  volumes:
    - redis_data:/data

# Connection
REDIS_URL: redis://redis:6379
```

---

## 16. Non-Functional Requirements

### 15.1 Performance

- Cache-Adapter für schnelle Datenbankabfragen
- Midnight-Expiration für Kalenderdaten
- Versionierte Cache-Keys (für Multi-Site)
- Warmup-Funktion für Cache-Preloading

### 15.2 Internationalisierung

- WordPress Locale System
- Deutsche Datumsformatierung
- Mehrsprachige GBFS-Responses

### 15.3 Security

- API Key Authentication
- Capability-basierte Berechtigungen
- Sanitization aller Inputs
- Nonce-Validierung für AJAX

---

## 17. Data Flow Diagrams

### 16.1 Calendar Availability Calculation

```
[Request: item, location, startDate, endDate]
         ↓
[TimeframeRepository::getBookableTimeframes()]
         ↓
[Filter: only bookable, matching item/location, in range]
         ↓
[RestrictionRepository::get()]- check restrictions
         ↓
[Calendar::addWeek()]- build week grid
         ↓
[Calendar::addDay()]- aggregate day slots
         ↓
[Day::processSlot()]- mark slots booked/locked
         ↓
[Response: JSON with slots, bookedDays, lockDays, holidays]
```

### 16.2 Booking Code Generation

```
[Trigger: Admin confirms OR cron (Timeframe END date reached)]
         ↓
[BookingCodesRepository::generate()]
         ↓
[Check: already generated?]- skip if exists
         ↓
[Get: Timeframe with advanceGenerationDays setting]
         ↓
[DatePeriod: start → advanceGenerationDays]
         ↓
[For each day:]
  [Generate: 8-char random alphanumeric]
  [Persist: to cb_bookingcodes table]
         ↓
[BookingCodes::sendBookingCodesMessage()]
         ↓
[Send: Email with all codes for timeframe]
```

---

## 18. Component Inventory

### 17.1 Backend Components

| Component | Responsibility | Public API |
|-----------|---------------|------------|
| **Booking\Service** | Cleanup, Reminders | `sendReminder()`, `cleanup()` |
| **BookingRule\Service** | Rule Definition | `getRules()`, `addRule()` |
| **BookingCodes\Service** | Code Generation/Email | `generate()`, `sendCodes()` |
| **Cache\Service** | Symfony Cache Wrapper | `get()`, `set()`, `clear()` |
| **iCalendar\Service** | ICS Generation | `getCalendar()`, `downloadICS()` |
| **Scheduler\Service** | WP Cron Management | `schedule()`, `unschedule()` |
| **TimeframeExport\Service** | CSV Export | `getCSV()`, `cronExport()` |
| **Upgrade\Service** | Version Migration | `run()`, `isAJAXUpgrade()` |
| **Holiday\Service** | German Holidays | `getStates()`, `renderFields()` |
| **MassOperations\Service** | Bulk Operations | `migrateOrphaned()` |

### 17.2 Frontend Components

| Component | Shortcode | Output |
|-----------|-----------|--------|
| **Item\View** | `[cb_items]` | Item-Liste mit Link |
| **Location\View** | `[cb_locations]` | Standort-Liste mit Karte |
| **Booking\View** | `[cb_bookings]` | Nutzer-Buchungsliste |
| **Calendar\View** | `[cb_items_table]` | Kalender-Tabelle |
| **Map\View** | `[commonsbooking_map]` | Leaflet-Karte |
| **UserWidget** | Widget | Login/Logout Links |

### 17.3 Repository Components

| Repository | Query Focus |
|------------|-------------|
| **Item\Repository** | Item by Location, Cloaked ID |
| **Location\Repository** | Location by Item |
| **Booking\Repository** | User bookings, Status filter |
| **Timeframe\Repository** | Grid data, Date ranges |
| **Restriction\Repository** | Complex multi-filter query |
| **BookingCodes\Repository** | Code generation/retrieval |
| **UserRepository** | Role-based queries |

---

## 19. Glossary

| Term | Definition |
|------|------------|
| **Timeframe** | Zeitfenster das Verfügbarkeit oder Blockierung definiert |
| **Bookable Post** | Item oder Location, die via Timeframes verbunden werden |
| **Grid** | Zeitraster (in Minuten) für slots |
| **Slot** | Einzelnes Zeitintervall innerhalb eines Timeframes |
| **Restriction** | Sperrung (Holiday, Repair, etc.) |
| **Booking Rule** | Konfigurierbare Geschäftsregel für Buchungen |
| **GBFS** | General Bikeshare Feed Specification - offener Standard für Bike-Sharing |
| **Cloaked ID** | Verschleierter Vehicle ID für GBFS Privacy |
| **Advance Booking Days** | Wie viele Tage im Voraus gebucht werden kann |

---

## 20. API Best Practices (Mobile App Support)

### 21.1 API Versioning

**Empfohlen: URL Path Versioning**

```
https://api.commonsbooking.org/v1/bookings
https://api.commonsbooking.org/v2/bookings
```

| Vorteil | Nachteil |
|---------|----------|
| Explizit, transparent | Breaking Changes erfordern Client-Update |
| Einfach zu cachen/proxien | URLs ändern sich bei Major-Releases |
| Leicht zu dokumentieren | |

**Versioning Strategy:**

```
Major Version (v1, v2):
- Breaking Changes (API-Structure, Required Fields)
- Wird im URL-Path angegeben
- Clients müssen explizit migrieren

Minor Version (v1.1, v1.2):
- Neue optionale Felder
- Neue optionale Endpoints
- Rückwärtskompatibel
- Wird im Accept-Header angegeben (optional)

Patch Version:
- Bug Fixes
- Keine Änderung der API-Signatur
- Keine Versionierung nötig
```

### 21.2 Rate Limiting

**Rate Limit Tiers:**

| Tier | Limit | Window | Use Case |
|------|-------|--------|----------|
| Anonymous | 30/hour | 1h | Werbepartner, public data |
| Free | 100/hour | 1h | Testen, Preview |
| Basic | 1,000/hour | 1h | Normale Nutzer |
| Premium | 5,000/hour | 1h | Power-User, App-Entwickler |
| Enterprise | 50,000/hour | 1h | Admins, Integrationen |

**Response Headers (Standard):**

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1622548800
X-RateLimit-Window: 1h
Retry-After: 3600  (nur bei 429 Too Many Requests)
```

**Rate Limit Implementation (TypeScript):**

```typescript
class RateLimiter {
  private redis: Redis;
  private limits = {
    'anonymous': { requests: 30, window: 3600 },
    'free': { requests: 100, window: 3600 },
    'basic': { requests: 1000, window: 3600 },
    'premium': { requests: 5000, window: 3600 },
    'enterprise': { requests: 50000, window: 3600 }
  };

  async check(clientId: string, plan: string): Promise<RateLimitResult> {
    const limit = this.limits[plan];
    const key = `rate_limit:${clientId}:${plan}`;

    const current = await this.redis.incr(key);
    if (current === 1) {
      await this.redis.expire(key, limit.window);
    }

    const ttl = await this.redis.ttl(key);

    return {
      allowed: current <= limit.requests,
      remaining: Math.max(0, limit.requests - current),
      reset: Date.now() + (ttl * 1000),
      window: limit.window
    };
  }

  async consume(clientId: string, plan: string): Promise<void> {
    const result = await this.check(clientId, plan);
    if (!result.allowed) {
      throw new RateLimitExceededError({
        retryAfter: Math.ceil((result.reset - Date.now()) / 1000),
        limit: this.limits[plan].requests,
        window: this.limits[plan].window
      });
    }
  }
}
```

### 21.3 Robustness Patterns

#### Circuit Breaker

```typescript
// Verhindert Kaskaden-Ausfälle bei externen Services
class CircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private threshold = 5;
  private timeout = 30000; // 30s

  async call<T>(service: string, fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailure > this.timeout) {
        this.state = 'half-open'; // Probe
      } else {
        throw new ServiceUnavailableError(`${service} unavailable (circuit open)`);
      }
    }

    try {
      const result = await fn();
      this.failures = 0;
      this.state = 'closed';
      return result;
    } catch (e) {
      this.failures++;
      this.lastFailure = Date.now();
      if (this.failures >= this.threshold) {
        this.state = 'open'; // Öffnet Circuit
      }
      throw e;
    }
  }
}

// Usage
const result = await breaker.call('geocoding', () => geocodeService.geocode(address));
```

#### Retry with Exponential Backoff

```typescript
// Automatische Wiederholung bei transienten Fehlern
class RetryHandler {
  async withRetry<T>(
    fn: () => Promise<T>,
    options = { maxRetries: 3, baseDelay: 1000, maxDelay: 30000 }
  ): Promise<T> {
    let lastError: Error;

    for (let attempt = 0; attempt < options.maxRetries; attempt++) {
      try {
        return await fn();
      } catch (e) {
        lastError = e;

        // Nur Retry bei transienten Fehlern
        if (!this.isRetryable(e)) {
          throw e;
        }

        if (attempt < options.maxRetries - 1) {
          const delay = Math.min(
            options.baseDelay * Math.pow(2, attempt),
            options.maxDelay
          );
          // Jitter hinzufügen um Thundering Herd zu vermeiden
          await sleep(delay + Math.random() * 1000);
        }
      }
    }

    throw lastError!;
  }

  private isRetryable(error: Error): boolean {
    // 5xx Errors, Timeout, Network Error
    const retryableCodes = ['ECONNRESET', 'ETIMEDOUT', 'ENOTFOUND', '500', '502', '503', '504'];
    return retryableCodes.some(code => error.message.includes(code));
  }
}
```

#### Request Deduplication

```typescript
// Verhindert doppelte Requests bei gleichzeitigen Retries
class RequestDeduplicator {
  private pending = new Map<string, Promise<any>>();

  async deduplicate<T>(key: string, fn: () => Promise<T>): Promise<T> {
    const existing = this.pending.get(key);
    if (existing) {
      return existing as Promise<T>;
    }

    const promise = fn().finally(() => this.pending.delete(key));
    this.pending.set(key, promise);
    return promise;
  }
}

// Usage: Verhindert 10 gleichzeitige Availability-Checks
const result = await deduplicator.deduplicate(
  `availability:${itemId}:${date}`,
  () => api.checkAvailability(itemId, date)
);
```

### 21.4 API Middleware Stack

```typescript
// Express.js / Fastify Middleware Chain
import express from 'express';
import rateLimit from 'express-rate-limit';
import { circuitBreaker } from 'opossum';

const app = express();

// 1. Trust Proxy (für Rate Limiting hinter Load Balancer)
app.set('trust proxy', 1);

// 2. Global Rate Limiting
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,
  skip: (req) => req.path === '/health'
});
app.use(globalLimiter);

// 3. JWT Authentication Middleware
app.use('/api', authenticateJWT({
  issuer: process.env.KEYCLOAK_ISSUER,
  audience: 'commonsbooking-api',
  jwksUri: process.env.KEYCLOAK_JWKS_URI
}));

// 4. API-spezifische Rate Limits
app.use('/api/v1', rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 5000,
  keyGenerator: (req) => req.user?.sub || req.ip,
  standardHeaders: true,
  handler: (req, res) => {
    res.status(429).json({
      error: 'TOO_MANY_REQUESTS',
      message: 'Rate limit exceeded',
      retryAfter: res.get('Retry-After') || 3600
    });
  }
}));

// 5. Response Caching
app.use('/api/v1/availability', cacheMiddleware({
  maxAge: 300,
  etag: true,
  varyBy: ['itemId', 'locationId', 'startDate', 'endDate']
}));

// 6. Schema Validation
app.post('/api/v1/bookings',
  validateBody(bookingSchema),
  bookingController.create
);
```

### 21.5 Token Management (Keycloak)

```typescript
// Token Manager mit Auto-Refresh
class TokenManager {
  private accessToken: string | null = null;
  private refreshToken: string | null = null;
  private tokenExpires: number = 0;

  constructor(private keycloakUrl: string) {}

  async getAccessToken(): Promise<string> {
    if (this.isExpired()) {
      await this.refresh();
    }
    return this.accessToken!;
  }

  async refresh(): Promise<void> {
    if (!this.refreshToken) {
      throw new AuthError('No refresh token available');
    }

    const response = await fetch(`${this.keycloakUrl}/realms/commonsbooking/protocol/openid-connect/token`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        grant_type: 'refresh_token',
        refresh_token: this.refreshToken,
        client_id: 'commonsbooking-mobile'
      })
    });

    if (!response.ok) {
      throw new AuthError('Token refresh failed');
    }

    const data = await response.json();
    this.accessToken = data.access_token;
    this.refreshToken = data.refresh_token;
    this.tokenExpires = Date.now() + (data.expires_in * 1000);

    // Secure Storage für Mobile
    await SecureStorage.set('refresh_token', this.refreshToken);
  }

  private isExpired(): boolean {
    return !this.accessToken || Date.now() >= this.tokenExpires - 60000; // 1min buffer
  }
}

// API Client mit Auto-Retry und Token-Refresh
class ApiClient {
  constructor(
    private baseUrl: string,
    private tokenManager: TokenManager
  ) {}

  async request<T>(config: RequestConfig): Promise<T> {
    const token = await this.tokenManager.getAccessToken();

    const response = await fetch(`${this.baseUrl}${config.path}`, {
      method: config.method,
      headers: {
        ...config.headers,
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: config.body ? JSON.stringify(config.body) : undefined
    });

    if (response.status === 401) {
      await this.tokenManager.refresh();
      return this.request(config); // Retry mit neuem Token
    }

    if (response.status === 429) {
      const retryAfter = parseInt(response.headers.get('Retry-After') || '60');
      await sleep(retryAfter * 1000);
      return this.request(config); // Retry nach Wartezeit
    }

    if (!response.ok) {
      const error = await response.json();
      throw new ApiError(error.code, error.message);
    }

    return response.json();
  }
}
```

### 21.6 Response Format

**Success Response:**

```json
{
  "data": { ... },
  "meta": {
    "version": "1.0",
    "timestamp": "2026-05-13T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

**Error Response:**

```json
{
  "error": {
    "code": "BOOKING_SLOT_UNAVAILABLE",
    "message": "The requested time slot is no longer available",
    "details": {
      "slot": "2026-05-15",
      "itemId": "123"
    }
  },
  "meta": {
    "requestId": "req_abc123",
    "version": "1.0"
  }
}
```

**Pagination Response:**

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "total": 156,
    "totalPages": 8,
    "links": {
      "self": "/v1/bookings?page=1",
      "next": "/v1/bookings?page=2",
      "prev": null,
      "first": "/v1/bookings?page=1",
      "last": "/v1/bookings?page=8"
    }
  }
}
```

### 21.7 Error Codes

| Code | HTTP Status | Beschreibung |
|------|-------------|--------------|
| `UNAUTHORIZED` | 401 | Keine gültige Authentifizierung |
| `FORBIDDEN` | 403 | Keine Berechtigung für Resource |
| `NOT_FOUND` | 404 | Resource nicht gefunden |
| `RATE_LIMIT_EXCEEDED` | 429 | Rate Limit erreicht |
| `VALIDATION_ERROR` | 400 | Ungültige Request-Daten |
| `BOOKING_SLOT_UNAVAILABLE` | 409 | Slot nicht mehr verfügbar |
| `BOOKING_CONFLICT` | 409 | Buchungskonflikt (Overlapping) |
| `INTERNAL_ERROR` | 500 | Interner Server-Fehler |
| `SERVICE_UNAVAILABLE` | 503 | Service temporär nicht verfügbar |

### 21.8 Offline Support (Mobile)

```typescript
// Offline Cache für Mobile App
class OfflineCache {
  constructor(private storage: MMKV) {}

  async get<T>(key: string): Promise<T | null> {
    const entry = this.storage.getString(key);
    if (!entry) return null;

    const { data, expires } = JSON.parse(entry);
    if (expires && Date.now() > expires) {
      this.storage.delete(key);
      return null;
    }
    return data as T;
  }

  async set<T>(key: string, data: T, ttlSeconds = 300): Promise<void> {
    const entry = {
      data,
      expires: ttlSeconds ? Date.now() + (ttlSeconds * 1000) : null
    };
    this.storage.set(key, JSON.stringify(entry));
  }

  async clear(): Promise<void> {
    this.storage.clearAll();
  }
}

// Mobile App Strategy
class AvailabilityService {
  async getAvailability(itemId: string, date: string): Promise<Availability> {
    const cacheKey = `availability:${itemId}:${date}`;

    // 1. Show cached immediately
    const cached = await this.cache.get<Availability>(cacheKey);
    if (cached) {
      this.refreshInBackground(cacheKey); // Non-blocking refresh
      return cached;
    }

    // 2. Wait for fresh data
    return await this.api.getAvailability(itemId, date);
  }

  private async refreshInBackground(key: string): Promise<void> {
    try {
      const fresh = await this.api.getAvailability(...extractIds(key));
      await this.cache.set(key, fresh, 300);
    } catch (e) {
      // Silent fail - cached data still shown
    }
  }
}
```

### 21.9 Health Checks

```typescript
// Health Endpoint für Kubernetes/Load Balancer
app.get('/health', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    keycloak: await checkKeycloak()
  };

  const healthy = Object.values(checks).every(c => c.status === 'healthy');

  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'healthy' : 'unhealthy',
    checks,
    timestamp: new Date().toISOString(),
    version: process.env.API_VERSION
  });
});

// Readiness Check (für K8s readinessProbe)
app.get('/ready', async (req, res) => {
  const dbReady = await checkDatabase();
  if (dbReady.status === 'healthy') {
    res.json({ ready: true });
  } else {
    res.status(503).json({ ready: false });
  }
});
```

---

## 21. Open Questions / To Clarify

1. **User Registration**: Soll das System eigene User-Registrierung unterstützen oder nur bestehende WordPress-User?
2. **Payment**: Ist Payment-Integration geplant? Aktuell keine Payment-Funktionalität.
3. **Multi-location**: Sollen Items an mehreren Standorten gleichzeitig verfügbar sein?
4. **Recurring Bookings**: Wird automatische Wiederholung von Buchungen benötigt?
5. **Waitlist**: Soll ein Wartelisten-System implementiert werden?
6. **API Rate Limits**: Sollen API-Limits implementiert werden?
7. **Webhooks**: Für welche Events sollen Webhooks ausgelöst werden?

### Mobile App spezifisch

8. **Mobile App**: Ist eine native iOS/Android App geplant? Wenn ja, wird die bestehende WP-Installation als Backend genutzt oder ein komplett neues Backend?
9. **Offline Support**: Soll die App offline-fähig sein (z.B. Buchungen zwischenspeichern)?
10. **Push Notifications**: Welche Push-Provider (FCM/APNS)?
11. **Biometric Auth**: FaceID/TouchID für App-Login?
12. **Deep Linking**: Sollen Deeplinks für Buchungen (z.B. `commonsbooking://booking/123`) unterstützt werden?
13. **QR Code Scanning**: Sollen Booking-Codes per QR gescannt werden können (Admin-Funktion)?
14. **Barcode/Booking Code Display**: Sollen Codes in der App angezeigt werden für Abholung?

---

## Appendix A: File Structure (Original WP Plugin)

```
src/
├── API/
│   ├── AvailabilityRoute.php
│   ├── BaseRoute.php
│   ├── GBFS/
│   │   ├── BaseRoute.php
│   │   ├── Discovery.php
│   │   ├── StationInformation.php
│   │   ├── StationStatus.php
│   │   ├── SystemInformation.php
│   │   ├── VehicleAvailability.php
│   │   └── VehicleStatus.php
│   ├── ItemsRoute.php
│   ├── LocationsRoute.php
│   ├── OwnersRoute.php
│   ├── ProjectsRoute.php
│   └── Share.php
├── CB/
│   ├── CB.php
│   └── CB1UserFields.php
├── Exception/
│   ├── BookingCodeException.php
│   ├── BookingDeniedException.php
│   ├── BookingRuleException.php
│   ├── ExportException.php
│   ├── OverlappingException.php
│   ├── PostException.php
│   └── TimeframeInvalidException.php
├── Helper/
│   ├── API.php
│   ├── GeoCodeService.php
│   ├── GeoHelper.php
│   ├── Helper.php
│   ├── NominatimGeoCodeService.php
│   └── WordPress.php
├── Map/
│   ├── BaseShortcode.php
│   ├── LocationMapAdmin.php
│   ├── MapData.php
│   ├── MapFilter.php
│   ├── MapItemAvailable.php
│   ├── MapShortcode.php
│   └── SearchShortcode.php
├── Messages/
│   ├── AdminMessage.php
│   ├── BookingCodesMessage.php
│   ├── BookingMessage.php
│   ├── BookingReminderMessage.php
│   ├── LocationBookingReminderMessage.php
│   ├── Message.php
│   └── RestrictionMessage.php
├── Migration/
│   ├── Booking.php
│   └── Migration.php
├── Model/
│   ├── BookablePost.php
│   ├── Booking.php
│   ├── BookingCode.php
│   ├── Calendar.php
│   ├── CustomPost.php
│   ├── Day.php
│   ├── Item.php
│   ├── Location.php
│   ├── Map.php
│   ├── MessageRecipient.php
│   ├── Restriction.php
│   ├── Timeframe.php
│   └── Week.php
├── Plugin.php
├── Repository/
│   ├── ApiShares.php
│   ├── BookablePost.php
│   ├── Booking.php
│   ├── BookingCodes.php
│   ├── CB1.php
│   ├── Item.php
│   ├── Location.php
│   ├── PostRepository.php
│   ├── Restriction.php
│   ├── Timeframe.php
│   └── UserRepository.php
├── Service/
│   ├── Booking.php
│   ├── BookingCodes.php
│   ├── BookingRule.php
│   ├── BookingRuleApplied.php
│   ├── Cache.php
│   ├── Holiday.php
│   ├── iCalendar.php
│   ├── MassOperations.php
│   ├── Scheduler.php
│   ├── TimeframeExport.php
│   └── Upgrade.php
├── Settings/
│   └── Settings.php
└── View/
    ├── Admin/
    │   └── Filter.php
    ├── Booking.php
    ├── BookingCodes.php
    ├── Calendar.php
    ├── Dashboard.php
    ├── Item.php
    ├── Location.php
    ├── Map.php
    ├── MassOperations.php
    ├── Migration.php
    ├── Restriction.php
    ├── TimeframeExport.php
    └── View.php
```

---

## Appendix B: Database Schema (WordPress Meta)

```sql
-- Location Meta
_meta _cb_location_lat = float
_meta _cb_location_lng = float
_meta _cb_location_address = string
_meta _cb_location_pickup_instructions = string

-- Item Meta
_meta _cb_item_category = int
_meta _cb_item_template = string

-- Booking Meta
_meta _cb_booking_code = string
_meta _cb_booking_user_id = int
_meta _cb_booking_start = timestamp
_meta _cb_booking_end = timestamp

-- Timeframe Meta
_meta _cb_timeframe_grid = json
_meta _cb_timeframe_advance = int
_meta _cb_timeframe_min_days = int
_meta _cb_timeframe_max_days = int
_meta _cb_timeframe_repeat = string

-- Restriction Meta
_meta _cb_restriction_type = string
_meta _cb_restriction_repeat = string
_meta _cb_restriction_state = string

-- Booking Codes Table
CREATE TABLE cb_bookingcodes (
  date DATE NOT NULL,
  timeframe BIGINT UNSIGNED NOT NULL,
  location BIGINT UNSIGNED NOT NULL,
  item BIGINT UNSIGNED NOT NULL,
  code VARCHAR(100) NOT NULL,
  PRIMARY KEY (date, timeframe, location, item, code)
);
```

---

*Document Version: 1.0*
*Generated from: CommonsBooking WordPress Plugin v2.10.10*
*Date: 2026-05-13*