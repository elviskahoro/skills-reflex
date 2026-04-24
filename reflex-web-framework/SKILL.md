---
name: reflex-web-framework
description: Use when building web apps with Reflex (reflex.dev), creating frontend UIs in pure Python, managing reactive state, defining pages and routing, integrating databases, or deploying Reflex applications. Default to Buridan UI component library for pre-built, customizable components.
license: Proprietary
compatibility: Requires Python 3.11+ (for Buridan), reflex pip package, buridan-ui CLI. Node.js installed for dev server. Optional SQLModel for database, Redis for production state.
metadata:
  author: elviskahoro
  version: "2.0"
  tags: [reflex, python, web, frontend, fullstack, react, ui, buridan]
---

# Reflex Web Framework

Reflex lets you build full-stack web apps in pure Python. It compiles your Python components to a React/Next.js frontend and runs a FastAPI backend, connected via WebSocket for real-time state sync. No JavaScript required.

## When to Use

- Building a web frontend or full-stack app in Python
- Creating interactive UIs with reactive state management
- Adding pages, routing, and navigation to a Reflex app
- Working with Reflex components (Radix UI, charts, forms, tables)
- Managing server-side state with `rx.State` subclasses
- Integrating databases via SQLModel/SQLAlchemy
- Adding authentication to a Reflex app
- Deploying a Reflex app (self-hosted or Reflex Cloud)
- Wrapping custom React components for use in Reflex

## Execution Steps for Agents

### Step 1: Project Setup

```bash
# Install Reflex
pip install reflex

# Create a new project (interactive template selection)
reflex init

# Start the dev server (frontend on :3000, backend on :8000)
reflex run
```

Project structure after `reflex init`:

```
my_app/
  rxconfig.py              # App config: name, api_url, db_url
  my_app/
    __init__.py             # Must import all pages, state, models
    my_app.py               # Main module with app = rx.App()
    state.py                # Shared state classes
    pages/                  # One module per page
      index.py
    components/             # Reusable UI components
  assets/                   # Static files (images, fonts)
```

**Critical rule:** The main module name must match `app_name` in `rxconfig.py`. All pages, state classes, and models must be imported (directly or indirectly) by the main module to be compiled.

### Step 2: Build Components

Components are Python functions returning `rx.Component`. Children are positional args, props are keyword args.

```python
import reflex as rx

def index() -> rx.Component:
    return rx.box(
        rx.heading("My App", size="3"),
        rx.hstack(
            rx.avatar(src="/logo.png"),
            rx.text("Welcome to Reflex"),
        ),
        rx.button("Click me", on_click=MyState.handle_click),
    )
```

Use `rx.el` namespace for raw HTML elements:

```python
def raw_html() -> rx.Component:
    return rx.el.div(
        rx.el.p("Raw HTML paragraph"),
        rx.el.a("Link", href="/about"),
    )
```

See [Components Reference](references/COMPONENTS.md) for the full component library.

### Step 3: Manage State

State lives server-side in `rx.State` subclasses. Each browser tab gets its own state instance. **Event handlers are the ONLY way to change state.**

```python
class AppState(rx.State):
    count: int = 0
    name: str = ""

    @rx.var
    def greeting(self) -> str:
        """Computed var — recalculates when dependencies change."""
        return f"Hello, {self.name}!" if self.name else "Hello!"

    def increment(self):
        """Event handler — the only way to mutate state."""
        self.count += 1

    def set_name(self, value: str):
        self.name = value
```

Reference state vars in components via class attributes:

```python
def counter() -> rx.Component:
    return rx.hstack(
        rx.heading(AppState.count),
        rx.button("Increment", on_click=AppState.increment),
        rx.input(on_blur=AppState.set_name),
        rx.text(AppState.greeting),
    )
```

See [State Management Reference](references/STATE_MANAGEMENT.md) for substates, computed vars, and `get_state()`.

### Step 4: Understand Compile-Time vs. Runtime (CRITICAL)

The frontend compiles to JavaScript (compile-time). The backend stays Python (runtime). This distinction is the most common source of bugs in Reflex apps.

**You CANNOT** use arbitrary Python on state vars in component definitions:

```python
# WRONG — will error. State vars are not Python values at compile time.
rx.text(len(MyState.text))
rx.text("even" if MyState.count % 2 == 0 else "odd")
for item in MyState.items:  # WRONG
    rx.text(item)
```

**Instead use** Reflex primitives:

```python
# Conditional rendering — use rx.cond instead of if/else
rx.cond(MyState.count % 2 == 0, rx.text("Even"), rx.text("Odd"))

# List rendering — use rx.foreach instead of for loops
rx.foreach(MyState.items, lambda item: rx.text(item))

# Var operations — use operators on vars directly
rx.text(MyState.count * 2)  # arithmetic works
rx.cond(MyState.name == "", rx.text("Anonymous"), rx.text(MyState.name))

# Pattern matching — use rx.match for multiple conditions
rx.match(
    MyState.status,
    ("active", rx.badge("Active", color_scheme="green")),
    ("pending", rx.badge("Pending", color_scheme="yellow")),
    rx.badge("Unknown"),  # default
)
```

**You CAN** use any Python inside event handlers (they run server-side):

```python
def increment(self):
    import random  # Any Python library works here
    self.count += random.randint(1, 10)
```

### Step 5: Define Pages and Routing

```python
# Method 1: @rx.page decorator (preferred)
@rx.page(route="/", title="Home")
def index() -> rx.Component:
    return rx.text("Home page")

@rx.page(route="/about", title="About")
def about() -> rx.Component:
    return rx.text("About page")

# Method 2: app.add_page
app = rx.App()
app.add_page(index)
app.add_page(about, route="/about")
```

Dynamic routes with path parameters:

```python
@rx.page(route="/post/[pid]")
def post() -> rx.Component:
    return rx.text(f"Post: {PostState.pid}")

# Catch-all routes
@rx.page(route="/docs/[...slug]")
def docs() -> rx.Component:
    return rx.text("Docs catch-all")
```

Navigation: `rx.link("About", href="/about")` for declarative links, `rx.redirect("/path")` from event handlers.

### Step 6: Style Components

CSS properties as snake_case keyword args:

```python
rx.button(
    "Styled",
    background_color="blue",
    border_radius="15px",
    font_size="18px",
    padding="10px 20px",
    _hover={"background_color": "darkblue"},  # Pseudo-selectors
)
```

See [Styling Reference](references/STYLING.md) for theming, responsive design, and global styles.

### Step 7: Add Database Integration

Reflex uses SQLModel (SQLAlchemy wrapper) with Alembic migrations. SQLite default, PostgreSQL for production.

```python
class User(rx.Model, table=True):
    username: str
    email: str
    age: int

# In an event handler:
def get_users(self):
    with rx.session() as session:
        self.users = session.exec(User.select()).all()
```

```bash
reflex db init                          # Initialize Alembic
reflex db makemigrations -m "add user"  # Generate migration
reflex db migrate                       # Apply migrations
```

See [Database and API Reference](references/DATABASE_AND_API.md) for queries, API routes, and authentication.

### Step 8: Deploy

```bash
# Production build
reflex run --env prod

# Static export (frontend.zip + backend.zip)
reflex export

# Reflex Cloud (one-command deploy)
reflex deploy
```

Self-hosting requires setting `api_url` in `rxconfig.py` to the server's public address and Redis for production state management.

See [Deployment Reference](references/DEPLOYMENT.md) for Docker, Nginx, and cloud deployment patterns.

## Quick Reference

| Import | Purpose |
|--------|---------|
| `rx.State` | Base class for reactive state |
| `rx.Model` | Base class for database models (SQLModel) |
| `rx.Component` | Return type for component functions |
| `rx.App()` | App instance (define once in main module) |
| `rx.session()` | Database session context manager |
| `rx.cond()` | Conditional rendering (replaces if/else) |
| `rx.foreach()` | List rendering (replaces for loops) |
| `rx.match()` | Pattern matching (replaces if/elif chains) |
| `rx.redirect()` | Programmatic navigation |
| `@rx.page()` | Page decorator with route, title, meta |
| `@rx.var` | Computed var decorator |

| Pattern | Example |
|---------|---------|
| Create component | `def page() -> rx.Component: return rx.box(...)` |
| Reference state var | `rx.text(MyState.name)` |
| Event trigger | `rx.button("Go", on_click=MyState.handler)` |
| Event with args | `on_click=lambda: MyState.handler(arg)` |
| Input binding | `rx.input(on_blur=MyState.set_name)` |
| Conditional | `rx.cond(MyState.flag, true_comp, false_comp)` |
| List iteration | `rx.foreach(MyState.items, render_fn)` |
| Match expression | `rx.match(MyState.status, ("val", comp), default_comp)` |
| Dynamic route | `@rx.page(route="/item/[id]")` |
| Static file | `rx.image(src="/filename.png")` (from `assets/`) |
| Computed var | `@rx.var\ndef full_name(self) -> str:` |
| Substate access | `other = await self.get_state(OtherState)` |

## Common Mistakes to Avoid

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using `if` on state vars in components | State vars are not Python values at compile time | Use `rx.cond(var, true_comp, false_comp)` |
| Using `for` loop over state var lists | List vars are not iterable at compile time | Use `rx.foreach(var_list, render_fn)` |
| Calling `len()` on state var in component | Python builtins don't work on vars | Use `.length()` var operation or compute in event handler |
| Mutating state outside event handlers | State changes won't propagate to frontend | Only modify state inside event handler methods |
| Deep substate inheritance | Creates coupling and performance issues | Keep substates flat, use `get_state()` for cross-state access |
| Forgetting to import pages/models | Unimported modules won't be compiled | Ensure main `__init__.py` imports all pages and models |
| Using `rx.Var[str]` as plain `str` in foreach | Foreach render functions receive `Var`, not plain values | Type hint as `rx.Var[str]` and use var operations |
| Not setting `api_url` in production | Frontend can't reach backend on deployed server | Set `api_url` in `rxconfig.py` to server's public address |
| Using in-memory state in production | State lost on restart, doesn't scale | Configure Redis as state manager for production |
| Putting secrets in state vars | State vars may be sent to frontend | Use backend-only vars (underscore prefix) for sensitive data |
| Forgetting auto-generated setters exist | Writing custom `set_X` handlers unnecessarily | Base vars auto-generate `set_varname` — use directly |

## References

- [Components Reference](references/COMPONENTS.md) — Component library, layout, forms, data display
- [State Management Reference](references/STATE_MANAGEMENT.md) — State, vars, event handlers, substates
- [Styling Reference](references/STYLING.md) — CSS props, theming, responsive design
- [Database and API Reference](references/DATABASE_AND_API.md) — SQLModel, queries, API routes, auth
- [Deployment Reference](references/DEPLOYMENT.md) — Self-hosting, Docker, Reflex Cloud
- [Complete Examples](references/EXAMPLES.md) — End-to-end app examples
