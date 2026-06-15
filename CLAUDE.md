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

Todo —estructura HTML, CSS y JavaScript— está en `index.html` (~1593 líneas). El archivo está organizado con banners de comentario claros:

| Sección | Líneas (aprox.) | Propósito |
|---|---|---|
| `<style>` | 11–470 | Todo el CSS con propiedades personalizadas (tema oscuro) |
| Vistas HTML | 473–660 | Tres paneles `<div class="view">`: `#view-dashboard`, `#view-history`, `#view-settings` |
| Nav inferior | 663–700 | Barra de pestañas fija que llama a `switchView()` |
| `// STATE` | 739–748 | Variables globales con todo el estado en tiempo de ejecución |
| `// ISO WEEK HELPERS` | 750–789 | Aritmética de fechas ISO 8601 (sin atajos UTC — ver más abajo) |
| `// INIT` | 791–835 | `window.onload`: restaura localStorage y arranca el render |
| `// FILE HANDLING` | 836–918 | `handleFile()` — parseo Excel con SheetJS → `allData` |
| `// BUILD WEEKLY STATS` | 919–933 | `buildWeeklyStats()` — `allData` → `weeklyStats` |
| `// GET PROJECTION` | 946–1071 | Algoritmo principal (ver más abajo) |
| `// RENDER *` | 1091–1225 | Funciones de actualización del DOM para cada vista |
| `// CONFIRMADOS` | 1226–1325 | Confirmación, exclusión de clientes, KPI y exportación |
| `// SETTINGS / FESTIVOS` | 1360–1545 | Sliders de umbrales y pesos, `saveConfig()`, festivos |
| `// UI HELPERS` | 1547–1593 | `switchView()`, `showToast()`, `showConfirm()` |

## Estado global

```js
let allData = [];         // {fecha: Date, codigo: string, nombre: string}[]
let weeklyStats = {};     // {isoKey: {week, year, count, clients: {}}}
let threshRed = 100;      // pedidos absolutos → semáforo rojo
let threshGreen = 50;     // pedidos absolutos → semáforo verde
let festivosData = [];    // Date[] — festivos (solo lun-vie afectan la capacidad)
let currentWeekOffset = 0;
let weightConfig = { w1: 70, w2: 30, w3: 20 }; // pesos de los 3 años más recientes (w1=más reciente)
let confirmados = {};     // {"YYYY-WNN": {codigo: nombre, ...}} — clientes que han pedido
let excluidos = {};       // {"YYYY-WNN": {codigo: nombre, ...}} — clientes que no van a pedir
```

Las claves de `weeklyStats` siguen el patrón `"YYYY-WNN"` (ej. `"2025-W03"`). El campo `clients` de cada entrada es un objeto plano indexado por `codigo` con `{nombre, count}`.

## Persistencia (localStorage)

| Clave | Formato | Contenido |
|---|---|---|
| `wl_data` | Array JSON | `allData` con `fecha` como cadena ISO |
| `wl_festivos` | `YYYY-MM-DD` separados por comas | Fechas de festivos |
| `wl_thresh` | JSON | `{red, green}` — umbrales del semáforo |
| `wl_pesos` | JSON | `weightConfig` — pesos de ponderación por recencia |
| `wl_confirmados_YYYY-WNN` | JSON | `{codigo: nombre, ...}` — confirmados por semana |
| `wl_excluidos_YYYY-WNN` | JSON | `{codigo: nombre, ...}` — excluidos por semana |

Umbrales y pesos **no** se auto-guardan; se persisten explícitamente al pulsar **💾 Guardar configuración** en Settings. Confirmados y excluidos se guardan automáticamente al instante en cada acción.

## Flujo de confirmación semanal

Cada tarjeta de cliente en el Dashboard tiene dos acciones:

- **Clic en la tarjeta / ✓** → `toggleConfirmado()` — marca al cliente como confirmado (ha pedido). Se resalta en verde. El **KPI de veracidad** sube.
- **Botón ✕** → `excluirCliente()` — oculta al cliente de la lista (no va a pedir). Si estaba confirmado, se des-confirma primero. Aparece el enlace **"N excluido(s) · Restaurar"**.
- **Restaurar** → `restaurarExcluidos()` — elimina todas las exclusiones de la semana actual y re-renderiza.
- **📥 Exportar semana** → `exportarSemana()` — genera un `.xlsx` con columnas **Fecha** (lunes de la semana ISO), **Código** y **Nombre** de los clientes confirmados. Solo aparece cuando hay al menos un confirmado.

El estado de cada semana es independiente: al navegar con las flechas ‹ › cada semana carga sus propios confirmados y excluidos desde localStorage.

## KPI de veracidad

Barra de progreso visible bajo el encabezado "ESPERAMOS A:". Fórmula:

```
veracidad = confirmados / total_proyectados_original × 100
```

- **Denominador**: `proj.clients.length` capturado al inicio del render, antes de filtrar excluidos. Se guarda en `data-total` del elemento `#kpiDetail` para que `toggleConfirmado()` y `excluirCliente()` puedan actualizarlo sin re-renderizar.
- **Color**: rojo < 40 %, amarillo 40–70 %, verde > 70 %.
- `updateKPI(weekKey, totalProyectados?)` — actualiza la barra. Si `totalProyectados` se omite, lee el valor del `data-total` guardado.

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
- Las actualizaciones de UI tras acciones del usuario (confirmar, excluir) se hacen manipulando el DOM directamente sin re-render completo para mayor fluidez. Solo `restaurarExcluidos()` y `changeWeek()` provocan un `renderDashboard()` completo.
