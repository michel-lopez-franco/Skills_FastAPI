
# Ampliar las habilidades de Claude

https://code.claude.com/docs/en/skills


Crea, gestiona y comparte habilidades para ampliar las capacidades de Claude en Claude Code. Incluye comandos de barra personalizados.

Las habilidades amplían las capacidades de Claude. Crea un SKILL.mdarchivo con instrucciones y Claude lo añade a su conjunto de herramientas. Claude usa habilidades cuando es necesario, o puedes invocarlas directamente con /skill-name.
Para comandos integrados como /helpy /compact, consulte modo interactivo .
Los comandos de barra personalizados se han fusionado en las habilidades. Un archivo en .claude/commands/review.mdy una habilidad en .claude/skills/review/SKILL.mdse crean /reviewy funcionan de la misma manera. Tus .claude/commands/archivos existentes siguen funcionando. Las habilidades añaden funciones opcionales: un directorio para archivos de apoyo, un frontmatter para controlar si tú o Claude las invocan , y la capacidad de Claude para cargarlas automáticamente cuando sea necesario.
Las habilidades de Claude Code siguen el estándar abierto de Habilidades de Agente , compatible con múltiples herramientas de IA. Claude Code amplía el estándar con funciones adicionales como el control de invocación , la ejecución de subagentes y la inyección de contexto dinámico .
​


## Intentar aprender a usar skills

La skill actuaría como un "profesor" que sigue siempre la misma estructura pedagógica. Aquí cómo quedaría:

```sh
Usa la skill fastapi-tutor para enseñarme middlewares en FastAPI

Genera una guía de middlewares en FastAPI siguiendo la skill fastapi-tutor
```




# Comandos basicos

```sh
uv init --python 3.12 nombre_del_proyecto

uv python pin 3.12 --global


fastapi dev main.py
```
