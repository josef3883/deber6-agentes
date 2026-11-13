# Equipo de agentes

## Parte 1 — Equipo y flujo

| Agente | Misión | Herramientas autorizadas | Entrada | Salida | Límites |
|---|---|---|---|---|---|
| Orquestador | Repartir en paralelo la revisión y consolidar las respuestas | Delegación a agentes; no inspecciona ni modifica el archivo por cuenta propia | `usuarios.py` | Tabla consolidada de hallazgos y casos de prueba | No modifica archivos ni ejecuta cambios sugeridos |
| Especialista 1 — Seguridad | Buscar claves/tokens escritos en el código y entradas de usuario sin validar | Leer y buscar | Ruta exacta: `usuarios.py` | Lista de hallazgos con línea, problema y severidad | Solo revisa `usuarios.py`; no escribe ni modifica |
| Especialista 2 — Pruebas | Proponer 3 casos de prueba que faltan | Leer | Ruta exacta: `usuarios.py` | Lista de 3 casos con nombre y qué verifica | Solo propone; no escribe ni ejecuta pruebas |

### Flujo y checkpoint humano

```text
                         ┌─> Especialista 1 — Seguridad ─┐
Objetivo + usuarios.py ─> Orquestador                     ├─> Consolidar tabla
                         └─> Especialista 2 — Pruebas ───┘
                                                            │
                                      checkpoint humano <───┘
                                                            │
                                  aprobar/rechazar/revisar
                                                            │
                                    ejecución posterior
```

Los especialistas trabajan en paralelo porque sus revisiones son independientes. El checkpoint ocurre después de consolidar y antes de aprobar cualquier cambio derivado de las recomendaciones. En este deber no se modifica `usuarios.py`.

### Ejemplo de consolidación para `usuarios.py`

Los hallazgos de seguridad se contrastan con el archivo; los casos de prueba son propuestas, no pruebas ejecutadas.

| Especialista | Línea/caso | Resultado |
|---|---|---|
| Seguridad | 10 | Constante con apariencia de clave API escrita en el código; el archivo la identifica como ejemplo. |
| Seguridad | 11, 42 | Contraseña de administrador fija en código y comparada directamente con el valor recibido. |
| Seguridad | 21 | `buscar_usuario` concatena el nombre recibido en la consulta SQL, sin validación ni parámetros. |
| Seguridad | 30–34 | `crear_usuario` parametriza la consulta, pero no valida datos como edad o formato de correo. |
| Pruebas | `buscar_usuario`, nombre existente | Comprueba que se devuelve el usuario con el nombre exacto. |
| Pruebas | `buscar_usuario`, nombre con apóstrofo | Comprueba que el nombre se trata como dato literal y la consulta no se altera. |
| Pruebas | `crear_usuario`, edad inválida | Comprueba que una edad negativa o no entera se rechaza cuando se añada validación. |

### Revisión humana

Antes de aprobar, la persona comprueba que cada hallazgo exista en la línea indicada, que la severidad sea razonable, que no se hayan copiado valores de credenciales a la entrega, y que cada caso de prueba corresponda al comportamiento real del archivo. Rechaza o pide corrección si hay líneas inventadas, conclusiones sin evidencia, casos irrelevantes o una recomendación que exceda el alcance.

### Detección y tratamiento de fallos

El orquestador detecta una respuesta vacía, líneas inexistentes, hallazgos sin evidencia o incumplimientos del límite de solo lectura. Solicita una corrección concreta al especialista y vuelve a comprobar las referencias. Si la respuesta sigue incompleta o contradictoria, marca el resultado como no verificado y avisa a la persona; no inventa resultados ni aprueba cambios automáticamente.

## A2A

> *Simplificación didáctica; no es JSON conforme a una versión específica de A2A.*

```json
{
  "identity": {
    "name": "Agente remoto de diseno de pruebas",
    "description": "Propone casos de prueba para una funcion indicada."
  },
  "skills": [
    {
      "id": "proponer-casos",
      "description": "Propone casos normales, limites y errores a partir del codigo descrito."
    }
  ],
  "interface": {
    "endpoint": "https://agente.ejemplo/a2a",
    "protocol": "JSON-RPC 2.0 sobre HTTPS"
  },
  "capabilities": {
    "streaming": true
  },
  "security": {
    "authentication": "OAuth 2.0; token de acceso con audiencia limitada",
    "requirements": [
      "TLS obligatorio",
      "El cliente valida identidad, permisos y origen antes de enviar datos"
    ]
  }
}
```

### Descubrimiento y validación

El cliente podría obtener la tarjeta desde una URL configurada por el equipo remoto, desde `/.well-known/agent-card.json` en un origen conocido, o desde un registro de agentes aprobado. Antes de enviar el archivo o una solicitud, valida que el origen esté permitido, que la tarjeta tenga la estructura esperada, que exista la skill `proponer-casos`, que la interfaz declarada sea compatible y que el mecanismo de autenticación requerido esté disponible. La tarjeta anuncia capacidades; no demuestra que el agente sea confiable.

### Secuencia y estados

```text
obtener Agent Card
  → verificar origen, skill, interfaz y seguridad
  → enviar mensaje con la solicitud compatible
  → recibir respuesta inmediata o identificador de Task
  → si hay Task: consultar/recibir estado
  → obtener el resultado (Artifact, si corresponde)
```

Una solicitud puede recibir un mensaje directo y no siempre crea una Task. En el recorrido simplificado `submitted → working → completed`, el cliente espera el resultado y valida que tenga el formato solicitado. Si el estado es `input-required`, pausa la revisión y pide a la persona la información faltante antes de continuar. Si es `failed`, registra el fallo, informa a la persona y ofrece reintentar o continuar sin el resultado remoto; no lo presenta como éxito.

### Diferencias con el subagente local y con MCP

El subagente local corre dentro del harness de la herramienta y usa su mecanismo de invocación y permisos. A2A requiere comunicación por una interfaz de red, descubrimiento e interoperabilidad entre implementaciones; agrega latencia y límites de confianza, por lo que hay que validar origen, autenticación, datos y respuestas. MCP conecta un agente con herramientas o recursos: si el orquestador necesita leer archivos directamente, puede usar una herramienta local o una expuesta mediante MCP. Si necesita delegar una revisión autónoma a otro agente remoto, puede usar A2A. La elección depende de la arquitectura y de la interfaz, no de que la tarea lea archivos.
