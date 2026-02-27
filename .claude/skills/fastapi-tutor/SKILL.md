---
name: fastapi-tutor
description: Genera guías de aprendizaje de temas de FastAPI. Úsala cuando el usuario pida aprender, entender o explorar un tema de FastAPI como autenticación, base de datos, middlewares, etc.
allowed-tools: Read, Write, Bash
---

# FastAPI Tutor

Cuando el usuario pida aprender un tema de FastAPI, genera una carpeta `<tema>` dentro con un siempre un archivo `guia-<tema>.md` y un archivo `retos-<tema>.md`.

## Pasos obligatorios EN ORDEN

1. Extrae el nombre del tema y conviértelo a minúsculas con guiones (ej: "middlewares en FastAPI" → `middlewares`)
2. Crea la carpeta con bash:
```bash
   mkdir -p temas/<tema>
```
3. Genera el archivo `temas/<tema>/guia-<tema>.md`
4. Genera el archivo `temas/<tema>/retos-<tema>.md`


## Estructura obligatoria de guia-<tema>.md

### 1. ¿Qué es y para qué sirve?
Explicación breve del concepto. Sin código aún.

### 2. ¿Cuándo usarlo?
Casos de uso reales y concretos.

### 3. Instalación y requisitos
Solo los paquetes necesarios para este tema:
```bash
pip install <paquetes>
```

### 4. Conceptos clave
Lista de los términos importantes con explicación simple.
Por ejemplo: qué es un token, qué es OAuth2, etc.

### 5. Ejemplo funcional
Código completo que se pueda copiar y ejecutar con:
```bash
fastapi dev main.py
```
El código debe:
- Ser un solo archivo `main.py`
- Tener comentarios explicando cada parte importante
- Cubrir el caso más común del tema

### 6. ¿Qué hace cada parte?
Desglose del ejemplo anterior sección por sección.
Explicar el "por qué" no solo el "qué".

### 7. Cómo probarlo
Instrucciones para verificar que funciona:
- Usando Swagger UI en `/docs`
- O con curl/httpie si aplica

### 8. Errores comunes
Los 2-3 errores más frecuentes de este tema y cómo resolverlos.

### 9. Conclusión
Qué aprendiste y cómo conecta con otros temas de FastAPI.

### 10. Siguientes pasos
Qué tema conviene aprender después de este.

---

## Estructura obligatoria de retos-<tema>.md

### Reto 1 - Básico
Descripción del reto (sin dar el código).
Qué debe lograr el usuario.
Pista opcional.

### Reto 2 - Intermedio
Más complejo, combina el tema con algo visto antes.

### Reto 3 - Avanzado
Requiere investigar algo no cubierto en la guía.

---
Al final del archivo de retos, incluir las soluciones en una sección separada con un aviso claro de "SOLUCIONES - No ver antes de intentarlo".
```

---

## Cómo lo usarías

Le dices a Claude Code:
```
aprende: autenticación JWT en FastAPI
```

Y genera automáticamente:
```
guia-autenticacion-jwt.md
retos-autenticacion-jwt.md
```

Otros ejemplos:
```
aprende: conexión a base de datos con SQLModel
aprende: middlewares en FastAPI
aprende: background tasks
aprende: WebSockets en FastAPI
aprende: testing con pytest en FastAPI