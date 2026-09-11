# Aprender SDD con SightPause: de la idea a la primera extensión

Tutorial práctico en español · Entorno revisado: **8 de septiembre de 2026** · Fuentes externas consultadas: **7 de septiembre de 2026**.

Esta guía está escrita para alguien que nunca ha usado desarrollo guiado por especificaciones (SDD). Aprenderás a convertir una idea en requisitos, trabajar con un agente de IA, revisar sus resultados y hacer una segunda iteración sin perder las decisiones de la primera.

El ejemplo es **SightPause**, una extensión Chromium que recuerda hacer descansos visuales mediante una notificación y un pequeño panel, llamado *popup*. Se propone una interfaz minimalista, bonita y accesible con HTML, CSS y JavaScript.

**Estado del proyecto en esta revisión:** la CLI y los archivos de Spec Kit están en **1.0.4**, con la integración verificada; todavía no hay una extensión implementada ni una especificación de funcionalidad. Los prompts, requisitos y árboles futuros de este documento son ejemplos para construirla. Se ha actualizado la infraestructura y comprobado el flujo básico de scripts en una copia temporal; el desarrollo de SightPause sigue pendiente.

## Índice

1. [Cómo seguir el tutorial](#lectura)
2. [Qué es SDD y qué aporta cada herramienta](#conceptos)
3. [Reconocer el repositorio y comprobar el entorno](#entorno)
4. [Entender las versiones y actualizar Spec Kit](#actualizacion)
5. [Trabajar con un LLM: conversación, Plan, agente y objetivos](#llm)
6. [Definir el primer MVP y sus dependencias](#mvp)
7. [Recorrer el primer ciclo de Spec Kit](#ciclo)
8. [Probar la extensión y decidir si está terminada](#pruebas)
9. [Iterar: error, cambio funcional, mejora visual y nueva función](#iteraciones)
10. [Git, contexto y recuperación de sesiones](#sesiones)
11. [Qué recomiendan usuarios de Spec Kit](#experiencias)
12. [Resolver problemas frecuentes](#problemas)
13. [Comprobar lo aprendido y consultar las fuentes](#fuentes)

<a id="lectura"></a>
## 1. Cómo seguir el tutorial

**Aprenderás:** a distinguir una explicación de una acción que modifica el proyecto.

**Antes de empezar:** abre `C:\dev\sightpause` como proyecto en Codex. Necesitas poder abrir una terminal PowerShell, leer un Markdown y cargar una extensión local en un navegador. No necesitas conocer SDD. Si un término de programación te resulta desconocido, usa el glosario y pide una explicación antes de aceptar una decisión técnica.

**Dónde actuar:** cada bloque indica su destino:

| Etiqueta | Dónde se utiliza | Ejemplo |
|---|---|---|
| PowerShell | Terminal, desde la raíz del repositorio | `specify version` |
| Codex — conversación | Cuadro de mensaje del agente | «Explícame este requisito» |
| Codex — skill | Cuadro de mensaje, seleccionando la skill si aparece | `$speckit-specify ...` |
| Archivo | Editor de texto o petición explícita al agente | Revisar `spec.md` |
| Navegador | Interfaz del navegador | Cargar la extensión sin empaquetar |

Los bloques `text` destinados a Codex **no se pegan en PowerShell**. En PowerShell, `$` tiene otro significado. Copia el contenido del bloque, sin las marcas de Markdown que lo delimitan.

**Primera acción — Codex, conversación:**

```text
Estoy aprendiendo SDD con docs/guia-sdd.md en SightPause.
Explícame con un ejemplo la diferencia entre un requisito, un plan técnico
y una tarea. No modifiques archivos. Después pídeme que clasifique tres frases.
```

**Resultado esperado:** puedes reconocer que «el recordatorio sigue funcionando al cerrar el popup» es un requisito; «guardar el vencimiento en chrome.storage.local» es una decisión técnica; «implementar la persistencia del vencimiento» es una tarea.

**Revisa:** que el agente explique los términos sin introducir nuevas herramientas para resolver un ejemplo sencillo.

**Continúa o retoma:** sigue las secciones en orden la primera vez. Para volver otro día, usa el [prompt de recuperación](#sesiones). No ejecutes todos los bloques de la guía de una vez: los de actualización y los cuatro ejercicios de iteración tienen requisitos propios.

<a id="conceptos"></a>
## 2. Qué es SDD y qué aporta cada herramienta

### La idea con un ejemplo

SDD significa *Spec-Driven Development*, o desarrollo guiado por especificaciones. Primero expresas qué debe hacer el producto y cómo reconocerás que lo hace bien. Después diseñas la solución, organizas el trabajo y construyes el código. Cuando cambia el comportamiento deseado, mantienes esos documentos alineados con la implementación.

Una petición como «haz un temporizador bonito» deja muchas decisiones abiertas. ¿Sigue contando cuando cierras el popup? ¿Qué ocurre al pausar? ¿Qué pasa después de suspender el ordenador? Una especificación transforma esas preguntas en comportamientos revisables.

Por ejemplo:

> Como persona que trabaja con el navegador, quiero recibir un aviso periódico aunque el popup esté cerrado, para recordar hacer una pausa sin tener que vigilar un contador.

Y un criterio de aceptación:

> Dado un recordatorio activo, cuando cierro y vuelvo a abrir el popup antes de su vencimiento, el próximo aviso conserva la misma hora prevista.

Puedes pedir al LLM que te ayude a redactarlo. La persona sigue decidiendo si ese es el comportamiento que necesita. Un documento largo o una respuesta convincente no prueban que la extensión funcione.

El recorrido de Spec Kit se apoya en documentos que se pasan de una fase a la siguiente. Su [guía oficial de inicio](https://github.github.com/spec-kit/quickstart.html) describe el proceso y sus comandos. Los ejemplos de este tutorial adaptan ese recorrido a SightPause y a las skills instaladas en este repositorio.

### Glosario de trabajo

| Término | Significado en este proyecto |
|---|---|
| Especificación, o *spec* | Documento del comportamiento esperado, alcance y criterios de aceptación; normalmente `spec.md`. |
| Constitución | Principios del proyecto que deben respetar todas las funcionalidades. |
| Historia de usuario | Necesidad expresada desde la perspectiva de quien usa el producto. |
| Criterio de aceptación | Condición observable que permite comprobar un requisito. |
| Plan técnico | Decisiones sobre cómo construir lo especificado; normalmente `plan.md`. |
| Tarea | Unidad concreta de trabajo con resultado verificable, dentro de `tasks.md`. |
| Funcionalidad, o *feature* | Capacidad del producto con alcance propio; puede tener varias historias y tareas. |
| MVP | Primera versión pequeña que ya aporta valor y se puede probar. |
| Dependencia de software | Herramienta, biblioteca o plataforma que necesita el proyecto. |
| Dependencia entre tareas | Trabajo que debe terminar antes de que otro pueda empezar. |
| Contrato | Acuerdo sobre una interfaz: mensajes, entradas, respuestas o comportamiento de una pantalla. No exige un servidor. |
| Prueba de regresión | Comprobación que detecta si reaparece un error ya corregido. |
| LLM o modelo | Sistema que interpreta y genera lenguaje o código. |
| Agente | Sistema que usa un modelo y herramientas para leer archivos, ejecutar comandos y realizar trabajo. |
| Contexto | Información que recibe el agente: petición, conversación, archivos e instrucciones. |
| Skill | Instrucciones especializadas que el agente puede seguir para una tarea. |
| Diff | Comparación de cambios en archivos. |
| Commit | Punto guardado en el historial Git con una descripción del cambio. |
| Rama | Línea de trabajo separada dentro del repositorio Git. |
| Hook | Acción asociada a una fase, por ejemplo crear una rama antes de especificar. |

### Herramientas que conviene distinguir

| Elemento | Responsabilidad |
|---|---|
| Git | Guarda versiones de archivos y permite trabajar con ramas localmente. |
| GitHub | Aloja repositorios y ofrece revisión mediante pull requests e issues. |
| Spec Kit | Organiza un proceso de desarrollo mediante plantillas, comandos y extensiones. |
| CLI Specify | Programa de terminal `specify` para instalar y gestionar esa infraestructura. |
| Integración Codex | Archivos que enseñan a Codex cómo ejecutar las fases de Spec Kit. |
| Extensión Git de Spec Kit | Añade operaciones Git al proceso; no es una extensión del navegador. |
| Extensión SightPause | El producto Chromium que construirás siguiendo el tutorial. |

**Ejercicio — Codex, conversación:**

```text
Comprueba si he entendido SDD. Propón tres comportamientos ambiguos de
SightPause y ayúdame a convertir uno en una historia y dos criterios de
aceptación. Mantén las decisiones de implementación fuera de los requisitos.
No generes todavía archivos.
```

**Resultado, revisión y siguiente paso:** conserva un ejemplo que entiendas y puedas comprobar. Si solo dice «rápido», «bonito» o «fiable», pide que lo concrete. Continúa cuando puedas explicar la diferencia entre qué construir y cómo hacerlo.

<a id="entorno"></a>
## 3. Reconocer el repositorio y comprobar el entorno

**Aprenderás:** dónde vive cada parte y qué está instalado realmente.

**Antes:** proyecto abierto y terminal en PowerShell. **Dónde:** terminal y editor. Estos comandos son de consulta:

```powershell
Set-Location C:\dev\sightpause
git status --short --branch
git log -3 --oneline
git remote -v
specify version
specify check
specify integration status
specify extension list
Get-Content .specify/init-options.json
Get-Content .specify/integration.json
```

### Estado verificado tras la actualización

Esta tabla refleja la comprobación del 08/09/2026, después de actualizar los archivos de Spec Kit y antes de editar esta revisión de la guía. Si ejecutas el tutorial después de cambiar el proyecto, tu salida puede ser distinta.

| Elemento | Estado observado |
|---|---|
| Rama y commit | `main`; `f4c7ec1 upgrade Spec Kit project files to v1.0.4`; árbol Git limpio antes de esta revisión documental |
| Remoto | `https://github.com/rublaman/sightpause.git` |
| CLI | Specify 1.0.4, instalada mediante uv |
| Integración e infraestructura | Codex y archivos compartidos en 1.0.4; skills habilitadas |
| Extensión Git | Git Branching Workflow 1.0.0, habilitada, 5 comandos y 18 hooks |
| Auto-commit | `auto_commit.default: false` y todos los eventos desactivados en `git-config.yml` |
| Scripts del núcleo | Bash, selección `script: sh` |
| Constitución | Plantilla con campos pendientes |
| Funcionalidad activa | No existe todavía `.specify/feature.json` |
| Producto | No existen `specs/`, código de extensión ni pruebas |
| Herramientas auxiliares | Git Bash, Python 3.13, uv y Node 24.19.0 disponibles; npm no localizado en esta sesión |
| Diagnóstico de integración | `OK`; 0 archivos gestionados modificados, 0 ausentes y 0 rutas inválidas |

No hace falta que `specify check` encuentre todos los agentes que enumera. En este proyecto importa la integración Codex y las herramientas que utilicen sus scripts. Ese comando no prueba el funcionamiento de la futura extensión.

### Qué carpetas son infraestructura y cuáles contendrán tu trabajo

**Estructura actual comprobada en `C:\dev\sightpause`:** el árbol siguiente muestra carpetas y archivos que ya existen. Se omite el contenido de las carpetas marcadas con `…`. No es la estructura final de la extensión.

```text
sightpause/
├── .agents/
│   └── skills/                  Skills de las fases y comandos Git
│       └── …
├── .git/                        Historial Git local (carpeta oculta)
│   └── …
├── .specify/
│   ├── extensions/
│   │   ├── .registry            Registro de extensiones instaladas
│   │   └── git/
│   │       ├── commands/
│   │       │   └── …
│   │       ├── scripts/
│   │       │   ├── bash/
│   │       │   │   └── …
│   │       │   ├── powershell/
│   │       │   │   └── …
│   │       │   └── python/
│   │       │       └── …
│   │       ├── config-template.yml
│   │       ├── extension.yml
│   │       ├── git-config.yml   Configuración de la extensión Git
│   │       └── README.md
│   ├── integrations/
│   │   ├── codex.manifest.json
│   │   └── speckit.manifest.json
│   ├── memory/
│   │   ├── .constitution-template.json
│   │   └── constitution.md      Principios del proyecto, aún pendientes
│   ├── scripts/
│   │   └── bash/                Scripts del núcleo seleccionados: sh
│   │       └── …
│   ├── templates/               Moldes para los documentos
│   │   └── …
│   ├── workflows/
│   │   ├── speckit/
│   │   │   └── workflow.yml
│   │   └── workflow-registry.json
│   ├── .gitignore
│   ├── extensions.yml           Configuración de extensiones y hooks
│   ├── init-options.json        Opciones y versión registradas
│   └── integration.json         Integración y configuración registrada
├── docs/
│   └── guia-sdd.md               Este tutorial
└── .gitignore
```

En un árbol abreviado, `.agents/skills/` significa una carpeta `skills` dentro de `.agents`; no es una carpeta con ese nombre completo. Lo mismo ocurre con `scripts/bash/` o `memory/constitution.md`. Algunos exploradores agrupan esos niveles en una sola línea. Además, `.git` puede no verse si el explorador oculta carpetas ocultas. Para consultar el árbol local desde PowerShell puedes ejecutar `Get-ChildItem -Force` y `tree /F /A .specify`.

**Qué pertenece a cada parte:** `.agents/` y la mayor parte de `.specify/` son infraestructura de Spec Kit. Dentro de ella, `.specify/memory/constitution.md` es el documento de principios que sí completarás para SightPause; los moldes de `.specify/templates/` no se rellenan como si fueran las especificaciones del producto. `docs/` contiene documentación propia y `.git/` guarda el historial del repositorio. La extensión Git distribuye scripts para Bash, PowerShell y Python, aunque el núcleo de este proyecto esté configurado con `script: sh`; ver esas tres carpetas no indica una configuración incorrecta.

**Lo que se creará después:** actualmente no existen `specs/` ni `.specify/feature.json`. Al ejecutar `$speckit-specify` para la primera funcionalidad, se creará una estructura como esta; `001-nombre-funcionalidad` es un nombre ilustrativo:

```text
sightpause/
├── .specify/
│   └── feature.json              Puntero a la funcionalidad activa
└── specs/
    └── 001-nombre-funcionalidad/
        ├── spec.md               Requisitos de esa funcionalidad
        └── checklists/
            └── requirements.md   Revisión de calidad de la especificación
```

Este segundo árbol muestra solo las incorporaciones, no sustituye al anterior. Después, `$speckit-plan` añadirá `plan.md` y los documentos de diseño que correspondan, y `$speckit-tasks` añadirá `tasks.md` dentro de la funcionalidad. La [skill de planificación instalada](../.agents/skills/speckit-plan/SKILL.md) explica esa generación. La ubicación del código de la extensión y sus pruebas se decidirá en el plan; todavía no hay una estructura final del producto que debas reproducir manualmente.

### Windows: elegir el Bash correcto

En esta sesión `bash` resuelve a `C:\WINDOWS\system32\bash.exe`, el lanzador de Windows. Git Bash está disponible en otra ruta. Puedes invocarlo explícitamente desde PowerShell:

```powershell
& 'C:/Program Files/Git/bin/bash.exe' --version
& 'C:/Program Files/Git/bin/bash.exe' .specify/scripts/bash/check-prerequisites.sh --paths-only --json
```

Antes de crear una funcionalidad, el segundo comando devuelve `Feature directory not found`. En este punto es el resultado esperado: falta seleccionar o crear la spec. No demuestra que Bash esté mal instalado.

Para orientar al agente, copia en Codex:

```text
Este proyecto usa scripts sh en Windows. Para los scripts Bash de Spec Kit
utiliza C:/Program Files/Git/bin/bash.exe explícitamente. Trabaja desde
C:/dev/sightpause y comprueba el directorio de funcionalidad activa antes
de generar documentos. No cambies el tipo de scripts durante este tutorial.
```

**Revisa:** rutas, versiones y estado de Git. **Continúa o retoma:** si están disponibles Specify y Git Bash, pasa a versiones; si el error es de funcionalidad inexistente, se resolverá en `specify`. Ante errores con `\r` o `bad interpreter`, consulta [problemas frecuentes](#problemas).

<a id="actualizacion"></a>
## 4. Entender las versiones y actualizar Spec Kit

**Aprenderás:** a actualizar la herramienta y sus archivos de proyecto por separado.

**Antes:** guarda y revisa tu trabajo con Git. Esta lección es opcional para comprender SDD. **Dónde:** PowerShell. Los comandos de actualización sí modifican herramientas o archivos; los de consulta no.

### CLI, archivos del proyecto y extensión Git

La CLI es el programa que ejecutas al escribir `specify`. Los archivos del proyecto son copias que se instalaron al preparar el repositorio. Puedes actualizar la CLI y conservar esas copias anteriores: ese era el estado del 07/09/2026, con CLI 1.0.4 y archivos 1.0.1. El 08/09/2026 se actualizaron también los archivos a 1.0.4. La versión 1.0.0 de la extensión Git tiene su propia numeración y se conserva.

Los valores de [init-options.json](../.specify/init-options.json) guardan opciones del proyecto y la versión registrada, que la actualización también modifica; para evaluar el estado actual consulta además [integration.json](../.specify/integration.json), los manifiestos de instalación y `specify integration status`. No cambies un número en un JSON para simular una actualización.

**Actualización ya realizada:** se ejecutó `specify integration upgrade codex --force` después de comprobar que las diferencias gestionadas eran únicamente CRLF/LF. La CLI reinstaló los archivos y actualizó los metadatos y manifiestos. Se conservaron los scripts `sh`, las skills, la numeración secuencial y el auto-commit desactivado. No necesitas repetir la actualización para comenzar el tutorial.

### Consultar, actualizar y revisar

La [guía oficial de actualización](https://github.github.com/spec-kit/upgrade.html) distingue CLI, integración y extensiones. Estos comandos se han contrastado con la ayuda de la CLI instalada:

```powershell
Set-Location C:\dev\sightpause
specify version
specify self check
specify integration status
specify extension list
git status --short --branch
```

El 07/09/2026, `specify self check` respondió `Up to date: 1.0.4`. Esa consulta es histórica: usa la salida que obtengas ahora para saber si existe otra versión. Los comandos siguientes quedan como referencia para futuras actualizaciones; no son pasos pendientes para llegar a 1.0.4.

```powershell
# Consulta lo que haría la actualización, sin aplicarla
specify self upgrade --dry-run

# Ejecuta solo cuando quieras actualizar el programa
specify self upgrade

# Actualiza los archivos de integración de este proyecto
specify integration upgrade codex

# Actualiza la extensión Git si hay una versión aplicable
specify extension update git
```

El último comando puede informar que no hay una actualización aplicable. No significa que la extensión esté ausente. No vuelvas a ejecutar `specify init --here --force` como procedimiento habitual de actualización de este repositorio.

### Caso histórico resuelto: 22 archivos modificados por los saltos de línea

Antes de actualizar, se observó `core.autocrlf=true`, archivos LF en el índice y CRLF en el directorio de trabajo. Git consideraba limpio el contenido; los hashes de Spec Kit detectaban diferencias de bytes. Después de actualizar, `specify integration status` devuelve `OK`, sin esas diferencias. Se conserva este caso para diagnosticar si vuelven a aparecer.

Se compararon los 10 archivos gestionados de Codex y los 12 de infraestructura compartida: todos coincidían con el hash registrado al convertir CRLF a LF en memoria. No había diferencias adicionales. Puedes repetir esta comprobación de lectura; no escribe archivos:

```powershell
@'
import hashlib
import json
from pathlib import Path

for name in ("codex", "speckit"):
    manifest = json.loads(
        Path(f".specify/integrations/{name}.manifest.json").read_text(encoding="utf-8")
    )
    counts = {"identicos": 0, "solo_crlf": 0, "otros_o_ausentes": 0}
    for filename, expected in manifest["files"].items():
        path = Path(filename)
        if not path.is_file():
            counts["otros_o_ausentes"] += 1
            print("AUSENTE:", filename)
            continue
        data = path.read_bytes()
        if hashlib.sha256(data).hexdigest() == expected:
            counts["identicos"] += 1
        elif hashlib.sha256(data.replace(b"\r\n", b"\n")).hexdigest() == expected:
            counts["solo_crlf"] += 1
        else:
            counts["otros_o_ausentes"] += 1
            print("REVISAR:", filename)
    print(name, counts)
'@ | python -
```

**Resultado observado inicial:** Codex: 10 `solo_crlf`; infraestructura: 12 `solo_crlf`; ambos con 0 `otros_o_ausentes`. Esta comprobación no audita los archivos de la extensión Git.

**Resultado tras actualizar a 1.0.4:** Codex: 10 `identicos`; infraestructura: 12 `identicos`; ambos con 0 `solo_crlf` y 0 `otros_o_ausentes`.

Si la actualización de integración se bloquea, los archivos siguen teniendo únicamente esas diferencias y tienes guardado el trabajo que quieras conservar, puedes ejecutar:

```powershell
specify integration upgrade codex --force
```

`--force` permite reemplazar archivos gestionados modificados. Si aparecen otras diferencias, revisa su contenido antes de decidir qué reemplazar. El diagnóstico previo a esta actualización no sirve para justificar sobrescribir personalizaciones futuras.

### Comprobar el resultado de cualquier actualización

```powershell
specify version
specify self check
specify integration status
specify extension list
specify check
git diff --stat
git diff
git status --short
Get-Content .specify/extensions.yml
Get-Content .specify/extensions/git/git-config.yml
```

**Revisa:** integración Codex, scripts `sh`, extensión Git habilitada y política de auto-commit conservada. Si Git transforma otra vez los saltos de línea, el aviso puede reaparecer; comprueba su causa. Si una skill no aparece tras actualizarla, vuelve a abrir la sesión para recargar las instrucciones.

**Validación realizada el 08/09/2026:** integración `OK` y comprobaciones de sintaxis Bash superadas. En una copia temporal se verificaron la creación de una funcionalidad, la generación del plan, la preparación de tareas, la resolución de la plantilla de checklist y los prerrequisitos de análisis/convergencia. Los resultados JSON se pudieron leer y la ausencia de `spec.md` se rechazó como corresponde. Estas comprobaciones validan el flujo básico de scripts; no equivalen a ejecutar todas las fases con un agente ni a probar la futura extensión.

**Cambios de comportamiento relevantes en 1.0.4:** `analyze` y `converge` comprueban explícitamente la existencia de `spec.md` mediante `--require-spec`; necesitan `spec.md`, `plan.md` y `tasks.md`. Además, `setup-plan.sh` rechaza argumentos desconocidos que antes ignoraba. Las skills instaladas usan los argumentos compatibles y los prompts de este tutorial siguen siendo válidos. La extensión Git 1.0.0 declara `speckit_version: ">=0.2.0"`, requisito que 1.0.4 cumple.

**Continúa o retoma:** con el diff revisado y el estado entendido. Registra qué versiones utilizas. No describas la extensión como probada por haber pasado `specify check`.

<a id="llm"></a>
## 5. Trabajar con un LLM: conversación, Plan, agente y objetivos

**Aprenderás:** qué pedir a la IA, cuánta autonomía darle y cómo comprobar su trabajo.

**Antes:** conoce el glosario y abre el proyecto correcto. **Dónde:** conversación de Codex. No necesitas una API propia ni una clave de OpenAI dentro de SightPause; la IA se utiliza para desarrollar el producto.

### Modelo, agente y modo son decisiones diferentes

El modelo interpreta y genera contenido. El agente combina el modelo con acceso a archivos y herramientas. El modo condiciona qué trabajo puede hacer en esa sesión. Puedes usar el mismo modelo para explicar un requisito y para editar código; la petición, las herramientas y el modo cambian.

| Necesidad | Forma de trabajo | Resultado que debes pedir |
|---|---|---|
| Aprender o comparar alternativas | Conversación; indica que no edite | Explicación, opciones y consecuencias |
| Decidir alcance o resolver incertidumbre | Modo Plan, mediante `/plan` si está disponible | Propuesta revisable antes de editar |
| Crear constitución, spec o documentos técnicos | Modo de ejecución del agente y skill correspondiente | Archivos guardados y resumen del diff |
| Implementar una funcionalidad ya definida | Agente con `$speckit-implement` | Código y evidencia de pruebas |
| Sostener varias iteraciones hasta un resultado | Objetivo persistente, si está disponible | Progreso contra una condición de finalización |
| Revisar | Petición de revisión con límites explícitos | Hallazgos y evidencias; cambios solo si los pides |

La [documentación de prompting de Codex](https://learn.chatgpt.com/docs/prompting) recomienda indicar el comportamiento, contexto, restricciones y forma de verificarlo, y presenta `/plan` para investigar antes de editar. Los nombres y controles pueden variar entre aplicación, CLI y extensión del editor: usa la superficie que exponga tu versión.

### Modo Plan no es `$speckit-plan`

**Modo Plan de Codex:** conversas para acordar una propuesta antes de modificar archivos. Incluso si pides «impleméntalo», tendrás que salir de ese modo para ejecutar cambios.

**Skill `$speckit-plan`:** lee una especificación existente y crea archivos de diseño como `plan.md`, `research.md`, `data-model.md` y `quickstart.md`. Necesita permiso de escritura propio del modo de ejecución. No crea por sí sola la primera especificación ni implementa la extensión.

Por eso el recorrido de aprendizaje puede ser: conversar en Plan → revisar el alcance → pasar a ejecución → invocar las skills que escriben documentos → revisar esos documentos → implementar. La fase de planificación técnica sigue ocurriendo antes del código, aunque utilice el agente en modo de ejecución para guardar Markdown.

### Plantilla para pedir trabajo útil

```text
Objetivo: [comportamiento o resultado concreto].
Contexto: [funcionalidad activa, archivos y situación actual].
Restricciones: [alcance, decisiones que debemos conservar y límites].
Verificación: [prueba o evidencia que demuestra que está terminado].
Trabaja en [explicación / propuesta / edición / revisión].
Si falta una decisión que cambie el comportamiento, explícala antes de asumirla.
Al terminar, indica qué cambió, qué comprobaste y qué sigue pendiente.
```

No copies los corchetes como si fueran datos reales. Sustitúyelos o utiliza los prompts completos del primer ciclo.

**Ejemplo — Codex en modo Plan:**

```text
Quiero definir el primer MVP de SightPause, una extensión Chromium para
recordar descansos visuales mediante notificaciones y un popup minimalista.
Lee docs/guia-sdd.md y la estructura actual. Ayúdame a decidir qué ocurre al
pausar, reiniciar el navegador o no poder mostrar notificaciones.
Explícame las alternativas con lenguaje sencillo. No escribas archivos aún.
Terminamos cuando tengamos un alcance pequeño y criterios que yo pueda probar.
```

**Resultado esperado:** decisiones entendibles. **Revisa:** que no aparezcan cuentas, backend, estadísticas o permisos que no has pedido. Si una propuesta no se entiende, pide «explícamela con un ejemplo y una alternativa más sencilla».

### Objetivos persistentes: cuándo y cómo

Un objetivo permite mantener un resultado pendiente entre iteraciones del agente. Es útil cuando necesitas investigar, corregir y volver a probar hasta resolver algo. Para una pequeña edición de texto, una petición normal basta. La [guía oficial de objetivos](https://developers.openai.com/cookbook/examples/codex/using_goals_in_codex) describe su ciclo y recomienda condiciones comprobables.

Después de definir la spec y las tareas, y fuera del modo Plan, un ejemplo sería:

```text
/goal Completar la funcionalidad activa de recordatorios de SightPause según
su spec.md, plan.md y tasks.md. Verifica las pruebas automatizadas y registra
en quickstart.md cuáles de los escenarios manuales se han comprobado realmente.
Conserva el alcance local, la interfaz mínima y los permisos acordados.
Si falta una decisión de producto o una comprobación requiere mi intervención,
expón lo que necesitas; no declares que una prueba pasó sin ejecutarla.
```

Comandos del ciclo de objetivos, cuando la superficie los admita:

```text
/goal
/goal pause
/goal resume
/goal clear
```

El primero consulta el objetivo; los siguientes lo pausan, reanudan o eliminan. Eliminar el objetivo no revierte cambios de código. Usa el control equivalente de la aplicación si tu versión presenta otra interfaz. Estos comandos pertenecen a Codex, no a Spec Kit, y no se ejecutan en PowerShell.

### Contexto, consumo y revisión humana

- Empieza con el modelo configurado que permita leer, editar y usar herramientas; no necesitas cambiarlo para cada fase. Si una decisión es difícil, pide comparar opciones y razonar sobre consecuencias antes de aumentar la complejidad del proceso.
- Referencia los archivos de la funcionalidad. Evita pegar todas las specs o toda la conversación en cada mensaje.
- Trabaja primero con un agente y una funcionalidad. La marca `[P]` en tareas significa que no dependen entre sí; no obliga a crear subagentes.
- Pide resultados observables: archivos, comandos ejecutados y salidas relevantes. «Hecho» no es una prueba.
- Al cambiar de modelo o sesión, conserva contexto en los artefactos y un resumen de estado. No dependas solo de la memoria del chat.
- Evita regenerar todo `tasks.md` por un ajuste menor si ya contiene progreso. Pide cambios concretos que conserven identificadores y estado.

Estas son recomendaciones de trabajo para este tutorial, no garantías de ahorro o de corrección. **Continúa o retoma:** cuando sepas elegir entre explicar, proponer, editar y revisar, define el MVP.

<a id="mvp"></a>
## 6. Definir el primer MVP y sus dependencias

**Aprenderás:** a limitar el producto y separar decisiones de uso de decisiones técnicas.

**Antes:** has elegido notificación y popup como experiencia inicial. **Dónde:** conversación y después `spec.md` mediante la fase de especificación.

### Propuesta funcional para los ejercicios

Los siguientes valores son **supuestos didácticos para confirmar en `clarify`**, no requisitos ya aprobados ni prestaciones existentes:

| Tema | Propuesta inicial |
|---|---|
| Inicio | La primera vez, la persona pulsa «Iniciar recordatorios». |
| Frecuencia | 20 minutos por defecto; intervalo configurable en minutos enteros entre 1 y 120. |
| Popup | Estado, tiempo hasta el próximo aviso, intervalo y control Iniciar/Pausar/Reanudar según corresponda. |
| Aviso | Notificación breve que recuerda apartar la vista y hacer una pausa; sin bloquear páginas. |
| Repetición | Tras procesar un vencimiento, programar el siguiente intervalo. Cerrar el aviso no confirma que se haya realizado un descanso. |
| Pausa inicial | Conservar el tiempo restante; no emitir avisos mientras está pausado; reanudar desde ese tiempo. |
| Cambio de intervalo | Activo: empezar un intervalo completo. Pausado: conservar la pausa y guardar el nuevo intervalo completo como tiempo restante. |
| Reapertura del popup | Mantener el vencimiento; abrir el panel no inicia otro temporizador. |
| Reinicio y suspensión | Conservar preferencias y pausa. Si hay un vencimiento atrasado al volver, procesar como máximo un aviso y comenzar un nuevo intervalo, sin acumular avisos históricos. |
| Notificaciones no disponibles | Conservar controles y estado; mostrar explicación cuando se detecte el bloqueo o falle la API. No asegurar que el sistema mostró un aviso solo porque la API aceptó crearlo. |
| Privacidad | Datos locales; sin cuenta, telemetría ni lectura de webs. |
| Idioma y alcance | Interfaz en español; primera comprobación en Chrome y otra en Edge si está disponible. |

El intervalo mide tiempo transcurrido, no tiempo efectivo mirando la pantalla. Detectar inactividad, programar horarios o cronometrar un descanso de 20 segundos son capacidades distintas y quedan fuera del primer ejemplo. Este producto es un recordatorio; no se presentará como tratamiento ni prometerá beneficios clínicos.

### Diseño agradable sin complicar la tecnología

HTML define la estructura, CSS el aspecto y JavaScript el comportamiento. TypeScript añade comprobaciones sobre tipos de datos; no mejora por sí solo el aspecto visual. Para este MVP se propone CSS organizado con variables de color, tipografía y espaciado, botones con estados claros y un número pequeño de elementos visibles.

Traduce «bonito» en criterios revisables: no hay texto recortado, el estado principal se identifica rápidamente, el foco del teclado se ve, los controles tienen nombre y el estado no se comunica solo mediante color. Mantén los detalles visuales en el plan o contrato de interfaz y las necesidades de accesibilidad en la spec.

### Dependencias y componentes

| Necesidad | Base propuesta | Qué se instala o configura |
|---|---|---|
| Ejecutar SightPause | Navegador Chromium con Manifest V3 | Extensión local; sin Node dentro del navegador |
| Interfaz | HTML, CSS y JavaScript con módulos | Sin framework, CDN ni compilación |
| Eventos en segundo plano | Service worker de la extensión | Declaración en `manifest.json` |
| Próximo aviso | `chrome.alarms` | Permiso `alarms` |
| Preferencias y estado | `chrome.storage.local` | Permiso `storage` |
| Aviso del sistema | `chrome.notifications` | Permiso `notifications` e iconos locales |
| Desarrollo con SDD | Specify, Codex, Git y Git Bash | Ya localizados en el entorno auditado |
| Pruebas de lógica | Node y `node:test` con `node:assert/strict` | Node disponible; no requiere instalar paquetes npm |
| Pruebas en navegador | Chrome; Edge como segunda comprobación | Carga local y lista de escenarios |

El [service worker puede detenerse](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle). Por ello, la arquitectura propuesta separa el estado persistente de la vista. El popup consulta el estado y solicita cambios al worker; no es el dueño del temporizador.

Las [alarmas](https://developer.chrome.com/docs/extensions/reference/api/alarms) pueden retrasarse y no despiertan el equipo. La disponibilidad de opciones nuevas de persistencia depende de la versión y del navegador. Para este ejemplo, comprueba y reconstruye la alarma necesaria al activar el worker desde el estado guardado, sin depender de opciones recientes para mantener compatibilidad. No diseñes la corrección alrededor de avisos exactos al segundo ni de alarmas rápidas que solo funcionen en carga sin empaquetar.

La [API de almacenamiento](https://developer.chrome.com/docs/extensions/reference/api/storage) conserva datos de la extensión. La [API de notificaciones](https://developer.chrome.com/docs/extensions/reference/api/notifications) requiere su permiso y opciones como título, mensaje e icono. El sistema operativo también influye en su presentación.

### Acción para preparar la especificación

```text
Revisa conmigo la tabla de propuesta funcional de docs/guia-sdd.md.
Identifica qué decisiones afectan al uso diario de SightPause y explica
sus consecuencias. Confirma especialmente pausa, cambio de intervalo,
reinicio y notificaciones bloqueadas. No añadas nuevas funciones.
```

**Resultado esperado:** una versión entendida del alcance. **Revisa:** qué has confirmado y qué sigue siendo propuesta. **Continúa o retoma:** usa las decisiones como entrada del ciclo; no instales dependencias opcionales antes de necesitarlas.

<a id="ciclo"></a>
## 7. Recorrer el primer ciclo de Spec Kit

El objetivo del primer recorrido es aprender cada fase con una funcionalidad pequeña. Las skills se ejecutan **una a una**. Sal del modo Plan de Codex cuando vayas a escribir los documentos.

```text
Idea y decisiones
       ↓
constitution → specify → clarify → plan → checklist → tasks → analyze
                                                             ↓
                                                         implement
                                                             ↓
                                                  pruebas + converge
                                                             ↓
                                             correcciones o revisión final
```

Los comandos y sus efectos se basan en las [skills instaladas](../.agents/skills/) y en el [recorrido oficial](https://github.github.com/spec-kit/quickstart.html). La constitución se establece para el proyecto; no se recrea completa en cada funcionalidad. El archivo de workflow instalado permite automatizar fases, pero este tutorial las ejecuta por separado para aprender a revisar sus resultados.

### Paso 0. Preparar un punto de partida

**Aprenderás:** a saber qué trabajo pertenece al ciclo. **Antes:** decisiones iniciales y entorno disponible. **Dónde:** PowerShell.

```powershell
Set-Location C:\dev\sightpause
git status --short --branch
git diff
```

**Resultado esperado:** identificas todos los cambios pendientes. **Revisa:** no mezclar archivos de otra tarea con el primer ciclo. Guarda la guía y otros cambios revisados según la [sección de Git](#sesiones). **Continúa o retoma:** con un punto del historial identificable; no necesitas volver a inicializar el repositorio.

### Paso 1. Constitución: principios que durarán

**Aprenderás:** a establecer reglas útiles sin sobrediseñar. **Antes:** conocer el alcance general. **Dónde:** Codex en ejecución, skill.

```text
$speckit-constitution
Establece la constitución de SightPause, una extensión Chromium de descanso
visual. Principios: interfaz mínima y accesible; funcionamiento local;
permisos justificados; dependencias mínimas; temporización resistente al cierre
del popup y a la suspensión del worker; requisitos y decisiones alineados con
el código. Exige pruebas de la lógica temporal y comprobaciones manuales de
notificaciones, persistencia y teclado. Usa lenguaje verificable y proporcional
a un MVP. Redacta en español conservando la estructura de la plantilla.
No implementes todavía la extensión.
```

**Resultado esperado:** `.specify/memory/constitution.md` con principios, gobierno y versión, sin marcadores como `[PROJECT_NAME]`. El hook Git de inicialización puede intervenir; debe reconocer el repositorio existente.

**Revisa:** que puedas explicar cada principio y comprobarlo. Un ejemplo de plantilla que habla de microservicios no es una obligación de SightPause. Una constitución que impone herramientas innecesarias debe corregirse antes de continuar.

**Continúa o retoma:** lee el archivo y pide cambios concretos si hace falta. No hace falta ejecutar de nuevo `constitution` por cada botón que añadas. Referencia local: [skill de constitución](../.agents/skills/speckit-constitution/SKILL.md).

### Paso 2. Specify: describir qué debe ocurrir

**Aprenderás:** a transformar la idea en historias y criterios. **Antes:** constitución y decisiones iniciales. **Dónde:** Codex, skill.

```text
$speckit-specify
Crear el primer recordatorio de SightPause para personas que trabajan con
un navegador Chromium. Debe avisar periódicamente de que hagan una pausa visual
sin tener que mantener abierto su panel. El popup muestra estado, tiempo hasta
el próximo aviso, intervalo y controles para iniciar, pausar y reanudar.
Usa como propuesta la tabla funcional de docs/guia-sdd.md y deja identificadas
las decisiones que todavía deba confirmar. El diseño debe ser minimalista,
legible y utilizable con teclado. Los ajustes se conservan entre sesiones.
Incluye situaciones de popup cerrado, reinicio, suspensión y avisos bloqueados.
Limita esta funcionalidad a recordatorios y controles; no añadas cuentas,
estadísticas, lectura de páginas ni publicación en una tienda.
Redacta la especificación en español, centrada en qué y por qué.
```

**Resultado esperado:** directorio de funcionalidad con `spec.md`, checklist de calidad inicial y selección de funcionalidad activa. El hook de la extensión Git crea una rama numerada antes de especificar. No ejecutes además el comando de creación de rama para la misma petición.

Un nombre como `001-visual-break-reminder` es ilustrativo; el nombre real lo determina la ejecución. Desde aquí, cuando la guía diga «directorio activo», utiliza la ruta real que devuelva el agente, no copies un nombre supuesto.

**Revisa:** historias ordenadas por valor y criterios comprobables. «Debe ser fiable» necesita ejemplos de comportamiento. «Usar React» sería una decisión técnica que no hemos elegido. Comprueba la rama y el puntero:

```powershell
git branch --show-current
Get-Content .specify/feature.json
```

**Continúa o retoma:** abre el `spec.md` indicado. Si la funcionalidad ya existe, pide revisar esa spec; no repitas `specify` sin contexto para corregir una frase, pues puedes iniciar otro directorio o rama. Referencia: [skill de especificación](../.agents/skills/speckit-specify/SKILL.md).

### Paso 3. Clarify: cerrar ambigüedades

**Aprenderás:** a tomar decisiones antes de diseñar su implementación. **Antes:** spec creada y funcionalidad activa correcta. **Dónde:** Codex, skill.

```text
$speckit-clarify
Revisa la spec activa de SightPause. Ayúdame a confirmar el intervalo y sus
límites, inicio manual, pausa y reanudación, cambio de intervalo, tratamiento
de vencimientos tras reinicio o suspensión y aviso cuando las notificaciones
no están disponibles. Explica las opciones sin jerga. Guarda las decisiones
en la spec y corrige los criterios afectados. No introduzcas otra funcionalidad.
```

**Resultado esperado:** preguntas focalizadas y respuestas incorporadas a la especificación. Puede haber un límite de preguntas por ejecución; si quedan decisiones relevantes, haz otra ronda centrada en ellas.

**Revisa:** que no haya contradicciones entre una respuesta nueva y una historia anterior. Si eliges otra política que la propuesta en la sección 6, las pruebas y el plan posteriores deben seguir la spec aprobada.

**Continúa o retoma:** cuando puedas explicar qué ocurrirá en cada situación crítica. La decisión debe quedar escrita, no solo en el chat. Referencia: [skill de aclaración](../.agents/skills/speckit-clarify/SKILL.md).

### Paso 4. Plan: decidir cómo construirlo

**Aprenderás:** a leer un diseño técnico y reconocer sus dependencias. **Antes:** spec aclarada y constitución real. **Dónde:** Codex en ejecución, skill; no modo Plan de la aplicación.

```text
$speckit-plan
Diseña la funcionalidad activa de SightPause con Manifest V3, HTML, CSS y
JavaScript modular, sin framework ni compilación. Propón extension/ como
carpeta cargable y tests/ para pruebas. Usa un service worker como responsable
del estado, chrome.alarms para vencimientos y chrome.storage.local para
persistencia. El popup consulta el estado y solicita cambios mediante mensajes.
Permisos previstos: alarms, storage y notifications; justifica cualquier cambio.
No dependas de un setInterval en el worker ni de que el popup siga abierto.
Documenta estados, fechas de vencimiento y recuperación según la spec aprobada.
Define mensajes entre popup y worker y el comportamiento de la interfaz.
Usa node:test y node:assert/strict para lógica temporal con reloj controlable,
sin dependencias npm. Explica si necesitas package.json para módulos.
Incluye iconos locales y validación manual en Chrome; segunda comprobación
en Edge si está disponible. Establece y justifica la versión mínima compatible.
Redacta los documentos en español y el quickstart con comandos ejecutables.
```

**Resultado esperado:** en el directorio activo, `plan.md`, `research.md`, `data-model.md`, contratos de las interfaces que lo requieran y `quickstart.md`. Todavía no se crea `tasks.md` en esta fase.

| Documento | Qué debes poder encontrar |
|---|---|
| `plan.md` | Componentes, dependencias, estructura y comprobación de principios |
| `research.md` | Decisiones técnicas, razones y alternativas descartadas |
| `data-model.md` | Preferencias y estado temporal, con transiciones coherentes con la spec |
| `contracts/` | Acuerdos para mensajes internos y/o interfaz del popup, según el diseño |
| `quickstart.md` | Cómo cargar, ejecutar pruebas y verificar escenarios; no una implementación completa |

**Revisa:** un solo responsable del temporizador, persistencia antes de perder memoria, eventos repetidos sin duplicar avisos y ausencia de permisos para leer páginas. El contador visual puede actualizarse mientras el popup está abierto, calculándose desde el estado temporal; eso no sustituye a la alarma de fondo.

Si aparece una arquitectura con servidor, autenticación o varias bibliotecas de interfaz, pide justificarla frente al MVP. Si quedan decisiones relevantes sin resolver, acláralas antes de generar tareas.

**Continúa o retoma:** con el diseño comprendido. La [skill `speckit-plan`](../.agents/skills/speckit-plan/SKILL.md) necesita una spec previa y termina tras el diseño. Si muestra `Feature directory not found`, vuelve al paso de selección o creación de funcionalidad.

### Paso 5. Checklist: revisar la calidad de los requisitos

**Aprenderás:** a distinguir revisión documental de pruebas del producto. **Antes:** spec y diseño disponibles. **Dónde:** Codex, skill, y revisión del Markdown.

```text
$speckit-checklist
Crea una checklist breve para revisar si los requisitos de SightPause definen
sin ambigüedad la temporización, pausa, recuperación, notificaciones,
privacidad y accesibilidad. Revisa calidad y cobertura de requisitos;
no la presentes como evidencia de que el código ya funciona.
```

**Resultado esperado:** una checklist personalizada dentro de `checklists/` del directorio activo; usa el nombre que genere la skill.

**Revisa:** responde cada punto leyendo los documentos y anota dónde está la evidencia. Por ejemplo, «¿se especifica qué pasa tras reiniciar?» se satisface cuando la spec lo define con claridad, aunque no exista todavía el código.

Las checklists personalizadas son del revisor: marca `[x]` al comprobar ese criterio documental, o pide al agente registrar los puntos que tú has revisado. `checklists/requirements.md` es una checklist inicial mantenida por `specify` y `clarify`; no confundas su papel con el de tus pruebas.

**Continúa o retoma:** corrige primero los requisitos incompletos. La [skill de implementación](../.agents/skills/speckit-implement/SKILL.md) puede detenerse si encuentra casillas pendientes. No las marques todas para saltar ese control. Referencia: [skill de checklist](../.agents/skills/speckit-checklist/SKILL.md).

### Paso 6. Tasks: convertir decisiones en trabajo

**Aprenderás:** a leer el orden de ejecución y la cobertura. **Antes:** spec y plan revisados. **Dónde:** Codex, skill.

```text
$speckit-tasks
Genera las tareas de la funcionalidad activa desde su spec y plan. Incluye
explícitamente pruebas automatizadas de cálculo de vencimiento, pausa,
reanudar, recuperación y prevención de avisos duplicados, con reloj controlable.
Incluye pruebas manuales de notificación, popup cerrado, reinicio y teclado.
Organiza por historias de usuario y dependencias, empezando por un recorrido
vertical que entregue un aviso con estado persistente y control desde el popup.
Incluye preparación de iconos y manifest, revisión visual y guía de carga local.
Usaremos un agente; conserva identificadores claros para revisar el progreso.
```

**Resultado esperado:** `tasks.md` con tareas identificadas, orden y rutas. Solicitamos pruebas expresamente porque la [skill de tareas](../.agents/skills/speckit-tasks/SKILL.md) no las genera obligatoriamente en todos los casos.

Una tarea ilustrativa podría ser `- [ ] T012 [US1] Verificar persistencia del vencimiento al reabrir el popup`. El número y formato final los establece la skill. Una marca `[P]` identifica oportunidades de trabajo independiente; una dependencia debe completarse antes de consumir su resultado.

**Revisa:** cada requisito tiene trabajo y verificación asociados; ninguna tarea dice únicamente «hacer el frontend». Las tareas de base habilitan historias, pero crear un manifest vacío no entrega por sí solo valor al usuario.

**Continúa o retoma:** usa los identificadores reales para pedir cambios. Conserva los ya completados cuando hagas modificaciones posteriores.

### Paso 7. Analyze: encontrar contradicciones antes de programar

**Aprenderás:** a contrastar los tres documentos principales. **Antes:** `spec.md`, `plan.md` y `tasks.md`. **Dónde:** Codex, skill.

```text
$speckit-analyze
Revisa la consistencia de la funcionalidad activa. Prioriza requisitos sin
tareas o pruebas, contradicciones en pausa y recuperación, permisos excesivos
y decisiones que amplíen el MVP. Presenta hallazgos con referencias concretas.
```

**Resultado esperado:** informe de problemas, no una reparación automática de todos los archivos. La [skill de análisis](../.agents/skills/speckit-analyze/SKILL.md) realiza una revisión no destructiva de los artefactos.

**Revisa:** si un hallazgo cambia el comportamiento, decide qué quieres antes de pedir la corrección. Después utiliza una petición normal de edición:

```text
Corrige los hallazgos aceptados del análisis anterior en los documentos de
la funcionalidad activa. Conserva su alcance e identificadores de tareas.
Explícame el diff y no implementes todavía código de la extensión.
```

**Continúa o retoma:** vuelve a analizar los documentos afectados y resuelve los problemas críticos. Una segunda revisión sin hallazgos no equivale a una prueba del producto.

### Paso 8. Implement: construir y verificar

**Aprenderás:** a encargar trabajo contra documentos y comprobar progreso. **Antes:** documentos coherentes, checklists revisadas y modo de ejecución. **Dónde:** Codex, skill.

```text
$speckit-implement
Implementa las tareas de la funcionalidad activa siguiendo su spec y plan.
Ejecuta las pruebas previstas y verifica cada historia antes de marcarla
como terminada. Mantén la estructura y dependencias acordadas y no añadas
funciones nuevas. Diferencia pruebas ejecutadas, fallidas y manuales pendientes.
Si hay una decisión de producto sin resolver, describe su impacto antes de
asumirla. Al terminar, indica archivos cambiados, comandos de prueba y cómo
cargar la extensión para revisar su funcionamiento y aspecto.
```

**Resultado esperado:** código de la extensión y pruebas según el plan, con progreso en `tasks.md`. La skill procesa las tareas de esa funcionalidad; mantén por eso el alcance inicial pequeño.

**Revisa:** diff, comandos reales y resultado de pruebas. Si un test falla, pide reproducir el fallo y corregir su causa. No aceptes eliminar el test solo para obtener una salida verde.

**Continúa o retoma:** ejecuta la [validación del navegador](#pruebas). Si se interrumpe, utiliza el prompt de recuperación y continúa sobre las tareas existentes. Referencia: [skill de implementación](../.agents/skills/speckit-implement/SKILL.md).

### Paso 9. Converge: localizar trabajo todavía sin completar

**Aprenderás:** a comprobar cobertura de lo construido. **Antes:** disponer de `spec.md`, `plan.md` y `tasks.md`, y haber ejecutado implementación sobre esas tareas. **Dónde:** Codex, skill.

```text
$speckit-converge
Compara el código actual de SightPause con la spec, el plan y las tareas de
la funcionalidad activa. Identifica requisitos o escenarios aún incompletos.
Añade trabajo pendiente trazable sin ampliar el alcance ni tratar pruebas
manuales no ejecutadas como pruebas superadas.
```

**Resultado esperado:** evaluación de cobertura y, si procede, nuevas tareas al final de `tasks.md`. La [skill de convergencia](../.agents/skills/speckit-converge/SKILL.md) contrasta el estado actual con los artefactos; no es un comparador de ramas ni sustituye las pruebas.

**Revisa:** las tareas añadidas deben corresponder a requisitos existentes. Una nueva idea de producto va al siguiente ciclo. **Continúa o retoma:** si hay trabajo, ejecuta `implement` y verifica de nuevo; si no, comprueba las pruebas reales y prepara el commit. No repitas ciclos sin leer qué falta ni qué evidencia ha cambiado.

<a id="pruebas"></a>
## 8. Probar la extensión y decidir si está terminada

**Aprenderás:** a verificar comportamiento, integración y aspecto. **Antes:** implementación disponible y `quickstart.md` de la funcionalidad. **Dónde:** terminal y navegador.

### Pruebas automatizadas

El diseño propuesto utiliza el [ejecutor de pruebas de Node](https://nodejs.org/api/test.html). Si el agente lo ha implementado según el plan, desde la raíz:

```powershell
node --version
node --test
```

No ejecutes este bloque como validación del producto antes de que existan las pruebas. Revisa cuántas se descubrieron y qué escenarios cubren: una ejecución con cero pruebas no verifica SightPause. Si el plan revisado eligió otro comando, `quickstart.md` debe indicar el comando real y por qué cambió.

El reloj controlable permite probar «han pasado 20 minutos» sin esperar 20 minutos reales. Comprueba cálculos y transiciones en funciones separadas de las APIs del navegador. Las simulaciones de `chrome.*` ayudan a probar integración lógica, pero no demuestran que Windows presente una notificación.

### Carga local y actualización durante el desarrollo

La carpeta `extension/` es la propuesta de esta guía. Si el plan final utiliza otra, selecciona la que contenga su `manifest.json`.

1. Abre Chrome y escribe `chrome://extensions` en la barra de direcciones.
2. Activa **Modo de desarrollador**.
3. Pulsa **Cargar descomprimida / Load unpacked** y selecciona `C:\dev\sightpause\extension`.
4. Revisa si la tarjeta muestra errores. Fija el icono de SightPause en la barra y abre su popup.
5. Después de cambiar el código, recarga la extensión desde su tarjeta y vuelve a abrir el popup.

La [guía de Chrome para una primera extensión](https://developer.chrome.com/docs/extensions/get-started/tutorial/hello-world) explica este procedimiento. Para la comprobación secundaria en Edge, utiliza `edge://extensions` y sus controles equivalentes; registra la versión y cualquier diferencia. Una comprobación en Chrome no acredita por sí sola todos los navegadores Chromium.

### Matriz de aceptación manual

Estos resultados siguen los supuestos de la sección 6. Si `clarify` estableció otros, actualiza la matriz de tu `quickstart.md` para que coincida con la spec aprobada.

| Escenario | Acción | Resultado esperado |
|---|---|---|
| Primera instalación | Abrir el popup sin iniciar | Estado inicial claro; no se emiten avisos todavía |
| Iniciar | Activar con un intervalo breve permitido | Aparece un próximo vencimiento y se recibe el aviso cuando el entorno lo permite |
| Popup cerrado | Cerrar el panel antes del vencimiento | El aviso no requiere que el panel siga abierto |
| Reabrir | Abrir antes del vencimiento | Conserva la hora prevista; no empieza un intervalo nuevo |
| Pausar | Pausar antes del vencimiento y esperar | No llega el aviso mientras está pausado |
| Reanudar | Reanudar una pausa | Usa el tiempo restante acordado, sin alarmas duplicadas |
| Cambiar intervalo | Guardar otro intervalo activo y luego pausado | Aplica la política definida para cada estado |
| Entrada inválida | Vacío, decimal, 0 o valor fuera del límite | Explica el error sin corromper la preferencia anterior |
| Reinicio | Cerrar todos los procesos del navegador y abrirlo | Recupera preferencias y estado según la spec; no acumula avisos |
| Suspensión del equipo | Suspender más allá del vencimiento y volver | Procesa como máximo un vencimiento atrasado y programa el siguiente |
| Worker detenido | Dejarlo inactivo y reabrir el popup | Reconstruye estado y alarma si hace falta, sin reiniciar indebidamente el ciclo |
| Avisos bloqueados | Deshabilitar notificaciones y provocar un vencimiento | Controles operativos; explicación si se detecta bloqueo/error; registrar lo que el sistema oculta |
| Teclado | Navegar con Tab, Shift+Tab, Enter o Espacio según el control | Orden comprensible, foco visible y controles operables |
| Diseño | Revisar textos largos, error y distintos estados | Sin recortes; jerarquía clara y contraste suficiente |
| Privacidad | Revisar permisos y actividad de red de la extensión | Solo permisos acordados; funcionamiento sin servicios remotos |

Para ensayar notificaciones usa, por ejemplo, un intervalo permitido de un minuto; restaura después el valor normal. No añadas un modo de desarrollo permanente al producto solo para acelerar la prueba.

Al comprobar la suspensión real del worker, cierra sus DevTools: inspeccionarlo puede alterar su ciclo de vida. Documenta las condiciones de la prueba. Si el navegador conserva procesos en segundo plano, cerrar una ventana no prueba necesariamente un reinicio completo.

Registra resultados en el `quickstart.md` activo con este formato:

```text
Escenario:
Fecha, navegador y versión:
Estado previo y pasos:
Resultado esperado según la spec:
Resultado observado:
Estado: PASA / FALLA / PENDIENTE
Evidencia o incidencia:
```

**Resultado esperado:** sabes qué funciona y qué no se ha comprobado. **Revisa:** un resultado pendiente nunca se presenta como pasado. **Continúa o retoma:** corrige fallos, vuelve a probar lo afectado y termina cuando se cumplan los criterios acordados. Si necesitas tu intervención para observar un aviso, el agente debe indicarlo.

<a id="iteraciones"></a>
## 9. Iterar: error, cambio funcional, mejora visual y nueva función

**Aprenderás:** a elegir el tamaño del proceso para cada cambio. **Antes:** primera funcionalidad implementada o un comportamiento reproducible. **Dónde:** Codex, artefactos y pruebas.

La política de este tutorial es conservar los documentos de una funcionalidad mientras la corriges y crear otra cuando introduces una capacidad nueva con criterios propios. Es una convención de trabajo, no una imposición universal de SDD.

| Tipo de cambio | Documentos | Recorrido recomendado |
|---|---|---|
| El código incumple un requisito vigente | Revisar spec; añadir tarea de corrección y evidencia | Reproducir → corregir → prueba de regresión |
| Cambia el comportamiento deseado | Actualizar spec y criterios, después plan y tareas afectados | Aclarar cambio → alinear documentos → analizar → implementar |
| Ajuste visual sin cambio de comportamiento | Plan/contrato visual y tarea si procede | Ajustar → revisar visualmente y accesibilidad |
| Nueva capacidad con alcance propio | Nueva spec y documentos asociados | Nuevo ciclo; conservar referencias a la funcionalidad anterior |

### Ejercicio A. Error: el contador se reinicia al cerrar el popup

**Antes:** la spec dice que debe conservarse el vencimiento. **Acción — Codex, ejecución:**

```text
Hay un error en SightPause: al cerrar y reabrir el popup, el contador vuelve
al intervalo completo. Localiza la funcionalidad de recordatorios y lee su spec,
plan y tareas. Reproduce el problema y comprueba el requisito de persistencia.
Si el comportamiento esperado está claro, añade una tarea de corrección con
identificador nuevo, corrige la causa y añade una prueba de regresión.
No cambies el requisito para justificar el error. Ejecuta las pruebas y registra
la comprobación de cerrar/reabrir, diferenciando pruebas manuales pendientes.
```

**Resultado esperado:** misma funcionalidad y requisito, nueva tarea trazable y corrección. **Revisa:** la spec solo cambia si era ambigua; el plan se ajusta si la solución técnica real lo requiere. **Pruebas:** cerrar/reabrir, vencer con popup cerrado, pausa y recuperación del worker. **Continúa o retoma:** confirma el fallo corregido y guarda un commit; no recrees todo el proyecto.

### Ejercicio B. Cambio funcional: reanudar con un intervalo completo

**Antes:** la spec inicial conserva el tiempo restante al reanudar. Ahora decides cambiarlo. **Acción — Codex, primero propuesta:**

```text
Quiero cambiar SightPause: al reanudar una pausa debe empezar un intervalo
completo, en vez de conservar el tiempo restante. Lee los artefactos de la
funcionalidad de recordatorios. Propón el cambio de requisitos y criterios,
su impacto en estado guardado, interfaz y pruebas. No implementes aún.
```

Tras revisar la propuesta, **Codex en ejecución:**

```text
Aplica la decisión revisada de reanudar con un intervalo completo a la spec,
plan, modelo de datos, contratos y quickstart donde corresponda. Conserva el
historial de tareas completadas y añade las tareas necesarias con nuevos IDs.
Define qué ocurre con una pausa guardada por la versión anterior. No cambies
el código hasta comprobar la consistencia de los documentos.
```

**Resultado esperado:** la misma funcionalidad evoluciona mediante un cambio explícito. **Revisa:** criterios antiguos incompatibles actualizados y cualquier transición de datos definida. **Pruebas:** nueva reanudación, pausa persistente tras reinicio, cambio de intervalo pausado y ausencia de avisos durante la pausa. **Continúa o retoma:** ejecuta `analyze`, resuelve hallazgos, después `implement` y validación. No ejecutes `tasks` desde cero para borrar el progreso.

### Ejercicio C. Mejora visual: espaciado y estados de botones

**Antes:** comportamiento correcto y un problema visual concreto. **Acción — Codex, ejecución:**

```text
Mejora la presentación del popup de SightPause: da más separación entre el
contador y los controles, unifica la altura de botones y haz visible el foco
del teclado. Lee primero el contrato visual y los requisitos de accesibilidad.
Conserva textos, controles, temporización y permisos. Usa el CSS existente.
Actualiza el contrato o plan visual si cambia una decisión documentada y añade
una tarea de mejora sin regenerar las tareas anteriores. Revisa los estados
inicial, activo, pausado y error, y explica cómo comprobaste el resultado.
```

**Resultado esperado:** ajuste pequeño en la misma funcionalidad. **Revisa:** el diff no cambia eventos del temporizador ni añade bibliotecas. **Pruebas:** inspección visual, teclado, estados y un recorrido básico de controles; no hacen falta tests que repitan cada regla CSS. **Continúa o retoma:** compara el aspecto antes/después. Si introduces un control o comportamiento nuevo, actualiza también sus requisitos.

### Ejercicio D. Nueva funcionalidad: modo oscuro

**Antes:** primera funcionalidad guardada y revisada. Vuelve a la rama base que contiene ese trabajo; sigue la sección de Git antes de abrir el siguiente ciclo. **Acción — Codex en ejecución, skill:**

```text
$speckit-specify
Añadir a SightPause una preferencia de apariencia con opciones Sistema,
Claro y Oscuro. Debe conservarse entre sesiones, respetar la elección del
usuario y mantener legibles todos los estados del popup. No debe modificar
temporización, controles ni permisos de los recordatorios. Reutiliza la base
existente y referencia la funcionalidad de recordatorios como dependencia.
Redacta en español historias y criterios que permitan comprobar la apariencia
y su persistencia. Mantén fuera del alcance los temas personalizados.
```

**Resultado esperado:** nueva rama y directorio numerados, con spec propia; no se elimina la anterior. **Revisa:** comportamiento de Sistema al cambiar la preferencia del sistema, valor inicial y persistencia. **Pruebas:** tres opciones, reinicio, contraste, teclado y regresión de recordatorios.

**Continúa o retoma:** usa `clarify` para cerrar esas decisiones; `plan` para reutilizar CSS; `tasks`, `analyze`, `implement` y `converge` según el mismo procedimiento. No reescribas la constitución completa salvo que cambien principios del proyecto.

<a id="sesiones"></a>
## 10. Git, contexto y recuperación de sesiones

**Aprenderás:** a guardar avances y retomar la funcionalidad correcta. **Antes:** entender el diff y el directorio activo. **Dónde:** PowerShell y Codex.

### Qué hace la extensión Git instalada

La configuración vive en [extensions.yml](../.specify/extensions.yml) y [git-config.yml](../.specify/extensions/git/git-config.yml). Sus cinco skills son inicialización, creación de rama, validación de rama, detección de remoto y commit. El [README de la extensión](../.specify/extensions/git/README.md) describe su uso.

En el estado auditado, los hooks de inicialización y creación de rama están registrados como obligatorios; los hooks de commit son opcionales. Además, `auto_commit.default` y los eventos concretos están desactivados. Que un hook figure como `enabled: true` en el registro no significa que el comando vaya a crear un commit: el comando consulta su propia configuración.

Si activaras el auto-commit, la skill puede preparar todos los cambios mediante `git add .`. Este tutorial conserva commits manuales para aprender qué estás guardando. No hace falta ejecutar las skills Git por separado si el hook correspondiente ya interviene.

### Revisar y guardar un avance

```powershell
git status --short --branch
git diff
```

Para guardar esta guía, por ejemplo:

```powershell
git add -- docs/guia-sdd.md
git diff --cached
git commit -m "docs: add SDD learning guide"
```

Para el código, selecciona las rutas reales que has revisado. `git diff` no muestra por sí solo el contenido de archivos nuevos sin seguimiento; ábrelos antes de prepararlos. `git diff --cached` muestra lo preparado para el próximo commit. No incluyas archivos ajenos al avance.

Los commits locales guardan historial; `push` lo envía al remoto. Puedes aprender todo el primer ciclo localmente. Si quieres revisión en GitHub, publica la rama y abre una pull request con problema, cambio y pruebas. Antes del siguiente ciclo, incorpora el trabajo revisado a la rama base mediante tu flujo Git, cambia a esa base y verifica que contiene la extensión. No basta con volver a una `main` que todavía no incluye la primera funcionalidad.

### La rama Git y la funcionalidad activa son estados distintos

Spec Kit utiliza `.specify/feature.json`, que aquí está ignorado por Git, y permite una selección explícita mediante `SPECIFY_FEATURE_DIRECTORY`. Cambiar de rama con Git no cambia automáticamente ese puntero. Así lo documenta la [guía oficial](https://github.github.com/spec-kit/quickstart.html).

Consulta antes de continuar:

```powershell
git branch --show-current
Get-ChildItem specs -Directory
Get-Content .specify/feature.json
Get-Item Env:SPECIFY_FEATURE_DIRECTORY -ErrorAction SilentlyContinue
```

Antes de la primera spec es normal que las rutas no existan. Después, una variable de entorno antigua puede apuntar a otra funcionalidad; no ignores una discrepancia. Las variables que defines en una terminal no se propagan necesariamente a otra sesión de Codex ya abierta.

Para cambiar de funcionalidad de manera explícita, pide al agente:

```text
Quiero retomar la funcionalidad de recordatorios de SightPause. Examina las
ramas y specs existentes y mi estado de Git. Identifica el directorio correcto
y comprueba .specify/feature.json y SPECIFY_FEATURE_DIRECTORY. Si la elección
es inequívoca, selecciona la rama apropiada sin perder cambios pendientes y
actualiza el puntero local a ese directorio existente. Si hay varias candidatas,
muéstramelas antes de elegir. No crees una nueva spec ni implementes nada aún.
```

### Prompt completo para retomar otra sesión o modelo

```text
Estoy retomando SightPause. Lee docs/guia-sdd.md como guía y consulta la
constitución y los artefactos de la funcionalidad activa como requisitos reales.
Comprueba rama, cambios pendientes y puntero de funcionalidad antes de actuar.
Resume: alcance aprobado, tareas completadas, pendientes y pruebas ejecutadas
o por realizar. Contrasta las casillas con el código; no des por hecho que todo
está validado. Indica el siguiente paso de SDD y no regeneres documentos ni
edites código hasta que hayamos confirmado que retomas el trabajo correcto.
```

**Resultado esperado:** resumen breve con rutas y siguiente paso. **Revisa:** que las decisiones vengan de los documentos actuales, no de los ejemplos de esta guía. **Continúa:** pide la fase o tarea pendiente con su identificador real.

Si necesitas un resumen al terminar una sesión, pide guardarlo dentro del directorio activo, por ejemplo en `session-notes.md`, con decisiones recientes, siguiente tarea y pruebas pendientes. Es una ayuda opcional de este tutorial; no sustituye `spec.md`, `plan.md` ni `tasks.md`. `AGENTS.md`, si se añade en el futuro, debe contener instrucciones estables para el agente, no duplicar toda la especificación.

<a id="experiencias"></a>
## 11. Qué recomiendan usuarios de Spec Kit

**Aprenderás:** a aprovechar experiencias ajenas sin confundirlas con capacidades garantizadas. **Antes:** haber leído el ciclo. **Dónde:** lectura de fuentes y revisión de tu forma de trabajo.

Las siguientes personas describen su propio uso. Son testimonios, no un estudio comparativo de productividad. Todos los enlaces se consultaron el **07/09/2026**. Para Reddit, la página mostraba antigüedades relativas; no se ha verificado una fecha exacta de publicación.

| Fuente y fecha visible | Experiencia que relata | Aplicación propuesta en SightPause |
|---|---|---|
| **Organic_Ad1162**, publicación con «4mo ago», [conversación sobre Spec Kit](https://www.reddit.com/r/vibecoding/comments/1sykoln/i_cannot_say_enough_good_things_about_githubs/) | Recomienda preparar la idea, ejecutar aclaraciones, analizar después de las tareas y guardar avances pequeños. | Conversar sobre pausa y reinicio antes del plan; comprobar cobertura antes del código; revisar commits por avance. |
| **jokiruiz**, comentario con «3mo ago», [misma conversación](https://www.reddit.com/r/vibecoding/comments/1sykoln/i_cannot_say_enough_good_things_about_githubs/) | Destaca leer la arquitectura propuesta y relata desajustes al cambiar código sin actualizar la spec. | Rechazar un backend innecesario en `plan`; registrar el cambio de política de pausa antes de implementarlo. |
| **h____**, comentario con «4mo ago», [misma conversación](https://www.reddit.com/r/vibecoding/comments/1sykoln/i_cannot_say_enough_good_things_about_githubs/) | Prefiere ajustar el grado de estructura al tamaño y riesgo del cambio. | Para espaciado CSS, revisar el cambio y sus estados sin recrear toda la documentación. |
| **orapha.dev**, artículo del **03/04/2026**, [experiencia con SDD y Spec Kit](https://orapha.dev/2026/04/03/como-eu-tenho-usado-spec-driven-development-com-o-spec-kit-nos-meus-projetos/) | Señala que el proceso puede pesar en cambios pequeños, el agente puede complicar la solución y las tareas terminadas no garantizan una buena experiencia. | Mantener el MVP pequeño, revisar dependencias y probar manualmente el popup y los avisos. |

**Qué adoptamos:** aclaración temprana, revisión de decisiones, cambios pequeños y validación real. **Qué no convertimos en regla universal:** comentarios descriptivos en todo el código, porcentajes arbitrarios de cobertura, muchos agentes o un ciclo completo para cada ajuste. Son decisiones que deben justificarse por el proyecto.

**Ejercicio — Codex, conversación:**

```text
Usa la sección de experiencias de docs/guia-sdd.md para revisar nuestro flujo.
Señala una práctica que nos ayude y una que sería excesiva para SightPause.
Distingue testimonio, documentación oficial y recomendación tuya. No cambies
el proyecto ni instales herramientas a partir de un comentario de comunidad.
```

**Resultado esperado:** decisiones razonadas. **Revisa:** los comandos de una experiencia antigua o de un fork pueden no existir en nuestra instalación. **Continúa o retoma:** conserva solo las prácticas que ayuden a cumplir requisitos o a revisar mejor el resultado.

<a id="problemas"></a>
## 12. Resolver problemas frecuentes

**Aprenderás:** a localizar la capa que falla. **Antes:** copia el comando y error reales, sin datos sensibles. **Dónde:** terminal para diagnóstico; Codex para interpretar o corregir.

| Síntoma | Qué comprobar | Siguiente acción |
|---|---|---|
| «No existe `$speckit-specify`» | ¿Lo pegaste en PowerShell? ¿La skill está en `.agents/skills/`? | Invócala en Codex; consulta integración y recarga sesión si corresponde. |
| El agente solo entrega un plan | ¿Sigue activo el modo Plan? | Cambia a ejecución antes de pedir creación de archivos. |
| `$speckit-plan` no encuentra spec | Puntero activo y existencia de `spec.md` | Ejecuta `specify` primero o selecciona una spec existente. |
| Hay dos funcionalidades parecidas | Se repitió la creación en vez de actualizar | Identifica cuál contiene el trabajo y alinea su puntero; no borres directorios sin revisar. |
| Cambiaste de rama pero usa otra spec | `.specify/feature.json` y variable de entorno | Selecciona explícitamente la funcionalidad correcta. |
| `bash` falla o intenta abrir WSL | Ruta del ejecutable | Usa Git Bash mediante su ruta explícita. |
| Errores `\r` o `bad interpreter` | Finales de línea de los scripts | Pide normalización LF solo de los scripts afectados y revisa el diff; trata la corrección del entorno como cambio separado. |
| Spec Kit ve modificaciones y Git no | Hashes del manifiesto frente a CRLF/LF | Repite el diagnóstico de la sección 4 antes de considerar `--force`. |
| `analyze` o `converge` indica que falta `spec.md` | Directorio de funcionalidad activa y sus tres documentos | Recupera la spec correcta o completa la fase pendiente; 1.0.4 comprueba su existencia explícitamente. |
| `setup-plan.sh` devuelve `Unknown option` | Argumentos del comando | Usa `--json` según la skill; 1.0.4 rechaza argumentos desconocidos. La descripción y las instrucciones van en el prompt de la skill. |
| `implement` pregunta por checklists | Casillas pendientes y su significado | Revisa los requisitos y documenta evidencia; no marques que el código funciona por cerrar una checklist. |
| No hay commits tras una fase | Configuración de auto-commit y hooks | En esta guía son manuales; revisa y guarda el avance. |
| No hay npm | ¿El plan realmente necesita paquetes? | El ejemplo usa APIs nativas y `node:test`; npm no es requisito de ejecución. |
| Pruebas verdes, cero tests | Descubrimiento y existencia de pruebas | Corrige el comando o incorpora las pruebas acordadas; no lo registres como validación. |
| La notificación no se ve | Permiso, icono, errores, preferencias del navegador y del sistema | Separa fallo de API de presentación del sistema; anota condiciones y no inventes resultados. |
| El temporizador solo funciona abierto | Responsabilidad del popup y worker | Reproduce el error y revisa persistencia y alarmas. |
| Una recarga no muestra los cambios | Carpeta cargada, botón de recarga y popup anterior | Recarga desde extensiones, cierra y abre el panel y comprueba la ruta. |
| El agente añade funciones no pedidas | Spec, plan y tareas vigentes | Pide volver al alcance y proponer esas ideas por separado. |

Prompt de diagnóstico:

```text
Estoy siguiendo docs/guia-sdd.md y este paso falló. Te proporcionaré el comando
exacto y su salida. Comprueba primero el entorno y los archivos relevantes.
Explica la causa probable y la comprobación mínima para confirmarla. No
reinicialices el proyecto ni fuerces actualizaciones para ocultar el error.
```

**Resultado esperado:** causa localizada y siguiente acción concreta. **Revisa:** que un problema de scripts no se confunda con un error del producto. **Continúa o retoma:** vuelve al paso original después de comprobar la corrección; no repitas fases que ya produjeron documentos válidos.

<a id="fuentes"></a>
## 13. Comprobar lo aprendido y consultar las fuentes

### Checklist de aprendizaje

Marca estos puntos cuando puedas demostrarlos. Esta es una lista del tutorial, no la checklist de calidad de una funcionalidad.

- [ ] Puedo explicar la diferencia entre SDD, Spec Kit, Git, GitHub y el agente.
- [ ] Distingo CLI instalada, integración del proyecto y extensión Git.
- [ ] Sé qué comandos consultan y cuáles modifican herramientas o archivos.
- [ ] Puedo elegir conversación, modo Plan, ejecución u objetivo persistente.
- [ ] Sé que `$speckit-plan` guarda diseño y necesita una spec previa.
- [ ] Puedo escribir una historia y al menos dos criterios observables.
- [ ] Puedo localizar la constitución y la spec, plan y tareas de mi funcionalidad.
- [ ] He comprobado que cada requisito relevante tiene trabajo y verificación.
- [ ] Distingo checklists de requisitos, tareas completadas y pruebas ejecutadas.
- [ ] Puedo cargar la extensión y comprobarla con el popup cerrado.
- [ ] He registrado pausa, persistencia, reinicio, suspensión y notificaciones bloqueadas como PASA, FALLA o PENDIENTE.
- [ ] Puedo corregir un error sin alterar el requisito para hacerlo desaparecer.
- [ ] Puedo cambiar un requisito y actualizar sus documentos y pruebas.
- [ ] Puedo retomar otra sesión sin apuntar a la funcionalidad incorrecta.
- [ ] He realizado una segunda iteración y revisado su diff y evidencia.

### Fuentes y cómo interpretarlas

**Fuentes oficiales de herramientas:**

- [Inicio de Spec Kit](https://github.github.com/spec-kit/quickstart.html): fases, invocaciones y funcionalidad activa.
- [Actualización de Spec Kit](https://github.github.com/spec-kit/upgrade.html): actualización de CLI, integración y extensiones.
- [Extensiones de Spec Kit](https://github.github.com/spec-kit/reference/extensions.html): registro y configuración.
- [Prompting en Codex](https://learn.chatgpt.com/docs/prompting): contexto, verificación y planificación.
- [Objetivos de Codex](https://developers.openai.com/cookbook/examples/codex/using_goals_in_codex): objetivos persistentes y controles.
- [Primera extensión Chrome](https://developer.chrome.com/docs/extensions/get-started/tutorial/hello-world): carga local.
- [Ciclo de vida del worker](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle): suspensión y persistencia.
- [Alarmas](https://developer.chrome.com/docs/extensions/reference/api/alarms), [almacenamiento](https://developer.chrome.com/docs/extensions/reference/api/storage) y [notificaciones](https://developer.chrome.com/docs/extensions/reference/api/notifications): APIs de la extensión.
- [Pruebas integradas de Node](https://nodejs.org/api/test.html): ejecución de pruebas sin una biblioteca adicional.

**Evidencia local:** metadatos de `.specify/`, configuración de la extensión Git, ayuda de Specify y skills enlazadas en cada fase. Estos archivos explican el comportamiento de la instalación auditada; pueden diferir de documentación más reciente.

**Experiencias:** los autores y enlaces de la sección 11 son observaciones personales con aplicación propuesta al tutorial. Los prompts de SightPause, la política de iteración y las decisiones iniciales de producto son ejemplos propios de esta guía. Los requisitos definitivos se guardarán en la constitución y los artefactos de cada funcionalidad cuando recorras el proceso.

Esta revisión actualiza el estado local y las indicaciones afectadas por Spec Kit 1.0.4; conserva la fecha de consulta de las fuentes externas. La actualización y las comprobaciones de infraestructura ejecutadas se detallan en la sección 4. Los demás bloques y ejercicios son instrucciones para el lector; su presencia aquí no acredita que se hayan ejecutado ni que SightPause esté construido.
