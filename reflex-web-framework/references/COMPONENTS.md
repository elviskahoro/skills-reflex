# Components Reference

Complete reference for Reflex's component library. Components are Python functions that compile to React components.

## Component Fundamentals

All components follow the same pattern: children as positional args, props as keyword args.

```python
rx.component_name(
    child1,            # Positional: child components or text
    child2,
    prop1="value",     # Keyword: component-specific props
    css_prop="value",  # Keyword: CSS styling props
    on_event=Handler,  # Keyword: event triggers
)
```

## Layout Components

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `rx.box` | Generic container (div) | All CSS props |
| `rx.flex` | Flexbox container | `direction`, `align`, `justify`, `gap`, `wrap` |
| `rx.hstack` | Horizontal stack | `spacing`, `align`, `justify` |
| `rx.vstack` | Vertical stack | `spacing`, `align`, `justify` |
| `rx.grid` | CSS Grid container | `columns`, `rows`, `gap`, `flow` |
| `rx.container` | Centered max-width container | `size` (1-4) |
| `rx.section` | Semantic section with padding | `size` (1-3) |
| `rx.center` | Centers children | All CSS props |
| `rx.spacer` | Flexible space (use in stacks) | — |
| `rx.divider` | Horizontal rule | `size`, `color` |
| `rx.stack` | Generic stack (h or v) | `direction`, `spacing` |

### Layout Examples

```python
# Responsive two-column layout
rx.grid(
    rx.box("Sidebar", width="250px"),
    rx.box("Main content"),
    columns="250px 1fr",
    gap="4",
)

# Centered card layout
rx.container(
    rx.vstack(
        rx.heading("Title"),
        rx.text("Content"),
        spacing="4",
        align="center",
    ),
    size="2",
)

# Navbar with spacer
rx.hstack(
    rx.heading("Logo"),
    rx.spacer(),
    rx.link("Home", href="/"),
    rx.link("About", href="/about"),
    width="100%",
    padding="16px",
)
```

## Typography

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `rx.heading` | Heading (h1-h6) | `size` (1-9), `as_` ("h1"-"h6") |
| `rx.text` | Paragraph text | `size` (1-9), `weight`, `color`, `as_` |
| `rx.code` | Inline code | `variant`, `color` |
| `rx.blockquote` | Block quote | — |
| `rx.link` | Anchor link | `href`, `is_external` |
| `rx.markdown` | Render Markdown string | `component_map` for custom rendering |

```python
rx.vstack(
    rx.heading("Page Title", size="7", as_="h1"),
    rx.text("Body text", size="3", color="gray"),
    rx.text(rx.text.strong("Bold"), " and ", rx.text.em("italic")),
    rx.code("inline_code()"),
    rx.link("Reflex Docs", href="https://reflex.dev", is_external=True),
)
```

## Form Components

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `rx.input` | Text input | `placeholder`, `default_value`, `on_change`, `on_blur`, `type` |
| `rx.text_area` | Multi-line input | `placeholder`, `default_value`, `on_change`, `on_blur` |
| `rx.button` | Button | `variant`, `color_scheme`, `size`, `on_click`, `loading` |
| `rx.checkbox` | Checkbox | `checked`, `on_change` |
| `rx.switch` | Toggle switch | `checked`, `on_change` |
| `rx.slider` | Range slider | `min`, `max`, `default_value`, `on_change` |
| `rx.select` | Dropdown select | `items`, `default_value`, `on_change` |
| `rx.radio_group` | Radio buttons | `items`, `default_value`, `on_change` |
| `rx.form` | HTML form | `on_submit` (receives dict of form data) |
| `rx.upload` | File upload zone | `on_upload`, `accept`, `max_files` |

### Form Examples

```python
# Controlled input with on_blur (recommended for text inputs)
rx.input(
    placeholder="Enter name",
    default_value=MyState.name,
    on_blur=MyState.set_name,
)

# Select dropdown
rx.select(
    items=["Option A", "Option B", "Option C"],
    default_value="Option A",
    on_change=MyState.set_option,
)

# Form submission
rx.form(
    rx.vstack(
        rx.input(name="email", placeholder="Email"),
        rx.input(name="password", type="password", placeholder="Password"),
        rx.button("Submit", type="submit"),
    ),
    on_submit=MyState.handle_submit,
)

# Event handler for form:
def handle_submit(self, form_data: dict):
    self.email = form_data.get("email", "")
    self.password = form_data.get("password", "")
```

### File Upload

```python
rx.upload(
    rx.vstack(
        rx.button("Select File"),
        rx.text("Drag and drop files here or click to select"),
    ),
    id="my_upload",
    accept={".pdf": "application/pdf", ".png": "image/png"},
    max_files=5,
)
rx.button("Upload", on_click=MyState.handle_upload(rx.upload_files(upload_id="my_upload")))

# Event handler:
async def handle_upload(self, files: list[rx.UploadFile]):
    for file in files:
        data = await file.read()
        outfile = rx.get_upload_dir() / file.filename
        outfile.write_bytes(data)
        self.uploaded_files.append(file.filename)
```

## Data Display

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `rx.badge` | Status badge | `variant`, `color_scheme` |
| `rx.avatar` | User avatar | `src`, `fallback`, `size` |
| `rx.callout` | Callout/alert box | `icon`, `variant`, `color` |
| `rx.card` | Card container | `size`, `variant` |
| `rx.tooltip` | Hover tooltip | `content` |
| `rx.progress` | Progress bar | `value` (0-100) |
| `rx.spinner` | Loading spinner | `size` |
| `rx.skeleton` | Loading placeholder | `loading` (bool) |
| `rx.image` | Image | `src`, `alt`, `width`, `height` |
| `rx.icon` | Lucide icon | `tag` (icon name), `size` |

```python
# Card with avatar
rx.card(
    rx.hstack(
        rx.avatar(src="/user.png", fallback="JD", size="3"),
        rx.vstack(
            rx.text("John Doe", weight="bold"),
            rx.text("john@example.com", size="2", color="gray"),
        ),
    ),
)

# Icon (uses Lucide icons)
rx.icon(tag="settings", size=24)
rx.icon(tag="chevron-right")
```

## Table

```python
rx.table.root(
    rx.table.header(
        rx.table.row(
            rx.table.column_header_cell("Name"),
            rx.table.column_header_cell("Email"),
            rx.table.column_header_cell("Role"),
        ),
    ),
    rx.table.body(
        rx.foreach(
            MyState.users,
            lambda user: rx.table.row(
                rx.table.cell(user.name),
                rx.table.cell(user.email),
                rx.table.cell(user.role),
            ),
        ),
    ),
)
```

## Overlay Components

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `rx.dialog` | Modal dialog | Use `.root`, `.trigger`, `.content`, `.title`, `.close` |
| `rx.alert_dialog` | Confirmation dialog | Same subcomponent pattern as dialog |
| `rx.popover` | Popover content | `.root`, `.trigger`, `.content` |
| `rx.drawer` | Slide-in drawer | `.root`, `.trigger`, `.content` |
| `rx.hover_card` | Card on hover | `.root`, `.trigger`, `.content` |
| `rx.toast` | Toast notification | `rx.toast("message")` from event handler |

### Dialog Example

```python
rx.dialog.root(
    rx.dialog.trigger(rx.button("Open Dialog")),
    rx.dialog.content(
        rx.dialog.title("Edit Profile"),
        rx.dialog.description("Make changes to your profile."),
        rx.input(placeholder="Name", on_blur=MyState.set_name),
        rx.flex(
            rx.dialog.close(rx.button("Cancel", variant="soft")),
            rx.dialog.close(rx.button("Save", on_click=MyState.save)),
            spacing="3",
            justify="end",
        ),
    ),
)
```

### Toast Notifications

```python
# In an event handler:
def save_data(self):
    # ... save logic ...
    return rx.toast("Saved successfully!")

# With options:
def save_data(self):
    return rx.toast("Saved!", duration=3000, position="top-right")
```

## Charts (Recharts)

Reflex wraps Recharts for data visualization:

```python
rx.recharts.bar_chart(
    rx.recharts.bar(data_key="value", fill="#8884d8"),
    rx.recharts.x_axis(data_key="name"),
    rx.recharts.y_axis(),
    rx.recharts.tooltip(),
    data=[
        {"name": "A", "value": 400},
        {"name": "B", "value": 300},
        {"name": "C", "value": 500},
    ],
    width=500,
    height=300,
)
```

Available chart types: `bar_chart`, `line_chart`, `area_chart`, `pie_chart`, `radar_chart`, `scatter_chart`, `composed_chart`, `funnel_chart`, `treemap`.

## Navigation

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `rx.link` | Anchor/router link | `href`, `is_external` |
| `rx.breadcrumb` | Breadcrumb nav | Use `.root`, `.item`, `.link` |
| `rx.tabs` | Tab navigation | `.root`, `.list`, `.trigger`, `.content` |

```python
# Internal navigation
rx.link("Go to About", href="/about")

# External link (opens new tab)
rx.link("Docs", href="https://reflex.dev", is_external=True)

# Tabs
rx.tabs.root(
    rx.tabs.list(
        rx.tabs.trigger("Tab 1", value="1"),
        rx.tabs.trigger("Tab 2", value="2"),
    ),
    rx.tabs.content(rx.text("Content 1"), value="1"),
    rx.tabs.content(rx.text("Content 2"), value="2"),
    default_value="1",
)
```

## Raw HTML Elements

Use `rx.el` namespace for any standard HTML element:

```python
rx.el.div(
    rx.el.h1("Title"),
    rx.el.p("Paragraph with ", rx.el.strong("bold"), " text"),
    rx.el.ul(
        rx.el.li("Item 1"),
        rx.el.li("Item 2"),
    ),
    rx.el.input(type="email", placeholder="Email"),
)
```

## Wrapping Custom React Components

```python
class ColorPicker(rx.Component):
    library = "react-colorful"
    tag = "HexColorPicker"

    color: rx.Var[str]
    on_change: rx.EventHandler[lambda color: [color]]

# Usage:
ColorPicker(color=MyState.color, on_change=MyState.set_color)
```
