# Epic: Trackwise FlutterFlow to Clean Architecture Migration

**Epic ID**: flutterflow-to-clean-architecture-migration
**Status**: backlog
**Progress**: 0%
**PRD**: .claude/prds/flutterflow-to-clean-architecture-migration.md
**Created**: 2026-01-02
**Target Completion**: Week 10 (8-10 weeks)

---

## Overview

Transform Trackwise from a FlutterFlow-generated Flutter app into a production-ready, commercially-deployable application using Clean Architecture and BLoC pattern. This migration prepares the app for deployment to Google Play Store and Apple App Store as a free application with all features, GDPR compliance, comprehensive testing, monitoring, and automated versioning.

**Core Value Proposition**: Replace FlutterFlow technical debt with a robust, testable, maintainable codebase ready for commercial success while maintaining 100% ESP32 firmware compatibility.

---

## Epic Goals

### Primary Goals
1. **Architecture Migration**: Implement Clean Architecture (Presentation → Domain → Data) with BLoC pattern
2. **Feature Completeness**: Migrate all 7 core features (Items, Bluetooth, EventLog, Charts, Export, Auth, Profile)
3. **Commercial Readiness**: Prepare for App Store deployment (Google Play + Apple App Store)
4. **GDPR Compliance**: Implement data export and account deletion
5. **Quality Assurance**: 70%+ test coverage for business logic, 50%+ for UI
6. **Production Infrastructure**: Monitoring, analytics, automated versioning

### Success Criteria
- ✅ 100% feature parity with FlutterFlow version
- ✅ Clean architecture implemented (0 FlutterFlow dependencies)
- ✅ ESP32 Bluetooth working reliably (95%+ connection success rate)
- ✅ All tests passing (70%+ coverage)
- ✅ Ready for internal testing (Google Play + TestFlight)
- ✅ GDPR compliant (data export + deletion working)
- ✅ Monitoring and analytics configured
- ✅ Clean build (0 warnings, 0 errors)

---

## Architecture Design

### Clean Architecture Layers

```
┌──────────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │
│  │ Items BLoC │  │  BT BLoC   │  │ Auth BLoC  │  │ Events     │ │
│  └────────────┘  └────────────┘  └────────────┘  │ BLoC       │ │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  └────────────┘ │
│  │   Pages    │  │  Widgets   │  │  Routing   │                 │
│  └────────────┘  └────────────┘  └────────────┘                 │
└──────────────────────────────────────────────────────────────────┘
                            ↓ Uses
┌──────────────────────────────────────────────────────────────────┐
│                       DOMAIN LAYER                                │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │
│  │  Entities  │  │ Use Cases  │  │  Repos     │  │   Errors   │ │
│  │  (Item,    │  │ (Create,   │  │ (Interface)│  │ (Failures) │ │
│  │   Event)   │  │  Update)   │  │            │  │            │ │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘ │
└──────────────────────────────────────────────────────────────────┘
                            ↓ Implements
┌──────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │
│  │   Models   │  │ Data       │  │ Repos      │  │ Exceptions │ │
│  │ (Firestore)│  │ Sources    │  │ (Impl)     │  │            │ │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘ │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                 │
│  │ Firestore  │  │ Bluetooth  │  │SharedPrefs │                 │
│  │   Client   │  │  Service   │  │   Client   │                 │
│  └────────────┘  └────────────┘  └────────────┘                 │
└──────────────────────────────────────────────────────────────────┘
                            ↓ External
┌──────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SERVICES                              │
│  Firebase Firestore  │  Firebase Auth  │  ESP32 BLE  │  Mail    │
└──────────────────────────────────────────────────────────────────┘
```

### Key Architectural Decisions

**1. State Management: BLoC Pattern**
- **Rationale**: Separates business logic from UI, testable, reactive, industry-standard
- **Implementation**: flutter_bloc package
- **Structure**: One BLoC per feature (ItemsBloc, BluetoothBloc, AuthBloc, EventsBloc, ExportBloc, ProfileBloc)

**2. Dependency Injection: GetIt**
- **Rationale**: Simple service locator, already in project, solo-developer friendly
- **Lifecycle**:
  - Singletons: Repositories, Data Sources, Firebase/Bluetooth clients
  - Factories: BLoCs (per-screen lifecycle)

**3. Error Handling: Dartz Either**
- **Rationale**: Forces explicit error handling, clear success/failure paths, testable
- **Pattern**: All use cases return `Either<Failure, Success>`
- **Failures**: ServerFailure, CacheFailure, BluetoothFailure, AuthFailure, ValidationFailure

**4. Repository Pattern**
- **Rationale**: Abstracts data sources, enables mocking, supports multiple data sources
- **Interfaces**: Domain layer defines contracts
- **Implementations**: Data layer implements with Firestore, SharedPreferences, Bluetooth

**5. Feature-Based Organization**
- **Rationale**: Easy navigation, clear boundaries, scalable
- **Structure**: `features/{feature_name}/{data|domain|presentation}`

---

## Feature Breakdown

### Feature 1: Items CRUD
**Priority**: P0 (Critical)
**Timeline**: Week 2-3
**Dependencies**: Foundation (GetIt, base classes)

**Domain Layer**:
- **Entity**: `Item` (id, name, count, todayCount, incrementBy, reminder, reminderValue, lastResetTime, lastUpdated, userId)
- **Repository Interface**: `ItemRepository` (getItems, createItem, updateItem, deleteItem, activateItem)
- **Use Cases**:
  - GetItemsUseCase: Fetch all items for user
  - CreateItemUseCase: Create item with validation
  - UpdateItemUseCase: Update existing item
  - DeleteItemUseCase: Delete item
  - ActivateItemUseCase: Toggle activation state

**Data Layer**:
- **Model**: `ItemModel` with Firestore serialization
- **Data Source**: `ItemFirestoreDataSource` (CRUD operations)
- **Repository Impl**: `ItemRepositoryImpl` implements `ItemRepository`

**Presentation Layer**:
- **BLoC**: ItemsBloc (events: LoadItems, CreateItem, UpdateItem, DeleteItem, ActivateItem, ToggleCountView)
- **States**: ItemsInitial, ItemsLoading, ItemsLoaded, ItemsError
- **Pages**: ItemsPage (list), ItemSetupPage (create), ItemUpdatePage (edit)
- **Widgets**: ItemListTile, ItemSlidableActions

**Testing**:
- Unit tests: Use cases, repository, BLoC
- Widget tests: Item list, create/edit forms

---

### Feature 2: Bluetooth (ESP32 Integration)
**Priority**: P0 (Critical)
**Timeline**: Week 3-4
**Dependencies**: Items feature (to send item list to device)

**Domain Layer**:
- **Entities**:
  - `BluetoothDevice` (id, name, rssi)
  - `ConnectionState` (status, device, error)
- **Repository Interface**: `BluetoothRepository` (scanDevices, connectDevice, disconnectDevice, sendData, receiveData, watchConnectionState)
- **Use Cases**:
  - ScanDevicesUseCase: Stream available BLE devices
  - ConnectDeviceUseCase: Connect to ESP32
  - DisconnectDeviceUseCase: Disconnect from ESP32
  - SendItemListUseCase: Send item list to device (chunked, 180-byte MTU)
  - WatchConnectionStateUseCase: Monitor connection status

**Data Layer**:
- **Service**: `BluetoothServiceImpl` using flutter_blue_plus
- **Mock Service**: `MockBluetoothService` for testing without hardware
- **Repository Impl**: `BluetoothRepositoryImpl`
- **Protocol**: JSON messages (see PRD Appendix for format)

**Presentation Layer**:
- **BLoC**: BluetoothBloc (events: StartScan, ConnectDevice, DisconnectDevice, SendItemList)
- **States**: BluetoothInitial, BluetoothScanning, BluetoothConnected, BluetoothError
- **Pages**: BluetoothSearchPage, DeviceManagementPage
- **Widgets**: DeviceListTile, ConnectionStatusWidget

**Testing**:
- Unit tests: Use cases, repository (with mock Bluetooth service)
- Integration tests: Mock BLE protocol testing
- Manual tests: Real ESP32 hardware testing

**ESP32 Protocol**:
- **UUIDs**: Service `...789000`, Read `...789001`, Notify `...789002`, SetItems `...789008`, Write `...789010`
- **Chunking**: 180-byte MTU, 30ms delay, newline delimiters
- **Messages**: `items`, `prefs`, `event` types (JSON)

---

### Feature 3: EventLog (Real-time Event Ingestion)
**Priority**: P0 (Critical)
**Timeline**: Week 4-5
**Dependencies**: Bluetooth (receives events from ESP32)

**Domain Layer**:
- **Entity**: `EventLog` (id, createdTime, itemId, eventName, increment, currentCount, userId)
- **Repository Interface**: `EventLogRepository` (getEvents, insertEvent, insertEventBatch, deleteEventsForItem)
- **Use Cases**:
  - GetEventsByDateRangeUseCase: Query events for date range
  - InsertEventsUseCase: Batch insert events from ESP32
  - GetEventsByItemUseCase: Query events for specific item

**Data Layer**:
- **Model**: `EventLogModel` with Firestore serialization
- **Data Source**: `EventLogFirestoreDataSource`
- **Repository Impl**: `EventLogRepositoryImpl`
- **Special Handling**: `switch` events update Item.lastUpdated (don't create EventLog)

**Presentation Layer**:
- **BLoC**: EventsBloc (events: LoadEvents, FilterByDateRange, FilterByItem)
- **States**: EventsInitial, EventsLoading, EventsLoaded, EventsError
- **Pages**: EventLogPage (list view on detail page)
- **Widgets**: EventListTile, DateRangePicker

**Testing**:
- Unit tests: Use cases, repository, batch insertion logic
- Integration tests: Firestore emulator with batch writes

---

### Feature 4: Detail Page & Charts
**Priority**: P1 (High)
**Timeline**: Week 5
**Dependencies**: EventLog (data source for charts)

**Domain Layer**:
- **Use Cases**:
  - GetChartDataUseCase: Aggregate events by date for charting
  - GetCumulativeChartDataUseCase: Calculate cumulative totals

**Presentation Layer**:
- **BLoC**: ChartsBloc (events: LoadBarChart, LoadCumulativeChart, ChangeDateRange)
- **States**: ChartsInitial, ChartsLoading, ChartsLoaded, ChartsError
- **Pages**: ItemDetailPage (detail view with charts)
- **Widgets**: BarChartWidget, CumulativeChartWidget, DateRangeSelector
- **Library**: fl_chart

**Chart Types**:
1. **Bar Chart**: Daily event counts (X: dates, Y: count)
2. **Cumulative Chart**: Running totals over time (X: dates, Y: cumulative)

**Testing**:
- Unit tests: Chart data aggregation logic
- Widget tests: Chart rendering with sample data

---

### Feature 5: Data Export (CSV)
**Priority**: P1 (High)
**Timeline**: Week 5-6
**Dependencies**: EventLog (data source)

**Domain Layer**:
- **Use Cases**:
  - GenerateCSVUseCase: Create CSV from event data
  - SendEmailWithCSVUseCase: Trigger Firebase Mail Extension

**Data Layer**:
- **Data Source**: FirestoreMailDataSource (writes to `/mail` collection)
- **CSV Generator**: Utility for CSV formatting

**Presentation Layer**:
- **BLoC**: ExportBloc (events: ExportCSV)
- **States**: ExportInitial, ExportInProgress, ExportSuccess, ExportError
- **Pages**: ExportPage (date range, aggregation level, email input)
- **Widgets**: AggregationLevelSelector, EmailInputField

**CSV Format**:
```csv
Item Name,Date,Event Count
Coffee,2026-01-01,5
```

**Email Implementation**:
- Firebase Extension: Trigger Email from Firestore
- Collection: `/mail/{mailId}`
- Attachment: Base64-encoded CSV
- Filename: `tally_export_{startDate}_to_{endDate}_{aggregation}.csv`

**Testing**:
- Unit tests: CSV generation logic
- Integration tests: Firebase Mail Extension (manual verification)

---

### Feature 6: Authentication
**Priority**: P0 (Critical)
**Timeline**: Week 6-7
**Dependencies**: None (can be developed in parallel)

**Domain Layer**:
- **Entity**: `User` (id, email, displayName, photoUrl)
- **Repository Interface**: `AuthRepository` (signInWithEmail, signInWithGoogle, signInWithApple, signUp, signOut, resetPassword, watchAuthState)
- **Use Cases**:
  - SignInWithEmailUseCase
  - SignInWithGoogleUseCase
  - SignInWithAppleUseCase
  - SignUpUseCase
  - SignOutUseCase
  - ResetPasswordUseCase
  - WatchAuthStateUseCase

**Data Layer**:
- **Data Source**: `AuthFirebaseDataSource` (wraps firebase_auth)
- **Repository Impl**: `AuthRepositoryImpl`

**Presentation Layer**:
- **BLoC**: AuthBloc (events: SignInWithEmail, SignInWithGoogle, SignInWithApple, SignUp, SignOut, ResetPassword)
- **States**: AuthInitial, Authenticated, Unauthenticated, AuthLoading, AuthError
- **Pages**: LoginPage, SignupPage, ForgotPasswordPage
- **Widgets**: EmailInputField, PasswordInputField, SocialSignInButtons

**Testing**:
- Unit tests: Use cases, repository with mocked Firebase Auth
- Widget tests: Auth forms, validation

---

### Feature 7: Profile & Settings (including GDPR)
**Priority**: P1 (High)
**Timeline**: Week 7
**Dependencies**: Auth (user must be authenticated)

**Domain Layer**:
- **Entity**: `UserProfile` (userId, email, displayName, createdAt, lastLogin)
- **Repository Interface**: `ProfileRepository` (getProfile, updateProfile, exportUserData, deleteAccount)
- **Use Cases**:
  - GetProfileUseCase
  - UpdateProfileUseCase
  - ExportUserDataUseCase: GDPR data export (JSON)
  - DeleteAccountUseCase: GDPR account deletion

**Data Layer**:
- **Data Source**: `ProfileFirestoreDataSource`
- **Repository Impl**: `ProfileRepositoryImpl`
- **GDPR Export**: JSON with all user data (items, events, profile)
- **GDPR Deletion**: Delete all Firestore collections + Firebase Auth account

**Presentation Layer**:
- **BLoC**: ProfileBloc (events: LoadProfile, UpdateProfile, ExportData, DeleteAccount)
- **States**: ProfileInitial, ProfileLoading, ProfileLoaded, ProfileError
- **Pages**: ProfilePage, SettingsPage
- **Widgets**: ProfileFormWidget, DataExportWidget, AccountDeletionWidget

**GDPR Features**:
1. **Data Export**: JSON file with all user data (emailed via Firebase Extension)
2. **Account Deletion**: Double confirmation, deletes all data, signs out user

**Testing**:
- Unit tests: GDPR data export logic, account deletion logic
- Integration tests: Verify all user data deleted

---

### Infrastructure 1: RTC Synchronization
**Priority**: P2 (Medium)
**Timeline**: Week 4 (alongside Bluetooth)
**Dependencies**: Bluetooth (sends timezone to ESP32)

**Implementation**:
- Send timezone offset to ESP32 on connection
- ESP32 uses RTC (DS3231) for timestamps
- App converts ESP32 Unix timestamps to local DateTime
- Handle daylight saving time

**Testing**:
- Unit tests: Timezone conversion logic
- Manual tests: Verify timestamps with ESP32

---

### Infrastructure 2: Privacy Policy & App Store Setup
**Priority**: P1 (High)
**Timeline**: Week 8
**Dependencies**: None (documentation task)

**Privacy Policy**:
- Markdown file: `/assets/privacy_policy.md`
- In-app viewer: markdown_widget package
- External URL: GitHub Pages (placeholder)
- Content: Data collection, usage, sharing, retention, GDPR rights, contact

**App Store Setup Guides**:
- `/docs/google-play-setup.md`: Step-by-step guide for Google Play
- `/docs/apple-app-store-setup.md`: Step-by-step guide for App Store
- Checklists for each platform
- Screenshot templates

**Deliverables**:
- Privacy policy (Markdown + external URL)
- Google Play setup guide
- Apple App Store setup guide
- Screenshot templates (5-8 images per platform)

---

### Infrastructure 3: Testing Infrastructure
**Priority**: P0 (Critical)
**Timeline**: Week 1 (foundation), ongoing throughout migration
**Dependencies**: None (setup early)

**Testing Strategy**:
1. **Unit Tests** (70%+ coverage for business logic):
   - All use cases tested with mocked repositories
   - All repositories tested with mocked data sources
   - All BLoCs tested with bloc_test

2. **Widget Tests** (50%+ coverage for UI):
   - Critical UI components (item list, detail page, charts)
   - Form validation
   - Navigation flows
   - Error states

3. **Integration Tests**:
   - End-to-end user flows (sign up → create item → connect device → view events)
   - Firestore emulator tests
   - Mock Bluetooth tests (no hardware required)

**Tools**:
- mocktail: Test doubles
- bloc_test: BLoC testing
- Firestore emulator: Local Firebase testing

**CI/CD**:
- GitHub Actions workflow: Run tests on every commit
- Coverage reports: Codecov or similar

---

### Infrastructure 4: Monitoring & Analytics
**Priority**: P1 (High)
**Timeline**: Week 8
**Dependencies**: All features complete

**Crashlytics**:
- Firebase Crashlytics integration
- Crash reporting with stack traces
- Custom logs for debugging
- User identification (anonymized)

**Analytics**:
- Firebase Analytics integration
- Events: item_created, item_updated, device_connected, export_csv, account_deleted
- User properties: item_count, device_type

**Performance Monitoring**:
- Firebase Performance Monitoring
- Track screen load times
- Track Bluetooth operation durations
- Track Firestore query times

**Implementation**:
- Add Firebase packages: firebase_crashlytics, firebase_analytics, firebase_performance
- Configure in main.dart
- Add event tracking throughout app

**Testing**:
- Verify in debug mode
- Test in release mode
- Check Firebase console for data

---

### Infrastructure 5: Automated Versioning & Release Notes
**Priority**: P2 (Medium)
**Timeline**: Week 8
**Dependencies**: None

**Versioning**:
- Semantic versioning: MAJOR.MINOR.PATCH
- Version in pubspec.yaml
- Auto-increment build number

**Release Notes**:
- CHANGELOG.md file
- Format:
  ```markdown
  ## [1.0.0] - 2026-03-15
  ### Added
  - Feature list
  ### Changed
  - Changes
  ### Fixed
  - Bug fixes
  ```

**CI/CD**:
- GitHub Actions workflow for releases
- Auto-increment build number
- Generate release notes from git commits
- Tag releases in git

---

## Implementation Timeline

### Week 1: Foundation & Setup
**Goal**: Establish clean architecture foundation

**Tasks**:
1. Create clean architecture folder structure
2. Set up GetIt dependency injection
3. Create base classes (UseCase, Failure, Exception)
4. Add required dependencies (flutter_bloc, dartz, mocktail, bloc_test)
5. Set up testing infrastructure
6. Create architectural decision records (ADRs)

**Deliverables**:
- Project structure in place
- GetIt configured
- Base classes ready
- Testing framework ready

---

### Week 2-3: Items Feature
**Goal**: Migrate Items CRUD to clean architecture

**Tasks**:
1. Implement Items domain layer (entity, repository interface, use cases)
2. Implement Items data layer (model, data source, repository impl)
3. Implement Items BLoC (events, states, bloc logic)
4. Migrate Items UI (list, create, edit pages)
5. Write unit tests (use cases, repository, BLoC)
6. Write widget tests (item list, forms)
7. Deploy and verify

**Deliverables**:
- Items feature fully migrated
- 70%+ test coverage for Items
- Items CRUD working with Firestore

---

### Week 3-4: Bluetooth Feature
**Goal**: Migrate ESP32 Bluetooth integration

**Tasks**:
1. Implement Bluetooth domain layer (entities, repository interface, use cases)
2. Implement Bluetooth data layer (ESP32 service, repository impl)
3. Create mock Bluetooth service for testing
4. Implement Bluetooth BLoC (events, states, bloc logic)
5. Migrate Bluetooth UI (scan, connect, manage)
6. Write unit tests with mocks
7. Test with real ESP32 device
8. Deploy and verify

**Deliverables**:
- Bluetooth feature fully migrated
- Mock Bluetooth service for testing
- ESP32 connection working (95%+ success rate)

---

### Week 4-5: EventLog & Charts
**Goal**: Implement event logging and visualization

**Tasks**:
1. Implement EventLog domain layer (entity, repository, use cases)
2. Implement EventLog data layer (model, data source, repository impl)
3. Implement EventLog BLoC
4. Implement Charts BLoC (bar chart, cumulative chart)
5. Migrate detail page UI with charts
6. Write unit tests
7. Write widget tests (charts)
8. Deploy and verify

**Deliverables**:
- EventLog feature working
- Charts displaying correctly
- Real-time event updates from ESP32

---

### Week 5-6: Data Export
**Goal**: Implement CSV export functionality

**Tasks**:
1. Implement Export domain layer (use cases)
2. Implement CSV generation logic
3. Integrate Firebase Mail Extension
4. Implement Export BLoC
5. Create export UI (date range, aggregation, email)
6. Write unit tests (CSV generation)
7. Test email delivery
8. Deploy and verify

**Deliverables**:
- CSV export working
- Email delivery functional
- Aggregation levels implemented

---

### Week 6-7: Authentication
**Goal**: Migrate authentication to clean architecture

**Tasks**:
1. Implement Auth domain layer (entity, repository, use cases)
2. Implement Auth data layer (Firebase Auth data source, repository impl)
3. Implement Auth BLoC (events, states)
4. Migrate auth UI (login, signup, forgot password)
5. Write unit tests (use cases, repository)
6. Write widget tests (auth forms)
7. Deploy and verify

**Deliverables**:
- Auth feature fully migrated
- Email, Google, Apple Sign-In working
- Password reset functional

---

### Week 7: Profile & GDPR
**Goal**: Implement profile management and GDPR compliance

**Tasks**:
1. Implement Profile domain layer (entity, repository, use cases)
2. Implement GDPR data export logic (JSON generation)
3. Implement GDPR account deletion logic
4. Implement Profile BLoC
5. Migrate profile UI
6. Write unit tests (GDPR features)
7. Test data export and deletion
8. Deploy and verify

**Deliverables**:
- Profile feature working
- GDPR data export functional
- GDPR account deletion functional

---

### Week 8: Infrastructure & Cleanup
**Goal**: Set up monitoring, versioning, privacy policy, and cleanup

**Tasks**:
1. Write privacy policy (Markdown + external URL)
2. Create App Store setup guides (Google Play + Apple)
3. Integrate Crashlytics, Analytics, Performance Monitoring
4. Set up automated versioning (CHANGELOG.md, GitHub Actions)
5. Remove all FlutterFlow dependencies
6. Clean up unused packages
7. Replace FlutterFlowTheme with custom AppTheme
8. Clean up navigation (GoRouter)
9. Final testing (full test suite)
10. Performance profiling

**Deliverables**:
- Privacy policy ready
- App Store guides ready
- Monitoring configured
- All FlutterFlow code removed
- Clean build (0 warnings)

---

### Week 9: Internal Testing
**Goal**: Prepare for App Store deployment

**Tasks**:
1. Create Google Play Developer account
2. Create Apple Developer account
3. Set up Google Play Internal Testing track
4. Set up TestFlight Internal Testing
5. Build APK/AAB for Android
6. Build IPA for iOS
7. Upload builds to internal testing
8. Invite testers (user + 5-10 others)
9. Test for 3-5 days
10. Collect feedback and fix critical bugs

**Deliverables**:
- App Store accounts created
- Internal testing builds uploaded
- Feedback collected
- Critical bugs fixed

---

### Week 10: Production Release
**Goal**: Launch on Google Play and Apple App Store

**Tasks**:
1. Address feedback from internal testing
2. Prepare app store listings (descriptions, screenshots, keywords)
3. Upload privacy policy URL
4. Configure data safety (Google Play)
5. Configure App Privacy questionnaire (Apple)
6. Submit to Google Play Production
7. Submit to Apple App Store Production
8. Monitor Crashlytics for crashes
9. Monitor user reviews
10. Celebrate launch! 🎉

**Deliverables**:
- App live on Google Play Store
- App live on Apple App Store
- Monitoring active
- User feedback being collected

---

## Risk Management

### High-Priority Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|-----------|
| Breaking Bluetooth during refactor | **Critical** | Medium | Extensive testing with ESP32; keep old code until verified; mock Bluetooth service |
| Underestimating task complexity | High | Medium | Buffer time in 8-10 week timeline; prioritize MVP features; incremental deployment |
| User availability for testing | High | Medium | Flexible testing schedule (1-2 hours/day); focus on critical flows |
| App Store rejection | High | Low | Follow guidelines strictly; privacy policy; test data safety disclosures |
| Learning curve (BLoC, Clean Arch) | Medium | Medium | Study before starting (4-6 hours); start with simplest feature; iterate |

### Medium-Priority Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|-----------|
| ESP32 protocol changes mid-migration | Medium | Low | Bluetooth abstraction makes protocol changes easy; minimal firmware changes |
| Testing takes too long | Medium | Medium | Focus on critical paths first; expand coverage over time |
| Data migration issues | Low | Low | No production users; can reset data freely |
| Firebase costs exceed expectations | Low | Low | Monitor usage; optimize queries; free tier sufficient for Phase 1 |

---

## Dependencies & Prerequisites

### External Dependencies

**Required Packages** (add to pubspec.yaml):
```yaml
dependencies:
  flutter_bloc: ^8.1.3
  equatable: ^2.0.5
  dartz: ^0.10.1
  get_it: ^7.6.0  # Already exists
  injectable: ^2.3.0
  firebase_core: ^2.20.0  # Already exists
  cloud_firestore: ^4.13.0  # Already exists
  firebase_auth: ^4.14.0  # Already exists
  flutter_blue_plus: ^1.32.0  # Already exists
  go_router: ^13.0.0  # Already exists
  fl_chart: ^0.66.0
  firebase_crashlytics: ^3.4.0
  firebase_analytics: ^10.7.0
  firebase_performance: ^0.9.3
  markdown_widget: ^2.3.0

dev_dependencies:
  mocktail: ^1.0.1
  bloc_test: ^9.1.4
  build_runner: ^2.4.0
  injectable_generator: ^2.4.0
```

**Can Remove** (after migration):
- All `/flutter_flow/` custom dependencies
- 40+ unused packages from FlutterFlow generation

### Hardware Dependencies
- **ESP32 Device**: For Bluetooth testing (critical for Week 3-4)
- **Android/iOS Device**: For testing (emulator acceptable for early phases)

### Service Dependencies
- **Firebase Project**: Firestore, Auth, Extensions configured
- **Firebase Mail Extension**: Installed and configured
- **Google Play Developer Account**: Created in Week 9 ($25 one-time fee)
- **Apple Developer Account**: Created in Week 9 ($99/year)

### Knowledge Dependencies

**Required Learning** (before starting):
- Clean Architecture principles (2-3 hours)
- BLoC pattern basics (2-3 hours)
- GetIt dependency injection (1 hour)
- Flutter testing fundamentals (1-2 hours)

**Helpful Resources**:
- BLoC documentation: https://bloclibrary.dev
- Clean Architecture (Reso Coder): https://resocoder.com/flutter-clean-architecture
- GetIt documentation: https://pub.dev/packages/get_it

---

## Testing Strategy

### Unit Testing (70%+ coverage for business logic)

**What to Test**:
- All use cases (with mocked repositories)
- All repositories (with mocked data sources)
- All BLoCs (with bloc_test)
- All models (serialization/deserialization)

**Tools**:
- mocktail for mocking
- bloc_test for BLoC testing
- test package for Dart tests

**Example**:
```dart
// Use case test
test('GetItemsUseCase returns items on success', () async {
  when(() => mockRepository.getItems()).thenAnswer((_) async => Right([mockItem]));

  final result = await useCase(NoParams());

  expect(result, Right([mockItem]));
  verify(() => mockRepository.getItems()).called(1);
});

// BLoC test
blocTest<ItemsBloc, ItemsState>(
  'emits [ItemsLoading, ItemsLoaded] when LoadItems succeeds',
  build: () => ItemsBloc(getItemsUseCase),
  act: (bloc) => bloc.add(LoadItems()),
  expect: () => [ItemsLoading(), ItemsLoaded([mockItem])],
);
```

---

### Widget Testing (50%+ coverage for UI)

**What to Test**:
- Item list rendering
- Create/Edit forms (validation)
- Bluetooth connection UI
- Charts rendering
- Auth forms

**Tools**:
- flutter_test package
- mockito or mocktail for mocking BLoCs

**Example**:
```dart
testWidgets('ItemsPage displays list of items', (tester) async {
  when(() => mockItemsBloc.state).thenReturn(ItemsLoaded([mockItem]));

  await tester.pumpWidget(MaterialApp(
    home: BlocProvider.value(
      value: mockItemsBloc,
      child: ItemsPage(),
    ),
  ));

  expect(find.text('Coffee'), findsOneWidget);
});
```

---

### Integration Testing

**What to Test**:
- End-to-end user flows:
  - Sign up → Create item → Connect device → View events
  - Export CSV → Receive email
  - Delete account → Data removed
- Firestore operations (with emulator)
- Bluetooth operations (with mock service)

**Tools**:
- integration_test package
- Firestore emulator
- Mock Bluetooth service

---

## Deployment Strategy

### Incremental Deployment Approach

**Phase Completion = Deployment**:
- After Items: Deploy with new Items feature
- After Bluetooth: Deploy with Items + Bluetooth
- After EventLog: Deploy with Items + Bluetooth + EventLog
- After Export: Deploy with all core features
- After Auth: Deploy with all features
- After Profile: Full migration complete

**Benefits**:
- Continuous progress visible
- Early detection of issues
- No "big bang" risk
- Can pause/resume at phase boundaries

---

### Production Release Checklist

**Pre-Release**:
- [ ] All features working
- [ ] All tests passing (70%+ coverage)
- [ ] ESP32 Bluetooth tested (95%+ connection success)
- [ ] Clean build (0 warnings, 0 errors)
- [ ] Privacy policy accessible
- [ ] Monitoring configured (Crashlytics, Analytics)
- [ ] Versioning set up (1.0.0)

**Internal Testing** (Week 9):
- [ ] Google Play Internal Testing configured
- [ ] TestFlight Internal Testing configured
- [ ] Builds uploaded
- [ ] Testers invited
- [ ] 3-5 days of testing completed
- [ ] Critical bugs fixed

**Production Release** (Week 10):
- [ ] App Store listings prepared (descriptions, screenshots)
- [ ] Privacy policy URL uploaded
- [ ] Data safety configured (Google Play)
- [ ] App Privacy configured (Apple)
- [ ] Submitted to Google Play Production
- [ ] Submitted to Apple App Store Production
- [ ] Monitoring active
- [ ] User reviews being monitored

---

## Success Metrics

### Technical Metrics

**Code Quality**:
- Test coverage: ≥ 70% for business logic, ≥ 50% for UI
- Dart analyzer warnings: 0
- FlutterFlow dependencies: 0
- Build time: < 30 seconds (incremental)

**Performance**:
- App launch time: < 3 seconds (cold start)
- Bluetooth connection: < 5 seconds
- Firestore queries: < 1 second
- UI frame rate: 60 FPS sustained

**Reliability**:
- Crash-free rate: ≥ 99%
- Bluetooth auto-reconnect success: ≥ 95%
- Firestore write success: ≥ 99.9%

### User Metrics (First 3 Months)

**Adoption**:
- Google Play installs: 100+
- Apple App Store installs: 50+
- Daily active users: 20+
- Weekly active users: 50+

**Engagement**:
- Average items per user: 3-5
- Average events per day: 10+
- Bluetooth sessions per week: 5+

**Retention**:
- Day 1 retention: ≥ 80%
- Day 7 retention: ≥ 50%
- Day 30 retention: ≥ 30%

**Satisfaction**:
- Google Play rating: ≥ 4.0 stars
- Apple App Store rating: ≥ 4.0 stars
- Critical bugs: < 5 in first month

---

## Post-Migration Benefits

### Code Quality Improvements
- ✅ **Testable**: 70%+ test coverage vs 0% currently
- ✅ **Maintainable**: Clear architecture vs FlutterFlow spaghetti
- ✅ **Debuggable**: Isolated layers vs mixed concerns
- ✅ **Flexible**: Easy to swap implementations

### Developer Experience Improvements
- ✅ **Fast Feature Development**: Add item field in 30 min vs 2+ hours
- ✅ **Confident Changes**: Tests catch regressions
- ✅ **Easy Debugging**: Clear data flow through layers
- ✅ **ESP32 Protocol Flexibility**: Change protocol without UI changes

### Commercial Readiness
- ✅ **App Store Ready**: Google Play + Apple App Store deployment guides
- ✅ **GDPR Compliant**: Data export + account deletion
- ✅ **Production Monitoring**: Crashlytics + Analytics + Performance
- ✅ **Privacy Policy**: Legal compliance for App Store

---

## Notes & Assumptions

1. **User Data Reset**: All existing data can be wiped (no migration scripts needed)
2. **No Production Users**: Breaking changes acceptable during migration
3. **ESP32 Protocol**: Current protocol maintained, architecture allows evolution
4. **User Availability**: 1-2 hours/day for testing and review
5. **Timeline**: 8-10 weeks is achievable with focused effort
6. **Testing Priority**: Focus on critical paths (Items CRUD, Bluetooth, Auth)
7. **FlutterFlow Reference**: Keep old code until migration verified complete
8. **App Store Accounts**: User will create during Week 9 (not yet created)
9. **Free App**: No monetization in Phase 1 (deferred to Phase 2)
10. **English Only**: No internationalization in Phase 1

---

**Epic Ready for Task Decomposition**

Next step: Decompose this epic into 30-35 actionable tasks that can be synced to GitHub Issues for tracking and execution.

---

**End of Epic**
