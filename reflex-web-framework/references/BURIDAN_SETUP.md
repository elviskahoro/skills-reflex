# Buridan UI Setup & Installation

Buridan is a component library for Reflex built on shadcn/ui principles. Unlike traditional packages, Buridan components are added to your project via CLI, making them yours to own, customize, and extend.

## Why Buridan Over Raw Reflex?

- **Components You Own**: Code is added directly to your project, no black boxes
- **Fully Customizable**: Modify styles, logic, and behavior to match your needs
- **Pre-built UI System**: Components follow Reflex best practices and state management
- **Theming Support**: Built-in CSS variable system for consistent design
- **Chart Library**: 7 chart types with advanced customization
- **Wrapped React Components**: Easy access to popular React libraries

## Prerequisites

- Python 3.11+ (Buridan requires 3.11, Reflex supports 3.10+)
- Reflex project with `rxconfig.py`
- Node.js installed for dev server

## Installation Steps

### Step 1: Create or Navigate to Your Reflex Project

If starting fresh:

```bash
reflex init my_app
cd my_app
```

For existing projects, ensure `rxconfig.py` exists in the root.

### Step 2: Install the Buridan CLI

```bash
pip install buridan-ui
```

Verify installation:

```bash
buridan --version
```

### Step 3: Ensure Latest Reflex

```bash
pip install --upgrade reflex
```

## Using the Buridan CLI

All commands must run from the directory containing `rxconfig.py`.

### List Available Items

See all components, wrapped React components, and themes:

```bash
buridan list
```

Example output shows categories:
- Standard Components (button, card, input, etc.)
- Wrapped React Components (react_countup, simple_icon, etc.)
- Themes (blue, red, green, amber, purple)

### Add a Component

```bash
buridan add component button
buridan add component card
buridan add component input
```

This command:
- Downloads component files from Buridan repository
- Places files in `your_app_name/components/ui/`
- Adds utility dependencies (like `twmerge.py`)
- Updates your local component cache

### Add a Wrapped React Component

For React components wrapped for Reflex:

```bash
buridan add wrapped-react simple_icon
buridan add wrapped-react react_countup
```

### Add a Theme

```bash
buridan add theme blue
buridan add theme red
```

This command:
- Extracts CSS rules for light and dark variants
- Creates `assets/css/` directory if needed
- Saves as `blue.css`, `red.css`, etc.
- CSS is ready to import in `rx.App()`

## Project Structure After Adding Components

```
your_app/
├── your_app_name/
│   ├── components/
│   │   └── ui/                 # Buridan components live here
│   │       ├── button.py
│   │       ├── card.py
│   │       ├── input.py
│   │       ├── twmerge.py      # Utility for class merging
│   │       └── ...
│   ├── __init__.py
│   └── your_app.py
├── assets/
│   └── css/
│       ├── blue.css            # Theme CSS
│       └── theme.css           # Main theme file
├── rxconfig.py
└── requirements.txt
```

## Importing Buridan Components

After adding a component, import and use it:

```python
import reflex as rx
from your_app_name.components.ui.button import button

def index() -> rx.Component:
    return rx.vstack(
        button("Click me"),
        rx.spacer(),
    )
```

## Setting Up Themes

### Basic Theme Setup

1. Create or update `assets/theme.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Theme color variables */
:root {
    --chart-1: oklch(0.81 0.1 252);
    --chart-2: oklch(0.62 0.19 260);
    --chart-3: oklch(0.55 0.22 263);
    --chart-4: oklch(0.49 0.22 264);
    --chart-5: oklch(0.42 0.18 266);
}

/* Import Buridan theme */
@import "./blue.css";
```

2. Import in your app:

```python
app = rx.App(stylesheets=["/theme.css"])
```

### Switching Themes

Apply theme classes to parent components:

```python
def themed_container():
    return rx.box(
        rx.heading("Blue Theme"),
        class_name="theme-blue",  # Applies blue color palette
    )
```

Available themes: `theme-blue`, `theme-red`, `theme-green`, `theme-amber`, `theme-purple`

## Dependencies & Node Modules

Some Buridan components require JavaScript dependencies. The CLI handles this automatically by:

1. Adding dependencies to your project
2. Installing them when you run `reflex run`

Common dependencies:
- `clsx` - Class merging utility
- `recharts` - Charting library (for charts)
- Various UI component libraries (loaded per component)

## Troubleshooting

### "Command not found: buridan"

Solution: Ensure buridan-ui is installed in the correct Python environment:

```bash
pip install buridan-ui
which buridan
```

### Components not importing

Solution: Ensure the component was successfully added:

```bash
ls your_app_name/components/ui/
```

If missing, try again:

```bash
buridan add component button
```

### Theme CSS not loading

Solution: Verify the CSS file path and import:

1. Check `assets/theme.css` exists
2. Verify stylesheet import in app declaration
3. Restart dev server: `reflex run`

### Python version error

Buridan requires Python 3.11+. Check your version:

```bash
python --version
```

Update if needed before installing buridan-ui.

## Next Steps

- [Using Components](BURIDAN_COMPONENTS.md) - Component patterns and props
- [Building Charts](BURIDAN_CHARTS.md) - Data visualization with Recharts
- [Advanced Features](BURIDAN_ADVANCED.md) - ClientStateVar, theming, utilities
