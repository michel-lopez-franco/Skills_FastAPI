# Guía: Middlewares en FastAPI

---

## 1. ¿Qué es y para qué sirve?

Un **middleware** es una función que se ejecuta **antes y/o después de cada request** que llega a tu aplicación. Se coloca "en medio" entre el cliente y tus rutas.

Piénsalo así:

```
Cliente → [Middleware] → Tu ruta → [Middleware] → Cliente
```

Sirve para ejecutar lógica que aplica a **todas** (o muchas) las rutas sin repetir código en cada una.

---

## 2. ¿Cuándo usarlo?

- Medir el tiempo que tarda cada request (logging de rendimiento)
- Agregar cabeceras HTTP a todas las respuestas (ej. CORS, seguridad)
- Verificar autenticación de forma global
- Registrar en logs cada request que llega
- Comprimir respuestas (GZip)
- Limitar acceso por IP o rate limiting

---

## 3. Instalación y requisitos

No se necesita instalar nada extra. FastAPI ya incluye soporte de middlewares. Solo necesitas:

```bash
pip install fastapi uvicorn
```

---

## 4. Conceptos clave

| Término | Significado |
|---|---|
| **Middleware** | Función que intercepta requests y responses |
| **`request`** | El objeto que representa lo que el cliente envió |
| **`call_next`** | Función que pasa el request a la ruta correspondiente |
| **`response`** | Lo que tu ruta devuelve al cliente |
| **ASGI** | Estándar que usa FastAPI; los middlewares trabajan a este nivel |
| **`@app.middleware("http")`** | Decorador para registrar un middleware personalizado |

---

## 5. Ejemplo funcional

Copia este archivo como `main.py` y ejecútalo con `uvicorn main:app --reload`.

```python
import time
from fastapi import FastAPI, Request

app = FastAPI()


# ── Middleware personalizado ──────────────────────────────────────────────────
@app.middleware("http")
async def agregar_tiempo_proceso(request: Request, call_next):
    # Código que corre ANTES de llegar a la ruta
    inicio = time.time()

    # Pasamos el request a la ruta y esperamos la respuesta
    response = await call_next(request)

    # Código que corre DESPUÉS de que la ruta respondió
    duracion = time.time() - inicio
    response.headers["X-Process-Time"] = str(round(duracion, 4))

    return response


# ── Rutas normales ────────────────────────────────────────────────────────────
@app.get("/")
def inicio():
    return {"mensaje": "Hola desde FastAPI"}


@app.get("/lento")
async def ruta_lenta():
    # Simulamos una operación que tarda
    import asyncio
    await asyncio.sleep(1)
    return {"mensaje": "Esta ruta tardó 1 segundo"}


@app.get("/usuarios/{user_id}")
def obtener_usuario(user_id: int):
    return {"usuario_id": user_id, "nombre": "Michel"}
```

---

## 6. ¿Qué hace cada parte?

### El decorador `@app.middleware("http")`
Le dice a FastAPI: "esta función debe ejecutarse en cada request HTTP". El argumento `"http"` indica que aplica a todo el tráfico HTTP (que es lo normal).

### Los parámetros `request` y `call_next`
- `request`: contiene toda la info del request entrante (método, URL, headers, body, etc.)
- `call_next`: es una función que debes llamar para que el request llegue a tu ruta. Si no la llamas, la ruta nunca se ejecuta.

### El flujo
```
1. Cliente hace GET /lento
2. Middleware empieza → guarda tiempo de inicio
3. call_next(request) → FastAPI ejecuta la ruta /lento
4. La ruta devuelve {"mensaje": "..."}
5. Middleware continúa → calcula duración → agrega header
6. Response llega al cliente con el header X-Process-Time
```

### Por qué agregar headers en la response
Los headers personalizados (los que empiezan con `X-`) son una forma estándar de enviar metadatos extra al cliente sin afectar el body de la respuesta.

---

## 7. Cómo probarlo

### Con Swagger UI
1. Corre: `fastapi dev main.py`
2. Abre: [http://localhost:8000/docs](http://localhost:8000/docs)
3. Ejecuta cualquier endpoint
4. En la respuesta, busca la sección **Response headers** — verás `x-process-time`

### Con curl
```bash
# Ver los headers de respuesta
curl -v http://localhost:8000/

# Ver el tiempo en la ruta lenta
curl -v http://localhost:8000/lento
```

Busca en la salida la línea:
```
< x-process-time: 1.0012
```

### Con httpie (más legible)
```bash
http GET http://localhost:8000/lento
```

---

## 8. Errores comunes

### Error 1: Olvidar `await call_next(request)`
```python
# ❌ MAL — la ruta nunca se ejecuta
async def mi_middleware(request: Request, call_next):
    response = call_next(request)  # falta await
    return response

# ✅ BIEN
async def mi_middleware(request: Request, call_next):
    response = await call_next(request)
    return response
```
**Por qué pasa:** `call_next` es una corrutina async. Sin `await`, obtienes un objeto coroutine sin ejecutar, y FastAPI lanza un error o devuelve una respuesta vacía.

### Error 2: El middleware no es `async`
```python
# ❌ MAL — no puede usar await
@app.middleware("http")
def mi_middleware(request: Request, call_next):  # falta async
    response = await call_next(request)
    return response

# ✅ BIEN
@app.middleware("http")
async def mi_middleware(request: Request, call_next):
    response = await call_next(request)
    return response
```

### Error 3: Modificar headers después de devolverlos
```python
# ❌ MAL — no puedes modificar headers de StreamingResponse después de call_next
# en algunos casos de streaming
response.headers["X-Custom"] = "valor"  # puede fallar con ciertos tipos de response
```
**Solución:** Para respuestas de streaming, usa `BackgroundTasks` o un middleware basado en `BaseHTTPMiddleware` de Starlette.

---

## 9. Conclusión

Aprendiste que un middleware en FastAPI:
- Se declara con `@app.middleware("http")`
- Recibe `request` y `call_next`
- Puede ejecutar código **antes y después** de cada ruta
- Se usa para lógica transversal (logging, headers, timing, auth global)

Esto conecta con lo que ya sabes: es como un "interceptor" que envuelve todas tus rutas GET/POST sin tocar cada una individualmente.

---

## 10. Siguientes pasos

Temas que conviene ver después de middlewares:

1. **CORS** — FastAPI incluye `CORSMiddleware` listo para usar, que es un middleware especial muy común
2. **Dependencias globales** (`dependencies` en `app = FastAPI(dependencies=[...])`) — alternativa a middlewares para lógica de autenticación
3. **Background Tasks** — ejecutar código después de enviar la respuesta
4. **Manejo de excepciones con `@app.exception_handler`** — similar a middlewares pero solo para errores
