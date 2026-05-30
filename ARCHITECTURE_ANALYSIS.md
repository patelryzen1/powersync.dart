# PowerSync Flutter Architecture Analysis

This document provides a comprehensive architectural analysis of the PowerSync Flutter codebase. Because this repository (`powersync-ja/powersync.dart`) is a monorepo containing both the PowerSync SDK (`packages/powersync`) and various sample applications (`demos/`), this analysis specifically focuses on how the SDK is used within the primary reference application: `demos/supabase-todolist`.

## 1. High-Level Architecture

*   **Overall architectural pattern:** The application uses an **Offline-First Layered Architecture** with a strong emphasis on reactive programming. It doesn't strictly adhere to Clean Architecture or BLoC; instead, it leans heavily on **Feature-First/Data-First** principles powered by Streams.
*   **Why it's structured this way:** PowerSync's core value proposition is instant local responsiveness. To achieve this, the local SQLite database acts as the single source of truth for the UI. Updates are made locally and synced in the background. The reactive nature (using `db.watch()`) ensures the UI reflects local database changes instantly.
*   **Main layers:**
    *   **Presentation Layer (UI/Widgets):** Consumes `Stream`s of data and renders `StreamBuilder`s. Triggers local database operations.
    *   **Domain/Data Layer (Models):** Wraps SQLite rows into Dart objects (e.g., `TodoList`, `TodoItem`) and provides static methods to watch queries.
    *   **Sync/Infrastructure Layer (PowerSync SDK + Supabase):** Handles authentication (Supabase), remote data fetching, local SQLite storage, and background synchronization (`PowerSyncDatabase`, `SupabaseConnector`).
*   **Dependency Direction:** Presentation -> Domain/Models -> Infrastructure (SQLite/PowerSync).

**Architecture Diagram:**

```text
+---------------------------------------------------------+
|                       Presentation                      |
|  (Widgets: ListsPage, TodoListPage, StreamBuilder)      |
+---------------------------------------------------------+
                          | (Reads Streams, Calls static methods)
                          v
+---------------------------------------------------------+
|                       Domain/Models                     |
|            (TodoList, TodoItem, schema.dart)            |
+---------------------------------------------------------+
                          | (SQL Queries, CRUD operations)
                          v
+---------------------------------------------------------+
|                Infrastructure / Data Layer              |
|        (PowerSyncDatabase, sqlite_async, Supabase)      |
+---------------------------------------------------------+
           | (Local SQLite)                | (Sync)
           v                               v
+------------------+             +--------------------+
| Local SQLite DB  |  <========> | Remote Supabase DB |
+------------------+             +--------------------+
```

## 2. Application Startup Flow

The startup flow prioritizes local database availability before the UI renders.

1.  `main()` (in `demos/supabase-todolist/lib/main.dart`)
2.  `WidgetsFlutterBinding.ensureInitialized()` - Required to get the SQLite file path.
3.  `openDatabase()` (in `demos/supabase-todolist/lib/powersync.dart`)
    *   Initializes the `PowerSyncDatabase` with the local schema.
    *   Loads Supabase (`loadSupabase()`).
    *   If logged in, connects PowerSync to Supabase via `SupabaseConnector`.
    *   Sets up FTS (Full-Text Search) with `configureFts()`.
4.  `isLoggedIn()` checks the current Supabase session.
5.  `runApp(MyApp(loggedIn: loggedIn))` starts the Flutter app.
6.  The router pushes `homePage` (which is `ListsPage`) if logged in, or `loginPage` if not.

**Startup Sequence Diagram:**

```text
main()
  |--> WidgetsFlutterBinding.ensureInitialized()
  |--> openDatabase()
  |      |--> PowerSyncDatabase.initialize() (Local DB ready)
  |      |--> loadSupabase() (Auth ready)
  |      |--> if logged in: db.connect(SupabaseConnector) (Background sync starts)
  |--> runApp(MyApp)
         |--> MaterialApp
                |--> if logged in: ListsPage
                |--> else: LoginPage
```

## 3. Folder Structure Analysis

### Monorepo Root
*   `packages/powersync/`: Core Dart/Flutter SDK for PowerSync.
*   `packages/powersync_attachments_helper/`: Helper library for file syncing.
*   `demos/`: Various reference implementations.

### `demos/supabase-todolist/lib/`
*   **`lib/` root:**
    *   `main.dart`: App entry point and global UI setup.
    *   `powersync.dart`: The core PowerSync integration. Defines `SupabaseConnector` and the global `db` variable.
    *   `supabase.dart`: Initialization logic for the Supabase SDK.
    *   `app_config_template.dart`: Configuration constants.
*   **`models/`**:
    *   **Purpose:** Defines the local database schema and Dart entity representations.
    *   **Key files:** `schema.dart` (PowerSync schema definition), `todo_list.dart`, `todo_item.dart`.
*   **`widgets/`**:
    *   **Purpose:** The Presentation layer containing all UI screens and components.
    *   **Key files:** `login_page.dart`, `lists_page.dart` (Lists view), `todo_list_page.dart` (Items view).
*   **`attachments/`**:
    *   **Purpose:** Logic for handling image attachments, syncing photos via PowerSync and Supabase Storage.
*   **`migrations/`**:
    *   **Purpose:** Handles custom SQLite migrations (e.g., FTS setup in `fts_setup.dart`).

## 4. State Management Analysis

*   **Solution Used:** Native Flutter `StreamBuilder`s powered by PowerSync's `db.watch()` and localized `StatefulWidget` / `setState` for form states. There is no external state management library (like Riverpod or BLoC).
*   **Where states are created:** The database is the source of truth. The state is represented as a stream of SQL query results originating from `PowerSyncDatabase.watch()`.
*   **Where states are consumed:** Within UI widgets using `StreamBuilder`.
*   **Propagation:** Local UI action -> `db.execute(INSERT/UPDATE)` -> `db.watch()` stream emits new data -> `StreamBuilder` rebuilds UI.

**State Flow Diagram:**

```text
User Taps "Add Todo"
       |
       v
TodoList.add(description) (demos/supabase-todolist/lib/models/todo_list.dart)
       |
       v
db.execute('INSERT INTO todos...')
       |
       +--> PowerSync Background Queue -> Uploads to Supabase
       |
       v
SQLite Database updates
       |
       v
db.watch() Stream emits new List<TodoItem>
       |
       v
StreamBuilder (in TodoListPage) receives new snapshot
       |
       v
ListView rebuilds with new TodoItemWidget
```

## 5. Navigation Analysis

*   **Package used:** Standard Flutter `Navigator` (`Navigator.of(context).push()`). No named routing framework (like go_router) is used.
*   **Route definitions:** Routes are defined inline using `MaterialPageRoute`.
*   **Auth redirects:** Handled dynamically via conditional logic in `MyApp` (`loggedIn ? homePage : loginPage`) and programmatic replacements on login/logout (e.g., `pushReplacement` to `loginPage`).
*   **Navigation Map:**
    *   `MyApp`
        *   `LoginPage` -> `SignupPage`
        *   `ListsPage` (Home) -> `TodoListPage` (Detail) -> `TodoItemDialog` (Edit)
        *   Drawer -> `sqlConsolePage` (`QueryWidget`)

## 6. Data Flow Analysis

**Scenario:** Adding a new Todo List.

1.  **User Input:** User taps floating action button on `ListsPage` (`lib/widgets/lists_page.dart`).
2.  **Dialog:** `ListItemDialog` prompts for list name.
3.  **Action:** Form submits, calling `TodoList.create(name)` (`lib/models/todo_list.dart`).
4.  **Local DB Action:** `db.execute('INSERT INTO lists...RETURNING *')` writes to the local SQLite database.
5.  **State Update:** The stream `TodoList.watchListsWithStats()` detects the change and fires.
6.  **UI Refresh:** The `StreamBuilder` in `ListsWidget` rebuilds, displaying the new list.
7.  **Background Sync:** The `SupabaseConnector` (`lib/powersync.dart`) pulls the transaction from `db.getNextCrudTransaction()` and executes `table.upsert()` via the Supabase REST API.

## 7. Dependency Injection

*   **Framework used:** None.
*   **Strategy:** **Global Variables & Singletons**.
*   **Location:**
    *   `late final PowerSyncDatabase db;` in `lib/powersync.dart`.
    *   `Supabase.instance.client` (Supabase singleton).
*   **Service Lifecycle:** Services (`db`, Supabase) are initialized once during startup (`main()`) and remain alive for the application's lifetime.

## 8. Backend Communication

*   **Backend Services:** Supabase (PostgreSQL) and PowerSync Service.
*   **Protocol:** REST (via Supabase SDK for writes/auth) and WebSocket/Postgres replication (via PowerSync SDK for reads/sync).
*   **Request Flow:** Reads are always local (SQLite). Writes are made locally and then queued for upload via `SupabaseConnector.uploadData()`.
*   **Authentication:** Supabase Auth is used. Token handling is managed by `SupabaseConnector.fetchCredentials()`, which passes the Supabase session token to PowerSync to authorize sync.
*   **Error handling & Retry:** If `uploadData()` throws a non-fatal exception (e.g., network error), PowerSync automatically retries. Fatal errors (like constraint violations) are caught, logged, and the transaction is discarded to avoid blocking the queue (`powersync.dart:97`).

## 9. Database Layer

*   **Local Storage:** SQLite via `powersync` and `sqlite_async`.
*   **Schema:** Defined in `lib/models/schema.dart` using PowerSync's schema builder.
    *   Tables: `lists`, `todos`, `attachments`.
*   **Models:** `TodoList` (`todo_list.dart`) and `TodoItem` (`todo_item.dart`).
*   **Data synchronization flow:** PowerSync automatically syncs down changes from the server based on sync rules defined on the PowerSync Dashboard. Local changes are synced up via the `SupabaseConnector`.

## 10. Authentication Flow

**Trace:**
1.  **User Input:** Enters email/password on `LoginPage` (`lib/widgets/login_page.dart`).
2.  **API Call:** `Supabase.instance.client.auth.signInWithPassword()`.
3.  **Token Storage:** Handled internally by `supabase_flutter`.
4.  **State Update:** The `onAuthStateChange` listener in `powersync.dart` detects `AuthChangeEvent.signedIn`.
5.  **Service Init:** The listener instantiates `SupabaseConnector` and calls `db.connect()`, initiating the PowerSync connection.
6.  **Route Change:** `Navigator.of(context).pushReplacement` routes the user to `ListsPage`.

## 11. Feature Breakdown

### 1. Auth Feature
*   **Entry point:** `login_page.dart`, `signup_page.dart`
*   **State:** Local `setState` for forms.
*   **APIs:** Supabase Auth.

### 2. Todo Lists Feature
*   **Entry point:** `lists_page.dart`
*   **Main screens:** `ListsPage`, `ListItemDialog`
*   **State:** `TodoList.watchListsWithStats()`
*   **Models:** `TodoList`, `schema.dart`

### 3. Todo Items Feature
*   **Entry point:** `todo_list_page.dart`
*   **Main screens:** `TodoListPage`, `TodoItemWidget`
*   **State:** `TodoList.watchItems()`
*   **Models:** `TodoItem`

## 12. Dependency Graph

```text
UI: ListsPage / TodoListPage
       |
       v
Models: TodoList / TodoItem (Provides DB methods)
       |
       v
Global: PowerSyncDatabase (db) (Local SQLite + Queue)
       |
       v
Connector: SupabaseConnector (Sync Logic)
       |
       v
Backend: Supabase (REST API + Auth) / PowerSync Service (Sync Stream)
```

## 13. Important Models

*   **Core Entities:**
    *   `Schema` (`schema.dart`): Defines the local tables and columns required by PowerSync to map remote data to local SQLite.
    *   `TodoList` (`todo_list.dart`): Immutable class mapping to the `lists` table. Contains static methods for watching data (`watchLists`).
    *   `TodoItem` (`todo_item.dart`): Immutable class mapping to the `todos` table.

## 14. Cross-Cutting Concerns

*   **Logging:** The native `logging` package is used. Configured in `main.dart` with `Logger.root.onRecord.listen(...)`.
*   **Error Handling:** Localized `try/catch` in UI for auth (`LoginPage`), and global exception catching within the PowerSync upload queue (`SupabaseConnector`).
*   **Offline Support:** Native to the architecture. The application continues functioning seamlessly when offline, queuing mutations and executing them when connectivity is restored.

## 15. Architectural Strengths & Weaknesses

### Good Decisions
*   **Instant UI Updates:** Utilizing local SQLite as the source of truth provides an exceptionally fast user experience.
*   **Simplified State Management:** By treating the database queries as reactive Streams, the codebase avoids the boilerplate of complex state management libraries like BLoC or Redux.

### Potential Problems
*   **Global Variables:** The use of `late final PowerSyncDatabase db;` and static model methods makes unit testing harder. Moving to a dependency injection approach would improve testability.
*   **Navigation:** Hardcoded `MaterialPageRoute` calls are fine for a demo, but scale poorly for deep linking.

### Refactoring Opportunities
*   Implement dependency injection (e.g., `get_it` or `Provider`) to pass the database instance to the UI and Models.
*   Adopt a routing library like `go_router` for a more declarative navigation structure.

## 16. Learning Guide

**Most Important Files to Read First:**
1.  `lib/powersync.dart` - Understanding how the app connects to PowerSync and Supabase is the crux of the application.
2.  `lib/models/schema.dart` - To understand what data the app handles.
3.  `lib/models/todo_list.dart` - To see how SQLite queries are transformed into Dart streams.
4.  `lib/widgets/lists_page.dart` - To see how the UI consumes those streams.

**Safe to Ignore Initially:**
*   `lib/attachments/` (Complex photo syncing logic)
*   `lib/widgets/sql_console_page.dart` (Debugging utility)

**Roadmap to Productivity:**
1.  Read the PowerSync architecture overview on their website to understand *why* local SQLite is used.
2.  Review `schema.dart` to understand the domain.
3.  Trace the `watchLists()` stream from `todo_list.dart` to the `StreamBuilder` in `lists_page.dart`.
4.  Trace the write path: `TodoList.create()` to the `INSERT` SQL query, and observe how `SupabaseConnector.uploadData()` handles it.

## 17. Evidence Requirement

*   **Architecture / Global DB:** `demos/supabase-todolist/lib/powersync.dart` contains `late final PowerSyncDatabase db;`.
*   **Startup Flow:** `demos/supabase-todolist/lib/main.dart` contains `openDatabase()` before `runApp()`.
*   **State Management:** `demos/supabase-todolist/lib/models/todo_list.dart` contains `db.watch('SELECT * FROM lists...').map(...)`. `demos/supabase-todolist/lib/widgets/lists_page.dart` uses `StreamBuilder(stream: TodoList.watchListsWithStats())`.
*   **Backend Connector:** `demos/supabase-todolist/lib/powersync.dart` defines `SupabaseConnector extends PowerSyncBackendConnector`.
*   **Auth Flow:** `demos/supabase-todolist/lib/widgets/login_page.dart` line 30 calls `Supabase.instance.client.auth.signInWithPassword()`.
