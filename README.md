# Autos Colombia — Iteración 1: entrada y salida de vehículos

Sistema de información del parqueadero por mensualidad **Autos Colombia**. Esta iteración cubre la
gestión de la entrada y salida de vehículos (RF-01 a RF-10 del documento de diseño), más lo mínimo
de usuarios, vehículos y celdas para poder operar.

- **Backend:** Python 3.11+ · FastAPI · SQLAlchemy 2 · JWT · SQLite (PostgreSQL opcional)
- **Frontend:** Angular 22 (componentes standalone, signals) · TypeScript
- **Diseño:** mockups en Figma (v2 accesible, contraste WCAG AA)

## Arquitectura

```
Angular (navegador) ──/api──▶ FastAPI
                               ├─ routers/        capa de aplicación (HTTP, permisos)
                               ├─ services/       reglas del negocio (entrada, salida, corrección)
                               ├─ models.py       dominio y persistencia (SQLAlchemy)
                               └─ base de datos   SQLite / PostgreSQL
```

Las reglas viven en `backend/app/services/movimientos.py` y las hacen cumplir además índices únicos
parciales en la base (una sola entrada abierta por placa y por celda, RNF-05).

## Cómo ejecutarlo

### 1. Backend

```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate      Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

La primera vez crea las tablas y carga datos de ejemplo. Documentación interactiva de la API en
<http://localhost:8000/docs>.

| Usuario | Rol | Contraseña (solo desarrollo) |
| --- | --- | --- |
| `ana` | Administrador | `demo1234` |
| `carlos`, `luis` | Operario | `demo1234` |

Placas de ejemplo: `ABC123`, `XYZ987`, `MNP32A`, `KLM456` (mensualidad vigente) y `QRS789`
(vencida, sirve para ver el error).

### 2. Frontend

```bash
cd frontend
npm install
npm start
```

Abre <http://localhost:4200>. El servidor de desarrollo reenvía `/api` a `http://localhost:8000`
(`proxy.conf.json`).

### 3. Pruebas

```bash
cd backend && python -m pytest        # 23 pruebas de la API y las reglas del negocio
cd frontend && npx ng test --watch=false
```

## Qué hace cada pantalla

| Pantalla | Quién | Historia |
| --- | --- | --- |
| Inicio de sesión | todos | HU-08 |
| Panel principal: celdas libres/ocupadas y vehículos dentro | todos | HU-06 |
| Registrar entrada: valida mensualidad y sugiere celda | todos | HU-01, HU-02, HU-03 |
| Registrar salida: libera la celda y muestra la permanencia | todos | HU-04 |
| Registrar novedad sobre un vehículo que está dentro | todos | HU-05 |
| Historial por placa y fechas | administrador | HU-07 |
| Corregir horas de un movimiento (queda quién y cuándo) | administrador | HU-08 |

## API principal

| Método y ruta | Descripción |
| --- | --- |
| `POST /api/auth/login` | Devuelve el token JWT y el empleado |
| `GET /api/movimientos/validar-entrada?placa=` | Indica si la entrada es posible y qué celda se asignaría, sin registrar |
| `POST /api/movimientos/entrada` · `POST /api/movimientos/salida` | Registran la entrada y la salida |
| `GET /api/movimientos/abiertos` | Vehículos dentro |
| `GET /api/movimientos` · `PATCH /api/movimientos/{id}` | Historial y corrección (solo administrador) |
| `POST /api/novedades` · `GET /api/novedades` | Novedades asociadas a la entrada vigente |
| `GET /api/celdas/resumen` | Totales de celdas y siguientes libres |
| `GET/POST /api/usuarios`, `PATCH /api/usuarios/{id}/mensualidad`, `GET/POST /api/vehiculos` | Clientes y vehículos (alta solo administrador) |

Los errores de negocio responden `{"detail": {"codigo": "...", "mensaje": "..."}}` con el código HTTP
que corresponde (404 placa no registrada, 409 ya está dentro o sin celdas, 422 mensualidad vencida).

## Decisiones a tener en cuenta

- **Horas:** se guardan en UTC y el frontend las muestra en hora de Colombia (UTC-5, sin horario de verano).
- **Tipo de vehículo:** sale del vehículo registrado, por eso en la pantalla de entrada es de solo lectura.
- **Sesión:** el token se guarda en `sessionStorage` (se cierra con la pestaña). Dura 8 horas.
- **Sin migraciones aún:** las tablas se crean al arrancar (`create_all`). Antes de llevarlo a
  producción conviene añadir Alembic.
- **Pendiente para producción:** definir `AUTOS_SECRET_KEY`, desactivar `AUTOS_SEED_DEMO`, servir
  Angular y la API detrás del mismo dominio con HTTPS y limitar los intentos de inicio de sesión.
- **Iteraciones siguientes:** gestión completa de usuarios y celdas (hoy solo alta básica) y pagos.
