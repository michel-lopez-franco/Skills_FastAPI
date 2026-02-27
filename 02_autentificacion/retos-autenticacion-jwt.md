# Retos: Autenticación y Autorización en FastAPI (JWT)

---

## Reto 1 — Básico: Ruta protegida con mensaje personalizado

### ¿Qué debes lograr?
Tomando el ejemplo de la guía como base, agrega una ruta `GET /saludo` que:
- Esté protegida (requiera token válido)
- Devuelva un mensaje personalizado con el nombre del usuario autenticado

Ejemplo de respuesta:
```json
{"mensaje": "Hola Michel López, bienvenido a la API"}
```

### Condiciones
- Usa `Depends(obtener_usuario_actual)` para obtener el usuario
- El nombre debe venir del objeto `Usuario` que devuelve la dependencia (campo `nombre`)
- Pruébalo en Swagger UI: sin token debe dar 401, con token debe saludar por nombre

### Pista
La dependencia ya te entrega un objeto `Usuario` con todos sus campos. Solo accede a `usuario_actual.nombre`.

---

## Reto 2 — Intermedio: Registro de nuevos usuarios + roles

### ¿Qué debes lograr?
Extiende el ejemplo de la guía para agregar:

1. Un endpoint `POST /registro` que permita crear nuevos usuarios
2. Un campo `rol` en cada usuario con valores `"usuario"` o `"admin"`
3. Una ruta `GET /admin/panel` que solo puedan ver usuarios con `rol == "admin"`

### Condiciones
- El body de `/registro` debe recibir: `username`, `nombre`, `email`, `password`, `rol`
- Hashea la contraseña antes de guardarla
- Si el `username` ya existe, devuelve `400 Bad Request`
- Para `/admin/panel`, si el usuario tiene rol `"usuario"`, devuelve `403 Forbidden`
- El rol debe incluirse en el JWT payload (junto al `sub`)

### Pista
- Agrega `rol` al payload al crear el token: `{"sub": username, "rol": rol}`
- En `obtener_usuario_actual`, extrae el `rol` del payload y agrégalo al objeto de retorno
- Crea una segunda dependencia `solo_admin` que use `Depends(obtener_usuario_actual)` internamente y verifique el rol

---

## Reto 3 — Avanzado: Expiración y renovación de tokens (Refresh Token)

### ¿Qué debes lograr?
Implementa un sistema de **dos tokens**:

- **Access token**: corta duración (2 minutos para facilitar las pruebas)
- **Refresh token**: larga duración (7 días), solo sirve para obtener un nuevo access token

### Endpoints a implementar:
- `POST /token` → devuelve ambos tokens al hacer login
- `POST /token/refresh` → recibe el refresh token, devuelve un nuevo access token
- `GET /yo` → ruta protegida con access token

### Condiciones
- Diferencia los tokens con un campo `tipo` en el payload: `"access"` o `"refresh"`
- `POST /token/refresh` debe rechazar tokens de tipo `"access"` (aunque sean válidos)
- Si el refresh token expiró, devuelve `401` con mensaje claro
- Investiga cómo pasar el refresh token: puedes usarlo en el body como JSON `{"refresh_token": "..."}`

### Pista
- El refresh token se crea igual que el access token pero con `timedelta(days=7)` y `tipo: "refresh"` en el payload
- En `/token/refresh`, decodifica el token, verifica que `payload["tipo"] == "refresh"`, luego crea y devuelve un nuevo access token

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

## Solución Reto 1 — Ruta protegida con saludo personalizado

```python
# Agrega esta ruta al main.py de la guía

@app.get("/saludo")
def saludo(usuario_actual: Annotated[Usuario, Depends(obtener_usuario_actual)]):
    return {"mensaje": f"Hola {usuario_actual.nombre}, bienvenido a la API"}
```

Así de simple. `Depends(obtener_usuario_actual)` ya maneja el 401 si no hay token o es inválido. Solo usas el objeto que te llega.

---

## Solución Reto 2 — Registro de usuarios + roles

```python
from datetime import datetime, timedelta
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel

SECRET_KEY = "mi-clave-super-secreta"
ALGORITHM = "HS256"
TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# BD con campo rol
db_usuarios = {
    "michel": {
        "username": "michel",
        "nombre": "Michel López",
        "email": "michel@ejemplo.com",
        "password_hash": pwd_context.hash("secreto123"),
        "rol": "admin",
    },
    "ana": {
        "username": "ana",
        "nombre": "Ana García",
        "email": "ana@ejemplo.com",
        "password_hash": pwd_context.hash("clave456"),
        "rol": "usuario",
    },
}


class Usuario(BaseModel):
    username: str
    nombre: str
    email: str
    rol: str  # campo nuevo


class TokenResponse(BaseModel):
    access_token: str
    token_type: str


class RegistroForm(BaseModel):
    username: str
    nombre: str
    email: str
    password: str
    rol: str = "usuario"


app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


def crear_token(datos: dict) -> str:
    payload = datos.copy()
    payload["exp"] = datetime.utcnow() + timedelta(minutes=TOKEN_EXPIRE_MINUTES)
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def obtener_usuario_actual(token: Annotated[str, Depends(oauth2_scheme)]):
    error = HTTPException(status_code=401, detail="Token inválido",
                          headers={"WWW-Authenticate": "Bearer"})
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if not username:
            raise error
    except JWTError:
        raise error

    usuario = db_usuarios.get(username)
    if not usuario:
        raise error
    return Usuario(**{k: v for k, v in usuario.items() if k != "password_hash"})


def solo_admin(usuario: Annotated[Usuario, Depends(obtener_usuario_actual)]):
    """Dependencia que verifica que el usuario sea admin."""
    if usuario.rol != "admin":
        raise HTTPException(status_code=403, detail="Solo administradores")
    return usuario


@app.post("/token", response_model=TokenResponse)
def login(form_data: Annotated[OAuth2PasswordRequestForm, Depends()]):
    usuario = db_usuarios.get(form_data.username)
    if not usuario or not pwd_context.verify(form_data.password, usuario["password_hash"]):
        raise HTTPException(status_code=401, detail="Credenciales incorrectas",
                            headers={"WWW-Authenticate": "Bearer"})
    token = crear_token({"sub": usuario["username"], "rol": usuario["rol"]})
    return {"access_token": token, "token_type": "bearer"}


@app.post("/registro", status_code=201)
def registrar(datos: RegistroForm):
    if datos.username in db_usuarios:
        raise HTTPException(status_code=400, detail="El username ya existe")
    db_usuarios[datos.username] = {
        "username": datos.username,
        "nombre": datos.nombre,
        "email": datos.email,
        "password_hash": pwd_context.hash(datos.password),
        "rol": datos.rol,
    }
    return {"mensaje": f"Usuario {datos.username} creado con rol {datos.rol}"}


@app.get("/usuarios/yo", response_model=Usuario)
def mi_perfil(usuario: Annotated[Usuario, Depends(obtener_usuario_actual)]):
    return usuario


@app.get("/admin/panel")
def panel_admin(admin: Annotated[Usuario, Depends(solo_admin)]):
    return {"mensaje": f"Bienvenido al panel admin, {admin.nombre}"}
```

---

## Solución Reto 3 — Access token + Refresh token

```python
from datetime import datetime, timedelta
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel

SECRET_KEY = "clave-secreta-para-pruebas"
ALGORITHM = "HS256"
ACCESS_EXPIRE_MINUTES = 2    # corto para probar expiración fácilmente
REFRESH_EXPIRE_DAYS = 7

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

db_usuarios = {
    "michel": {
        "username": "michel",
        "nombre": "Michel López",
        "password_hash": pwd_context.hash("secreto123"),
    }
}


class TokenPar(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class RefreshRequest(BaseModel):
    refresh_token: str

class Usuario(BaseModel):
    username: str
    nombre: str


app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


def crear_token(datos: dict, expire_delta: timedelta) -> str:
    payload = datos.copy()
    payload["exp"] = datetime.utcnow() + expire_delta
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def obtener_usuario_actual(token: Annotated[str, Depends(oauth2_scheme)]):
    error = HTTPException(status_code=401, detail="Token inválido o expirado",
                          headers={"WWW-Authenticate": "Bearer"})
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        # Rechazar si no es access token
        if payload.get("tipo") != "access":
            raise error
        username = payload.get("sub")
        if not username:
            raise error
    except JWTError:
        raise error

    usuario = db_usuarios.get(username)
    if not usuario:
        raise error
    return Usuario(username=usuario["username"], nombre=usuario["nombre"])


@app.post("/token", response_model=TokenPar)
def login(form_data: Annotated[OAuth2PasswordRequestForm, Depends()]):
    usuario = db_usuarios.get(form_data.username)
    if not usuario or not pwd_context.verify(form_data.password, usuario["password_hash"]):
        raise HTTPException(status_code=401, detail="Credenciales incorrectas")

    access = crear_token(
        {"sub": usuario["username"], "tipo": "access"},
        timedelta(minutes=ACCESS_EXPIRE_MINUTES)
    )
    refresh = crear_token(
        {"sub": usuario["username"], "tipo": "refresh"},
        timedelta(days=REFRESH_EXPIRE_DAYS)
    )
    return {"access_token": access, "refresh_token": refresh}


@app.post("/token/refresh")
def renovar_token(body: RefreshRequest):
    error = HTTPException(status_code=401, detail="Refresh token inválido o expirado")
    try:
        payload = jwt.decode(body.refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        # Solo aceptar tokens de tipo "refresh"
        if payload.get("tipo") != "refresh":
            raise HTTPException(status_code=400, detail="El token no es un refresh token")
        username = payload.get("sub")
        if not username or username not in db_usuarios:
            raise error
    except JWTError:
        raise error

    nuevo_access = crear_token(
        {"sub": username, "tipo": "access"},
        timedelta(minutes=ACCESS_EXPIRE_MINUTES)
    )
    return {"access_token": nuevo_access, "token_type": "bearer"}


@app.get("/yo", response_model=Usuario)
def mi_perfil(usuario: Annotated[Usuario, Depends(obtener_usuario_actual)]):
    return usuario
```

**Flujo de prueba:**
```bash
# 1. Login → obtienes access_token y refresh_token
curl -X POST http://localhost:8000/token \
  -d "username=michel&password=secreto123"

# 2. Usa el access_token (válido 2 minutos)
curl http://localhost:8000/yo -H "Authorization: Bearer <access_token>"

# 3. Espera 2 minutos → el access_token expira → 401

# 4. Renueva con el refresh_token (válido 7 días)
curl -X POST http://localhost:8000/token/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "<refresh_token>"}'

# 5. Usa el nuevo access_token
curl http://localhost:8000/yo -H "Authorization: Bearer <nuevo_access_token>"
```
