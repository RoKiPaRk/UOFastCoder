# Build a CLIENTS CRUD App with UOFastCoder

A step-by-step walkthrough that takes you from a bare U2 `CLIENTS` file to a running Flask app with login, a landing page, and full create/read/update/delete.

---

## What you'll end up with

```
your-project/
├── app.py                          ← Flask app, login route, session guard
├── .env                            ← U2 connection credentials
├── models/
│   └── clients.py                  ← UopyModel (auto-generated)
├── services/
│   └── clients_service.py          ← list / get / create / update / delete
├── forms/
│   └── clients_form.py             ← UOFastForms FormModel
├── routes/
│   ├── auth.py                     ← login / logout Blueprint (hand-written)
│   └── clients.py                  ← CRUD Blueprint (auto-generated)
└── templates/
    ├── base.html                   ← shared layout with navbar (hand-written)
    ├── login.html                  ← login page (hand-written)
    ├── index.html                  ← landing page (hand-written)
    └── clients/
        ├── list.html               ← searchable table (auto-generated)
        └── edit.html               ← create / edit form (auto-generated)
```

---

## Prerequisites

- UOFastMCP server running (`uofast-mcp` on `localhost:8000`)
- UOFastCoder plugin installed
- Python packages installed:

```bash
pip install uofast-orm uofastforms flask python-dotenv
```

- `.env` file in your project root:

```
UNIDATA_HOST=localhost
UNIDATA_PORT=31438
UNIDATA_USERNAME=your_user
UNIDATA_PASSWORD=your_password
UNIDATA_ACCOUNT=your_account
UNIDATA_SERVICE=udcs
```

---

## Step 1 — Document the CLIENTS file

```
/uo-document CLIENTS
```

This reads the live DICT and writes `docs/u2-schema.md`. Every subsequent skill reads that file as memory so it always has accurate field names and attribute numbers.

Re-run any time the DICT changes.

---

## Step 2 — Explore the file (optional but recommended)

```
/uo-explore CLIENTS
```

Shows you the full field list, a sample record, and any detected foreign-key relationships before any code is generated. Good for confirming the DICT looks right.

---

## Step 3 — Generate the Python stack

```
/uo-python CLIENTS
```

Writes three files:

| File | What it contains |
|------|-----------------|
| `models/clients.py` | `ClientsModel` — field map, MV fields, write-safe field list |
| `services/clients_service.py` | `list_clients`, `get_client`, `create_client`, `update_client`, `delete_client` |
| `routes/clients.py` | Flask Blueprint with REST endpoints, session auth guard on every route |

**Note:** The generated routes use `session.get("authenticated")` as the auth check. You will wire this up in Step 6.

---

## Step 4 — Generate the UI

```
/uo-ui CLIENTS
```

Writes:

| File | What it contains |
|------|-----------------|
| `forms/clients_form.py` | `ClientsForm` — field types inferred from DICT (dates, emails, textareas, etc.) |
| `routes/clients.py` | Updated Blueprint with form-aware GET/POST edit handler |
| `templates/clients/list.html` | Bootstrap 5 table, search bar, Edit / Delete buttons |
| `templates/clients/edit.html` | Bootstrap 5 form with correct input type per field |

---

## Step 5 — Create the shared base template

Create `templates/base.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}My App{% endblock %}</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
  <nav class="navbar navbar-expand-lg navbar-dark bg-dark px-3">
    <a class="navbar-brand" href="{{ url_for('main.index') }}">My App</a>
    <div class="ms-auto">
      {% if session.get('authenticated') %}
        <a class="nav-link text-white d-inline" href="{{ url_for('auth.logout') }}">Logout</a>
      {% endif %}
    </div>
  </nav>
  <div class="container mt-4">
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% for category, message in messages %}
        <div class="alert alert-{{ category }}">{{ message }}</div>
      {% endfor %}
    {% endwith %}
    {% block content %}{% endblock %}
  </div>
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

---

## Step 6 — Create the login page

Create `templates/login.html`:

```html
{% extends "base.html" %}
{% block title %}Login{% endblock %}
{% block content %}
<div class="row justify-content-center">
  <div class="col-md-4">
    <h2 class="mb-4">Login</h2>
    <form method="post">
      <div class="mb-3">
        <label class="form-label">Username</label>
        <input type="text" name="username" class="form-control" required autofocus>
      </div>
      <div class="mb-3">
        <label class="form-label">Password</label>
        <input type="password" name="password" class="form-control" required>
      </div>
      <button type="submit" class="btn btn-primary w-100">Sign in</button>
    </form>
  </div>
</div>
{% endblock %}
```

Create `routes/auth.py`:

```python
from flask import Blueprint, render_template, request, redirect, url_for, session, flash
import os

auth_bp = Blueprint("auth", __name__)

# Replace with real credential validation or U2 session auth
VALID_USER = os.getenv("APP_USERNAME", "admin")
VALID_PASS = os.getenv("APP_PASSWORD", "changeme")


@auth_bp.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        if (request.form["username"] == VALID_USER
                and request.form["password"] == VALID_PASS):
            session["authenticated"] = True
            session["username"] = request.form["username"]
            return redirect(url_for("main.index"))
        flash("Invalid credentials", "danger")
    return render_template("login.html")


@auth_bp.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("auth.login"))
```

---

## Step 7 — Create the landing page

Create `templates/index.html`:

```html
{% extends "base.html" %}
{% block title %}Home{% endblock %}
{% block content %}
<div class="py-4">
  <h1>Welcome, {{ session.get('username', 'User') }}</h1>
  <p class="lead">Select a section to get started.</p>
  <div class="row mt-4">
    <div class="col-md-3">
      <div class="card text-center p-4">
        <h5>Clients</h5>
        <p class="text-muted">View and manage client records</p>
        <a href="{{ url_for('clients.list_clients') }}" class="btn btn-primary">Open</a>
      </div>
    </div>
  </div>
</div>
{% endblock %}
```

Create `routes/main.py`:

```python
from flask import Blueprint, render_template, redirect, url_for, session

main_bp = Blueprint("main", __name__)


@main_bp.route("/")
def index():
    if not session.get("authenticated"):
        return redirect(url_for("auth.login"))
    return render_template("index.html")
```

---

## Step 8 — Wire it all together in app.py

Create `app.py`:

```python
import os
from flask import Flask
from dotenv import load_dotenv
from uofast_orm import ConnectionPool

load_dotenv()

app = Flask(__name__)
app.secret_key = os.getenv("SECRET_KEY", "change-me-in-production")

# U2 connection pool — shared across all requests
pool = ConnectionPool(
    host=os.getenv("UNIDATA_HOST"),
    port=int(os.getenv("UNIDATA_PORT", "31438")),
    user=os.getenv("UNIDATA_USERNAME"),
    password=os.getenv("UNIDATA_PASSWORD"),
    account=os.getenv("UNIDATA_ACCOUNT"),
    service=os.getenv("UNIDATA_SERVICE", "udcs"),
)
app.config["UO_POOL"] = pool

# Register blueprints
from routes.auth import auth_bp
from routes.main import main_bp
from routes.clients import clients_bp  # generated by /uo-python + /uo-ui

app.register_blueprint(auth_bp)
app.register_blueprint(main_bp)
app.register_blueprint(clients_bp, url_prefix="/clients")

if __name__ == "__main__":
    app.run(debug=True)
```

---

## Step 9 — Run and test

```bash
python app.py
```

Visit `http://localhost:5000` — you'll be redirected to `/login`. After signing in you land on the home page. Click **Open** to reach the clients list.

| URL | What it does |
|-----|-------------|
| `/` | Landing page (requires login) |
| `/login` | Login form |
| `/logout` | Clears session, back to login |
| `/clients/` | List all CLIENTS records |
| `/clients/new` | Create a new client |
| `/clients/<id>` | Edit an existing client |
| `/clients/<id>/delete` | Delete a client |

---

## Updating after DICT changes

If fields are added or changed in the DICT, re-run the full chain:

```
/uo-document CLIENTS
/uo-python CLIENTS
/uo-ui CLIENTS
```

The skills check for existing files before overwriting — they update only what changed and leave hand-written code (like `routes/auth.py`) untouched.
