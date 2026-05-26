# CLAUDE.md

Este archivo proporciona orientación a Claude Code (claude.ai/code) cuando trabaja con el código de este repositorio.

## Descripción del proyecto

**WorkLoad — Proyección de Pedidos** es una aplicación web de una sola página (SPA) en español para la proyección de carga de pedidos. Carga datos históricos de pedidos desde archivos Excel, proyecta los volúmenes esperados por semana ISO y muestra un semáforo de carga. Toda la aplicación vive en un único archivo: `index.html`.

## Ejecutar la aplicación

Sin paso de compilación ni gestor de paquetes. Abrir `index.html` directamente en el navegador, o servirlo con cualquier servidor estático:

```bash
python3 -m http.server 8080
# abrir http://localhost:8080
```

No hay tests, linter ni configuración de CI.

## Arquitectura

Todo —estructura HTML, CSS y JavaScript— está en `index.html` (~1300 líneas). El archivo está organizado con banners de comentario claros:

| Sección | Líneas (aprox.) | Propósito |
|---|---|---|
| `<style>` | 11–453 | Todo el CSS con propiedades personalizadas (tema oscuro) |
| Vistas HTML | 456–609 | Tres paneles `<div class="view">`: `#view-dashboard`, `#view-history`, `#view-settings` |
| Nav inferior | 612–635 | Barra de pestañas fija que llama a `switchView()` |
| `// STATE` | 672–679 | Variables globales con todo el estado en tiempo de ejecución |
| `// ISO WEEK HELPERS` | 681–720 | Aritmética de fechas ISO 8601 (sin atajos UTC — ver más abajo) |
| `// INIT` | 722–750 | `window.onload`: restaura localStorage y arranca el render |
| `// FILE HANDLING` | 752–833 | `handleFile()` — parseo Excel con SheetJS → `allData` |
| `// BUILD WEEKLY STATS` | 835–848 | `buildWeeklyStats()` — `allData` → `weeklyStats` |
| `// GET PROJECTION` | 862–990 | Algoritmo principal (ver más abajo) |
| `// RENDER *` | 1012–1155 | Funciones de actualización del DOM para cada vista |
| `// SETTINGS / FESTIVOS` | 1156–1330 | Sliders de umbrales y pesos, `updatePesos()`, `loadPesos()`, festivos |
| `// UI HELPERS` | 1332–1380 | `switchView()`, `showToast()`, `showConfirm()` |

## Estado global

```js
let allData = [];         // {fecha: Date, codigo: string, nombre: string}[]
let weeklyStats = {};     // {isoKey: {week, year, count, clients: {}}}
let threshRed = 100;      // pedidos absolutos → semáforo rojo
let threshGreen = 50;     // pedidos absolutos → semáforo verde
let festivosData = [];    // Date[] — festivos (solo lun-vie afectan la capacidad)
let currentWeekOffset = 0;
let weightConfig = { w1: 70, w2: 30, w3: 20 }; // pesos de los 3 años más recientes (w1=más reciente)
```

Las claves de `weeklyStats` siguen el patrón `"YYYY-WNN"` (ej. `"2025-W03"`). El campo `clients` de cada entrada es un objeto plano indexado por `codigo` con `{nombre, count}`.

## Persistencia (localStorage)

| Clave | Formato | Contenido |
|---|---|---|
| `wl_data` | Array JSON | `allData` con `fecha` como cadena ISO |
| `wl_festivos` | `YYYY-MM-DD` separados por comas | Fechas de festivos |
| `wl_pesos` | JSON | `weightConfig` — pesos de ponderación por recencia |

Los umbrales **no** se persisten; vuelven a los valores predeterminados (100/50) al recargar. Los pesos **sí** se persisten en `wl_pesos`.

## Algoritmo principal: `getProjection(targetWeek, targetYear)`

1. Recopila todas las entradas de `weeklyStats` donde `week === targetWeek` en todos los años.
2. Separa en `historical` (años < targetYear) y `currentYearData` (año === targetYear).
3. **Ajuste por festivos históricos**: cada festivo fijo (mediante `festivosEnSemanaHistorica`) que caiga lun-vie en una ocurrencia pasada suma +20 al conteo de esa semana, normalizando lecturas artificialmente bajas.
4. **Ponderación por recencia configurable**: los pesos provienen de `weightConfig` (guardado en `wl_pesos`), normalizado automáticamente al 100%. El usuario los ajusta desde Settings con tres sliders. Para 1 año histórico→100%; 2 años→`w1/w2` normalizados; 3 años→`w1/w2/w3`; 4+ años→los más antiguos reparten el peso de `w3` equitativamente.
5. **Mezcla con datos reales del año actual**: si existe `currentYearData`, aporta el 50% y el promedio histórico ponderado el 50% restante.
6. **Reducción por festivos en la semana objetivo**: si `festivosEnSemana(targetYear, targetWeek)` devuelve festivos, se multiplica el conteo proyectado por `(5 - festivos) / 5`.
7. **Probabilidad de clientes**: la probabilidad de cada cliente es su ratio de aparición ponderado por recencia entre todas las fuentes.

El semáforo compara `proj.avgCount` con `capacidadAjustada()` (threshRed menos 20 por festivo) para el rojo, y con `threshGreen` para el verde.

## Manejo de fechas ISO 8601

Un patrón deliberado en todo el código: **nunca usar `new Date(string)` ni métodos UTC** para fechas provenientes de Excel, porque `new Date("2024-01-15")` se parsea como medianoche UTC, lo que desplaza la fecha al día anterior en zonas horarias con offset negativo.

La solución usada en todos lados:
```js
// Parsear las partes directamente en el constructor local
const [y, m, d] = s.split('-').map(Number);
new Date(y, m - 1, d);  // medianoche local, sin desplazamiento UTC
```

`isoCalendar(date)` usa aritmética anclada en el jueves (regla ISO 8601: la semana que contiene el jueves pertenece a ese año ISO). No reemplazar con llamadas a librerías que puedan usar UTC internamente.

## Formato de entrada Excel

El Excel de pedidos debe tener columnas que coincidan (sin distinción de mayúsculas, coincidencia parcial):
- **Fecha**: `fecha`, `date` o `fec`
- **Código de cliente**: `código`, `codigo`, `cod`, `code` o `id`
- **Nombre de cliente**: `nombre`, `cliente`, `name`, `client` o `nom`

El Excel de festivos puede tener cualquier estructura — `handleFestivos()` escanea cada celda buscando seriales numéricos de Excel (rango 40000–60000) o cadenas con formato de fecha.

## Dependencias externas (CDN)

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
```

SheetJS es la única dependencia en tiempo de ejecución. Google Fonts (Syne, JetBrains Mono) se cargan desde CDN solo para visualización y no son críticas.

## Convenciones de UI

- Propiedades CSS personalizadas definidas en `:root` — usar `var(--accent)`, `var(--red)`, `var(--green)`, `var(--yellow)`, etc. para cualquier nuevo elemento estilizado.
- `var(--font-mono)` (JetBrains Mono) para valores numéricos, etiquetas y códigos. `var(--font-display)` (Syne) para títulos y texto de interfaz.
- Las vistas se alternan añadiendo/quitando la clase `active` — solo se muestra un `.view.active` a la vez.
- `showConfirm()` se usa en lugar de `window.confirm()` porque el confirm nativo del navegador está bloqueado en contextos de iframe.
- `showToast(msg)` para feedback no bloqueante al usuario (desaparece automáticamente a los 3 segundos).
