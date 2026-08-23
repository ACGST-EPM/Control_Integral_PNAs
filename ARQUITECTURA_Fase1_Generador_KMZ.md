# Plataforma Control Integral PNAs — Arquitectura Fase 1 (Generador KMZ)
## Documento de decisiones de arquitectura
**Grupo EPM · Centro de Gestión Servicios Técnicos**
Repositorio: `ACGST-EPM/Control_Integral_PNAs`
Carpeta de trabajo local: `C:\Users\lmarinza\PLATAFORMA_PNAs`
Rama de desarrollo: `claude/plataforma-permisos-geoespaciales-6g19yn`

> **Propósito:** dejar registrada, de forma reproducible, la estructura de datos que sostiene toda la plataforma. Lo que se define aquí se refleja después en el código Python (Fase 2) y en el Dashboard (Fase 3).

---

## 1. Qué es esta plataforma (y en qué se diferencia de la de PMTs de obra)

Esta herramienta administra el **ciclo de vida de trámites de permisos (PNAs)** ubicados en un mapa. **No** detecta conflictos espaciales/temporales entre permisos (eso queda para una fase futura).

| | Plataforma PMTs de **obra** (base anterior) | Plataforma **Control Integral PNAs** (esta) |
|---|---|---|
| Objetivo | Detectar choques espaciales/temporales entre frentes | Seguir el **historial de gestión** de cada trámite |
| Etapa del proyecto | Obra (una fase específica) | Diversas fases y etapas (diseño en adelante) |
| Motor | Cruza geometrías (interferencias/cercanías) | Consolida estado e historial para el tablero |
| Permisos | Un solo tipo (PMT de obra) | Cinco tipos (PMT_Diseño, PUZV, Cierre_Via, PUOI-IVCF, PIV) |

> **Integración futura:** cuando ambas maduren, se buscará unirlas. Por ahora se desarrollan por separado.

---

## 2. Principio de diseño: "núcleo + etiqueta + extensión + catálogo único"

- **Núcleo común:** campos que todo permiso tiene siempre.
- **Etiqueta discriminadora (`PNA`):** el campo que dice de qué permiso se trata. Evita fragmentar la lógica: un solo sistema atiende los cinco permisos.
- **Campos específicos / historial:** datos que se acumulan por permiso.
- **Catálogo único (`tipos_permiso.json`):** describe cada permiso en un solo lugar (autoridad, plazo, estados, color, geometrías). El Generador, el Python y el Dashboard **leen de ese mismo archivo**. Para agregar un permiso se edita el catálogo, no los tres programas.

El KMZ guarda la información como parejas `campo: valor` separadas por ` | ` (igual que la plataforma anterior), leídas "por nombre" y no "por posición". Así se pueden añadir campos sin romper nada.

---

## 3. Estructura de campos

### Grupo A — Identidad del proyecto (la ingresa el ADMIN con clave)
Protegido por clave de administrador, como el numeral 1 del generador anterior.

| Campo | Tipo | Origen |
|---|---|---|
| `ID` | Identificador único | **Automático** (consecutivo generado por el sistema, ej. 0001) |
| `Código_Banco` | Texto | Lo agrega el admin |
| `Proyecto` | Texto | Lo agrega el admin |
| `Estado` | Lista (`estado_proyecto`) | Selección |
| `Fase` | Lista (`fase_proyecto`) | Selección |
| `Municipio` | Lista (`municipios`, ampliable) | Selección |
| `Contrato` | Texto | Lo agrega el admin |

### Grupo B — Etiqueta discriminadora
| Campo | Valores |
|---|---|
| `PNA` | PMT_Diseño · PUZV · Cierre_Via · PUOI-IVCF · PIV |

### Grupo C — Datos DEDUCIDOS del PNA (los pone el sistema automáticamente)
Al elegir el PNA, el sistema completa solo estos datos leyendo el catálogo:

| PNA | Autoridad | Tiempo_Gestión | ¿Concesión? | Estados de trámite |
|---|---|---|---|---|
| PMT_Diseño | Municipio (Sec. Movilidad) | 15 días hábiles | No | Radicado, Negado, Aprobado |
| PUZV | INVIAS | 12 meses | No | (los 8 estados completos) |
| Cierre_Via | INVIAS | 3 meses | No | (los 8 estados completos) |
| PUOI-IVCF | ANI | 12 meses | **Sí** | (los 8 estados completos) |
| PIV | Gobernación | 12 meses | No | (los 8 estados completos) |

**Los 8 estados completos:** Radicado · Observado · Revisión_Ajustes · Resolución_Aprobatoria · Gestión_Pólizas · Gestión_Acta_Inicio · Gestión_Suspensión · Gestión_Reinicio

### Grupo D — Historial de gestión (lo acumula el usuario del generador)
Cada permiso guarda una **lista de filas de historial** (una bitácora que crece en el tiempo). Cada fila tiene:

| Campo | Tipo | Regla |
|---|---|---|
| `Estado_Trámite` | Lista (según el PNA) | Selección |
| `Radicado` | Texto | Se ingresa solo en ciertos estados (ver `radicado_en_estados`) |
| `Fecha_radicación` | Fecha (DD/MM/AAAA) | Selector de fecha |
| `Fecha_Respuesta` | Fecha (DD/MM/AAAA) | Selector de fecha |
| `Observación` | Texto libre | Lo escribe el profesional asignado |
| `Concesión` | Texto | Solo si `PNA = PUOI-IVCF` (autoridad ANI) |

**Regla de `Radicado` por permiso:**
- PMT_Diseño → en cualquiera de sus estados.
- PUZV / Cierre_Via / PUOI-IVCF / PIV → solo en `Radicado`, `Observado`, `Resolución_Aprobatoria`.

---

## 4. Reglas de negocio confirmadas

1. **ID automático:** lo genera el sistema (consecutivo), el admin no lo escribe.
2. **Geometría:** los cinco permisos admiten **punto, línea y polígono**. (El generador anterior solo hacía punto y línea → en Fase 2 del generador se agrega el polígono.)
3. **Varios permisos por proyecto:** un mismo Proyecto/Código_Banco puede tener **varios PNA**, incluso **varios del mismo tipo**. Por eso el `ID` identifica cada *permiso* (trazado), no el proyecto.
4. **Sin detección de conflictos** entre permisos por ahora.
5. **Autoridad, Tiempo_Gestión, estados y "¿concesión?"** no se escriben: se deducen del PNA vía catálogo.

---

## 4.1 Reglas de plazos y semántica de estados (para la Fase 2 — pendiente de construir)

> Decisiones tomadas en sesión para el diseño del **semáforo** y la **línea de tiempo** del dashboard. Algunas quedan marcadas como *(por confirmar)*.

### Los dos "relojes" a contrastar
1. **Reloj normativo (autoridad):** cuánto tarda el permiso = `tiempo_gestion` del catálogo (15 días hábiles / 3 / 12 meses).
2. **Reloj del proyecto (cronograma):** **fecha límite** para tener el permiso sin afectar la contratación/ejecución de la obra → nuevo dato **`Fecha_Límite_Requerida`**.

### Nuevo campo `Fecha_Límite_Requerida`
- **Una por permiso** (cada permiso se necesita en un momento distinto del cronograma).
- Es **dato del proyecto/permiso**, NO del catálogo (no se toca `tipos_permiso.json`).
- **No** se enlaza aún a hitos del cronograma: primero se estandariza la gestión de permisos; cuando los programadores estandaricen sus cronogramas, se enlazará. (Fase futura.)

### Cálculo propuesto (planeación hacia atrás)
- **Última fecha viable para radicar** = `Fecha_Límite_Requerida` − `tiempo_gestion` (− holgura opcional).
- **Dos semáforos** juntos:
  - **De gestión:** ¿el trámite avanza dentro del plazo de la autoridad?
  - **De riesgo al proyecto:** dado lo avanzado y lo que falta, ¿llegará antes de la `Fecha_Límite_Requerida`?

### Semántica de los estados de trámite (PUZV, Cierre_Via, PUOI-IVCF, PIV)
Los 8 estados ya están completos en el generador. Su significado para el semáforo/línea de tiempo:

| Estado | Significado |
|---|---|
| Radicado | Inicia el trámite ante la autoridad (arranca el reloj normativo) |
| Observado | La autoridad pide ajustes |
| Revisión_Ajustes | Se atienden las observaciones |
| Resolución_Aprobatoria | **Permiso otorgado** (en mano) |
| Gestión_Pólizas | Trámite de pólizas posterior a la aprobación |
| **Gestión_Acta_Inicio** | **Detona el inicio de la ejecución** bajo el permiso (se "activa") |
| Gestión_Suspensión | **Opcional** — pausa (permiso listo pero aún no se ejecuta / corrimiento de cronograma) |
| Gestión_Reinicio | **Opcional** — reactivación tras una suspensión |

- **PMT_Diseño** mantiene sus 3 estados: Radicado · Negado · Aprobado.
- *(Por confirmar)* La `Fecha_Límite_Requerida` se compara, por defecto, contra alcanzar **Gestión_Acta_Inicio** (poder iniciar obra), no solo contra Resolución_Aprobatoria.
- *(Por confirmar)* Fecha de arranque del reloj normativo = **Fecha_radicación del primer "Radicado"**.
- *(Por confirmar)* Estados que "cierran" el trámite y detienen el reloj de gestión: PMT_Diseño → Aprobado/Negado; los demás → Resolución_Aprobatoria (o Gestión_Acta_Inicio, según lo anterior).
- **Nota de ortografía canónica:** los estados se guardan con tildes exactamente como en el catálogo (`Revisión_Ajustes`, `Resolución_Aprobatoria`, `Gestión_Pólizas`, `Gestión_Acta_Inicio`, `Gestión_Suspensión`, `Gestión_Reinicio`) para que el dashboard los compare sin errores.

---

## 5. Formato del KMZ (borrador — se cierra al construir el generador)

Cada trazado (`<Placemark>`) llevará en su `<description>` los datos de identidad + PNA, y el **historial serializado**. Propuesta:

- Campos simples: `campo: valor` separados por ` | ` (compatibilidad con el estilo anterior).
- Historial: un bloque estructurado (JSON) bajo la clave `historial:` para poder guardar varias filas dentro de un solo trazado.

> Este formato exacto se fija en la construcción del Generador (siguiente paso). El separador ` | ` no debe usarse dentro de textos libres (se reemplaza por `/`).

---

## 6. Estructura de carpetas local sugerida

```
C:\Users\lmarinza\PLATAFORMA_PNAs
├─ 01_KMZ_Entrada          (aquí caen los KMZ que genera la herramienta)
├─ 02_Proyecto_QGIS        (script Python de la Fase 2 — no se publica)
└─ PLATAFORMA_PNAs         (clon del repositorio GitHub — archivos web)
```

Los archivos que van al **repositorio** (carpeta `PLATAFORMA_PNAs`):
- `tipos_permiso.json` — catálogo maestro.
- `Generador_KMZ_PNAs.html` — Fase 1.
- `Plataforma_PNAs.html` — Fase 3 (a construir).
- `ARQUITECTURA_Fase1_Generador_KMZ.md` — este documento.

> **Convención de nombres:** todos los archivos de esta plataforma llevan el sufijo `_PNAs` (p. ej. `Generador_KMZ_PNAs.html`, `Plataforma_PNAs.html`, `proceso_pnas_qgis.py`) para distinguirlos de los de la plataforma de PMTs de obra. Así, si a futuro se integran ambas soluciones, no habrá choque de nombres.

---

## 7. Estado de avance

- [x] **Fase 1 – Definición de estructura y catálogo** (este documento + `tipos_permiso.json`).
- [x] **Fase 1 – Construcción del `Generador_KMZ_PNAs.html`** (con polígonos + historial + ID automático).
- [ ] Fase 2 – Script Python/QGIS (`proceso_pnas_qgis.py`) que lee los KMZ y consolida el reporte.
- [ ] Fase 3 – Dashboard (`Plataforma_PNAs.html`) adaptado a PNAs (filtros por PNA, autoridad, estado de trámite, semáforo de plazos).

---

## Anexo — Archivos de esta fase
- `tipos_permiso.json` — catálogo maestro (fuente única de verdad).
- `ARQUITECTURA_Fase1_Generador_KMZ.md` — este documento.
