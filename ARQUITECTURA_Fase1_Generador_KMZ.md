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
- `Generador_KMZ.html` — Fase 1 (a construir).
- `Plataforma.html` — Fase 3 (a construir).
- `ARQUITECTURA_Fase1_Generador_KMZ.md` — este documento.

---

## 7. Estado de avance

- [x] **Fase 1 – Definición de estructura y catálogo** (este documento + `tipos_permiso.json`).
- [ ] Fase 1 – Construcción del `Generador_KMZ.html` (con polígonos + historial + ID automático).
- [ ] Fase 2 – Script Python/QGIS que lee los KMZ y consolida el reporte.
- [ ] Fase 3 – Dashboard adaptado a PNAs (filtros por PNA, autoridad, estado de trámite, semáforo de plazos).

---

## Anexo — Archivos de esta fase
- `tipos_permiso.json` — catálogo maestro (fuente única de verdad).
- `ARQUITECTURA_Fase1_Generador_KMZ.md` — este documento.
