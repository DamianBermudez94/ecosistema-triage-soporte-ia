# Ecosistema de Automatización IA — Triage de Soporte con Memoria y HITL

Proyecto final del curso **AI Automation**. Un ecosistema de automatización que resuelve, de extremo a extremo y sin intervención manual, el **triage de tickets de soporte de un e-commerce**: recibe el correo del cliente, lo clasifica con IA, guarda todo en una base de datos como memoria, pide **aprobación humana** antes de ejecutar la acción crítica (reembolso) y responde por Gmail.

---

## 1. Caso de uso

Proceso de negocio: **atención al cliente / triage de soporte** para un e-commerce.

Cuando llega un correo a la casilla de soporte, el sistema interpreta el lenguaje natural del cliente y decide automáticamente qué hacer:

- Cliente molesto que pide devolución de dinero → **reembolso** (acción crítica, requiere aprobación humana).
- Cliente satisfecho / agradecido → **cupón de descuento** de fidelización.
- Consulta general → **respuesta asistida + escalado** a un agente humano.

## 2. Stack (tecnologías integradas)

| Capa | Herramienta |
|------|-------------|
| Orquestador | **n8n** |
| Base de datos (memoria) | **Airtable** |
| Motor de IA | **Anthropic Claude** (nodo AI Agent, modelo `claude-sonnet-5`) |
| Salida | **Gmail** (respuestas al cliente y avisos internos) |
| Human-in-the-loop | **Gmail** (Send and Wait / aprobación con botones) |

## 3. Estructura del repositorio

```
/
├── README.md                          <- este archivo
├── 1_Diagrama_Arquitectura.pdf        <- diagrama de arquitectura (entregable obligatorio)
├── Ecosistema_Triage_Soporte_n8n.json <- flujo de n8n exportado
├── /screenshots                       <- evidencias del flujo y las pruebas
│   ├── 01_flujo_completo.png
│   ├── 02_ejecucion_ok.png
│   ├── 03_hitl_email_aprobacion.png
│   ├── 04_airtable_estados.png
│   ├── 05_camino_infeliz_error.png
│   ├── 06_tabla_errores.png
│   └── 07_dashboard_kpis.png
├── /docs
│   └── link_base_datos.md             <- link en modo lectura a la base de Airtable
└── /mejoras                           <- documentación de mejoras
    ├── Mejora_1_Comparativa_Modelos.pdf
    ├── Mejora_2y3_Guia_Dashboard_Airtable.md
    └── Mejora_4_API_Batches.pdf
```

## 4. La base de datos (el "cerebro")

Base de Airtable **Soporte IA** con 3 tablas relacionadas.

### Tabla `Tickets`
| Campo | Tipo | Notas |
|-------|------|-------|
| Asunto | Single line text | campo principal |
| Email | Single line text | correo del cliente |
| Cuerpo | Long text | mensaje original |
| Sentimiento | Single line text | lo completa la IA (negativo / neutral / positivo) |
| Ruta | Single line text | lo completa la IA (reembolso / cupon / general) |
| Respuesta_IA | Long text | texto sugerido por la IA |
| Éxito | Formula | 1 si Estado es Resuelto/Escalado, 0 si no (para la tasa de éxito) |
| **Estado** | Single select | **campo de estado (ver ciclo abajo)** |
| Aprobado_por | Single line text | "Humano" cuando se aprueba un reembolso |
| Cliente | **Link to `Clientes`** | **relación entre tablas** |
| Fecha | Created time | automático |

> Nota de implementación: `Sentimiento` y `Ruta` se dejaron como **texto libre** (no Single select) para que acepten el valor que devuelve la IA sin conflictos de validación. Solo `Estado` es Single select, porque esos valores los controla el flujo.

### Tabla `Clientes`
| Campo | Tipo | Notas |
|-------|------|-------|
| Email | Single line text | clave de upsert (campo principal) |
| Nombre | Single line text | opcional |
| Tickets | **Link to `Tickets`** | historial (relación inversa) |

### Tabla `Errores`
| Campo | Tipo |
|-------|------|
| Nodo | Single line text |
| Mensaje | Long text |
| Payload | Long text |
| Fecha | Created time |

### Ciclo de estados (campo `Estado`)
```
Pendiente → Procesado por IA → Esperando aprobación
   → Aprobado por humano / Rechazado por humano
   → Resuelto / Escalado
```
Cumple el requisito de **campos de estado + relaciones entre tablas** para evitar datos aislados.

### Link en modo lectura
En Airtable: `Share → Share base → "Anyone with the link can view"`. Pegá el link en `docs/link_base_datos.md` y verificalo en ventana de incógnito antes de entregar.

### Dashboard de KPIs (Airtable Interfaces)
Además de las tablas, el sistema tiene un **panel de control** construido en Airtable Interfaces que agrupa los indicadores clave:

- **KPIs:** Total de tickets (volumen), Tasa de éxito (% de tickets Resueltos/Escalados sobre el total) y Tasa de errores (conteo de la tabla `Errores`).
- **Visualizaciones:** tickets por Estado, tickets por Sentimiento, distribución por Estado (torta), errores por Nodo y evolución de tickets por fecha.

> El link público de interfaces de Airtable requiere plan Team, por eso el dashboard se documenta con capturas (`screenshots/07_dashboard_kpis.png`) y la base queda accesible en modo lectura por el link de `docs/link_base_datos.md`.

## 5. El flujo en n8n (el "corazón")

### Cómo importarlo
1. n8n → menú `⋮` → **Import from File** → seleccioná `Ecosistema_Triage_Soporte_n8n.json`.
2. Cargá las **credenciales**: Gmail, Airtable (Personal Access Token) y Anthropic (Claude).
3. En cada nodo de Airtable, seleccioná tu **Base** y **Tabla**.
4. En el nodo de Claude, elegí el modelo `claude-sonnet-5`.

### Recorrido de nodos
1. **Trigger inteligente** — `Gmail Trigger` filtrado a correos nuevos (no procesa toda la bandeja). Hay además un `Trigger Manual` + `Email de ejemplo` para pruebas.
2. **Normalizar datos del email** — unifica `from / subject / body / messageId` venga del trigger real o del de prueba (sin datos hardcodeados).
3. **Filtro anti-bucle** — descarta auto-respuestas y correos ya procesados (evita loops infinitos).
4. **Airtable** — upsert de `Cliente` + crea `Ticket` con estado `Pendiente`.
5. **Motor de IA (AI Agent + Claude)** — prompt dinámico con las variables del correo, devuelve JSON estructurado (`sentimiento`, `ruta`, `respuesta`). **Max Tokens = 400** para optimizar costos. Con **On Error → Continue (error output)**: si la API falla, no rompe el flujo.
6. **Parsear respuesta IA** — mapea el `output` del Agent a campos usables (tolera mayúsculas/minúsculas en las claves).
7. **Airtable** — guarda la clasificación, estado `Procesado por IA`.
8. **Switch** — enruta según `ruta` (reembolso / cupón / general).
9. **HITL (Gmail Send and Wait)** — en la ruta de reembolso, envía un email con botones Aprobar/Rechazar y **pausa** hasta la respuesta humana. Tiene **Limit Wait Time = 2 días**: si nadie responde, se reanuda y cae en la rama "No aprobado".
10. **Salida** — Gmail responde al cliente (operación *reply*, mantiene el Thread ID) con plantillas HTML. Airtable cierra el ticket con el estado final.

### Human-in-the-loop (evita el "efecto metralleta")
El nodo de aprobación **detiene el flujo** antes de mover dinero. Solo si un humano aprueba, se ejecuta el reembolso y se confirma al cliente. Si rechaza (o vence el tiempo de espera), el ticket pasa a `Rechazado por humano` y se envía una respuesta alternativa.

### Gestión de errores (resiliencia)
- El nodo Claude tiene **`On Error → Continue (error output)`**: si la API de IA falla, el flujo **no se rompe**, registra el fallo en la tabla `Errores` de Airtable.
- **Error Trigger global** del proyecto para capturar cualquier fallo no controlado.
- Datos faltantes: el filtro y el nodo Normalizar evitan procesar correos incompletos.

> Distinción clave: un correo procesado con éxito (incluida una consulta general) termina en la tabla **Tickets**. La tabla **Errores** solo se llena ante un **fallo técnico** (API caída, credencial inválida, etc.).

## 6. Plan de test de estrés (mínimo 5 ejecuciones)

| # | Entrada | Camino esperado | Estado final |
|---|---------|-----------------|--------------|
| 1 | "Llegó roto, quiero mi reembolso" | Reembolso → HITL → **Aprobar** | Resuelto |
| 2 | "Llegó roto, quiero mi reembolso" | Reembolso → HITL → **Rechazar** | Rechazado por humano |
| 3 | "Gracias, excelente producto!" | Cupón | Resuelto |
| 4 | "¿Cuándo llega mi pedido?" | Consulta general → escalado | Escalado |
| 5 (camino infeliz) | Correo **sin asunto / cuerpo vacío** | Filtro / validación | no se procesa |
| 6 (camino infeliz) | **API de IA caída** (credencial inválida a propósito) | Salida de error del Agent | Registro en tabla `Errores` |

Ejecutá cada caso, sacá screenshot de la ejecución y del cambio de estado en Airtable.

## 7. Check de seguridad (antes de enviar)

1. **¿Filtro para evitar bucles infinitos?** Sí — el nodo `Filtro anti-bucle` descarta correos propios y marcados `[AUTO]`.
2. **¿Comparás tipos de datos correctos?** Sí — el `Switch` compara texto (`ruta`), el `IF` del HITL compara **booleano** (`data.approved`).
3. **¿Prompt de IA dinámico con variables del sistema?** Sí — usa `{{ asunto }}` y `{{ cuerpo }}` del correo normalizado, nada hardcodeado.

## 8. Mejoras aplicadas

Documentación adicional en la carpeta `/mejoras`:

- **Comparativa de modelos + matriz de decisión** (`Mejora_1_Comparativa_Modelos.pdf`): justifica la elección de modelo (Haiku vs Sonnet vs Opus) por tipo de tarea, con precios 2026 e impacto financiero estimado por volumen.
- **Guía de dashboard en Airtable** (`Mejora_2y3_Guia_Dashboard_Airtable.md`): paso a paso del panel de KPIs (volumen, tasa de éxito, tasa de errores) con gráficos y agregaciones.
- **API de Batches y costos** (`Mejora_4_API_Batches.pdf`): uso del procesamiento por lotes asincrónico (Anthropic/OpenAI) y su impacto de ~50% en reducción de costos operativos.

## 9. Checklist de entregables
- [ ] `1_Diagrama_Arquitectura.pdf`
- [ ] `Ecosistema_Triage_Soporte_n8n.json` (exportado desde tu n8n)
- [ ] Link a la base de Airtable en **modo lectura**
- [ ] Screenshots de evidencia (flujo, ejecución, HITL, estados, camino infeliz, tabla Errores, dashboard)
- [ ] Video demo (3 min, sin credenciales visibles)
- [ ] Documentos de mejoras en `/mejoras`
- [ ] Todo publicado en un repositorio público de GitHub
