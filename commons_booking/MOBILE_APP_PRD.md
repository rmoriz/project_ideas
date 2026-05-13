# CommonsBooking Mobile App - Product Requirements Document

## Version: 1.0
### Basierend auf: CommonsBooking PRD v1.0

---

## 1. Overview

### 1.1 Purpose

Die CommonsBooking Mobile App ermöglicht Nutzern das Browsen von Verleih-Stationen, das Buchen von Gegenständen und die Verwaltung ihrer Buchungen - mit einem nativen Look & Feel für sowohl iOS als auch Android, aus einer gemeinsamen Codebasis.

### 1.2 Core Value Proposition

- **Native Experience**: Die App fühlt sich wie eine echte native App an, nicht wie eine Web-App
- **Offline-First**: Buchungen und Verfügbarkeit können auch ohne Verbindung eingesehen werden
- **Einfachheit**: Schneller Zugriff auf das Wesentliche - Finden und Buchen

### 1.3 Zielgruppe

| Segment | Bedürfnisse |
|---------|-------------|
| **Nutzer (Bucher)** | Schnell ein Fahrrad/Werkzeug finden und buchen |
| **Team Members** | Ausleihe/Rücknahme bestätigen, QR-Codes scannen |
| **Station Manager** | Station-Statistiken, Buchungen verwalten |

---

## 2. Design Philosophy

### 2.1 "Write Once, Run Native Everywhere"

```
┌─────────────────────────────────────────────────────────────┐
│                    Shared Codebase                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              React Native / Expo                      │   │
│  │  - TypeScript                                         │   │
│  │  - Unified UI Components                              │   │
│  │  - Shared Business Logic                              │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│          iOS             │     │         Android          │
│  ┌───────────────────┐   │     │  ┌───────────────────┐   │
│  │   SwiftUI Bridge   │   │     │  │  Material Bridge   │   │
│  │  - Native Controls │   │     │  │  - Material You    │   │
│  │  - iOS Design Lang │   │     │  │  - Android Widgets │   │
│  │  - Haptics/Taptic  │   │     │  │  - Notifications   │   │
│  └───────────────────┘   │     │  └───────────────────┘   │
│  - App Store ready       │     │  - Play Store ready      │
└─────────────────────────┘     └─────────────────────────┘
```

### 2.2 Native Adaptation Layers

Die App verwendet **adaptive Komponenten**, die automatisch das native Verhalten der Zielplattform emulieren:

| Component | iOS Behavior | Android Behavior |
|-----------|--------------|------------------|
| **Buttons** | SF Symbols, rounded, blur bg | Material buttons, elevation |
| **Navigation** | Large titles, swipe back, navbar blur | Edge-to-edge, back gesture, dynamic colors |
| **Lists** | SF Style separators, swipe actions | Material ripple, dividers |
| **Forms** | Floating labels, grouped style | Outlined text fields |
| **Modals** | Card presentation, detents | Bottom sheets, Material dialogs |
| **Typography** | SF Pro, Dynamic Type | Roboto, sp units for scaling |
| **Colors** | Light/dark system appearance | Material You theming |
| **Haptics** | Impact, selection, notification | Vibration effects |

### 2.3 Skeleton Loading

Schnelle Ladezeiten durch Skeleton-Screens:
```
┌─────────────────────────────────┐
│  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  │
│  ▓▓▓▓▓▓▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓▓  │
│  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  │
└─────────────────────────────────┘
```

---

## 3. Technology Stack

### 3.1 Core Framework

| Technology | Rationale |
|------------|-----------|
| **React Native 0.76+** | New Architecture (Fabric), TypeScript, strong ecosystem |
| **Expo SDK 52+** | Simplified native builds, OTA updates, push notifications |
| **Expo Router** | File-based routing, deep linking |
| **TypeScript 5** | Type safety across codebase |

### 3.2 Key Libraries

| Category | Library | Purpose |
|----------|---------|---------|
| **State Management** | Zustand | Lightweight, TypeScript-first |
| **Data Fetching** | TanStack Query | Caching, offline support, background refetch |
| **Navigation** | expo-router | File-based routing, native stacks |
| **UI Components** | React Native Paper | Material Design components |
| **Adaptive UI** | react-native-ui-lib | Native feel components |
| **Forms** | React Hook Form + Zod | Schema validation |
| **Storage** | expo-secure-store | Secure token storage |
| **Offline** | WatermelonDB | Local SQLite with sync |
| **Maps** | react-native-maps | Leaflet/MapKit integration |
| **QR/Barcode** | expo-camera + zxing | Scanner for booking codes |
| **Notifications** | expo-notifications | Push notifications |
| **Haptics** | expo-haptics | Native haptic feedback |

### 3.3 Architecture

```
src/
├── app/                    # Expo Router pages
│   ├── (auth)/            # Auth screens (login, register)
│   ├── (tabs)/            # Tab navigation
│   │   ├── home/          # Home / Discover
│   │   ├── search/        # Search & Filter
│   │   ├── bookings/      # My Bookings
│   │   └── profile/       # Profile & Settings
│   └── (modals)/          # Modal screens
│
├── components/            # Shared UI components
│   ├── atoms/             # Basic components (Button, Text, Icon)
│   ├── molecules/         # Composed components (BookingCard, StationListItem)
│   ├── organisms/         # Complex components (AvailabilityCalendar)
│   └── templates/         # Page templates
│
├── features/              # Feature-based modules
│   ├── auth/              # Authentication feature
│   ├── booking/           # Booking flow feature
│   ├── stations/          # Station browsing feature
│   ├── scanner/           # QR code scanning feature
│   └── admin/             # Station manager feature
│
├── hooks/                 # Custom React hooks
├── services/              # API services (Keycloak, API client)
├── stores/                # Zustand stores
├── utils/                 # Utility functions
├── native/                # Native module bridges
│   ├── ios/               # iOS-specific adaptations
│   └── android/           # Android-specific adaptations
└── theme/                 # Theming (adaptive)
    ├── tokens/            # Design tokens
    ├── variants/          # Platform variants
    └── context/           # Theme context
```

---

## 4. Feature Requirements

### 4.1 Priority Matrix

| Feature | P0 (Must Have) | P1 (Should Have) | P2 (Nice to Have) |
|---------|----------------|------------------|-------------------|
| Station browsing | ✅ | | |
| Item availability view | ✅ | | |
| Booking creation | ✅ | | |
| User authentication | ✅ | | |
| My Bookings list | ✅ | | |
| Booking cancellation | ✅ | | |
| Push notifications | ✅ | | |
| Offline mode | ✅ | | |
| QR code display | | ✅ | |
| QR code scanner (Admin) | | ✅ | |
| Station Manager dashboard | | ✅ | |
| Deep linking | | ✅ | |
| Biometric auth | | | ✅ |
| Apple Watch companion | | | ✅ |
| Widgets | | | ✅ |

### 4.2 P0 Features (Must Have)

#### F1: Station Browser
```
User Story: Als Nutzer möchte ich Stationen in meiner Nähe sehen, um zu wissen wo ich einen Gegenstand ausleihen kann.

Akzeptanzkriterien:
- Karte zeigt alle Stationen mit Markern
- Listenansicht der Stationen
- Station-Detail mit Adresse, Öffnungszeiten, verfügbaren Items
- Navigation zu Station (Maps app integration)
```

#### F2: Availability Calendar
```
User Story: Als Nutzer möchte ich sehen welche Tage ein Item verfügbar ist, um eine Buchung zu planen.

Akzeptanzkriterien:
- Kalender zeigt verfügbare (grün) und blockierte (rot) Tage
- Kein Backend-Call für bereits geladene Daten (Offline)
- Slot-Auswahl für Start-/Enddatum
- Min/max booking days Info
```

#### F3: Booking Creation
```
User Story: Als Nutzer möchte ich eine Buchung in wenigen Schritten erstellen.

Akzeptanzkriterien:
- 3-Schritt Flow: Item wählen → Zeitraum wählen → Bestätigen
- Keine Registrierung für öffentliche Stationen (optional)
- Booking Code wird nach Bestätigung angezeigt
- Bestätigungs-E-Mail wird gesendet
```

#### F4: User Authentication
```
User Story: Als Nutzer möchte ich mich mit meinem CommonsBooking Konto anmelden.

Akzeptanzkriterien:
- Email + Password Login
- Keycloak als Identity Provider
- Token-Refresh automatisch
- Logout löscht alle lokalen Daten
```

#### F5: My Bookings
```
User Story: Als Nutzer möchte ich meine aktiven und vergangenen Buchungen sehen.

Akzeptanzkriterien:
- Liste sortiert nach Datum (bevorstehend zuerst)
- Status-Badge (unconfirmed, confirmed, canceled)
- Tap öffnet Buchungs-Detail
- Pull-to-refresh
```

#### F6: Offline Mode
```
User Story: Als Nutzer möchte ich auch ohne Internetverbindung meine Buchungen einsehen können.

Akzeptanzkriterien:
- Letzte 50 Buchungen cached lokal
- Station-Liste cached
- Availability nicht offline-editable (nur read)
- Sync-Status indicator wenn offline
```

### 4.3 P1 Features (Should Have)

#### F7: QR Code Display & Scanner
```
User Story: Als Nutzer möchte ich meinen Booking-Code als QR anzeigen, damit der Team Member scannen kann.

Akzeptanzkriterien:
- QR Code im Booking-Detail
- Fullscreen Mode für besseres Scannen
- Helles Display während Anzeige

Als Team Member möchte ich den Code scannen um die Ausleihe zu bestätigen.
- Kamera öffnet sich automatisch
- Code wird validiert
- Bestätigungs-Dialog
```

#### F8: Push Notifications
```
User Story: Als Nutzer möchte ich Erinnerungen und Status-Updates meiner Buchungen erhalten.

Akzeptanzkriterien:
- Erinnerung X Stunden vor Buchung
- Notification bei Bestätigung/Stornierung
- Station Manager: Notification bei neuer Buchung
```

#### F9: Station Manager Dashboard
```
User Story: Als Station Manager möchte ich meine Station verwalten und Buchungen bearbeiten.

Akzeptanzkriterien:
- Heutige Ausleihen/Rücknahmen
- Quick Actions: Bestätigen, Stornieren
- Tagesstatistik
- Team Members Liste
```

### 4.4 P2 Features (Nice to Have)

#### F10: Biometric Authentication
```
- FaceID / TouchID für App-Login
- Android: Fingerprint / Face Unlock
- Fallback auf PIN/Password
```

#### F11: Deep Linking
```
commonsbooking://bookings/123
commonsbooking://stations/456
```

---

## 5. UI/UX Specification

### 5.1 Screen Structure

```
┌─────────────────────────────────────────────┐
│                   iOS                        │
├─────────────────────────────────────────────┤
│                                             │
│  ┌─ Tab Bar ─────────────────────────────┐  │
│  │ Home │ Search │ Bookings │ Profile   │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  Screen Flow:                               │
│                                             │
│  (Unauthenticated)                          │
│  ├── Welcome Screen                         │
│  │   ├── Station Map Preview               │
│  │   └── "Anmelden" / "Registrieren"       │
│  │                                          │
│  └── Login / Register                       │
│      └── Forgot Password                    │
│                                             │
│  (Authenticated - Tab Navigation)           │
│  ├── Home Tab                               │
│  │   ├── Nearby Stations Map               │
│  │   ├── Featured Items                    │
│  │   └── Quick Book (favorites)            │
│  │                                          │
│  ├── Search Tab                             │
│  │   ├── Search Bar                        │
│  │   ├── Filter (Category, Date, Distance) │
│  │   └── Results List                      │
│  │                                          │
│  ├── Bookings Tab                           │
│  │   ├── Active Bookings                   │
│  │   ├── Past Bookings                     │
│  │   └── Booking Detail                    │
│  │       └── QR Code Display               │
│  │                                          │
│  └── Profile Tab                            │
│      ├── User Info                          │
│      ├── Station Manager Section (if role)  │
│      │   └── Station Dashboard              │
│      ├── Settings                           │
│      └── Logout                             │
│                                             │
│  (Modal Screens)                            │
│  ├── Station Detail                         │
│  │   ├── Map                                │
│  │   ├── Item List                          │
│  │   └── Availability Calendar             │
│  │                                          │
│  ├── Booking Creation Flow                  │
│  │   ├── Step 1: Select Item               │
│  │   ├── Step 2: Select Dates              │
│  │   └── Step 3: Confirm                   │
│  │                                          │
│  ├── QR Scanner (Admin)                     │
│  └── Booking Confirmation                   │
│                                             │
└─────────────────────────────────────────────┘
```

### 5.2 Color System

**Light Mode:**

| Token | Hex | Usage |
|-------|-----|-------|
| `primary` | #007AFF | CTA buttons, links |
| `primaryContainer` | #E3F2FD | Selected states |
| `secondary` | #5856D6 | Secondary actions |
| `success` | #34C759 | Confirmed, available |
| `warning` | #FF9500 | Pending, reminder |
| `error` | #FF3B30 | Error, canceled |
| `surface` | #FFFFFF | Cards, backgrounds |
| `background` | #F2F2F7 | Screen background |
| `onSurface` | #1C1C1E | Primary text |
| `onSurfaceVariant` | #8E8E93 | Secondary text |

**Dark Mode:** 
System colors follow iOS/Android dark mode automatically.

### 5.3 Typography

| Style | iOS | Android | Usage |
|-------|-----|---------|-------|
| `largeTitle` | 34pt, Bold | 32sp, Bold | Screen titles |
| `title1` | 28pt, Bold | 28sp, Bold | Section headers |
| `title2` | 22pt, Bold | 22sp, Bold | Card titles |
| `headline` | 17pt, Semibold | 16sp, Medium | List item titles |
| `body` | 17pt, Regular | 16sp, Regular | Body text |
| `callout` | 16pt, Regular | 14sp, Regular | Secondary text |
| `footnote` | 13pt, Regular | 12sp, Regular | Captions |
| `caption` | 12pt, Regular | 11sp, Regular | Badges, timestamps |

### 5.4 Spacing System (8pt Grid)

| Token | Value | Usage |
|-------|-------|-------|
| `xs` | 4px | Inline spacing |
| `sm` | 8px | Icon gaps |
| `md` | 16px | Component padding |
| `lg` | 24px | Section spacing |
| `xl` | 32px | Screen margins |
| `xxl` | 48px | Major sections |

### 5.5 Component Library

#### Atoms

| Component | Variants | States |
|-----------|----------|--------|
| `Button` | primary, secondary, outline, ghost | default, pressed, disabled, loading |
| `IconButton` | filled, outlined, ghost | default, pressed, disabled |
| `Text` | largeTitle, title1, title2, headline, body, callout, footnote, caption | bold, regular, medium |
| `Badge` | success, warning, error, info | - |
| `Avatar` | sm, md, lg | with image, initials, placeholder |
| `Input` | text, email, password, search | default, focused, error, disabled |
| `Checkbox` | unchecked, checked, indeterminate | default, pressed, disabled |

#### Molecules

| Component | Composition |
|-----------|-------------|
| `BookingCard` | StatusBadge + ItemImage + Title + DateRange + Location + ActionButton |
| `StationListItem` | Avatar + Name + Address + Distance + Chevron |
| `ItemCard` | Image + Title + AvailabilityBadge + CategoryBadge |
| `DateRangePicker` | Calendar icon + StartDate + Arrow + EndDate |
| `TimeSlotPicker` | Grid of TimeSlot items |
| `UserInfo` | Avatar + Name + Email + LogoutButton |

#### Organisms

| Component | Description |
|-----------|-------------|
| `StationMap` | Full-screen map with markers, current location, search |
| `AvailabilityCalendar` | Month view with color-coded days, slot selection |
| `BookingFlow` | Multi-step form with progress indicator |
| `BookingList` | FlatList with pull-to-refresh, section headers |
| `QRCodeDisplay` | Fullscreen QR with booking details, close button |
| `QRScanner` | Camera preview with overlay, flashlight toggle |

---

## 6. Technical Requirements

### 6.1 API Integration

Die App kommuniziert mit der CommonsBooking API (siehe PRD.md Section 4 & 20).

**Base Configuration:**
```typescript
const API_CONFIG = {
  baseUrl: __DEV__
    ? 'http://localhost:3000/api/v1'
    : 'https://api.commonsbooking.org/api/v1',
  timeout: 15000,
  retries: 3,
};
```

**Authentication Flow:**
```typescript
// 1. Login with Keycloak
const keycloak = new Keycloak({
  url: 'https://auth.commonsbooking.org',
  realm: 'commonsbooking',
  clientId: 'commonsbooking-mobile',
});

// 2. Get tokens
const { accessToken, refreshToken } = await keycloak.login({
  username: email,
  password: password,
});

// 3. Store securely
await SecureStorage.setAsync('accessToken', accessToken);
await SecureStorage.setAsync('refreshToken', refreshToken);

// 4. API Client with auto-refresh
const apiClient = new ApiClient({
  baseUrl: API_CONFIG.baseUrl,
  getAccessToken: () => SecureStorage.getAsync('accessToken'),
  onTokenRefresh: (newToken) => SecureStorage.setAsync('accessToken', newToken),
});
```

### 6.2 Offline Data Strategy

```typescript
// WatermelonDB Schema
const schema = appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'bookings',
      columns: [
        { name: 'server_id', type: 'string' },
        { name: 'status', type: 'string' },
        { name: 'start_date', type: 'number' },
        { name: 'end_date', type: 'number' },
        { name: 'item_name', type: 'string' },
        { name: 'location_name', type: 'string' },
        { name: 'booking_code', type: 'string' },
        { name: 'synced_at', type: 'number' },
      ],
    }),
    tableSchema({
      name: 'stations',
      columns: [
        { name: 'server_id', type: 'string' },
        { name: 'name', type: 'string' },
        { name: 'address', type: 'string' },
        { name: 'latitude', type: 'number' },
        { name: 'longitude', type: 'number' },
        { name: 'synced_at', type: 'number' },
      ],
    }),
    // ...
  ],
});

// Sync Strategy
// 1. On app start: fetch latest data in background
// 2. On pull-to-refresh: force sync
// 3. On network restore: sync pending changes
// 4. Conflict resolution: server wins (last-write-wins)
```

### 6.3 Push Notifications

```typescript
// Notification Types
enum NotificationType {
  BOOKING_CONFIRMED = 'booking_confirmed',
  BOOKING_REMINDER = 'booking_reminder',
  BOOKING_CANCELED = 'booking_canceled',
  STATION_NEW_BOOKING = 'station_new_booking',
}

// Setup
async function setupNotifications() {
  const { status } = await Notifications.requestPermissionsAsync();

  if (status === 'granted') {
    // Get Expo Push Token
    const token = await Notifications.getExpoPushTokenAsync({
      projectId: 'commonsbooking-app',
    });

    // Send to backend
    await api.post('/users/me/push-token', { token: token.data });

    // Setup notification handlers
    Notifications.setNotificationHandler({
      handleNotification: async (notification) => ({
        shouldShowAlert: true,
        shouldPlaySound: true,
        shouldSetBadge: true,
      }),
    });
  }
}
```

### 6.4 Performance Targets

| Metric | Target |
|--------|--------|
| App Launch (cold) | < 2 seconds |
| Screen Transition | < 300ms |
| API Response (cached) | < 100ms |
| API Response (network) | < 2 seconds |
| Offline data access | < 50ms |
| Memory Usage | < 150MB |
| Battery Impact | Low (background sync limited) |

---

## 7. Platform-Specific Features

### 7.1 iOS

| Feature | Implementation |
|---------|----------------|
| **Dynamic Type** | Text scales with system settings |
| **Dark Mode** | Automatic with system preference |
| **Haptic Feedback** | expo-haptics for confirmations |
| **Safe Area** | SafeAreaProvider for notch/home indicator |
| **Share Sheet** | Native share for booking details |
| **App Clip** | Lightweight version for proximity |

### 7.2 Android

| Feature | Implementation |
|---------|----------------|
| **Material You** | Dynamic color extraction from wallpaper |
| **Edge-to-Edge** | Full screen with system bar handling |
| **Widgets** | Home screen widget for quick booking status |
| **Notification Channel** | Separate channels for reminders, updates |
| **Splash Screen** | Adaptive icon with splash |
| **Biometric** | Native fingerprint/face unlock |

---

## 8. Security Requirements

| Requirement | Implementation |
|-------------|----------------|
| **Token Storage** | expo-secure-store (Keychain/Keystore) |
| **API Communication** | HTTPS only, certificate pinning |
| **Local Data** | Encrypted SQLite for offline cache |
| **Session Timeout** | 30 min inactivity → re-auth required |
| **Biometric Gate** | Optional additional auth layer |

---

## 9. Accessibility

| Requirement | Implementation |
|-------------|----------------|
| **VoiceOver/TalkBack** | Semantic labels on all components |
| **Dynamic Type** | Text respects system font size |
| **Color Contrast** | WCAG AA minimum (4.5:1) |
| **Touch Targets** | Minimum 44x44pt |
| **Motion** | Respect reduced motion preference |

---

## 10. Testing Strategy

### 10.1 Test Pyramid

```
        ▲
       /E\           E2E Tests (Playwright)
      /2E2\          - Full user flows
     /──────\        - Critical paths
    /  I T  \        Integration Tests
   /──────────\      - Feature modules
  /  U  U  U  \      Unit Tests
 /──────────────\    - Utility functions
──────────────────    - Business logic
```

### 10.2 Unit Tests (Vitest)
- Utility functions
- Date/time formatting
- Validation schemas
- Store actions

### 10.3 Integration Tests (Testing Library)
- Component rendering
- Hook behavior
- API mocking

### 10.4 E2E Tests (Playwright)
- Login flow
- Booking creation
- Booking cancellation
- Offline scenarios

---

## 11. Build & Deployment

### 11.1 Build Configuration

```json
// app.json (Expo)
{
  "expo": {
    "name": "CommonsBooking",
    "slug": "commonsbooking",
    "version": "1.0.0",
    "ios": {
      "bundleIdentifier": "org.commonsbooking.app",
      "supportsTablet": true,
      "infoPlist": {
        "NSCameraUsageDescription": "Camera is used to scan booking QR codes",
        "NSLocationWhenInUseUsageDescription": "Location is used to show nearby stations"
      }
    },
    "android": {
      "package": "org.commonsbooking.app",
      "permissions": [
        "CAMERA",
        "ACCESS_FINE_LOCATION"
      ]
    }
  }
}
```

### 11.2 Release Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Develop   │───▶│   Beta      │───▶│   Release   │
│  (Push)     │    │  (TestFlight│    │  (App Store │
│             │    │   / Play)   │    │   / Play)   │
└─────────────┘    └─────────────┘    └─────────────┘
```

---

## 12. Open Questions

1. **Deep Linking Schema**: Soll `commonsbooking://` oder `https://commonsbooking.org/` verwendet werden?
2. **Apple Watch**: Soll eine Companion Watch App entwickelt werden?
3. **Widget Support**: Welche Widget-Grössen (small, medium, large)?
4. **App Clip**: Soll ein App Clip für spontane Nutzung entwickelt werden?
5. **Payment**: Wird In-App Payment benötigt?
6. **Localization**: Welche Sprachen neben Deutsch?

---

## Appendix A: User Flow Diagrams

### Booking Flow

```
┌──────────────────────────────────────────────────────────────┐
│                     BOOKING FLOW                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [Home] ──▶ [Station Detail] ──▶ [Item Selection]           │
│                                      │                       │
│                                      ▼                       │
│                               [Calendar View]                │
│                                      │                       │
│                                      ▼                       │
│                               [Date Range Selected]          │
│                                      │                       │
│                                      ▼                       │
│                              [Booking Confirmation]          │
│                                      │                       │
│                         ┌────────────┴────────────┐         │
│                         ▼                         ▼         │
│                   [Success]                  [Error]        │
│                         │                         │         │
│                         ▼                         ▼         │
│                  [Show QR Code]            [Show Message]   │
│                         │                    [Retry]        │
│                         ▼                                        │
│              [Booking in My List]                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Station Manager Flow

```
┌──────────────────────────────────────────────────────────────┐
│                  STATION MANAGER FLOW                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [Profile Tab] ──▶ [Station Manager Section]                │
│                            │                                 │
│                            ▼                                 │
│                    [Station Dashboard]                       │
│                            │                                 │
│              ┌─────────────┼─────────────┐                  │
│              ▼             ▼             ▼                  │
│      [Today's List]  [Statistics]  [Team Members]           │
│              │             │             │                  │
│              └─────────────┼─────────────┘                  │
│                            │                                 │
│                            ▼                                 │
│                  [Booking Actions]                           │
│                            │                                 │
│              ┌─────────────┼─────────────┐                  │
│              ▼             ▼             ▼                  │
│        [Confirm]    [Check-in]    [Cancel]                  │
│              │             │             │                  │
│              └─────────────┼─────────────┘                  │
│                            │                                 │
│                            ▼                                 │
│                    [QR Scanner]                              │
│                            │                                 │
│                            ▼                                 │
│               [Scan Booking Code]                            │
│                            │                                 │
│                            ▼                                 │
│                  [Action Confirmed]                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Appendix B: Component State Specifications

### Button States

| State | Visual | Behavior |
|-------|--------|----------|
| `default` | Primary color, full opacity | - |
| `pressed` | Darker shade, scale 0.98 | 100ms ease-out |
| `disabled` | Gray, 50% opacity | No interaction |
| `loading` | Spinner replaces text | No interaction |

### Card States

| State | Visual |
|-------|--------|
| `default` | Surface color, shadow-sm |
| `pressed` | Elevated shadow, scale 0.99 |
| `selected` | Primary border, light primary bg |

### List Item States

| State | Visual |
|-------|--------|
| `default` | Surface bg |
| `pressed` | Slight opacity reduction |
| `swiped` | Action buttons revealed |
| `disabled` | Gray text, no interaction |

---

*Document Version: 1.0*
*Based on: CommonsBooking PRD v1.0*
*Date: 2026-05-13*