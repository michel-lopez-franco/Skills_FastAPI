# Retos: Middlewares en FastAPI

---

## Reto 1 — Básico: Middleware de logging

### ¿Qué debes lograr?
Crea un middleware que imprima en la consola una línea por cada request con el siguiente formato:

```
[GET] /usuarios/5 → 200
[POST] /items → 201
```

Es decir: el método HTTP, la ruta solicitada, y el código de status de la respuesta.

### Condiciones
- Usa `@app.middleware("http")`
- La impresión debe ocurrir **después** de que la ruta responde (para tener el status code)
- Agrega al menos 2 rutas de prueba para verificar que el log aparece en ambas

### Pista
El status code está en `response.status_code` y la URL en `request.url.path`.

---

## Reto 2 — Intermedio: Middleware de autenticación simple por header

### ¿Qué debes lograr?
Crea un middleware que proteja **todas** las rutas de tu app. El cliente debe enviar el header:

```
X-Token: mi-token-secreto
```

Si el header no está presente o el valor es incorrecto, el middleware debe devolver una respuesta `403 Forbidden` **sin llegar a ejecutar la ruta**.

Si el token es correcto, el request continúa normalmente.

### Condiciones
- El token válido puede estar hardcodeado como constante: `TOKEN_VALIDO = "mi-token-secreto"`
- Devuelve un JSON `{"error": "Token inválido o ausente"}` con status 403
- Agrega una ruta `/publico` que **no** requiera token (pista: revisa `request.url.path` para excluirla)
- Pruébalo con curl enviando y sin enviar el header

### Pista
Puedes devolver una respuesta temprana con `from fastapi.responses import JSONResponse` y retornarla directamente desde el middleware sin llamar `call_next`.

---

## Reto 3 — Avanzado: Middleware con BaseHTTPMiddleware y límite de tamaño

### ¿Qué debes lograr?
Crea un middleware usando la clase `BaseHTTPMiddleware` de Starlette (en lugar del decorador) que rechace requests cuyo body sea mayor a **1 KB (1024 bytes)**.

Si el body es muy grande, devuelve `413 Request Entity Too Large`.

### Condiciones
- Investiga cómo usar `BaseHTTPMiddleware` (no es el decorador `@app.middleware`)
- Lee el body del request con `await request.body()`
- Si supera 1024 bytes → responde 413 con JSON `{"error": "Body demasiado grande"}`
- Agrega una ruta POST `/datos` que acepte un body y devuelva su tamaño en bytes
- Prueba enviando un body pequeño y uno grande

### Pista
`BaseHTTPMiddleware` se importa desde `starlette.middleware.base` y se agrega con `app.add_middleware(MiMiddleware)`.

---

---

# SOLUCIONES — No ver antes de intentarlo

.

.

.

.

.

.

.

.

.

.

---

## Solución Reto 1 — Middleware de logging

```python
from fastapi import FastAPI, Request

app = FastAPI()


@app.middleware("http")
async def log_requests(request: Request, call_next):
    response = await call_next(request)
    print(f"[{request.method}] {request.url.path} → {response.status_code}")
    return response


@app.get("/")
def inicio():
    return {"mensaje": "inicio"}


@app.get("/usuarios/{user_id}")
def usuario(user_id: int):
    return {"id": user_id}
```

---

## Solución Reto 2 — Middleware de autenticación por header

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

TOKEN_VALIDO = "mi-token-secreto"
RUTAS_PUBLICAS = ["/publico"]


@app.middleware("http")
async def verificar_token(request: Request, call_next):
    # Rutas que no requieren autenticación
    if request.url.path in RUTAS_PUBLICAS:
        return await call_next(request)

    token = request.headers.get("X-Token")

    if token != TOKEN_VALIDO:
        return JSONResponse(
            status_code=403,
            content={"error": "Token inválido o ausente"}
        )

    return await call_next(request)


@app.get("/publico")
def ruta_publica():
    return {"mensaje": "Esta ruta es pública, no necesita token"}


@app.get("/privado")
def ruta_privada():
    return {"mensaje": "Accediste a una ruta privada"}
```

**Prueba con curl:**
```bash
# Sin token → 403
curl http://localhost:8000/privado

# Con token correcto → 200
curl -H "X-Token: mi-token-secreto" http://localhost:8000/privado

# Ruta pública → 200 sin token
curl http://localhost:8000/publico
```

---

## Solución Reto 3 — BaseHTTPMiddleware con límite de tamaño

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

LIMITE_BYTES = 1024  # 1 KB


class LimiteTamanioMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        body = await request.body()

        if len(body) > LIMITE_BYTES:
            return JSONResponse(
                status_code=413,
                content={"error": "Body demasiado grande"}
            )

        return await call_next(request)


app = FastAPI()
app.add_middleware(LimiteTamanioMiddleware)


@app.post("/datos")
async def recibir_datos(request: Request):
    body = await request.body()
    return {"tamanio_bytes": len(body)}
```

**Prueba con curl:**
```bash
# Body pequeño → 200
curl -X POST http://localhost:8000/datos \
  -H "Content-Type: text/plain" \
  -d "hola mundo"

# Body grande → 413 (genera string de 2000 caracteres)
curl -X POST http://localhost:8000/datos \
  -H "Content-Type: text/plain" \
  -d "$(python3 -c "print('x' * 2000)")"
```
