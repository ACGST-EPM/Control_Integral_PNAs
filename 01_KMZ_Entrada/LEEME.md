# Carpeta 01_KMZ_Entrada

Aquí van los **KMZ de cada proyecto** (uno por proyecto, nombrado con su **Código Banco**, p. ej. `CB123.kmz`).

## Flujo
1. En el generador (`Generador_KMZ_PNAs.html`) importas el KMZ actual del proyecto, agregas el avance del trámite (nueva fila de historial) o un permiso nuevo, y exportas.
2. Guardas el KMZ resultante **en esta carpeta**, reemplazando la versión anterior del mismo proyecto.
3. Haces **commit + push**.
4. El dashboard (`Plataforma_PNAs.html`), al recargarse o con el botón **🔄 Recargar del repositorio**, lee automáticamente todos los KMZ de esta carpeta y muestra la información actualizada.

> El KMZ es el **almacén oficial** del historial acumulado de cada proyecto: cada estado nuevo se añade como una fila más, conservando las anteriores.

> Nota: tras el push, GitHub puede tardar 1–2 minutos en servir la versión nueva por caché.
