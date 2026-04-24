# Database and API Reference

Complete reference for Reflex database integration (SQLModel/SQLAlchemy), API routes, and authentication patterns.

## Database Setup

### Configuration

Database URL is set in `rxconfig.py`:

```python
import reflex as rx

config = rx.Config(
    app_name="my_app",
    db_url="sqlite:///reflex.db",  # Default: SQLite
    # db_url="postgresql://user:pass@localhost:5432/mydb",  # PostgreSQL
)
```

### Defining Models

Models use SQLModel (SQLAlchemy wrapper). Subclass `rx.Model` with `table=True`:

```python
import reflex as rx
from sqlmodel import Field
from datetime import datetime

class User(rx.Model, table=True):
    username: str = Field(unique=True, index=True)
    email: str = Field(unique=True)
    hashed_password: str
    is_active: bool = Field(default=True)
    created_at: datetime = Field(default_factory=datetime.utcnow)

class Post(rx.Model, table=True):
    title: str
    content: str
    author_id: int = Field(foreign_key="user.id")
    published: bool = Field(default=False)
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

`rx.Model` automatically adds an `id` primary key field (auto-incrementing integer).

**Critical:** All model classes must be imported somewhere in your app's module tree. Unimported models won't be detected by the migration system.

### Migrations

```bash
# Initialize Alembic (run once)
reflex db init

# Generate migration after model changes
reflex db makemigrations --message "add user and post tables"

# Apply pending migrations
reflex db migrate
```

## CRUD Operations

All database operations use `rx.session()` context manager inside event handlers:

### Create

```python
class UserState(rx.State):
    def create_user(self, form_data: dict):
        with rx.session() as session:
            user = User(
                username=form_data["username"],
                email=form_data["email"],
                hashed_password=hash_password(form_data["password"]),
            )
            session.add(user)
            session.commit()
```

### Read

```python
class UserState(rx.State):
    users: list[User] = []
    current_user: User | None = None

    def load_users(self):
        with rx.session() as session:
            self.users = session.exec(User.select()).all()

    def load_user_by_id(self, user_id: int):
        with rx.session() as session:
            self.current_user = session.get(User, user_id)

    def search_users(self):
        with rx.session() as session:
            self.users = session.exec(
                User.select().where(User.is_active == True)
            ).all()
```

### Update

```python
def update_user(self, user_id: int, new_email: str):
    with rx.session() as session:
        user = session.get(User, user_id)
        if user:
            user.email = new_email
            session.add(user)
            session.commit()
```

### Delete

```python
def delete_user(self, user_id: int):
    with rx.session() as session:
        user = session.get(User, user_id)
        if user:
            session.delete(user)
            session.commit()
```

## Query Patterns

### Filtering

```python
from sqlmodel import select

# Single condition
users = session.exec(select(User).where(User.is_active == True)).all()

# Multiple conditions
users = session.exec(
    select(User).where(
        User.is_active == True,
        User.created_at > cutoff_date,
    )
).all()

# OR conditions
from sqlalchemy import or_
users = session.exec(
    select(User).where(or_(User.role == "admin", User.role == "moderator"))
).all()
```

### Ordering and Pagination

```python
# Order by
users = session.exec(
    select(User).order_by(User.created_at.desc())
).all()

# Pagination
page_size = 20
page = 1
users = session.exec(
    select(User)
    .order_by(User.id)
    .offset((page - 1) * page_size)
    .limit(page_size)
).all()
```

### Joins

```python
# Join posts with users
results = session.exec(
    select(Post, User)
    .join(User, Post.author_id == User.id)
    .where(Post.published == True)
).all()
```

### Aggregations

```python
from sqlalchemy import func

count = session.exec(select(func.count(User.id))).one()
```

## API Routes

Reflex runs on FastAPI internally. Add custom REST endpoints via the `api_transformer` parameter on `rx.App`. Use these for webhooks, third-party integrations, or REST APIs alongside the WebSocket-based state system.

### Defining API Routes

```python
from fastapi import FastAPI, HTTPException, Request

# Create a FastAPI instance for custom routes
api = FastAPI()

@api.get("/api/health")
async def health_check():
    return {"status": "ok"}

@api.get("/api/users")
async def list_users():
    with rx.session() as session:
        users = session.exec(User.select()).all()
        return [{"id": u.id, "username": u.username} for u in users]

@api.post("/api/users")
async def create_user(data: dict):
    with rx.session() as session:
        user = User(**data)
        session.add(user)
        session.commit()
        return {"id": user.id, "username": user.username}

@api.get("/api/users/{user_id}")
async def get_user(user_id: int):
    with rx.session() as session:
        user = session.get(User, user_id)
        if not user:
            raise HTTPException(status_code=404, detail="User not found")
        return {"id": user.id, "username": user.username}

# Pass to rx.App via api_transformer
app = rx.App(api_transformer=api)
```

### Webhook Endpoints

```python
@api.post("/api/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()
    sig = request.headers.get("stripe-signature")
    # Verify and process webhook
    event = verify_stripe_signature(payload, sig)
    return {"received": True}
```

### Reserved Routes

Do not override these Reflex-internal routes:
- `/ping/` — health check
- `/_event` — WebSocket event communication
- `/_upload` — file upload handling

## Authentication Patterns

Reflex does not include built-in authentication. Use one of these approaches:

### Option 1: reflex-local-auth (Recommended for Simple Auth)

```bash
pip install reflex-local-auth
```

```python
import reflex as rx
import reflex_local_auth

# Add auth pages
app = rx.App()
app.add_page(reflex_local_auth.pages.login_page, route="/login")
app.add_page(reflex_local_auth.pages.register_page, route="/register")
```

### Option 2: Custom Session-Based Auth

```python
class AuthState(rx.State):
    _user_id: int = 0  # Backend-only (underscore prefix)
    username: str = ""
    logged_in: bool = False

    def login(self, form_data: dict):
        with rx.session() as session:
            user = session.exec(
                select(User).where(User.username == form_data["username"])
            ).first()
            if user and verify_password(form_data["password"], user.hashed_password):
                self._user_id = user.id
                self.username = user.username
                self.logged_in = True
                return rx.redirect("/dashboard")
            return rx.toast("Invalid credentials", variant="error")

    def logout(self):
        self._user_id = 0
        self.username = ""
        self.logged_in = False
        return rx.redirect("/login")

    def require_login(self):
        """Call from on_load to protect pages."""
        if not self.logged_in:
            return rx.redirect("/login")
```

### Protecting Pages

```python
@rx.page(route="/dashboard", on_load=AuthState.require_login)
def dashboard() -> rx.Component:
    return rx.cond(
        AuthState.logged_in,
        rx.box(
            rx.heading(f"Welcome, {AuthState.username}"),
            rx.button("Logout", on_click=AuthState.logout),
        ),
        rx.spinner(),  # Brief flash while redirect happens
    )
```

### Option 3: OAuth (Google Sign-In Example)

```python
class GoogleAuthState(rx.State):
    _token: str = ""  # Backend-only
    user_email: str = ""
    logged_in: bool = False

    async def handle_google_callback(self, token_data: dict):
        """Process OAuth callback."""
        # Verify token with Google
        user_info = await verify_google_token(token_data["credential"])
        self._token = token_data["credential"]
        self.user_email = user_info["email"]
        self.logged_in = True
        return rx.redirect("/dashboard")
```

## Security Best Practices

| Practice | Implementation |
|----------|---------------|
| Backend-only secrets | Prefix sensitive vars with `_` (e.g., `_api_key: str`) |
| Validate in event handlers | Check permissions in every event handler, not just page on_load |
| Hash passwords | Use `bcrypt` or `passlib` — never store plaintext |
| Parameterized queries | SQLModel/SQLAlchemy handle this automatically |
| Rate limiting | Add via FastAPI middleware on API routes |
| CSRF protection | Reflex WebSocket events are inherently CSRF-resistant |
| Session isolation | Each browser tab gets its own state — verify user identity per action |

## Database Provider Notes

### SQLite (Development)

Default. No setup needed. Single-file database at `reflex.db`.

```python
config = rx.Config(app_name="my_app", db_url="sqlite:///reflex.db")
```

### PostgreSQL (Production)

```python
config = rx.Config(
    app_name="my_app",
    db_url="postgresql://user:password@host:5432/dbname",
)
```

Required package: `pip install psycopg2-binary` (or `asyncpg` for async).

### Connection Pooling

For production PostgreSQL, configure pool settings via SQLAlchemy engine args in `rxconfig.py` or via environment variables. Reflex manages the engine internally — set `db_url` and it handles connection pooling.
