# Manual del usuario — extractorGfwPvc

## Qué hace esta skill

`extractorGfwPvc` extrae datos de monitoreo marítimo desde la API v4 de Global
Fishing Watch (GFW) para un área geográfica y un período de fechas determinados:
presencia de embarcaciones por AIS, detecciones por radar satelital (SAR), y un
sub-flujo específico de "pesca aparente" que además descarga rasters TIF de
esfuerzo pesquero (fishing effort) en dos resoluciones espaciales (0.01° y 0.1°).

Produce archivos CSV listos para análisis tabular y rasters TIF listos para abrir
en QGIS u otro SIG. No depende de ningún otro componente del proyecto de origen —
es una skill autocontenida de un solo script.

## Instalación

1. Copia el bootstrap (`extractorGfwPvcBootstrap.md`) a la raíz del proyecto donde
   quieres usar la skill.
2. Sigue los pasos del propio bootstrap: crea las carpetas indicadas, guarda el
   script de extracción embebido en `/tmp/extractExtractorGfwPvc.py` y ejecútalo
   con `python3`. Esto escribe los archivos reales de la skill (`SKILL.md`,
   `scripts/01extractorGfw.py`, `requirements.txt`) en tu proyecto.
3. Instala las dependencias Python (en tu propio entorno virtual, no se instala
   nada automáticamente):
   ```bash
   pip install -r requirements.txt
   ```
4. Verifica que el número de archivos extraídos coincide con el reportado por el
   bootstrap.

## Cómo invocar

Activa esta skill cuando necesites datos de GFW para un período y área de interés:
"extrae datos de GFW", "corre el extractor GFW", "presencia AIS del período X",
"rasters de pesca aparente de GFW".

## Operación paso a paso

1. **Configura las 4 rutas placeholder** al inicio de `scripts/01extractorGfw.py`
   (reemplaza cada valor entre corchetes por tu propia ruta real):

   | Variable | Placeholder a reemplazar | Qué debe apuntar |
   |:---|:---|:---|
   | `RUTA_TOKEN` | `[RUTA_TOKEN_GFW]` | Un archivo `.env` con tu propio token de la API de GFW |
   | `RUTA_GEOJSON` | `[RUTA_GEOJSON_AOI]` | El geojson del área completa que vas a monitorear |
   | `RUTA_GEOJSON_AOI` | `[RUTA_GEOJSON_AOI_PESCA]` | El geojson del polígono usado en el sub-pipeline de pesca aparente |
   | `RUTA_RASTER_SALIDA` | `[RUTA_BASE_RASTERS_SALIDA]` | La carpeta base donde se guardarán los rasters TIF |

   **Importante:** esta skill no incluye geojson de ningún área específica — cada
   equipo debe traer sus propios polígonos de interés.

2. **Edita el período de consulta**: `PERIODO` (rango principal de presencia
   AIS/SAR) y `PERIODO_PESCA` (rango del sub-pipeline de pesca aparente, en el
   script original de origen se calcula como los ~3 meses previos a
   `PERIODO['start']` — ajusta según tu propio criterio de negocio).

3. **Ejecuta el script** desde la carpeta donde quieres recibir los outputs:
   ```bash
   python3 scripts/01extractorGfw.py
   ```
   Esto corre en secuencia: `gfwApiExtractor.ejecutar(...)` (presencia AIS/SAR) y
   dos llamadas a `gfwApiExtractor.ejecutarPesca(...)` (rasters de pesca, alta y
   baja resolución).

4. **Revisa los outputs**: CSV de presencia/pesca en la carpeta de ejecución,
   JSON crudos en `./json/`, rasters TIF en la carpeta configurada en
   `RUTA_RASTER_SALIDA`.

## Troubleshooting

- **HTTP 401/403 al llamar la API**: el token en `RUTA_TOKEN` es inválido, venció,
  o el archivo `.env` no tiene el formato esperado. Verifica el token en tu cuenta
  de GFW.
- **HTTP 422 "group-by option not valid"**: la API de GFW exige el parámetro
  `group-by` en todos los datasets de `/v3/4wings/report` (presencia, SAR, fishing
  effort) — sin él, retorna este error con un mensaje engañoso. El script ya
  envía `group-by: VESSEL_ID` en las 3 llamadas (presencia AIS, SAR, fishing
  effort) — si ves este error de todas formas, probablemente modificaste
  `_construirParams` o pasaste `groupBy=None` en una llamada nueva propia.
- **Rasters vacíos o `no solicitados`**: revisa que `RUTA_RASTER_SALIDA` y
  `RASTER_GROUP_BY_CONFIG` no sean `None` — el script solo genera rasters si
  ambos están configurados.
- **`ModuleNotFoundError: requests` / `pandas`**: no corriste
  `pip install -r requirements.txt`, o lo corriste en un entorno virtual distinto
  al que usas para ejecutar el script.
- **El geojson de área no aplica a tu zona**: recuerda que esta skill viene sin
  geojson — debes traer el tuyo y apuntar `RUTA_GEOJSON`/`RUTA_GEOJSON_AOI` a él.

## Recomendaciones

- Nunca subas tu archivo `.env` con el token de GFW a un repositorio — agrégalo a
  `.gitignore` del proyecto destino.
- Guarda tus geojson de área de interés junto al proyecto, pero fuera del control
  de versiones si contienen información sensible de ubicación de operaciones.
- No hay capas de detección óptica, night lights, ni VMS fishing effort
  disponibles vía la API pública de GFW — no esperes que este extractor las
  produzca.
- Si vas a monitorear varias áreas en paralelo, considera mantener una copia de
  este script por área con su propio set de 4 rutas configuradas, en vez de
  editar las mismas variables repetidamente.
