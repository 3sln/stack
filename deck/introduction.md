# The 3sln Stack

The 3sln stack is a cohesive ecosystem for building performant, scalable, and maintainable web applications. It emphasizes a strict separation of concerns, dependency injection, and a reactive, event-driven architecture.

## Core Libraries

The stack is composed of three primary libraries:

*   **[Ngin](https://ngin.3sln.com)**: A general-purpose state management and business logic abstraction layer using dependency injection.
*   **[Dodo](https://dodo.3sln.com)**: A high-performance, functional VDOM library for building UI components. Reactivity, context, element observation, animation and scoped styling come as optional modules alongside it — `@3sln/dodo/reactive`, `/context`, `/observe`, `/animate` and `/style` — which the core neither imports nor knows about.
*   **[Deck](https://deck.3sln.com)**: A documentation and playground tool for building and showcasing your project.

## Philosophy

The 3sln stack promotes:

1.  **Strict Separation of Concerns**: UI components are "dumb" and unaware of business logic. Business logic is pure and unaware of the UI.
2.  **Dependency Injection**: All resources and dependencies are managed explicitly via `ngin` Providers.
3.  **Event-Driven Communication**: Components signal intent via DOM events; Business Logic signals state changes via observables and event feeds.
4.  **Factory Pattern**: UI components are created via factory functions to allow for dependency injection (like themes or sub-components) at the composition root.
