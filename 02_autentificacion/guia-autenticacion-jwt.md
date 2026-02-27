# Guía: Autenticación y Autorización en FastAPI (OAuth2 + JWT)

---

## 1. ¿Qué es y para qué sirve?

La **autenticación** responde a: *¿quién eres?*
La **autorización** responde a: *¿qué puedes hacer?*

En APIs REST, el flujo más común es:

```
1. El usuario manda su usuario y contraseña → recibe un TOKEN
2. En cada request siguiente, el usuario manda ese TOKEN en el header
3. El servidor verifica el token → decide si dejar pasar o no
```

**JWT (JSON Web Token)** es el formato estándar para ese token. Se ve así:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJtaWNoZWwifQ.abc123xyz
```

Son tres partes separadas por puntos: `header.payload.firma`

**OAuth2 Password Flow** es el nombre del esquema que FastAPI usa nativamente para este patrón (usuario + contraseña → token).

---

## 2. ¿Cuándo usarlo?

- Cualquier API que tenga usuarios con login
- Rutas que solo ciertos usuarios pueden ver (ej. `/admin`, `/mi-perfil`)
- Apps móviles o SPAs que consumen tu API
- Cuando no quieres guardar sesiones en el servidor (los JWT son *stateless*)

---

## 3. Instalación y requisitos

```bash
pip install fastapi uvicorn python-jose[cryptography] passlib[bcrypt]

uv add fastapi uvicorn 'python-jose[cryptography]' 'passlib[bcrypt]'

```

| Paquete | Para qué |
|---|---|
| `python-jose` | Crear y verificar tokens JWT |
| `passlib[bcrypt]` | Hashear contraseñas de forma segura |

---

## 4. Conceptos clave

| Término | Significado |
|---|---|
| **JWT** | Token en formato JSON firmado digitalmente. No se puede falsificar sin la clave secreta |
| **Payload** | Datos dentro del token (ej. nombre de usuario, expiración). Son legibles pero no modificables |
| **`sub`** | Campo estándar del JWT que identifica al sujeto (normalmente el username) |
| **`exp`** | Campo estándar que indica cuándo expira el token |
| **SECRET_KEY** | Clave privada del servidor para firmar los tokens. Nunca se comparte |
| **Bearer token** | Forma de enviar el JWT en el header: `Authorization: Bearer <token>` |
| **`OAuth2PasswordBearer`** | Clase de FastAPI que extrae el token del header automáticamente |
| **`OAuth2PasswordRequestForm`** | Clase de FastAPI que recibe usuario+contraseña del formulario de login |
| **Hash de contraseña** | Nunca guardas la contraseña en texto plano, guardas su hash (bcrypt) |
| **Dependencia (`Depends`)** | Mecanismo de FastAPI para reutilizar lógica en rutas (ya lo conoces de SQLModel) |

---

## 5. Ejemplo funcional

Copia este archivo como `main.py`:

```python
from datetime import datetime, timedelta
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel

# ── Configuración ─────────────────────────────────────────────────────────────

# Clave secreta para firmar los tokens. En producción usa una clave larga y aleatoria.
# Genera una con: openssl rand -hex 32
SECRET_KEY = "mi-clave-super-secreta-cambiar-en-produccion"
ALGORITHM = "HS256"           # Algoritmo de firma
TOKEN_EXPIRE_MINUTES = 30     # El token dura 30 minutos

# ── Utilidades de contraseñas ─────────────────────────────────────────────────

# CryptContext maneja el hash y verificación de contraseñas con bcrypt
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hashear_password(password: str) -> str:
    return pwd_context.hash(password)

def verificar_password(password_plano: str, password_hash: str) -> bool:
    return pwd_context.verify(password_plano, password_hash)

# ── Base de datos falsa (en producción usa SQLModel) ──────────────────────────

# Simulamos una BD con un diccionario. Las contraseñas están hasheadas.
db_usuarios = {
    "michel": {
        "username": "michel",
        "nombre": "Michel López",
        "email": "michel@ejemplo.com",
        "password_hash": hashear_password("secreto123"),
    },
    "ana": {
        "username": "ana",
        "nombre": "Ana García",
        "email": "ana@ejemplo.com",
        "password_hash": hashear_password("clave456"),
    },
}

# ── Modelos Pydantic ───────────────────────────────────────────────────────────

class Usuario(BaseModel):
    username: str
    nombre: str
    email: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str

# ── Lógica de JWT ─────────────────────────────────────────────────────────────

def crear_token(datos: dict) -> str:
    """Crea un JWT firmado con expiración."""
    payload = datos.copy()
    expira = datetime.utcnow() + timedelta(minutes=TOKEN_EXPIRE_MINUTES)
    payload["exp"] = expira  # campo estándar de expiración
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

# ── FastAPI y OAuth2 ──────────────────────────────────────────────────────────

app = FastAPI()

# Le dice a FastAPI dónde está el endpoint de login para obtener el token.
# Esto activa el botón "Authorize" en Swagger UI.
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def autenticar_usuario(username: str, password: str):
    """Busca el usuario y verifica su contraseña. Retorna el usuario o None."""
    usuario = db_usuarios.get(username)
    if not usuario:
        return None
    if not verificar_password(password, usuario["password_hash"]):
        return None
    return usuario

def obtener_usuario_actual(token: Annotated[str, Depends(oauth2_scheme)]):
    """
    Dependencia reutilizable: extrae y valida el token JWT.
    Se usa con Depends() en cualquier ruta protegida.
    """
    credenciales_error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Token inválido o expirado",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        # Decodificamos el token con nuestra SECRET_KEY
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")  # "sub" es el campo estándar de identidad
        if username is None:
            raise credenciales_error
    except JWTError:
        raise credenciales_error

    usuario = db_usuarios.get(username)
    if usuario is None:
        raise credenciales_error

    return Usuario(**{k: v for k, v in usuario.items() if k != "password_hash"})

# ── Endpoints ─────────────────────────────────────────────────────────────────

@app.post("/token", response_model=TokenResponse)
def login(form_data: Annotated[OAuth2PasswordRequestForm, Depends()]):
    """
    Endpoint de login. Recibe usuario+contraseña, devuelve el JWT.
    OAuth2PasswordRequestForm espera un form con campos 'username' y 'password'.
    """
    usuario = autenticar_usuario(form_data.username, form_data.password)
    if not usuario:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Usuario o contraseña incorrectos",
            headers={"WWW-Authenticate": "Bearer"},
        )
    token = crear_token({"sub": usuario["username"]})
    return {"access_token": token, "token_type": "bearer"}


@app.get("/usuarios/yo", response_model=Usuario)
def mi_perfil(usuario_actual: Annotated[Usuario, Depends(obtener_usuario_actual)]):
    """Ruta protegida: solo accesible con token válido."""
    return usuario_actual


@app.get("/")
def inicio():
    """Ruta pública, no requiere token."""
    return {"mensaje": "API pública. Ve a /docs para hacer login."}
```

---

## 6. ¿Qué hace cada parte?

### El flujo completo paso a paso

```
1. POST /token  { username: "michel", password: "secreto123" }
        ↓
2. FastAPI recibe con OAuth2PasswordRequestForm
        ↓
3. autenticar_usuario() → verifica contraseña contra hash en BD
        ↓
4. crear_token() → genera JWT firmado con SECRET_KEY
        ↓
5. Respuesta: { "access_token": "eyJ...", "token_type": "bearer" }

─────────────────────────────────────────────

6. GET /usuarios/yo
   Header: Authorization: Bearer eyJ...
        ↓
7. oauth2_scheme extrae el token del header automáticamente
        ↓
8. obtener_usuario_actual() → decodifica JWT → obtiene username del campo "sub"
        ↓
9. Busca el usuario en BD → lo devuelve
        ↓
10. La ruta recibe el objeto Usuario ya validado
```

### ¿Por qué `Depends(obtener_usuario_actual)`?
Porque es código que quieres reutilizar en **muchas rutas**. En lugar de repetir la verificación del token en cada endpoint, lo declaras una vez y FastAPI lo ejecuta automáticamente antes de entrar a la ruta.

### ¿Por qué no guardar la contraseña en texto plano?
Si tu base de datos se filtra, el atacante solo ve hashes bcrypt. Romper bcrypt es computacionalmente muy costoso, dando tiempo a tus usuarios para cambiar contraseñas.

### ¿Por qué el token expira?
Si un token es robado, solo es válido por `TOKEN_EXPIRE_MINUTES`. Sin expiración, un token robado funciona para siempre.

---

## 7. Cómo probarlo

### Con Swagger UI (la forma más fácil)

1. Corre: `uvicorn main:app --reload`
2. Abre: [http://localhost:8000/docs](http://localhost:8000/docs)
3. Haz clic en el botón **Authorize** (candado arriba a la derecha)
4. Escribe `michel` / `secreto123` → clic en **Authorize**
5. Ahora prueba `GET /usuarios/yo` → debería devolver tu perfil

### Con curl

```bash
# 1. Hacer login y obtener token
curl -X POST http://localhost:8000/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=michel&password=secreto123"

# Respuesta: {"access_token":"eyJ...","token_type":"bearer"}

# 2. Usar el token en una ruta protegida (pega el token real)
curl http://localhost:8000/usuarios/yo \
  -H "Authorization: Bearer eyJ..."

# 3. Sin token → 401
curl http://localhost:8000/usuarios/yo
```

---

## 8. Errores comunes

### Error 1: `422 Unprocessable Entity` al hacer login
```
El endpoint /token espera Content-Type: application/x-www-form-urlencoded
NO application/json
```
**Por qué:** `OAuth2PasswordRequestForm` usa el estándar OAuth2 que define el login como un form, no JSON. En Swagger UI esto se maneja automáticamente. Con curl debes usar `-d "username=...&password=..."`.

### Error 2: `401 Unauthorized` con token recién generado
Causas más frecuentes:
- **SECRET_KEY diferente** entre la creación y la verificación del token
- **Token expirado** (cambiar `TOKEN_EXPIRE_MINUTES` para pruebas)
- **Copiar el token con espacios** extra al pegarlo en el header

```python
# Verifica que usas la misma SECRET_KEY en crear_token y jwt.decode
jwt.encode(payload, SECRET_KEY, ...)   # al crear
jwt.decode(token, SECRET_KEY, ...)     # al verificar ← misma clave
```

### Error 3: `ImportError` al importar jose o passlib
```bash
# Asegúrate de instalar los extras correctos
pip install python-jose[cryptography]   # no solo python-jose
pip install passlib[bcrypt]             # no solo passlib
```

---

## 9. Conclusión

Aprendiste el flujo completo de autenticación en FastAPI:
- **Login** con `OAuth2PasswordRequestForm` → genera JWT con `python-jose`
- **Proteger rutas** con `Depends(obtener_usuario_actual)`
- **Hashear contraseñas** con `passlib/bcrypt` (nunca texto plano)
- **Verificar tokens** decodificando el JWT con la `SECRET_KEY`

Esto conecta directamente con lo que ya sabes: usas **Pydantic** para los modelos de respuesta, **Depends** (que ya usabas con SQLModel) para inyectar el usuario actual, y tus rutas GET/POST normales, solo con una dependencia extra.

---

## 10. Siguientes pasos

1. **Roles y permisos** — agregar un campo `rol` al token para diferenciar admin/usuario
2. **Refresh tokens** — tokens de larga duración para renovar el `access_token` sin hacer login
3. **Guardar usuarios en base de datos** — reemplazar el diccionario por SQLModel + tabla `Usuario`
4. **Revocar tokens** — lista negra de tokens en Redis (los JWT son stateless, revocarlos requiere trabajo extra)
5. **OAuth2 con proveedores externos** — login con Google/GitHub usando `authlib`
