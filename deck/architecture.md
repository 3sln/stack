# Architecture

The 3sln stack enforces a layered architecture to maintain scalability and testability.

## Layers

### 1. Infrastructure Layer (Providers)
Managed by `ngin` Providers. This layer handles external resources, API connections, databases, and global configuration.

*   **Responsibility**: Lifecycle management of resources.
*   **Examples**: `UserProvider`, `DatabaseProvider`, `ConfigProvider`.

### 2. Business Logic Layer (Domains)
Managed by `ngin` Actions and Queries. This layer contains the core logic of the application, independent of any specific UI.

*   **Actions**: "Verbs" that perform side effects or state updates (e.g., `AddTodoAction`).
*   **Queries**: "Nouns" that provide reactive data streams (e.g., `TodosQuery`).
*   **Organization**: Grouped by domain in `src/bl/`.

### 3. Composition Layer (Compositions)
This is the "glue" layer. Compositions are UI factories that inject the `engine` and coordinate between the Business Logic and the pure UI Components.

*   **Responsibility**: Wiring state to UI and UI events to Actions.
*   **Access**: Has access to `engine` and `components`.
*   **Location**: `src/ui/compositions/`.

### 4. Presentation Layer (Components)
Pure, "dumb" UI components built with `dodo`. They rely solely on props (passed via factory arguments) and communicate up via custom DOM events.

*   **Responsibility**: Rendering UI and capturing user input.
*   **Restrictions**: No access to `engine` or business logic.
*   **Communication**: Dispatches standard or custom DOM events.
*   **Location**: `src/ui/components/`.

## Data Flow

1.  **User Interaction**: User clicks a button in a **Component**.
2.  **Event Dispatch**: Component dispatches a custom DOM event (e.g., `save-requested`).
3.  **Event Handling**: **Composition** listens for the event.
4.  **Action Dispatch**: Composition dispatches an `ngin` **Action** (e.g., `SaveDataAction`).
5.  **State Update**: Action executes, modifies state/resource via **Providers**.
6.  **Reactive Update**: State change triggers an `ngin` **Query**.
7.  **UI Update**: Composition subscribes to the Query and updates the **Component**.
