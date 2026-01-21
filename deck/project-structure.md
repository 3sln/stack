# Project Structure

Agents must enforce this folder structure.

```text
src/
├── bl/                  # Business Logic (Domain Modules)
│   ├── auth.js          # Exports AuthAction & AuthQuery classes
│   ├── todos.js         # Exports TodoAction & TodoQuery classes
│   └── ...
├── providers/           # Resource owners & State containers
│   ├── [resource].js    # e.g., authProvider.js (extends Provider)
│   └── state.js         # Singleton global state (if needed)
├── ui/                  # Dodo VDOM layer
│   ├── components/      # PURE UI Factories (No Engine, export default)
│   ├── compositions/    # APP UI Factories (Inject Engine, export default)
│   └── ...
├── deck/                # Documentation cards
├── config.dev.json      # Environment config
└── config.prod.json
```
