# PRD: Trackwise - FlutterFlow to Clean Architecture Migration

## Document Information
- **Product**: Trackwise Item Tracking App
- **Project Type**: Architecture Migration & Enhancement
- **Version**: 1.0
- **Date**: 2026-01-02
- **Status**: Ready for Epic Creation
- **Epic**: `flutterflow-to-clean-architecture-migration`

## Executive Summary

Trackwise is a production-ready Flutter application for tracking physical items using a custom ESP32-based Bluetooth device. The app currently runs on FlutterFlow-generated code and needs to be migrated to a clean architecture implementation using the BLoC pattern while maintaining full compatibility with the existing ESP32 firmware and Firebase backend.

This migration will transform Trackwise into a maintainable, scalable codebase ready for commercial deployment on Google Play Store and Apple App Store as a free application (monetization deferred to Phase 2).

### Key Objectives
1. Migrate from FlutterFlow to clean Flutter architecture with BLoC pattern
2. Maintain 100% compatibility with existing ESP32 firmware (minimal firmware changes only)
3. Preserve all existing features and add commercial-grade capabilities
4. Prepare for App Store deployment (Google Play + Apple App Store)
5. Implement GDPR compliance (data export, account deletion)
6. Set up automated testing, versioning, and monitoring
7. Launch as free app (Phase 1), defer monetization to Phase 2

### Timeline & Collaboration Model
- **Duration**: 8-10 weeks
- **User Commitment**: 1-2 hours/day for testing and review
- **Claude Commitment**: Primary development work (coding, architecture, testing setup)
- **Development Approach**: In-place migration using git branch (`feature/clean-architecture`)

---

## 1. Product Context

### 1.1 Current State

**Application**: Trackwise Flutter App (FlutterFlow-generated)
- **Platform**: Flutter 3.0+
- **Deployment**: Currently in development/testing
- **Users**: Test users with Firebase test data
- **Backend**: Firebase (Firestore + Auth + Extensions)
- **Hardware**: Custom ESP32-based Bluetooth item tracker

**Technical Debt**:
- FlutterFlow-generated code (limited maintainability)
- Global state management (FFAppState singleton with ChangeNotifier)
- 21 custom actions + 2 custom widgets tightly coupled to FlutterFlow imports
- No separation of concerns (UI, business logic, data layer mixed)
- Limited error handling and validation
- No comprehensive testing infrastructure

**ESP32 Firmware**: C++ Arduino code (764 lines)
- **BLE Protocol**: Bidirectional streaming with 5 characteristics
- **Storage**: NVS (Non-Volatile Storage) for item persistence
- **RTC**: DS3231 for accurate timestamps
- **Features**: Item tracking, event logging, reminders, daily auto-reset

**Firebase Collections**:
- `/users/{userId}` - User profiles
- `/Item/{itemId}` - Item definitions and counts
- `/EventLog/{eventId}` - Event history from ESP32
- `/mail/{emailId}` - Email queue (Firebase Mail Extension)

### 1.2 Target State

**Application**: Trackwise Clean Architecture
- **Architecture**: Clean Architecture (Presentation → Domain → Data)
- **State Management**: BLoC pattern (flutter_bloc)
- **Dependency Injection**: GetIt service locator
- **Error Handling**: Dartz Either<Failure, Success>
- **Testing**: Unit tests, widget tests, integration tests
- **CI/CD**: Automated versioning, testing, and release notes

**Deployment Targets**:
- Google Play Store (Internal Testing → Production)
- Apple App Store (TestFlight → Production)

**Compliance**:
- GDPR-compliant (data export, account deletion)
- Privacy policy and data disclosures
- App Store guidelines compliance

**Business Model** (Phase 1):
- Free app with all features
- No in-app purchases or subscriptions
- Focus on perfect core functionality
- Monetization deferred to Phase 2

---

## 2. Features & Requirements

### 2.1 Core Features (Phase 1)

#### 2.1.1 Items CRUD
**Description**: Manage trackable items with Firestore persistence

**User Stories**:
- As a user, I can create a new item with name, increment value, and reminder settings
- As a user, I can view all my items in a list with current counts
- As a user, I can edit an item's name, increment value, and reminder settings
- As a user, I can delete an item (with confirmation)
- As a user, I can see real-time updates when the ESP32 increments counts

**Technical Requirements**:
- Firestore collection: `/Item/{itemId}`
- Real-time synchronization with ESP32
- Optimistic UI updates with rollback on failure
- Validation: name (required, max 30 chars), increment (1-100), reminder value (0-1000)

**Data Model**:
```dart
class Item {
  final String id;
  final String name;
  final int count;
  final int todayCount;
  final int incrementBy;
  final ReminderType reminder; // NONE, TARGET, INTERVAL, EVERY_TIME
  final int reminderValue;
  final DateTime lastResetTime;
  final DateTime lastUpdated;
  final String userId;
}
```

**Acceptance Criteria**:
- ✅ Create item with validation
- ✅ Edit item with validation
- ✅ Delete item with confirmation
- ✅ View list with real-time updates
- ✅ Handle concurrent updates from ESP32
- ✅ Show loading states and error messages

#### 2.1.2 Bluetooth (ESP32 Communication)
**Description**: Bidirectional BLE communication with ESP32 device

**User Stories**:
- As a user, I can scan for nearby Trackwise devices
- As a user, I can connect to a device and see connection status
- As a user, I can send my item list to the device
- As a user, I can receive real-time updates when the device increments counts
- As a user, I can receive event logs from the device
- As a user, I can disconnect from the device manually
- As a user, I experience automatic reconnection if the connection drops unexpectedly

**Technical Requirements**:
- Library: flutter_blue_plus
- BLE Service UUID: `12345678-1234-1234-1234-123456789000`
- Characteristics:
  - Read (`...789001`): Read item list from device
  - Notify (`...789002`): Receive real-time updates
  - Set Items (`...789008`): Send item list to device
  - Write (`...789010`): Send commands to device
- Chunking: 180-byte MTU with 30ms delay between chunks
- Chunk reassembly: Queue-based processing with newline delimiters
- Auto-reconnect: Scan and reconnect on unexpected disconnection
- Manual disconnect flag: Prevent auto-reconnect when user disconnects intentionally

**Message Formats**:
```dart
// App → ESP32 (Set Items)
{
  "type": "items",
  "data": [
    {
      "id": "item_123",
      "name": "Coffee",
      "count": 5,
      "todaycount": 2,
      "increment": 1,
      "reminder": "TARGET",
      "reminder_value": 10,
      "lastResetTime": 1672531200
    }
  ]
}

// ESP32 → App (Prefs Update)
{
  "type": "prefs",
  "data": [
    {
      "id": "item_123",
      "name": "Coffee",
      "count": 6,
      "todaycount": 3,
      "increment": 1,
      "reminder": "TARGET",
      "reminder_value": 10,
      "lastResetTime": 1672531200
    }
  ],
  "selected_id": "item_123"
}

// ESP32 → App (Event)
{
  "type": "event",
  "data": {
    "itemId": "item_123",
    "event": "increment",
    "timestamp": 1672531200,
    "count": 6,
    "increment": 1
  }
}
```

**Acceptance Criteria**:
- ✅ Scan for devices with timeout (15 seconds)
- ✅ Connect to device and discover services
- ✅ Send item list to device (chunked, with progress indicator)
- ✅ Receive real-time prefs updates
- ✅ Receive real-time event updates
- ✅ Handle chunked messages correctly (reassemble)
- ✅ Auto-reconnect on unexpected disconnection
- ✅ Prevent auto-reconnect on manual disconnect
- ✅ Show connection status in UI
- ✅ Handle BLE errors gracefully (permissions, Bluetooth off, device not found)

#### 2.1.3 EventLog
**Description**: Real-time event ingestion from ESP32 to Firestore

**User Stories**:
- As a user, I see new events appear in real-time when I use the device
- As a user, I can view event history filtered by date range and item
- As a user, I can see event details (item name, event type, timestamp, count)

**Technical Requirements**:
- Firestore collection: `/EventLog/{eventId}`
- Event ID format: `{itemId}-{event}-{timestamp}-{count}`
- Batch writes for performance (up to 500 writes per batch)
- Real-time listeners on detail page
- Event types: `increment`, `decrement`, `reset`, `switch`

**Data Model**:
```dart
class EventLog {
  final String id;
  final DateTime createdTime;
  final String itemId;
  final String eventName;
  final int increment;
  final int currentCount;
  final String userId;
}
```

**Special Handling**:
- `switch` events update `lastUpdated` timestamp on Item (don't create EventLog entry)
- Batch processing for bulk event ingestion from device
- Activation state tracking (FFAppState.isactivated)

**Acceptance Criteria**:
- ✅ Receive events from ESP32 via BLE
- ✅ Batch write to Firestore
- ✅ Handle `switch` events (update item timestamp, skip EventLog creation)
- ✅ Handle other events (create EventLog entry)
- ✅ Real-time UI updates on detail page
- ✅ Error handling for Firestore failures

#### 2.1.4 Detail Page & Charts
**Description**: Item detail view with event visualization

**User Stories**:
- As a user, I can view an item's detail page with current count and today's count
- As a user, I can select a date range to filter events
- As a user, I can view a bar chart showing daily event counts
- As a user, I can view a cumulative chart showing running totals over time
- As a user, I can switch between chart types

**Technical Requirements**:
- Date range picker (calendar widget)
- Bar chart: Daily event counts (X-axis: dates, Y-axis: count)
- Cumulative chart: Running totals (X-axis: dates, Y-axis: cumulative count)
- Real-time updates when new events arrive
- Chart library: fl_chart or similar

**Acceptance Criteria**:
- ✅ Display item details (name, count, todaycount)
- ✅ Date range selection with calendar
- ✅ Bar chart visualization
- ✅ Cumulative chart visualization
- ✅ Chart updates in real-time
- ✅ Handle empty data gracefully
- ✅ Responsive layout (mobile and tablet)

#### 2.1.5 Data Export
**Description**: CSV generation and email export

**User Stories**:
- As a user, I can export event data for a date range as CSV
- As a user, I can choose aggregation level (daily, weekly, monthly)
- As a user, I receive the CSV file via email
- As a user, I can see export progress and confirmation

**Technical Requirements**:
- CSV generation: item name, date, event count, aggregation level
- Email via Firebase Mail Extension (Firestore-triggered Cloud Function)
- Base64 encoding for attachment
- Filename format: `tally_export_{startDate}_to_{endDate}_{aggregation}.csv`

**CSV Format**:
```csv
Item Name,Date,Event Count
Coffee,2026-01-01,5
Coffee,2026-01-02,3
Tea,2026-01-01,2
```

**Acceptance Criteria**:
- ✅ Select date range and aggregation level
- ✅ Generate CSV from EventLog data
- ✅ Send email with CSV attachment
- ✅ Show export progress
- ✅ Handle export errors (no data, email failure)
- ✅ Validate email address

#### 2.1.6 Authentication
**Description**: Multi-provider authentication with Firebase Auth

**User Stories**:
- As a user, I can sign up with email and password
- As a user, I can sign in with email and password
- As a user, I can sign in with Google
- As a user, I can sign in with Apple Sign-In
- As a user, I can reset my password via email
- As a user, I can sign out

**Technical Requirements**:
- Firebase Authentication
- Providers: Email/Password, Google, Apple Sign-In
- Email verification (optional, not enforced)
- Password reset flow
- Secure credential storage
- Auth state persistence

**Acceptance Criteria**:
- ✅ Sign up with email/password
- ✅ Sign in with email/password
- ✅ Sign in with Google
- ✅ Sign in with Apple
- ✅ Password reset flow
- ✅ Sign out
- ✅ Auth state persistence
- ✅ Error handling (invalid credentials, network errors)

#### 2.1.7 Profile & Settings
**Description**: User profile management and app settings

**User Stories**:
- As a user, I can view my profile information
- As a user, I can update my display name
- As a user, I can change my password
- As a user, I can export all my data (GDPR)
- As a user, I can delete my account and all data (GDPR)
- As a user, I can view the privacy policy
- As a user, I can view app version and contact support

**Technical Requirements**:
- Firestore: `/users/{userId}` for profile data
- GDPR data export: JSON file with all user data (items, events, profile)
- GDPR account deletion: Delete all Firestore data, delete Firebase Auth account
- Privacy policy: In-app web view or external link

**Acceptance Criteria**:
- ✅ View profile
- ✅ Update display name
- ✅ Change password
- ✅ Export data as JSON (email link)
- ✅ Delete account with confirmation
- ✅ View privacy policy
- ✅ Display app version
- ✅ Contact support (email link)

---

### 2.2 Infrastructure Features (Phase 1)

#### 2.2.1 RTC Synchronization
**Description**: Time synchronization between app and ESP32

**Technical Requirements**:
- Timezone support (user's local timezone)
- Send timezone offset to ESP32 when connecting
- ESP32 uses RTC (DS3231) for accurate timestamps
- App converts ESP32 Unix timestamps to local DateTime

**Acceptance Criteria**:
- ✅ Send timezone offset on connection
- ✅ Convert timestamps correctly
- ✅ Handle daylight saving time

#### 2.2.2 GDPR Compliance
**Description**: Data portability and right to erasure

**User Stories**:
- As a user, I can export all my data in JSON format
- As a user, I can delete my account and all associated data

**Data Export Format**:
```json
{
  "user": {
    "userId": "abc123",
    "email": "user@example.com",
    "displayName": "John Doe",
    "createdAt": "2025-01-01T00:00:00Z"
  },
  "items": [
    {
      "id": "item_123",
      "name": "Coffee",
      "count": 5,
      "todayCount": 2,
      ...
    }
  ],
  "events": [
    {
      "id": "event_456",
      "itemId": "item_123",
      "eventName": "increment",
      "timestamp": "2026-01-01T12:00:00Z",
      ...
    }
  ]
}
```

**Account Deletion Process**:
1. User confirms deletion (double confirmation)
2. Delete all `/Item/{itemId}` documents where `uid == userRef`
3. Delete all `/EventLog/{eventId}` documents where `uid == userRef`
4. Delete `/users/{userId}` document
5. Delete Firebase Auth account
6. Sign out user

**Acceptance Criteria**:
- ✅ Export all user data as JSON
- ✅ Send export via email
- ✅ Delete all Firestore data on account deletion
- ✅ Delete Firebase Auth account
- ✅ Prevent accidental deletion (double confirmation)

#### 2.2.3 Privacy Policy & Disclosures
**Description**: Legal compliance for App Store deployment

**Requirements**:
- Privacy policy covering:
  - Data collection (email, items, events)
  - Data usage (app functionality only)
  - Data sharing (none, except Firebase services)
  - Data retention (until account deletion)
  - User rights (GDPR: export, deletion)
  - Contact information
- In-app privacy policy access
- Link to privacy policy in App Store listings

**Implementation**:
- Markdown file: `/assets/privacy_policy.md`
- In-app viewer: markdown_widget package
- External URL: Host on GitHub Pages or similar (placeholder for now)

**Acceptance Criteria**:
- ✅ Privacy policy markdown file
- ✅ In-app privacy policy viewer
- ✅ External URL for App Store listing

#### 2.2.4 App Store Setup Guides
**Description**: Documentation for Google Play and Apple App Store setup

**Google Play Store Guide**:
1. Create Google Play Developer account ($25 one-time fee)
2. Set up app listing (title, description, screenshots)
3. Upload privacy policy URL
4. Configure data safety section
5. Create internal testing track
6. Upload APK/AAB
7. Invite testers
8. Prepare for production release

**Apple App Store Guide**:
1. Create Apple Developer account ($99/year)
2. Set up app in App Store Connect
3. Configure app metadata (name, description, keywords)
4. Upload privacy policy URL
5. Configure App Privacy questionnaire
6. Create TestFlight build
7. Add internal testers
8. Submit for App Review
9. Prepare for production release

**Deliverables**:
- `/docs/google-play-setup.md`
- `/docs/apple-app-store-setup.md`
- Checklist for each platform
- Screenshot requirements and templates

**Acceptance Criteria**:
- ✅ Google Play setup guide
- ✅ Apple App Store setup guide
- ✅ Checklists for each platform
- ✅ Screenshot templates

#### 2.2.5 Testing Infrastructure
**Description**: Automated testing and quality assurance

**Unit Tests**:
- BLoC tests (state transitions, event handling)
- Repository tests (Firestore operations, error handling)
- Use case tests (business logic)
- Utility function tests

**Widget Tests**:
- Critical UI components (item list, detail page, charts)
- Form validation
- Navigation flows
- Error states

**Integration Tests**:
- End-to-end user flows (sign up, create item, connect device, view events)
- Bluetooth mock tests
- Firestore emulator tests

**Test Coverage Goal**: 70%+ for business logic, 50%+ for UI

**Acceptance Criteria**:
- ✅ Unit tests for all BLoCs
- ✅ Unit tests for all repositories
- ✅ Widget tests for critical components
- ✅ Integration tests for main flows
- ✅ CI pipeline runs tests on every commit
- ✅ Coverage reports generated

#### 2.2.6 Monitoring & Analytics
**Description**: Production monitoring and user analytics

**Crashlytics**:
- Firebase Crashlytics integration
- Crash reporting with stack traces
- User identification (anonymized user ID)
- Custom logs for debugging

**Analytics**:
- Firebase Analytics integration
- Track key events:
  - `item_created`, `item_updated`, `item_deleted`
  - `device_connected`, `device_disconnected`
  - `export_csv`, `export_data`
  - `account_deleted`
- User properties: `item_count`, `device_type`

**Performance Monitoring**:
- Firebase Performance Monitoring
- Track screen load times
- Track network request durations
- Track Bluetooth operation durations

**Acceptance Criteria**:
- ✅ Crashlytics integrated
- ✅ Analytics events tracked
- ✅ Performance monitoring enabled
- ✅ Test in debug and release modes

#### 2.2.7 Automated Versioning & Release Notes
**Description**: Semantic versioning and automated release management

**Versioning Strategy**:
- Semantic versioning: `MAJOR.MINOR.PATCH`
- Version stored in `pubspec.yaml`
- Build number auto-incremented on release

**Release Notes**:
- Markdown file: `/CHANGELOG.md`
- Auto-generated from git commits (conventional commits)
- Format:
  ```markdown
  ## [1.0.0] - 2026-03-15
  ### Added
  - Initial release with clean architecture
  - Items CRUD with Firestore
  - Bluetooth ESP32 integration
  - EventLog with charts
  - CSV export
  - Multi-provider auth
  - GDPR compliance

  ### Changed
  - Migrated from FlutterFlow to clean architecture

  ### Fixed
  - None
  ```

**CI/CD Integration**:
- GitHub Actions workflow for releases
- Auto-increment build number
- Generate release notes from commits
- Tag releases in git

**Acceptance Criteria**:
- ✅ Semantic versioning implemented
- ✅ CHANGELOG.md file
- ✅ CI/CD workflow for releases
- ✅ Auto-increment build number
- ✅ Git tags for releases

---

### 2.3 Future Features (Phase 2 - Out of Scope)

**Monetization**:
- In-app purchases
- Subscription tiers
- Ad-supported free tier (alternative)

**Advanced ESP32 Features**:
- MTU negotiation optimization
- Battery monitoring
- Power-saving modes
- Firmware OTA updates

**Analytics Dashboard**:
- Web dashboard for data visualization
- Advanced reporting
- Export to PDF

**Multi-Device Support**:
- Connect multiple ESP32 devices per user
- Device management UI

---

## 3. Technical Architecture

### 3.1 Architecture Overview

**Clean Architecture Layers**:

```
┌─────────────────────────────────────────────────────────────┐
│                     Presentation Layer                       │
│  - Pages (UI)                                                │
│  - Widgets (Reusable UI Components)                         │
│  - BLoCs (State Management)                                 │
└─────────────────────────────────────────────────────────────┘
                            ↓ ↑
┌─────────────────────────────────────────────────────────────┐
│                      Domain Layer                            │
│  - Entities (Business Objects)                              │
│  - Use Cases (Business Logic)                               │
│  - Repository Interfaces (Contracts)                        │
└─────────────────────────────────────────────────────────────┘
                            ↓ ↑
┌─────────────────────────────────────────────────────────────┐
│                       Data Layer                             │
│  - Repository Implementations                                │
│  - Data Sources (Remote: Firestore, Local: Shared Prefs)   │
│  - Models (Data Transfer Objects)                           │
└─────────────────────────────────────────────────────────────┘
```

**Dependency Injection**:
- GetIt service locator
- Injectable package for code generation
- Singleton and factory registrations

**Error Handling**:
- Dartz Either<Failure, Success> for use cases
- Custom Failure classes: `ServerFailure`, `CacheFailure`, `BluetoothFailure`, etc.
- BLoC error states for UI feedback

### 3.2 Project Structure

```
lib/
├── core/
│   ├── error/
│   │   ├── failures.dart
│   │   └── exceptions.dart
│   ├── usecases/
│   │   └── usecase.dart
│   ├── utils/
│   │   ├── constants.dart
│   │   ├── bluetooth_constants.dart
│   │   └── validators.dart
│   └── di/
│       └── injection.dart (GetIt setup)
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── models/
│   │   │   ├── datasources/
│   │   │   └── repositories/
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   ├── repositories/
│   │   │   └── usecases/
│   │   └── presentation/
│   │       ├── bloc/
│   │       ├── pages/
│   │       └── widgets/
│   ├── items/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── bluetooth/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── events/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── export/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   └── profile/
│       ├── data/
│       ├── domain/
│       └── presentation/
├── main.dart
└── app.dart
```

---

## 4. Success Metrics

### 4.1 Technical Metrics

**Code Quality**:
- Test coverage: ≥ 70% for business logic, ≥ 50% for UI
- Dart analyzer warnings: 0
- Critical Crashlytics errors: 0 within 48 hours of detection

**Performance**:
- App launch time: < 3 seconds (cold start)
- Bluetooth connection time: < 5 seconds
- Firestore query time: < 1 second
- UI frame rate: 60 FPS sustained

**Reliability**:
- Crash-free rate: ≥ 99%
- Bluetooth auto-reconnect success rate: ≥ 95%
- Firestore write success rate: ≥ 99.9%

### 4.2 User Metrics

**Adoption** (Phase 1: First 3 months):
- Google Play installs: 100+
- Apple App Store installs: 50+
- Daily active users (DAU): 20+
- Weekly active users (WAU): 50+

**Engagement**:
- Average items per user: 3-5
- Average events per user per day: 10+
- Bluetooth connection sessions per week: 5+

**Retention**:
- Day 1 retention: ≥ 80%
- Day 7 retention: ≥ 50%
- Day 30 retention: ≥ 30%

**Satisfaction**:
- Google Play rating: ≥ 4.0 stars
- Apple App Store rating: ≥ 4.0 stars
- User-reported bugs: < 5 critical bugs in first month

---

## 5. Deployment & Release

### 5.1 Deployment Targets

**Google Play Store**:
- Internal Testing track (Phase 1: Week 9)
- Open Testing / Closed Testing (optional, Phase 2)
- Production (Phase 1: Week 10)

**Apple App Store**:
- TestFlight Internal Testing (Phase 1: Week 9)
- TestFlight External Testing (optional, Phase 2)
- Production (Phase 1: Week 10)

### 5.2 Release Process

**Version 1.0.0 (Initial Release)**:

**Pre-Release**:
1. Merge `feature/clean-architecture` → `main`
2. Tag release: `git tag v1.0.0`
3. Generate release notes (CHANGELOG.md)
4. Build APK/AAB for Android
5. Build IPA for iOS
6. Test builds on physical devices

**Internal Testing** (Week 9):
1. Upload to Google Play Internal Testing
2. Upload to TestFlight Internal Testing
3. Invite testers (user + 5-10 others)
4. Test for 3-5 days
5. Collect feedback and fix critical bugs

**Production Release** (Week 10):
1. Address feedback from internal testing
2. Submit to Google Play Production (review: 1-3 days)
3. Submit to App Store Production (review: 1-3 days)
4. Monitor Crashlytics for crashes
5. Monitor user reviews and ratings

**Post-Release**:
1. Monitor app performance (Firebase Performance)
2. Monitor crash reports (Crashlytics)
3. Gather user feedback
4. Plan Phase 2 features

---

## 6. Assumptions & Dependencies

### 6.1 Assumptions

1. **User Availability**: User can commit 1-2 hours/day for testing and review
2. **Single ESP32 Device**: User has one ESP32 device for testing (no multi-device support in Phase 1)
3. **Firebase Test Data**: Current Firebase data is test data only (no production data to preserve)
4. **No App Store Accounts**: User does not have Google Play or Apple Developer accounts yet (will create during Week 9)
5. **Free App**: App will be free with no monetization in Phase 1
6. **English Only**: App supports English language only in Phase 1
7. **Modern Devices**: Target Android 8.0+ and iOS 13.0+ (no legacy support)

### 6.2 Dependencies

**External Services**:
- Firebase Firestore (uptime: 99.95% SLA)
- Firebase Auth (uptime: 99.95% SLA)
- Firebase Extensions (Trigger Email)
- Google Play Store (for Android distribution)
- Apple App Store (for iOS distribution)

**Third-Party Libraries**:
- flutter_blue_plus (Bluetooth library)
- firebase_core, cloud_firestore, firebase_auth
- flutter_bloc, get_it, injectable
- dartz, equatable, json_serializable
- fl_chart (charting)

**User-Provided**:
- ESP32 device for testing
- Firebase project credentials
- App Store developer accounts (Google Play + Apple)
- Email for testing export functionality

---

## 7. Out of Scope (Phase 1)

The following features are explicitly **out of scope** for Phase 1 and deferred to Phase 2 or later:

1. **Monetization**: In-app purchases, subscriptions, ads
2. **Multi-Device Support**: Connecting multiple ESP32 devices per user
3. **Web Dashboard**: Web-based analytics and reporting
4. **Firmware OTA Updates**: Over-the-air firmware updates for ESP32
5. **Advanced ESP32 Features**: MTU negotiation, battery monitoring, power-saving modes
6. **Multi-Language Support**: Internationalization (i18n)
7. **Social Features**: Sharing data with other users, leaderboards
8. **Offline Mode**: Full offline support (partial offline via Firestore cache only)
9. **Data Migration Tools**: Importing data from other apps
10. **Admin Dashboard**: Managing users, monitoring system health

---

## Appendix: ESP32 Protocol Reference

### BLE Service Structure

**Service UUID**: `12345678-1234-1234-1234-123456789000`

**Characteristics**:
- **Read** (`12345678-1234-1234-1234-123456789001`): Read current state from ESP32
- **Notify** (`12345678-1234-1234-1234-123456789002`): Receive real-time updates
- **Set Items** (`12345678-1234-1234-1234-123456789008`): Send item list to ESP32
- **Write** (`12345678-1234-1234-1234-123456789010`): Send commands to ESP32

### Firestore Schema Reference

**users collection**:
```
/users/{userId}
  - email: string
  - displayName: string (optional)
  - createdAt: timestamp
  - lastLogin: timestamp
```

**Item collection**:
```
/Item/{itemId}
  - item_name: string (max 30 chars)
  - count: number (total count)
  - todaycount: number (today's count)
  - increment_by: number (1-100)
  - reminder: string (NONE, TARGET, INTERVAL, EVERY_TIME)
  - reminder_value: number (0-1000)
  - lastResetTime: timestamp
  - lastUpdated: timestamp
  - uid: reference to /users/{userId}
```

**EventLog collection**:
```
/EventLog/{eventId}  # Format: {itemId}-{event}-{timestamp}-{count}
  - created_time: timestamp
  - item: reference to /Item/{itemId}
  - event_name: string (increment, decrement, reset, switch)
  - increment: number
  - currentcount: number
  - uid: reference to /users/{userId}
  - id: string
```

**mail collection** (Firebase Extension):
```
/mail/{mailId}
  - to: array of strings (email addresses)
  - message: object
    - subject: string
    - text: string
    - attachments: array of objects
      - filename: string
      - type: string (text/csv)
      - content: string (base64)
```

---

**End of PRD**
