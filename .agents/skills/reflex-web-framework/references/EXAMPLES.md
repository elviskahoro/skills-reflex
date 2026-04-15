# Complete Examples

End-to-end Reflex application examples demonstrating common patterns.

## Example 1: Todo App

A full CRUD todo application with database persistence.

### rxconfig.py

```python
import reflex as rx

config = rx.Config(
    app_name="todo_app",
    db_url="sqlite:///reflex.db",
)
```

### todo_app/models.py

```python
import reflex as rx
from sqlmodel import Field
from datetime import datetime


class Todo(rx.Model, table=True):
    title: str
    completed: bool = Field(default=False)
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

### todo_app/state.py

```python
import reflex as rx
from .models import Todo


class TodoState(rx.State):
    todos: list[Todo] = []
    new_todo: str = ""
    filter_mode: str = "all"  # "all", "active", "completed"

    @rx.var
    def visible_todos(self) -> list[Todo]:
        if self.filter_mode == "active":
            return [t for t in self.todos if not t.completed]
        elif self.filter_mode == "completed":
            return [t for t in self.todos if t.completed]
        return self.todos

    @rx.var
    def active_count(self) -> int:
        return sum(1 for t in self.todos if not t.completed)

    def load_todos(self):
        with rx.session() as session:
            self.todos = session.exec(
                Todo.select().order_by(Todo.created_at.desc())
            ).all()

    def set_new_todo(self, value: str):
        self.new_todo = value

    def add_todo(self):
        if not self.new_todo.strip():
            return
        with rx.session() as session:
            todo = Todo(title=self.new_todo.strip())
            session.add(todo)
            session.commit()
        self.new_todo = ""
        self.load_todos()

    def toggle_todo(self, todo_id: int):
        with rx.session() as session:
            todo = session.get(Todo, todo_id)
            if todo:
                todo.completed = not todo.completed
                session.add(todo)
                session.commit()
        self.load_todos()

    def delete_todo(self, todo_id: int):
        with rx.session() as session:
            todo = session.get(Todo, todo_id)
            if todo:
                session.delete(todo)
                session.commit()
        self.load_todos()

    def set_filter(self, mode: str):
        self.filter_mode = mode
```

### todo_app/pages/index.py

```python
import reflex as rx
from ..state import TodoState


def todo_item(todo: rx.Var) -> rx.Component:
    return rx.hstack(
        rx.checkbox(
            checked=todo.completed,
            on_change=lambda: TodoState.toggle_todo(todo.id),
        ),
        rx.text(
            todo.title,
            text_decoration=rx.cond(todo.completed, "line-through", "none"),
            color=rx.cond(todo.completed, "gray", "inherit"),
            flex="1",
        ),
        rx.button(
            rx.icon(tag="trash-2", size=16),
            variant="ghost",
            color_scheme="red",
            size="1",
            on_click=lambda: TodoState.delete_todo(todo.id),
        ),
        width="100%",
        padding="8px",
        align="center",
    )


@rx.page(route="/", title="Todo App", on_load=TodoState.load_todos)
def index() -> rx.Component:
    return rx.container(
        rx.vstack(
            rx.heading("Todo App", size="7"),
            rx.hstack(
                rx.input(
                    placeholder="What needs to be done?",
                    value=TodoState.new_todo,
                    on_change=TodoState.set_new_todo,
                    on_key_down=rx.cond(
                        rx.Var.create("Enter") == rx.Var.create("Enter"),
                        TodoState.add_todo,
                    ),
                    flex="1",
                ),
                rx.button("Add", on_click=TodoState.add_todo),
                width="100%",
            ),
            rx.hstack(
                rx.button(
                    "All",
                    variant=rx.cond(TodoState.filter_mode == "all", "solid", "outline"),
                    on_click=lambda: TodoState.set_filter("all"),
                    size="1",
                ),
                rx.button(
                    "Active",
                    variant=rx.cond(TodoState.filter_mode == "active", "solid", "outline"),
                    on_click=lambda: TodoState.set_filter("active"),
                    size="1",
                ),
                rx.button(
                    "Completed",
                    variant=rx.cond(TodoState.filter_mode == "completed", "solid", "outline"),
                    on_click=lambda: TodoState.set_filter("completed"),
                    size="1",
                ),
                spacing="2",
            ),
            rx.text(TodoState.active_count, " items left", size="2", color="gray"),
            rx.divider(),
            rx.foreach(TodoState.visible_todos, todo_item),
            spacing="4",
            width="100%",
            max_width="500px",
            margin="0 auto",
            padding_y="48px",
        ),
        size="2",
    )
```

### todo_app/todo_app.py

```python
import reflex as rx
from . import models  # noqa: F401 — ensures models are registered
from .pages import index  # noqa: F401 — ensures pages are compiled

app = rx.App(
    theme=rx.theme(accent_color="blue", radius="medium"),
)
```

---

## Example 2: Dashboard with Charts

A data dashboard with charts, tables, and loading states.

### dashboard/state.py

```python
import reflex as rx


class DashboardState(rx.State):
    loading: bool = False
    metrics: list[dict] = []
    chart_data: list[dict] = []
    date_range: str = "7d"

    async def load_data(self):
        self.loading = True
        yield  # Show loading state immediately

        # Simulate data fetch (replace with real data source)
        self.metrics = [
            {"label": "Total Users", "value": "12,340", "change": "+12%"},
            {"label": "Revenue", "value": "$45,678", "change": "+8%"},
            {"label": "Active Sessions", "value": "1,234", "change": "-3%"},
            {"label": "Conversion", "value": "3.2%", "change": "+0.5%"},
        ]
        self.chart_data = [
            {"date": "Mon", "users": 120, "revenue": 4500},
            {"date": "Tue", "users": 150, "revenue": 5200},
            {"date": "Wed", "users": 180, "revenue": 4800},
            {"date": "Thu", "users": 140, "revenue": 5600},
            {"date": "Fri", "users": 200, "revenue": 6100},
            {"date": "Sat", "users": 90, "revenue": 3200},
            {"date": "Sun", "users": 110, "revenue": 3800},
        ]
        self.loading = False

    def set_date_range(self, value: str):
        self.date_range = value
        return DashboardState.load_data
```

### dashboard/pages/index.py

```python
import reflex as rx
from ..state import DashboardState


def metric_card(metric: rx.Var) -> rx.Component:
    return rx.card(
        rx.vstack(
            rx.text(metric["label"], size="2", color="gray"),
            rx.heading(metric["value"], size="6"),
            rx.badge(
                metric["change"],
                color_scheme=rx.cond(
                    metric["change"].contains("+"), "green", "red"
                ),
            ),
            spacing="2",
        ),
    )


@rx.page(route="/", title="Dashboard", on_load=DashboardState.load_data)
def index() -> rx.Component:
    return rx.container(
        rx.vstack(
            # Header
            rx.hstack(
                rx.heading("Dashboard", size="7"),
                rx.spacer(),
                rx.select(
                    items=["7d", "30d", "90d"],
                    default_value="7d",
                    on_change=DashboardState.set_date_range,
                ),
                width="100%",
                align="center",
            ),
            # Metric cards
            rx.cond(
                DashboardState.loading,
                rx.hstack(
                    *[rx.skeleton(width="200px", height="100px") for _ in range(4)],
                    spacing="4",
                ),
                rx.grid(
                    rx.foreach(DashboardState.metrics, metric_card),
                    columns="4",
                    gap="4",
                    width="100%",
                ),
            ),
            # Chart
            rx.card(
                rx.vstack(
                    rx.heading("Weekly Trends", size="4"),
                    rx.recharts.line_chart(
                        rx.recharts.line(
                            data_key="users", stroke="#8884d8", name="Users"
                        ),
                        rx.recharts.line(
                            data_key="revenue", stroke="#82ca9d", name="Revenue"
                        ),
                        rx.recharts.x_axis(data_key="date"),
                        rx.recharts.y_axis(),
                        rx.recharts.tooltip(),
                        rx.recharts.legend(),
                        data=DashboardState.chart_data,
                        width="100%",
                        height=300,
                    ),
                    width="100%",
                ),
                width="100%",
            ),
            spacing="6",
            width="100%",
            padding_y="32px",
        ),
        size="4",
    )
```

---

## Example 3: Multi-Page App with Auth

A multi-page application with login, protected routes, and shared layout.

### app/state.py

```python
import reflex as rx


class AuthState(rx.State):
    _user_id: int = 0
    username: str = ""
    logged_in: bool = False

    def login(self, form_data: dict):
        # Replace with real auth logic
        if form_data.get("username") and form_data.get("password"):
            self._user_id = 1
            self.username = form_data["username"]
            self.logged_in = True
            return rx.redirect("/dashboard")
        return rx.toast("Invalid credentials", variant="error")

    def logout(self):
        self._user_id = 0
        self.username = ""
        self.logged_in = False
        return rx.redirect("/login")

    def require_auth(self):
        if not self.logged_in:
            return rx.redirect("/login")
```

### app/components/layout.py

```python
import reflex as rx
from ..state import AuthState


def navbar() -> rx.Component:
    return rx.hstack(
        rx.heading("MyApp", size="4"),
        rx.spacer(),
        rx.cond(
            AuthState.logged_in,
            rx.hstack(
                rx.link("Dashboard", href="/dashboard"),
                rx.link("Settings", href="/settings"),
                rx.text(AuthState.username, color="gray"),
                rx.button("Logout", variant="outline", on_click=AuthState.logout),
                spacing="4",
                align="center",
            ),
            rx.hstack(
                rx.link("Login", href="/login"),
                spacing="4",
            ),
        ),
        width="100%",
        padding="16px 24px",
        border_bottom="1px solid #eee",
    )


def layout(*children) -> rx.Component:
    return rx.box(
        navbar(),
        rx.container(
            *children,
            size="3",
            padding_y="32px",
        ),
        min_height="100vh",
    )
```

### app/pages/login.py

```python
import reflex as rx
from ..state import AuthState
from ..components.layout import layout


@rx.page(route="/login", title="Login")
def login() -> rx.Component:
    return layout(
        rx.center(
            rx.card(
                rx.vstack(
                    rx.heading("Sign In", size="5"),
                    rx.form(
                        rx.vstack(
                            rx.input(
                                name="username",
                                placeholder="Username",
                                required=True,
                            ),
                            rx.input(
                                name="password",
                                type="password",
                                placeholder="Password",
                                required=True,
                            ),
                            rx.button("Sign In", type="submit", width="100%"),
                            spacing="4",
                            width="100%",
                        ),
                        on_submit=AuthState.login,
                        width="100%",
                    ),
                    spacing="4",
                    width="100%",
                ),
                width="400px",
            ),
            min_height="60vh",
        ),
    )
```

### app/pages/dashboard.py

```python
import reflex as rx
from ..state import AuthState
from ..components.layout import layout


@rx.page(route="/dashboard", title="Dashboard", on_load=AuthState.require_auth)
def dashboard() -> rx.Component:
    return layout(
        rx.vstack(
            rx.heading("Dashboard", size="7"),
            rx.text(f"Welcome back, ", AuthState.username),
            rx.card(
                rx.text("Your dashboard content goes here."),
                width="100%",
            ),
            spacing="4",
        ),
    )
```

---

## Example 4: Chat Interface

A chat application pattern with streaming responses.

### chat/state.py

```python
import reflex as rx


class Message(rx.Base):
    role: str  # "user" or "assistant"
    content: str


class ChatState(rx.State):
    messages: list[Message] = []
    input_text: str = ""
    is_streaming: bool = False

    def set_input(self, value: str):
        self.input_text = value

    async def send_message(self):
        if not self.input_text.strip():
            return

        # Add user message
        user_msg = Message(role="user", content=self.input_text)
        self.messages.append(user_msg)
        prompt = self.input_text
        self.input_text = ""
        self.is_streaming = True
        yield

        # Simulate streaming assistant response
        # Replace with actual LLM API call
        assistant_msg = Message(role="assistant", content="")
        self.messages.append(assistant_msg)

        response_text = f"You said: {prompt}. This is a simulated response."
        for i, char in enumerate(response_text):
            self.messages[-1].content += char
            if i % 5 == 0:
                yield  # Stream partial updates

        self.is_streaming = False
```

### chat/pages/index.py

```python
import reflex as rx
from ..state import ChatState, Message


def message_bubble(msg: rx.Var[Message]) -> rx.Component:
    return rx.box(
        rx.text(msg.content),
        background_color=rx.cond(
            msg.role == "user", "blue.3", "gray.3"
        ),
        padding="12px 16px",
        border_radius="12px",
        max_width="70%",
        align_self=rx.cond(
            msg.role == "user", "flex-end", "flex-start"
        ),
    )


@rx.page(route="/", title="Chat")
def index() -> rx.Component:
    return rx.container(
        rx.vstack(
            rx.heading("Chat", size="5"),
            # Message list
            rx.vstack(
                rx.foreach(ChatState.messages, message_bubble),
                spacing="3",
                width="100%",
                min_height="400px",
                max_height="60vh",
                overflow_y="auto",
                padding="16px",
                border="1px solid #eee",
                border_radius="12px",
            ),
            # Input area
            rx.hstack(
                rx.input(
                    placeholder="Type a message...",
                    value=ChatState.input_text,
                    on_change=ChatState.set_input,
                    flex="1",
                    disabled=ChatState.is_streaming,
                ),
                rx.button(
                    rx.cond(ChatState.is_streaming, rx.spinner(size="1"), "Send"),
                    on_click=ChatState.send_message,
                    disabled=ChatState.is_streaming,
                ),
                width="100%",
            ),
            spacing="4",
            width="100%",
            max_width="700px",
            margin="0 auto",
            padding_y="32px",
        ),
        size="3",
    )
```

---

## Project Template Checklist

When starting a new Reflex project:

1. `reflex init` — scaffold project
2. Edit `rxconfig.py` — set app name, db_url, api_url
3. Create `models.py` — define database models
4. Create `state.py` — define shared state
5. Create `pages/` — one module per page with `@rx.page`
6. Create `components/` — reusable UI components
7. Import everything in `__init__.py` — pages, models, state
8. `reflex db init && reflex db makemigrations && reflex db migrate`
9. `reflex run` — start dev server and test
