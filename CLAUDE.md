# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

GestionStock is a Flask web app for small-business stock and sales management (in Spanish). It manages Productos, Clientes, Ventas (cash or "fiado"/credit), Pagos, and exports Informes (reports) to Excel/PDF. Server-rendered with Jinja2 + Bootstrap 5, no frontend build step or JS framework.

## Commands

Activate the venv first (Windows): `venv\Scripts\activate`

- Run the dev server: `python app.py` (runs on Flask's default port with `debug=True`)
- Install dependencies: `pip install -r requirements.txt`
- Create a migration after changing a model: `flask db migrate -m "description"`
- Apply migrations: `flask db upgrade`

There is no test suite, linter, or build step configured in this repo.

## Architecture

**Entry point** `app.py` creates the Flask app, configures the DB (`DATABASE_URL` env var if set, else `sqlite:///tienda.db`), initializes `db`/`Migrate`, and registers one Blueprint per module. It also owns the `/` dashboard route (totals for clientes, productos, ventas today, and revenue this month).

**Layering**: `routes/` (Blueprints, one per entity: `producto`, `cliente`, `venta`, `pago`, `informe`) → `models/` (SQLAlchemy models) → `templates/<entity>/` (Jinja2 views). Business logic that doesn't belong to a single route lives in `services/` (currently `deuda_service.calcular_deuda_cliente`, which computes a cliente's debt as sum of "fiado" ventas minus sum of pagos — this is the source of truth for debt and is reused across the dashboard, `clientesConDeudas` report, and its exports).

**Data model** (`models/`, all import the shared `db` instance from `models/db.py`):
- `Cliente` 1—N `Venta` and 1—N `Pago`
- `Venta` has `tipo_pago` ("efectivo" or "fiado") and 1—N `DetalleVenta`; `cliente_id` is nullable (walk-in sales have no cliente)
- `DetalleVenta` is the Venta↔Producto line-item join (cantidad, precio_unitario snapshot at sale time)
- `Producto` holds `stock`, decremented directly on `Venta` creation in `routes/ventas_routes.py`

**Venta creation flow** (`routes/ventas_routes.py: crear`) is the most involved route: it creates the `Venta` row first (`db.session.flush()` to get an id), then loops submitted producto/cantidad pairs, validates stock per item, decrements `Producto.stock`, creates `DetalleVenta` rows, and accumulates `total` before a single commit. "fiado" ventas require a `cliente_id`.

**Informes** (`routes/informes_routes.py`) follow a repeated pattern: an HTML view route (e.g. `/informes/ventasDelDia`) plus separate `/excel` and `/pdf` export routes that rebuild the same query and stream a file via `send_file`/`BytesIO` (openpyxl `Workbook` for Excel, reportlab `SimpleDocTemplate`/`Table` for PDF). When adding a new report, follow this three-route pattern (view + excel + pdf) rather than trying to unify them, to match the existing style.

**Migrations**: Flask-Migrate/Alembic in `migrations/`. Model changes need a corresponding migration generated via `flask db migrate`.
