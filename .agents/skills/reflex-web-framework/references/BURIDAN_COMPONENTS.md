# Buridan Components Guide

Buridan includes a comprehensive library of pre-built UI components. Since components are added to your project via CLI, you own and control all the code.

## Component Categories

| Category | Components | Use Case |
|----------|-----------|----------|
| **Inputs** | input, textarea, checkbox, radio, select, slider | Form data collection |
| **Layout** | card, container, box, vstack, hstack | Page structure |
| **Navigation** | breadcrumb, menu, tabs, sidebar | User navigation |
| **Feedback** | badge, alert, toast, tooltip | User feedback |
| **Display** | avatar, image, table | Content presentation |
| **Interactive** | accordion, collapsible, dialog, dropdown | User interactions |
| **Actions** | button | User actions |

## Adding Components

```bash
buridan add component button
buridan add component card
buridan add component input
buridan add component badge
buridan add component dialog
buridan add component avatar
```

## Common Components with Examples

### Button

```python
import reflex as rx
from your_app_name.components.ui.button import button

class State(rx.State):
    click_count: int = 0

    def increment(self):
        self.click_count += 1

    def reset(self):
        self.click_count = 0

def button_example() -> rx.Component:
    return rx.vstack(
        # Basic button
        button("Click me", on_click=State.increment),

        # Button with state display
        rx.text(f"Clicks: {State.click_count}"),

        # Multiple buttons
        rx.hstack(
            button("Increment", on_click=State.increment),
            button("Reset", on_click=State.reset),
            spacing="4",
        ),
        spacing="4",
    )
```

### Card

```python
import reflex as rx
from your_app_name.components.ui.card import card

def card_example() -> rx.Component:
    return rx.vstack(
        card(
            rx.heading("Welcome", size="3"),
            rx.text("This is a card component with content inside."),
            rx.text("Cards are great for organizing content."),
        ),

        # Multiple cards in a grid
        rx.grid(
            card(
                rx.heading("Card 1", size="2"),
                rx.text("First card"),
            ),
            card(
                rx.heading("Card 2", size="2"),
                rx.text("Second card"),
            ),
            card(
                rx.heading("Card 3", size="2"),
                rx.text("Third card"),
            ),
            columns="3",
            spacing="4",
        ),
        spacing="4",
    )
```

### Input with State

```python
import reflex as rx
from your_app_name.components.ui.input import input as text_input

class FormState(rx.State):
    username: str = ""
    email: str = ""

    def set_username(self, value: str):
        self.username = value

    def set_email(self, value: str):
        self.email = value

def input_example() -> rx.Component:
    return rx.vstack(
        rx.heading("User Form"),

        rx.vstack(
            rx.label("Username"),
            text_input(
                value=FormState.username,
                placeholder="Enter username",
                on_change=FormState.set_username,
            ),
            spacing="2",
        ),

        rx.vstack(
            rx.label("Email"),
            text_input(
                value=FormState.email,
                placeholder="Enter email",
                on_change=FormState.set_email,
                type_="email",
            ),
            spacing="2",
        ),

        rx.divider(),

        rx.vstack(
            rx.text(f"Username: {FormState.username}"),
            rx.text(f"Email: {FormState.email}"),
            spacing="2",
        ),

        spacing="4",
    )
```

### Badge

```python
import reflex as rx
from your_app_name.components.ui.badge import badge

def badge_example() -> rx.Component:
    return rx.vstack(
        rx.heading("Badge Variants"),

        rx.hstack(
            badge("Default"),
            badge("Success", class_name="bg-green-500"),
            badge("Warning", class_name="bg-yellow-500"),
            badge("Error", class_name="bg-red-500"),
            spacing="4",
        ),

        rx.heading("With Icons", size="3"),

        rx.hstack(
            badge("✓ Complete"),
            badge("⚠ Pending"),
            badge("✕ Failed"),
            spacing="4",
        ),

        spacing="4",
    )
```

### Accordion

```python
import reflex as rx
from your_app_name.components.ui.accordion import accordion, accordion_item

def accordion_example() -> rx.Component:
    return rx.vstack(
        rx.heading("FAQ Accordion"),

        accordion(
            accordion_item(
                header="What is Buridan?",
                content="Buridan is a UI component library for Reflex. You own the code.",
            ),
            accordion_item(
                header="How do I add components?",
                content="Use 'buridan add component <name>' from your project root.",
            ),
            accordion_item(
                header="Can I customize components?",
                content="Yes! All component code is in your project, fully customizable.",
            ),
            collapsible=True,
        ),

        spacing="4",
    )
```

### Dialog (Modal)

```python
import reflex as rx
from your_app_name.components.ui.dialog import dialog, dialog_content, dialog_header, dialog_body
from reflex.experimental import ClientStateVar

# Create client-side state for dialog visibility
dialog_open = ClientStateVar.create("dialog_open", False)

def dialog_example() -> rx.Component:
    return rx.vstack(
        rx.button(
            "Open Dialog",
            on_click=dialog_open.set_value(True),
        ),

        dialog(
            dialog_content(
                dialog_header("Confirm Action"),
                dialog_body(
                    rx.text("Are you sure you want to continue?"),
                    rx.hstack(
                        rx.button(
                            "Cancel",
                            on_click=dialog_open.set_value(False),
                        ),
                        rx.button(
                            "Confirm",
                            on_click=dialog_open.set_value(False),
                        ),
                        spacing="4",
                    ),
                ),
            ),
            is_open=dialog_open.value,
        ),

        spacing="4",
    )
```

### Avatar

```python
import reflex as rx
from your_app_name.components.ui.avatar import avatar

def avatar_example() -> rx.Component:
    return rx.vstack(
        rx.heading("User Avatars"),

        # With image
        rx.hstack(
            avatar(
                src="https://api.dicebear.com/7.x/avataaars/svg?seed=John",
                name="John Doe",
            ),
            rx.vstack(
                rx.text("John Doe", weight="bold"),
                rx.text("john@example.com", size="sm"),
                spacing="0",
            ),
            spacing="4",
        ),

        # Multiple avatars in a row
        rx.hstack(
            avatar(src="https://api.dicebear.com/7.x/avataaars/svg?seed=1"),
            avatar(src="https://api.dicebear.com/7.x/avataaars/svg?seed=2"),
            avatar(src="https://api.dicebear.com/7.x/avataaars/svg?seed=3"),
            spacing="2",
        ),

        spacing="4",
    )
```

### Checkbox with State

```python
import reflex as rx
from your_app_name.components.ui.checkbox import checkbox

class TodoState(rx.State):
    todos: list[dict] = [
        {"id": 1, "name": "Learn Reflex", "done": False},
        {"id": 2, "name": "Build app", "done": False},
        {"id": 3, "name": "Deploy", "done": False},
    ]

    def toggle_todo(self, todo_id: int):
        self.todos = [
            todo.copy(update={"done": not todo["done"]}) if todo["id"] == todo_id else todo
            for todo in self.todos
        ]

def checkbox_example() -> rx.Component:
    return rx.vstack(
        rx.heading("Todo List"),

        rx.foreach(
            TodoState.todos,
            lambda todo: rx.hstack(
                checkbox(
                    is_checked=todo["done"],
                    on_change=lambda: TodoState.toggle_todo(todo["id"]),
                ),
                rx.text(
                    todo["name"],
                    text_decoration=rx.cond(todo["done"], "line-through", "none"),
                    opacity=rx.cond(todo["done"], "0.6", "1"),
                ),
                spacing="4",
                width="100%",
            ),
        ),

        spacing="4",
        width="100%",
    )
```

## Component Props Cheat Sheet

### Common Props (Available on Most Components)

| Prop | Type | Purpose |
|------|------|---------|
| `class_name` | str | Tailwind CSS classes |
| `id` | str | HTML ID attribute |
| `style` | dict | Inline CSS styles |
| `on_click` | EventHandler | Click event |
| `on_change` | EventHandler | Change event |
| `on_blur` | EventHandler | Blur event |
| `on_focus` | EventHandler | Focus event |
| `width` | str | Component width |
| `height` | str | Component height |
| `disabled` | bool | Disable interaction |
| `visibility` | str | CSS visibility |

### Tailwind Utilities in Buridan

Most Buridan components support full Tailwind CSS via `class_name`:

```python
button(
    "Styled Button",
    class_name="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded-lg shadow-lg",
)
```

### Using the `cn()` Utility for Class Merging

When combining conditional classes, use the `cn()` utility:

```python
from your_app_name.components.ui.twmerge import cn

def conditional_button(is_active: rx.Var[bool]) -> rx.Component:
    return button(
        "Toggle",
        class_name=cn(
            "py-2 px-4",  # Base classes
            rx.cond(is_active, "bg-blue-500 text-white", "bg-gray-200 text-gray-800"),
        ),
    )
```

## Best Practices

### 1. Import Components at Module Level

```python
# ✅ Good
from your_app_name.components.ui.button import button
from your_app_name.components.ui.card import card
from your_app_name.components.ui.input import input as text_input

def my_component():
    return card(button("Click"))
```

### 2. Use State for Interactivity

```python
# ✅ Good - uses state for interaction
class MyState(rx.State):
    is_open: bool = False

    def toggle(self):
        self.is_open = not self.is_open

def my_component():
    return dialog(
        is_open=MyState.is_open,
        on_close=MyState.toggle,
    )
```

### 3. Combine Components for Complex UI

```python
# ✅ Good - builds reusable component from smaller pieces
def user_card(user_name: str, user_email: str):
    return card(
        rx.hstack(
            avatar(name=user_name),
            rx.vstack(
                rx.text(user_name, weight="bold"),
                rx.text(user_email, size="sm"),
                spacing="0",
            ),
            spacing="4",
        ),
    )
```

### 4. Use ClientStateVar for Ephemeral UI State

```python
# ✅ Good - dropdown toggle doesn't need server state
from reflex.experimental import ClientStateVar

menu_open = ClientStateVar.create("menu_open", False)

def menu():
    return rx.vstack(
        button("Menu", on_click=menu_open.set_value(True)),
        rx.cond(
            menu_open.value,
            rx.vstack(
                rx.text("Option 1"),
                rx.text("Option 2"),
            ),
        ),
    )
```

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using Python `if` on state vars in component | Doesn't work at compile time | Use `rx.cond(condition, true_comp, false_comp)` |
| Using `for` loop over state var list | List not iterable at compile time | Use `rx.foreach(list, render_fn)` |
| Forgetting to import component | Component not found error | Always import at module top: `from app.components.ui.button import button` |
| Mutating state vars directly | Changes don't propagate | Create event handler method and call it |
| Over-nesting components | Hard to read and maintain | Extract into separate functions |

## Next Steps

- [Using Charts](BURIDAN_CHARTS.md) - Data visualization
- [Advanced Features](BURIDAN_ADVANCED.md) - ClientStateVar, theming, utilities
- [Setup & Installation](BURIDAN_SETUP.md) - Getting started
