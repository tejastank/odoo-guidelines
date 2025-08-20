# Odoo 18.0 — Frontend & Web Client (Expert)

This guide covers advanced frontend development for Odoo 18.0: OWL, JavaScript patterns, widget development, RPC, assets, and UI performance tips.

## Table of contents
- OWL (Odoo Web Library) patterns
- Components and state management
- RPC and services
- Widgets and legacy JS
- Assets and bundling
- QWeb templates for client-side
- Performance and accessibility

---

## OWL Patterns

Odoo 18 uses OWL for the web client and front-end components. Prefer OWL components for new features.

Example basic OWL component:

```javascript
// static/src/js/components/counter.js
import { Component, useState } from 'owl';

export class Counter extends Component {
    setup() {
        this.state = useState({ count: 0 });
    }

    increment() {
        this.state.count += 1;
    }
}
Counter.template = 'my_module.Counter';
```

QWeb template:

```xml
<t t-name="my_module.Counter">
  <div>
    <button t-on-click="increment">+</button>
    <span t-esc="state.count"/>
  </div>
</t>
```

Tips:
- Use `useState` or `useRef` for reactive state.
- Create small, testable components and compose them.

---

## RPC and Services

- Use `this.env.services.rpc` or `rpc.query` to call server methods.
- Use `session.rpc` for session-aware RPC calls in legacy code.

Example:

```javascript
import { rpc } from 'web.rpc';

rpc.query({
    model: 'res.partner',
    method: 'search_read',
    args: [[['is_company', '=', true]] , ['name', 'country_id']],
}).then(result => console.log(result));
```

Use `await rpc.query(...)` in OWL async functions to keep code readable.

---

## Widgets & Legacy JS

- For small customizations on existing views, you might still use legacy widgets, but prefer OWL.
- Use `registry.category('views')` to register view components.

---

## Assets and Bundling

- Put JS/SCSS under `static/src/js` and `static/src/scss` and reference them in `__manifest__` under `web.assets_*`.
- Minimize bundle size and tree-shake unused modules.

---

## QWeb templates for client-side

- Keep templates small and logic-light. Heavy logic should be in JS components.
- Use `t-call` to reuse templates.

---

## Performance & Accessibility

- Avoid heavy DOM updates; use OWL reactivity.
- Use `requestAnimationFrame` for visual updates.
- Ensure ARIA attributes and keyboard navigation for widgets.

---

This file contains patterns and snippets for OWL and frontend development. See `theme.md` for website/theme specific guidance.
