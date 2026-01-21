# Coding Conventions

## 1. Providers (Ngin)

*   **Constructor**: `constructor(deps, { engineFeed })`
    *   `deps`: Object containing injected **Provider Instances**.
    *   `engineFeed`: The system-wide event emitter.
*   **Resource Management**: You must manually call `.obtain()` on injected providers to get their resource, and `.release()` when done.
*   **Lifecycle**:
    *   `obtain()`: Setup resources for a consumer.
    *   `dispose()`: **Called ONLY on `engine.dispose()**`. Permanent shutdown cleanup.

**Example Provider:**

```javascript
import { Provider } from 'ngin';

export class UserProvider extends Provider {
  static deps = ['db']; 

  constructor({ db }, { engineFeed }) {
    super();
    this.dbProvider = db;
  }

  async obtain() {
    const db = await this.dbProvider.obtain(); // Get resource
    const resource = { /* ... */ };
    this.dbProvider.release(); // Release dependency
    return resource;
  }

  async dispose() {
    console.log("Shutting down UserProvider...");
  }
}
```

## 2. Business Logic (Ngin Domains)

*   **Pattern**: Domain files exporting `Action` and `Query` classes.
*   **Execution**:
    *   **Actions**: `async execute({ dep })`. Use `constructor` for payloads.
    *   **Queries**: `async boot({ dep }, { notify })`. Use dynamic `this.kill` inside boot if closures are needed.

**Example Domain Module:**

```javascript
import { Action, Query } from 'ngin';

export class AddTodoAction extends Action {
  static deps = ['todoState']; 
  constructor(text) { super(); this.text = text; }
  async execute({ todoState }) {
    todoState.add({ text: this.text });
  }
}

export class TodosQuery extends Query {
  static deps = ['todoState']; 
  async boot({ todoState }, { notify }) {
    notify(todoState.get());
    const onUpdate = (data) => notify(data);
    todoState.subscribe(onUpdate);
    // Dynamic cleanup to access closure state
    this.kill = async () => todoState.unsubscribe(onUpdate);
  }
}
```

## 3. UI Layer: The Separation of Concerns

All UI modules are factories that must use **`export default`**.

### A. Components (Pure UI)

*   **Role**: Reusable, dumb building blocks.
*   **Restrictions**: **NEVER** inject the `engine`.
*   **Communication**: Dispatch **Custom DOM Events** using `this.dispatchEvent` (via `alias`).

```javascript
// src/ui/components/saveButton.js
import { alias, button } from '@3sln/dodo';

// Injects THEME, not ENGINE
export default ({ theme }) => {
  return alias((label) => {
    return button(label)
      .style({ color: theme.primary })
      .on({ 
        // Component is unaware of Actions/Ngin. Just signals intent.
        click: () => this.dispatchEvent(new CustomEvent('save-requested', { bubbles: true }))
      });
  });
};
```

### B. Compositions (App Fragments)

*   **Role**: Wires Components to Business Logic.
*   **Privileges**: **MUST** inject the `engine`.
*   **Communication**: Listens to Component events -> Dispatches Ngin Actions. Subscribes to Ngin Queries -> Updates UI.

```javascript
// src/ui/compositions/todoForm.js
import { alias, div } from '@3sln/dodo';
import { AddTodoAction } from '../../bl/todos.js';

// Injects ENGINE and COMPONENTS
export default ({ engine, saveButton, textInput }) => {
  
  return alias(() => {
    let currentText = '';

    return div(
      textInput().on({ input: (e) => currentText = e.target.value }),
      
      saveButton('Add Item').on({ 
        // Catch the custom DOM event from the pure component
        'save-requested': () => {
             if(currentText) engine.dispatch(new AddTodoAction(currentText));
        }
      })
    );
  });
};
```

## 4. Advanced: Special Components & Custom Elements

For performance-critical components or integration with complex third-party libraries (like map renderers or chart libraries), it is idiomatic to "drop down" to the DOM layer using `special`.

### `special` Components

A `special` component allows direct DOM manipulation via `attach`, `update`, and `detach` hooks.

```javascript
import { special } from '@3sln/dodo';

export default () => {
  return special({
    attach: (element) => {
      // Called when the element is first created
      element.innerHTML = '<canvas></canvas>';
      element.ctx = element.querySelector('canvas').getContext('2d');
    },
    update: (element, [data], [oldData]) => {
      // Called whenever arguments change
      // Perform efficient, imperative DOM updates here
      element.ctx.fillRect(0, 0, data.width, data.height);
    },
    detach: (element) => {
      // Cleanup
      element.ctx = null;
    }
  });
};
```

### Reconciling onto Custom Elements

Both `alias` and `special` VDOM nodes can be reconciled *directly* onto any HTML element, including Custom Elements. When you do this, the component "takes over" the host element.

This is the standard pattern for integrating `dodo` with Web Components.

```javascript
import { reconcile } from '@3sln/dodo';
import myChartComponent from './ui/components/chart.js'; // A special() component

class MyChartElement extends HTMLElement {
  connectedCallback() {
    this.chart = myChartComponent(); // Create the VDOM factory
    this.render();
  }

  render() {
    // Reconcile the 'special' component DIRECTLY onto 'this' (the custom element)
    // The 'special' component's attach/update hooks will receive 'this' as the element.
    reconcile(this, this.chart({ data: this.data }));
  }
  
  set data(val) {
    this._data = val;
    this.render();
  }
  get data() { return this._data; }
}

customElements.define('my-chart', MyChartElement);
```

## 🚫 Anti-Patterns (What to Avoid)

*   ❌ **PascalCase Components**: **NEVER** use `UpperCamelCase` for component names. We are not in React. Always use `lowerCamelCase` (e.g., `saveButton`, not `SaveButton`).
*   ❌ **Engine in Components**: Do not pass `engine` to `src/ui/components`. Only Compositions know about the engine.
*   ❌ **Provider Confusion**: Injected Provider dependencies are **Providers**, not resources. You must `.obtain()` them.
*   ❌ **No Action Chaining**: Actions cannot call other Actions. Use `engineFeed`.
*   ❌ **No React Props**: Do not pass `onClick`. Use `.on({ click: ... })`.
*   ❌ **No Folder Split**: Keep Actions and Queries together in `bl/[domain].js`.
*   ❌ **No Named Exports for Factories**: UI Factories should always be `export default`.
