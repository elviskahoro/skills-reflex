# Styling Reference

Complete reference for styling Reflex components: CSS props, theming, responsive design, and global styles.

## CSS Props (Inline Styling)

Pass CSS properties as snake_case keyword arguments to any component:

```python
rx.box(
    rx.text("Styled content"),
    background_color="#f0f0f0",
    border_radius="8px",
    padding="16px",
    margin="8px 0",
    box_shadow="0 2px 4px rgba(0,0,0,0.1)",
    width="100%",
    max_width="600px",
)
```

### Common CSS Props

| Prop | CSS Property | Example Values |
|------|-------------|----------------|
| `width` | width | `"100%"`, `"300px"`, `"auto"` |
| `height` | height | `"200px"`, `"100vh"`, `"auto"` |
| `min_width` / `max_width` | min/max-width | `"200px"`, `"1200px"` |
| `min_height` / `max_height` | min/max-height | `"50px"`, `"100vh"` |
| `padding` | padding | `"16px"`, `"8px 16px"` |
| `margin` | margin | `"0 auto"`, `"16px"` |
| `background_color` | background-color | `"blue"`, `"#ff0000"`, `"rgb(0,0,0)"` |
| `color` | color | `"white"`, `"gray.500"` |
| `font_size` | font-size | `"14px"`, `"1.2em"` |
| `font_weight` | font-weight | `"bold"`, `"600"` |
| `text_align` | text-align | `"center"`, `"left"` |
| `border` | border | `"1px solid gray"` |
| `border_radius` | border-radius | `"8px"`, `"50%"` |
| `box_shadow` | box-shadow | `"0 2px 4px rgba(0,0,0,0.1)"` |
| `display` | display | `"flex"`, `"grid"`, `"none"` |
| `position` | position | `"relative"`, `"absolute"`, `"fixed"` |
| `overflow` | overflow | `"hidden"`, `"auto"`, `"scroll"` |
| `cursor` | cursor | `"pointer"`, `"default"` |
| `opacity` | opacity | `"0.5"`, `"1"` |
| `z_index` | z-index | `"10"`, `"100"` |
| `transition` | transition | `"all 0.2s ease"` |

## Pseudo-Selectors

Use underscore-prefixed dict props for pseudo-selectors:

```python
rx.button(
    "Hover me",
    background_color="blue",
    color="white",
    _hover={
        "background_color": "darkblue",
        "cursor": "pointer",
        "transform": "scale(1.05)",
    },
    _active={
        "background_color": "navy",
        "transform": "scale(0.98)",
    },
    _focus={
        "outline": "2px solid blue",
        "outline_offset": "2px",
    },
    _disabled={
        "opacity": "0.5",
        "cursor": "not-allowed",
    },
)
```

Available pseudo-selectors: `_hover`, `_active`, `_focus`, `_disabled`, `_visited`, `_before`, `_after`, `_placeholder`, `_first_child`, `_last_child`.

## Responsive Design

Pass a list of values for responsive breakpoints (mobile-first):

```python
rx.box(
    rx.text("Responsive"),
    # [initial, sm (640px), md (768px), lg (1024px), xl (1280px)]
    width=["100%", "100%", "80%", "60%", "50%"],
    padding=["8px", "16px", "24px", "32px"],
    font_size=["14px", "14px", "16px", "18px"],
    display=["none", "block"],  # Hidden on mobile, visible on sm+
)

# Responsive stacking
rx.flex(
    sidebar(),
    main_content(),
    direction=["column", "column", "row"],  # Stack vertically on mobile
    gap="4",
)
```

## Theming

Configure app-wide theme via `rx.App()`:

```python
app = rx.App(
    theme=rx.theme(
        appearance="dark",        # "light" or "dark"
        accent_color="blue",      # Primary color
        gray_color="slate",       # Gray palette
        radius="medium",          # Border radius: "none", "small", "medium", "large", "full"
        scaling="100%",           # UI scaling
    ),
)
```

### Radix Color Tokens

Reflex uses Radix UI color scales. Reference colors by name + shade:

```python
rx.text("Colored", color="blue.9")       # Blue shade 9
rx.box(background_color="gray.3")         # Gray shade 3
rx.text("Accent", color="var(--accent-9)") # Theme accent color
```

Available color scales: `gray`, `blue`, `red`, `green`, `yellow`, `orange`, `purple`, `pink`, `cyan`, `teal`, and more. Shades range from 1 (lightest) to 12 (darkest).

### Color Mode

Access and toggle light/dark mode:

```python
rx.color_mode.button()  # Built-in toggle button

# Conditional styling based on color mode
rx.box(
    background_color=rx.color_mode_cond(
        light="white",
        dark="gray.900",
    ),
)
```

## Global Styles

Apply styles to all instances of a component type:

```python
style = {
    "body": {
        "font_family": "Inter, sans-serif",
        "background_color": "#fafafa",
    },
    rx.heading: {
        "font_weight": "700",
    },
    rx.button: {
        "border_radius": "8px",
        "cursor": "pointer",
    },
    "::selection": {
        "background_color": "blue.4",
    },
}

app = rx.App(style=style)
```

## Custom Stylesheets

Link external CSS files:

```python
app = rx.App(
    stylesheets=[
        "https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap",
        "/custom.css",  # File in assets/ directory
    ],
)
```

## Style Composition Patterns

### Reusable Style Dicts

```python
# Define reusable styles
card_style = {
    "background_color": "white",
    "border_radius": "12px",
    "padding": "24px",
    "box_shadow": "0 2px 8px rgba(0,0,0,0.08)",
    "border": "1px solid #eee",
}

button_primary = {
    "background_color": "blue.9",
    "color": "white",
    "border_radius": "8px",
    "padding": "8px 16px",
    "_hover": {"background_color": "blue.10"},
}

# Apply via unpacking
rx.box(rx.text("Card content"), **card_style)
rx.button("Primary", **button_primary)
```

### Component Wrapper Pattern

```python
def card(*children, **props) -> rx.Component:
    """Reusable card component with default styling."""
    defaults = {
        "background_color": "white",
        "border_radius": "12px",
        "padding": "24px",
        "box_shadow": "0 2px 8px rgba(0,0,0,0.08)",
    }
    defaults.update(props)
    return rx.box(*children, **defaults)

# Usage:
card(
    rx.heading("Title"),
    rx.text("Content"),
    max_width="400px",  # Override or extend defaults
)
```

## Animations

Use CSS transitions and keyframes:

```python
# Transition on hover
rx.box(
    "Animate me",
    transition="all 0.3s ease",
    _hover={"transform": "translateY(-4px)", "box_shadow": "0 4px 12px rgba(0,0,0,0.15)"},
)
```

For keyframe animations, define in a CSS file in `assets/` and reference the class:

```python
# assets/animations.css:
# @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
# .fade-in { animation: fadeIn 0.5s ease-in; }

rx.box("Fading in", class_name="fade-in")
```

## Common Styling Patterns

### Full-Page Layout

```python
def layout(*children) -> rx.Component:
    return rx.box(
        navbar(),
        rx.container(
            *children,
            size="3",
            padding_y="48px",
        ),
        min_height="100vh",
        display="flex",
        flex_direction="column",
    )
```

### Centered Content

```python
rx.center(
    rx.vstack(
        rx.heading("Centered"),
        rx.text("This is vertically and horizontally centered"),
        align="center",
    ),
    min_height="100vh",
)
```

### Sidebar Layout

```python
rx.flex(
    rx.box(
        sidebar_content(),
        width="250px",
        min_height="100vh",
        border_right="1px solid #eee",
        padding="16px",
        display=["none", "none", "block"],  # Hidden on mobile
    ),
    rx.box(
        main_content(),
        flex="1",
        padding="24px",
    ),
    direction="row",
)
```
