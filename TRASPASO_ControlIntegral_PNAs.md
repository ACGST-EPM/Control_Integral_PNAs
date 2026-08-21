# Plataforma Control Integral PNAs — Documento de traspaso y contexto
## Estado del desarrollo, limitaciones, hoja de ruta y guía para replicar en GitHub Copilot
**Grupo EPM · Centro de Gestión Servicios Técnicos**
Repositorio: `ACGST-EPM/Control_Integral_PNAs`
Rama de desarrollo: `claude/plataforma-permisos-geoespaciales-6g19yn`
Carpeta de trabajo local: `C:\Users\lmarinza\PLATAFORMA_PNAs`
Fecha: 2026-08-21

> **Propósito de este documento:** dejar en un solo lugar todo lo construido hasta ahora, con qué falta, hacia dónde vamos y cómo replicar el mismo proceso en **GitHub Copilot** si se desea trabajar en paralelo. Es autocontenido: sirve para retomar en una conversación nueva (aquí o en Copilot) sin perder contexto.

---

## 1. Objetivo final (hacia dónde vamos)

Construir una **plataforma web integral** que administre el **ciclo de vida de trámites de permisos geoespaciales (PNAs)** de EPM, con:

- **Captura estandarizada** de cada permiso y de su **historial de gestión** (bitácora que crece con el trámite), evitando la escritura manual libre.
- **Ubicación en mapa** de cada permiso (punto, línea o polígono).
- **Consolidación automática** de los datos en un reporte.
- **Dashboard** con filtros (por tipo de permiso, autoridad, municipio, estado del trámite), **semáforo de plazos** (según el tiempo de gestión de cada permiso) e informe ejecutivo.
- **Escalabilidad**: agregar un nuevo tipo de permiso debe requerir editar **un solo archivo de catálogo**, no reprogramar todo.

**Permisos que soporta (PNAs):**

| PNA | Nombre completo | Autoridad | Plazo | Concesión |
|---|---|---|---|---|
| PMT_Diseño | Plan de Manejo de Tránsito en etapa de Diseño | Municipio (Sec. Movilidad) | 15 días hábiles | No |
| PUZV | Permiso de Uso de Zona de Vía | INVIAS | 12 meses | No |
| Cierre_Via | Permiso de Cierre de Vía | INVIAS | 3 meses | No |
| PUOI-IVCF | Permiso de Uso, Ocupación e Intervención de Infraestructura Carretera Concesionada y Férrea | ANI | 12 meses | **Sí** |
| PIV | Permiso de Intervención Vial | Gobernación de Antioquia | 12 meses | No |

> **Alcance actual:** esta plataforma es independiente de la de **PMTs de obra** (que detecta interferencias espaciales/temporales). Aquí **no** hay detección de conflictos por ahora. La integración de ambas se evaluará al final.

---

## 2. Naturaleza de la solución (importante para no confundir)

| | Plataforma **PMTs de obra** (otro repo) | Plataforma **Control Integral PNAs** (esta) |
|---|---|---|
| Qué hace | Detecta choques espaciales/temporales entre frentes | Sigue el **historial de gestión** de cada trámite |
| Etapa | Obra (una fase específica) | Diversas fases/etapas (diseño en adelante) |
| Motor | Cruza geometrías (interferencias/cercanías) | Consolida estado e historial para el tablero |
| Permisos | Un solo tipo | Cinco tipos (ver tabla) |
| Convención de nombres | archivos sin sufijo | archivos con sufijo **`_PNAs`** |

---

## 3. Principio de arquitectura: "núcleo + etiqueta + catálogo único"

- **Núcleo común:** campos que todo permiso tiene siempre (identidad del proyecto).
- **Etiqueta discriminadora `PNA`:** dice de qué permiso se trata. Un solo sistema atiende los 5 permisos (no se fragmenta la lógica).
- **Catálogo único `tipos_permiso.json`:** describe cada permiso en un solo lugar (autoridad, plazo, estados de trámite, si pide concesión, color, geometrías). **El generador, el futuro Python y el futuro dashboard leen de este mismo archivo.** Para agregar un permiso se edita **solo el catálogo**.
- **Formato del KMZ:** cada trazado guarda sus datos como parejas `campo: valor` separadas por ` | `, leídas "por nombre" (no por posición), y el **historial serializado en JSON** al final. Así se pueden añadir campos sin romper nada.

### Estructura de campos (4 grupos)

**A. Identidad del proyecto (la ingresa el ADMIN con clave):**
`ID` (automático) · `Código_Banco` · `Proyecto` · `Estado` · `Fase` · `Municipio` · `Contrato`

**B. Etiqueta:** `PNA`

**C. Datos deducidos del PNA (los pone el sistema):** `Autoridad` · `Tiempo_Gestión` · `¿Concesión?` · lista de `Estado_Trámite` disponible.

**D. Historial de gestión (lo acumula el usuario, una fila por avance):**
`Estado_Trámite` · `Radicado` · `Fecha_radicación` · `Fecha_Respuesta` · `Observación` · (`Concesión` a nivel de permiso, solo para PUOI-IVCF).

**Reglas confirmadas:**
1. `ID` lo genera el sistema (consecutivo, ej. `0001`).
2. Geometría: punto, línea y **polígono**.
3. Un proyecto puede tener **varios permisos**, incluso varios del **mismo tipo**. El permiso (trazado) es la unidad; hereda la identidad del proyecto.
4. `Radicado` solo se captura en ciertos estados según el permiso (`radicado_en_estados` en el catálogo).
5. Sin detección de conflictos por ahora.

---

## 4. Lo construido hasta este punto (Fase 1 ✅)

| Archivo | Rol | Estado |
|---|---|---|
| `tipos_permiso.json` | **Catálogo maestro** (fuente única de verdad) con los 5 PNAs | ✅ Definitivo |
| `Generador_KMZ_PNAs.html` | **Generador de KMZ**: captura + historial + mapa, exporta KMZ | ✅ Funcional |
| `ARQUITECTURA_Fase1_Generador_KMZ.md` | Documento de decisiones de arquitectura | ✅ |
| `TRASPASO_ControlIntegral_PNAs.md` | Este documento | ✅ |

### Qué hace hoy el `Generador_KMZ_PNAs.html`
- **Base de proyectos con clave de administrador** (por defecto `EPM-PNA-2026`, guardada como huella SHA-256, no en texto plano). Import/export `proyectos_db.json`. **ID automático.**
- **Menús validados**: Estado, Fase y Municipio salen del catálogo.
- **Selección de PNA** que autocompleta **autoridad, plazo y estados de trámite**, y muestra el campo **Concesión solo para permisos ANI**.
- **Geometría**: dibujar/importar **punto, línea y polígono** (KMZ/KML de Google Earth).
- **Tabla de historial** acumulable; el campo `Radicado` se habilita solo en los estados permitidos por cada permiso.
- **Exporta el KMZ** con los campos simples + el historial en JSON dentro de cada trazado.
- **Autocontenido**: lee `tipos_permiso.json` si está publicado a su lado; si no, usa una copia embebida de respaldo.

---

## 5. Limitaciones conocidas (hoy)

### Del generador
- La **clave es un control de proceso**, no seguridad real (el archivo es público en GitHub Pages).
- La base de proyectos se **cachea por navegador** (localStorage). La versión compartida y persistente es `proyectos_db.json` en el repositorio, que hay que **exportar y subir manualmente** tras cada cambio (aún no hay base compartida en la nube).
- El **ID automático es por navegador/base local**: si dos personas generan proyectos en equipos distintos sin compartir el JSON, los ID podrían chocar. Mientras haya un solo administrador de la base, no hay problema.
- La **geometría dibujada a mano** es menos precisa que la importada de Google Earth.
- No hay **edición de vértices** dentro del generador (para corregir una geometría se vuelve a dibujar/importar).
- No hay **validación de duplicados** de permisos ni bitácora de quién generó cada KMZ.

### Del proceso general (aún no construido)
- **Fase 2 (Python/QGIS)** y **Fase 3 (Dashboard)** todavía no existen: hoy solo se genera el KMZ.
- No hay **automatización**: la corrida del análisis y la publicación son pasos manuales (como en la plataforma de PMTs).
- **Sin detección de conflictos** entre permisos (por diseño, por ahora).

### Operativa (esta sesión)
- El **push directo a GitHub está bloqueado** (acceso de solo lectura → error 403). Los cambios se entregan como archivos y se suben con **GitHub Desktop**. Para push directo se requiere habilitar **escritura** a la app de Claude sobre el repositorio (lo hace un administrador de la organización).

---

## 6. Hoja de ruta (pasos propuestos para llegar al objetivo)

### Fase 1 — Generador (✅ hecho, pendiente de pruebas del usuario)
- [x] Catálogo `tipos_permiso.json`.
- [x] `Generador_KMZ_PNAs.html` (polígonos + historial + ID automático).
- [ ] Pruebas de usuario y ajustes finos (colores, textos, flujo).

### Fase 2 — Motor de consolidación (`proceso_pnas_qgis.py`)
- [ ] Leer todos los `.kmz` de `01_KMZ_Entrada`.
- [ ] Parsear campos simples + **historial JSON** de cada trazado.
- [ ] Armar **capas del mapa** categorizadas por `PNA` (color del catálogo) y por geometría (punto/línea/polígono).
- [ ] Derivar el **estado actual** del trámite (última fila del historial) y calcular el **semáforo de plazos** según `tiempo_gestion` de cada permiso (días hábiles para PMT_Diseño; meses para los demás).
- [ ] Exportar `reporte_pnas.csv` con columnas estandarizadas para el dashboard.
- [ ] (Opcional) Exportar el **historial completo** a un CSV aparte (una fila por evento) para trazabilidad.

### Fase 3 — Dashboard (`Plataforma_PNAs.html`)
- [ ] Tabla + filtros cruzados: **PNA**, Autoridad, Municipio, Estado del trámite, Proyecto/Contrato.
- [ ] **Semáforo de vencimientos** por plazo de gestión.
- [ ] **Línea de tiempo** del trámite por permiso (con base en el historial).
- [ ] Mapa (qgis2web o Leaflet) coloreado por PNA.
- [ ] Informe **PDF ejecutivo** por autoridad / por proyecto.
- [ ] Botón para abrir el `Generador_KMZ_PNAs.html`.

### Fase 4 — Integración futura con PMTs de obra (a evaluar)
- [ ] Unificar catálogos y, si se requiere, activar detección de conflictos entre permisos.

---

## 7. Cómo continuar en ESTA plataforma (Claude Code + GitHub)

1. Adjunta este `.md` y (si aplica) los archivos actuales del repositorio.
2. Frase de contexto sugerida:
   > "Adjunto el traspaso de mi Plataforma Control Integral PNAs (EPM). Vamos a continuar con [Fase 2 / Fase 3 / ajuste X]. Respeta la arquitectura del catálogo único y pregúntame antes de cambiar decisiones ya tomadas."
3. Mantén la **convención de nombres `_PNAs`** y la **fuente única de verdad `tipos_permiso.json`**.

---

## 8. Guía para replicar el proceso en GitHub Copilot (símil)

> Objetivo: reproducir el mismo desarrollo en **GitHub Copilot** (Copilot Chat en VS Code o Copilot en el navegador), por si se quiere trabajar en paralelo.

### 8.1 Recomendaciones clave
1. **Lleva el contexto contigo.** Copilot no conoce esta conversación. Pega **este `.md` completo** al inicio del chat de Copilot; es el "cerebro" del proyecto. Sin él, Copilot improvisará y romperá la coherencia.
2. **Un archivo a la vez.** Copilot trabaja mejor enfocado en el archivo abierto. Pídele cambios sobre **un archivo concreto** (p. ej. "modifica solo `Generador_KMZ_PNAs.html`, no toques el resto").
3. **Respeta la fuente única de verdad.** Recuérdale que **autoridad, plazos, estados y colores viven en `tipos_permiso.json`** y que el código debe **leer del catálogo**, no tener valores "quemados".
4. **Nada de re-escribir todo.** Pídele **cambios mínimos y localizados**; que te muestre solo el bloque que cambia, para que tú lo pegues (igual que aquí).
5. **Formato del KMZ es sagrado.** El separador ` | `, los nombres de campo en minúscula y el `historial:` como JSON al final **no se cambian** sin actualizar también el futuro Python. Adviértele esto explícitamente.
6. **Convención de nombres `_PNAs`** en todos los archivos, para no chocar con la plataforma de PMTs de obra.
7. **Prueba local sin servidor.** Todo debe funcionar abriendo el HTML con doble clic; usa librerías por CDN (Leaflet, JSZip, Bootstrap) como aquí.
8. **Verifica siempre.** Copilot puede inventar funciones. Tras cada cambio, abre el archivo en el navegador y prueba el flujo completo (crear proyecto → dibujar → elegir PNA → historial → exportar KMZ).

### 8.2 Diferencias operativas a tener en cuenta
- **Copilot no "ejecuta" ni guarda archivos por sí solo** como aquí: tú aplicas los cambios en VS Code y haces commit/push (Copilot sí puede ayudarte a redactar los comandos de git).
- **Push a GitHub:** con Copilot en VS Code usas tu propia sesión de git, así que **no tendrás el bloqueo 403** que hay en esta sesión.
- **Copilot no genera archivos binarios** (KMZ) ni corre QGIS; para la Fase 2 te ayuda a **escribir el `.py`**, pero la ejecución sigue siendo manual en la consola de QGIS.

### 8.3 Prompt inicial sugerido para Copilot
Copia esto como primer mensaje en Copilot Chat (adjuntando este `.md`):

> "Actúa como asistente técnico de una plataforma de gestión geoespacial de permisos (PNAs) de EPM. Te adjunto el documento de traspaso con la arquitectura. Reglas: (1) la fuente única de verdad es `tipos_permiso.json`; el código debe leer de ahí, sin valores quemados. (2) Respeta el formato del KMZ (campos `campo: valor` separados por ` | ` y el `historial:` en JSON al final). (3) Usa la convención de nombres con sufijo `_PNAs`. (4) Haz cambios mínimos y localizados sobre el archivo que te indique, y muéstrame solo el bloque modificado. (5) No tengo experiencia en programación: dame instrucciones claras de dónde pegar cada cosa. Empecemos por [describe la tarea]."

### 8.4 Orden de trabajo sugerido en Copilot
1. Cargar `tipos_permiso.json` y `Generador_KMZ_PNAs.html` al repo/carpeta.
2. Pedir a Copilot que **verifique** que el generador lee bien el catálogo.
3. Construir `proceso_pnas_qgis.py` (Fase 2) pidiéndole que **parsee el historial JSON** y calcule el semáforo por plazo.
4. Construir `Plataforma_PNAs.html` (Fase 3) leyendo el CSV del paso anterior.

---

## 9. Inventario actual del repositorio

```
Control_Integral_PNAs/
├─ tipos_permiso.json                    (catálogo maestro — fuente única de verdad)
├─ Generador_KMZ_PNAs.html               (Fase 1 — generador de KMZ)
├─ ARQUITECTURA_Fase1_Generador_KMZ.md   (decisiones de arquitectura)
├─ TRASPASO_ControlIntegral_PNAs.md      (este documento)
└─ README.md
```

**Archivos que produce el usuario al usar la herramienta (no versionados necesariamente):**
- `proyectos_db.json` — base de proyectos exportada desde el generador.
- `*.kmz` — permisos generados (van a `01_KMZ_Entrada`).

---

## 10. Datos de referencia rápida

- **Clave de administrador del generador:** `EPM-PNA-2026` (huella SHA-256 en la constante `CLAVE_HASH`; para cambiarla se reemplaza por el SHA-256 de la nueva clave).
- **Colores por PNA** (definidos en `tipos_permiso.json`): PMT_Diseño `#009300` · PUZV `#0066CC` · Cierre_Via `#d56b00` · PUOI-IVCF `#8e44ad` · PIV `#e6a800`.
- **Separador de campos del KMZ:** ` | ` (no usar `|` en textos libres; se reemplaza por `/`).
- **Historial en el KMZ:** clave `historial:` seguida de un arreglo JSON `[{estado_tramite, radicado, fecha_radicacion, fecha_respuesta, observacion}, ...]` al final de la descripción.

---

*Fin del documento de traspaso.*
