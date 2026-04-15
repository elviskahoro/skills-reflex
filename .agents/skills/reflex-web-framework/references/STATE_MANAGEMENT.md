# State Management Reference

Complete reference for Reflex state management: vars, event handlers, substates, and cross-state access.

## Core Concepts

Reflex state lives **server-side**. Each browser tab gets a unique state instance. The framework tracks "dirty vars" and sends only changed state to the frontend via WebSocket.

## Defining State

```python
import reflex as rx

class AppState(rx.State):
    # Base vars — type-annotated fields with defaults
    count: int = 0
    name: str = ""
    items: list[str] = []
    user_data: dict[str, str] = {}
    loading: bool = False
```

### Var Types

| Type | Example | Notes |
|------|---------|-------|
| `int`, `float`, `str`, `bool` | `count: int = 0` | Basic types |
| `list[T]` | `items: list[str] = []` | Use `rx.foreach` to render |
| `dict[K, V]` | `data: dict[str, int] = {}` | Serializable values only |
| `Optional[T]` | `user: str \| None = None` | Nullable vars |
| Custom dataclass/BaseModel | `user: UserInfo = UserInfo()` | Must be serializable |

### Backend-Only Vars

Prefix with underscore to prevent sending to frontend. Use for secrets and sensitive data:

```python
class AuthState(rx.State):
    _api_key: str = ""        # Never sent to frontend
    _session_token: str = ""  # Never sent to frontend
    username: str = ""        # Sent to frontend for display
```

## Computed Vars

Computed vars automatically recalculate when their dependencies change. Use `@rx.var` decorator:

```python
class TodoState(rx.State):
    todos: list[dict] = []
    show_completed: bool = False

    @rx.var
    def visible_todos(self) -> list[dict]:
        if self.show_completed:
            return self.todos
        return [t for t in self.todos if not t["done"]]

    @rx.var
    def todo_count(self) -> int:
        return len(self.todos)

    @rx.var
    def completion_pct(self) -> float:
        if not self.todos:
            return 0.0
        done = sum(1 for t in self.todos if t["done"])
        return done / len(self.todos) * 100
```

Reference computed vars the same as base vars: `TodoState.todo_count`.

## Event Handlers

Event handlers are methods on state classes. They are the **only way** to change state.

```python
class FormState(rx.State):
    name: str = ""
    email: str = ""
    submitted: bool = False

    def set_name(self, value: str):
        """Called by on_blur or on_change — receives the input value."""
        self.name = value

    def set_email(self, value: str):
        self.email = value

    def submit(self):
        """Called by on_click — no extra arguments."""
        if self.name and self.email:
            self.submitted = True

    def reset_form(self):
        """Reset to defaults."""
        self.name = ""
        self.email = ""
        self.submitted = False
```

### Event Triggers

Common event triggers on components:

| Trigger | Component | Argument Passed |
|---------|-----------|-----------------|
| `on_click` | Most components | None |
| `on_change` | `input`, `select`, `checkbox`, `switch`, `slider` | New value |
| `on_blur` | `input`, `text_area` | Current value |
| `on_submit` | `form` | `dict` of form data (keyed by `name` prop) |
| `on_key_down` | `input` | Key event string |
| `on_mount` | Any page (via `@rx.page(on_load=...)`) | None |

### Passing Arguments to Event Handlers

Use lambda when the event trigger doesn't provide the right args:

```python
# No args from on_click, but handler needs one
rx.button("Add 5", on_click=lambda: MyState.increment(5))

# Multiple buttons with different args
rx.foreach(
    MyState.items,
    lambda item: rx.button(
        "Delete",
        on_click=lambda: MyState.delete_item(item),
    ),
)
```

### Yielding Multiple Updates

Event handlers can yield to send intermediate state updates:

```python
import asyncio

class LoadingState(rx.State):
    progress: int = 0
    loading: bool = False

    async def process_data(self):
        self.loading = True
        yield  # Send loading=True to frontend immediately
        for i in range(100):
            await asyncio.sleep(0.05)
            self.progress = i + 1
            yield  # Send progress update
        self.loading = False
```

### Returning Events

Event handlers can return special events:

```python
def submit(self):
    self.save_to_db()
    return rx.redirect("/success")  # Navigate after saving

def show_message(self):
    return rx.toast("Operation complete!")  # Show toast

def download(self):
    return rx.download(url="/files/report.pdf", filename="report.pdf")
```

### Page Load Events

Run event handlers when a page loads:

```python
@rx.page(route="/dashboard", on_load=DashboardState.load_data)
def dashboard() -> rx.Component:
    return rx.text("Dashboard")

class DashboardState(rx.State):
    data: list = []

    def load_data(self):
        with rx.session() as session:
            self.data = session.exec(MyModel.select()).all()
```

## Substates

For large apps, split state into focused classes. **Keep the hierarchy flat** — most substates should inherit directly from `rx.State`.

```python
class AuthState(rx.State):
    """Shared auth state."""
    user_id: str = ""
    logged_in: bool = False

class DashboardState(rx.State):
    """Dashboard-specific state. Flat — inherits from rx.State directly."""
    metrics: list[dict] = []
    date_range: str = "7d"

    async def load_metrics(self):
        # Access another state via get_state
        auth = await self.get_state(AuthState)
        if not auth.logged_in:
            return rx.redirect("/login")
        self.metrics = fetch_metrics(auth.user_id, self.date_range)

class SettingsState(rx.State):
    """Settings-specific state. Also flat."""
    theme: str = "light"
    notifications: bool = True
```

### Cross-State Access with `get_state()`

The `get_state()` API lets any event handler access and modify other substates without inheritance coupling:

```python
class CartState(rx.State):
    items: list[dict] = []
    total: float = 0.0

    async def checkout(self):
        auth = await self.get_state(AuthState)
        if not auth.logged_in:
            return rx.redirect("/login")

        # Modify another state
        order = await self.get_state(OrderState)
        order.create_order(auth.user_id, self.items, self.total)

        # Clear cart
        self.items = []
        self.total = 0.0
```

### When to Inherit vs. Use `get_state()`

| Pattern | Use When |
|---------|----------|
| Direct `rx.State` inheritance (flat) | Default for all new substates |
| Inherit from another state | Parent holds data the child always needs (rare) |
| `get_state()` | Accessing another state on demand |

## Dynamic Route Parameters

Dynamic route params are automatically available as state vars:

```python
class PostState(rx.State):
    @rx.var
    def post_id(self) -> str:
        return self.router.page.params.get("pid", "")

@rx.page(route="/post/[pid]")
def post_page() -> rx.Component:
    return rx.text(PostState.post_id)
```

## Router Data

Access current page info via `self.router`:

```python
class NavState(rx.State):
    @rx.var
    def current_path(self) -> str:
        return self.router.page.path

    @rx.var
    def query_params(self) -> dict:
        return self.router.page.params
```

## Private Helper Methods

Prefix with underscore to keep methods private (not exposed as event handlers):

```python
class MyState(rx.State):
    items: list[str] = []

    def _validate_item(self, item: str) -> bool:
        """Private — not callable from frontend."""
        return len(item) > 0 and item not in self.items

    def add_item(self, item: str):
        """Public event handler."""
        if self._validate_item(item):
            self.items.append(item)
```

## Auto-Generated Setters

Every base var automatically generates a `set_varname` event handler:

```python
class MyState(rx.State):
    name: str = ""
    count: int = 0

# These work without defining any handlers:
rx.input(on_change=MyState.set_name)      # Auto-generated
rx.slider(on_change=MyState.set_count)    # Auto-generated
```

## Background Tasks

Long-running operations that shouldn't block the UI use `background=True`:

```python
@rx.event(background=True)
async def fetch_data(self):
    result = await slow_api_call()
    async with self:  # Acquire state lock to modify vars
        self.data = result
        self.loading = False
```

Background tasks run concurrently and must use `async with self` to safely modify state.

## Client-Side Storage

Store data in cookies or browser localStorage:

```python
class MyState(rx.State):
    auth_token: str = rx.Cookie("")           # Stored in cookie
    preferences: str = rx.LocalStorage("")    # Stored in localStorage
    session_data: str = rx.Cookie(name="sid") # Custom cookie name
```

## Performance: rx.memo

Prevent unnecessary re-renders for components whose props haven't changed:

```python
@rx.memo
def expensive_card(title: str, content: str) -> rx.Component:
    return rx.card(rx.heading(title), rx.text(content))
```

`rx.memo` requires type-annotated keyword args. Use for components that render in `rx.foreach` or have expensive subtrees.
