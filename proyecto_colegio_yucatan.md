# Proyecto de Automatización Administrativa — Colegio Yucatan SCP

## Resumen ejecutivo

Sustitución de procesos manuales y propensos a errores en dos áreas críticas de administración escolar: **conciliación bancaria** y **seguimiento de cobranza**. Ambos sistemas están diseñados para integrarse eventualmente, pero se desarrollan y validan de forma independiente para reducir riesgo.

---

## Sistema 1: Conciliación Bancaria (n8n)

### Problema que resuelve

El cierre mensual se realizaba manualmente, cruzando pagos de tres fuentes heterogéneas con formatos distintos. El proceso era lento, repetible en errores y dependiente de una sola persona.

### Fuentes de datos

| Fuente | Formato | Particularidades |
|---|---|---|
| HSBC Global Payments | Excel (.xlsx) | Fechas invertidas por openpyxl; folio con zero-padding |
| Corte de caja (sistema interno) | TXT | Encabezados variables; parsing con regex |
| Santander | CSV | Campos envueltos en apóstrofos; fechas DDMMAAAA |

### Arquitectura

- **Orquestador:** n8n (self-hosted o cloud)
- **Almacenamiento y revisión:** Google Sheets
- **Ciclo:** Semanal (sustituyó el cierre mensual)

### Flujos

#### Flujo 1 — Extracción, normalización y cruce ✅ Completo y validado
1. Lectura de las tres fuentes desde Google Drive
2. Normalización de fechas, folios y montos por fuente
3. Cruce y deduplicación entre fuentes
4. Escritura en pestañas de revisión (`Clear + Append` paralelo)

#### Flujo 2 — Validación y conciliación final 🔄 En construcción
1. Re-exportación limpia del TXT para validación
2. Segunda pasada de validación contra datos ya revisados
3. Conciliación final en pestañas acumuladas (solo escritura de operadores)

### Lógica de deduplicación entre ejecuciones diarias

- Al iniciar cada ejecución: leer pestañas de revisión pendientes
- Fusionar con datos nuevos
- Deduplicar en nodo Code:
  - Santander → clave: `referencia`
  - HSBC y sistema → clave: `folio`
- Reemplazar contenido de pestañas de revisión (`Clear + Append`)
- **Nunca tocar** pestañas finales/acumuladas (son territorio exclusivo de operadores)

### Problemas de calidad de datos resueltos

- Apostrofos envolviendo campos CSV de Santander
- Formato de fecha `DDMMAAAA` en Santander
- Inversión de fechas inducida por openpyxl en HSBC
- Conteo variable de encabezados en TXT (resuelto con regex)
- Zero-padding inconsistente en folios
- Folios duplicados (agrupados y sumados antes del cruce)

### Principios de diseño

- **Pestañas de revisión:** `Clear + Append` en cada ejecución (seguro sobrescribir)
- **Pestañas finales:** intocables por automatización; movimiento manual y deliberado de operadores
- **Integración diferida:** la app de cobranza no se conecta hasta que este flujo sea confiable

---

## Sistema 2: Aplicación de Seguimiento de Cobranza (HTML/JS)

### Problema que resuelve

El seguimiento de adeudos de alumnos se llevaba en un Excel con codificación de colores, sin historial de contactos, sin filtros dinámicos y sin visibilidad consolidada del estado de la cartera.

### Estado actual

Prototipo funcional de una sola página (HTML + JavaScript puro), con 12 registros mock y cuatro vistas:

| Vista | Funcionalidad |
|---|---|
| **Kanban** | Drag-and-drop entre columnas; color por urgencia (`dias_atraso`) |
| **Dashboard** | KPIs, gráfica de barras por estatus, ranking de mayores adeudos |
| **Detalle** | Panel editable: estatus, comentarios, registro de contacto con un clic |
| **Tabla** | Ordenable y filtrable por columna |

### Esquema de datos — `Cobranza_Activa`

- `id_alumno`, `nombre`, `grado`, `nivel`
- `saldo_pendiente` (numérico)
- `dias_atraso` (calculado por fórmula, no entrada manual)
- `ultimo_contacto` (fecha)
- `comentarios` (texto libre)
- `estatus` (vocabulario controlado por dropdown)

### Progresión de estatus

```
SIN_CONTACTAR → EN_SEGUIMIENTO → COMPROMISO_DE_PAGO → PAGADO → SUSPENDIDO
```

### Hosting

El frontend vive como archivos estáticos servidos localmente o desde un servidor simple. Supabase actúa como base de datos y capa de autenticación. No se requiere backend propio — n8n hace el trabajo de transformación y escritura.

---

## Arquitectura del sistema completo

### Visión general

```
Fuentes brutas (HSBC xlsx, Santander CSV, Caja TXT)
        ↓
    Google Drive
        ↓
    n8n — Flujo 1 (extracción, normalización, cruce)
        ↓
    Google Sheets (pestañas de revisión — trabajo del operador)
        ↓
    n8n — Flujo 2 (validación y conciliación final)
        ↓
    Supabase (tablas definitivas — fuente de verdad)
        ↓
    Frontend Coleg.IO (dashboard.html → app completa)
        ↓
    Operador: revisión manual, aprobaciones, visualizaciones
```

### Rol de cada componente

| Componente | Rol |
|---|---|
| **n8n** | Orquestador. Extrae, limpia, cruza y escribe. Nunca expuesto al usuario final. |
| **Google Sheets** | Zona de revisión temporal. El operador puede editar antes de aprobar. |
| **Supabase** | Fuente de verdad definitiva. Recibe datos solo después de aprobación. Expone API REST para el frontend. |
| **Frontend (Coleg.IO)** | Interfaz del operador. Lee de Supabase. Permite revisión, aprobación, corrección manual y visualizaciones. |

### Flujo de datos — conciliación

```
1. n8n ejecuta Flujo 1 → escribe en Google Sheets (pestañas revisión)
2. Operador revisa en Sheets, corrige si hay errores
3. n8n ejecuta Flujo 2 → lee Sheets aprobadas, escribe en Supabase
4. Frontend lee Supabase → muestra estado de conciliación
5. Operador aprueba/rechaza movimientos directamente en el frontend
6. Supabase actualiza estado → n8n puede reaccionar vía webhook si es necesario
```

### Esquema de tablas Supabase (conciliación)

#### `conciliaciones`
| Campo | Tipo | Descripción |
|---|---|---|
| `id` | uuid | PK generado por Supabase |
| `folio` | text | Folio normalizado (clave de cruce) |
| `referencia` | text | Referencia Santander (clave alternativa) |
| `fuente` | text | `HSBC` / `SANTANDER` / `CAJA` |
| `fecha` | date | Fecha normalizada (YYYY-MM-DD) |
| `monto` | numeric | Monto del movimiento |
| `estatus` | text | `PENDIENTE` / `CONCILIADO` / `EN_REVISION` / `RECHAZADO` |
| `periodo` | text | Ej: `2025-06` — semana o mes de cierre |
| `notas` | text | Comentarios del operador |
| `created_at` | timestamptz | Auto |
| `updated_at` | timestamptz | Auto |

#### `cobranza_activa`
| Campo | Tipo | Descripción |
|---|---|---|
| `id_alumno` | text | ID del alumno |
| `nombre` | text | Nombre completo |
| `grado` | text | |
| `nivel` | text | |
| `saldo_pendiente` | numeric | |
| `fecha_vencimiento` | date | Para calcular `dias_atraso` en frontend |
| `ultimo_contacto` | date | |
| `comentarios` | text | |
| `estatus` | text | Vocabulario controlado (ver progresión) |
| `updated_at` | timestamptz | Auto |

---

## Integración futura

Una vez que Flujo 2 esté validado y estable:

```
Flujo 2 escribe pago confirmado en Supabase (conciliaciones)
        ↓
  Supabase webhook / n8n polling detecta nuevo CONCILIADO
        ↓
  n8n actualiza tabla cobranza_activa → estatus PAGADO
        ↓
  Frontend refleja cambio en tiempo real (Supabase Realtime)
        ↓
  Notificación al operador responsable
```

---

## Frontend — estructura de archivos

```
Coleg.io/
├── proyecto_colegio_yucatan.md   ← este archivo
├── dashboard.html                ← punto de partida (diseño base)
├── index.html                    ← shell principal del ERP (por construir)
└── modulos/
    └── conciliacion/
        ├── index.html            ← lista y KPIs de conciliaciones
        ├── detalle.html          ← revisión movimiento a movimiento
        └── reportes.html         ← visualizaciones y cierres
```

### Decisiones de diseño frontend

- **Sin framework por ahora:** HTML + JS + CSS vanilla, igual que el prototipo existente. Escalar a framework cuando la complejidad lo justifique.
- **Supabase JS SDK:** única dependencia externa. Se carga desde CDN para no necesitar build step.
- **Sin backend propio:** Supabase expone la API REST y gestiona autenticación. El frontend llama directamente.
- **Diseño tomado del `dashboard.html`:** paleta, tipografía (Inter + Playfair Display), componentes y variables CSS se reutilizan tal cual.

---

## Hoja de ruta

| Prioridad | Tarea | Estado |
|---|---|---|
| 1 | Completar lógica de deduplicación en Flujo 1 | 🔄 En progreso |
| 2 | Construir y validar Flujo 2 | 🔄 En construcción |
| 3 | Crear tablas en Supabase (`conciliaciones`, `cobranza_activa`) | ⏳ Pendiente |
| 4 | Conectar Flujo 2 para que escriba en Supabase en lugar de solo Sheets | ⏳ Pendiente |
| 5 | Construir módulo de conciliación en el frontend | ⏳ Pendiente |
| 6 | Visualizaciones y reportes de cierre | ⏳ Pendiente |
| 7 | Integrar eventos de pago confirmado → cobranza | ⏳ Diferido |

---

## Principios transversales

1. **Integración diferida:** cada sistema debe ser independientemente confiable antes de conectarlos.
2. **Validación por fases:** una fase se da por cerrada solo cuando opera correctamente en producción.
3. **Operadores en el centro:** la automatización asiste; el movimiento de registros críticos es siempre una acción humana y deliberada.
4. **Vocabularios controlados:** campos de estatus con dropdown para evitar inconsistencias entre usuarios.
5. **Fórmula sobre entrada manual:** campos derivados (como `dias_atraso`) se calculan, no se escriben.
6. **Supabase como fuente de verdad:** Google Sheets es zona de trabajo temporal; Supabase es el registro definitivo.
7. **Frontend sin backend propio:** Supabase SDK directo desde el navegador; n8n hace toda la transformación antes de escribir.
