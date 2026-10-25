# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Proyecto

Guía web no oficial, en español, para **DWARF** (juego de rol solitario de Tavern Lore). Acompaña al jugador en la creación de personaje y en el procedimiento diario de turnos: indica qué dado tirar, acepta el resultado (o tira por él), muestra la tabla consultada con la fila resaltada y pasa al siguiente paso. Todo el texto de la interfaz, los comentarios del código y las entradas de la bitácora están en español.

## Ejecución y desarrollo

No hay compilación, gestor de paquetes, linter ni tests. Toda la aplicación es `index.html` (`<style>` y `<script>` en línea): basta con abrirlo en el navegador o servir la carpeta de forma estática (p. ej. `python -m http.server`) para que carguen los fondos de `imagenes/`. Google Fonts es la única dependencia externa. Los cambios se verifican manualmente en el navegador.

## Referencia de reglas

`DWARF reglas es.md` es la traducción al español de las reglas y tablas del juego. Las constantes de tablas de `index.html` (`BG`, `WEATHER`, `POI`, `ENC`, `BEAST`, `SCROLLS`, `MAGIC`, etc.) deben mantenerse coherentes con ese archivo. Desviación intencionada conocida: la tabla de Tesoro usa 11–16 / 17–19 (el manual tiene la errata 11–17 / 17–19).

## Arquitectura (script de `index.html`)

El estilo del código es denso, en líneas largas casi minificadas; hay que respetarlo. Las secciones se separan con comentarios `/* ===== ... ===== */` (UTILIDADES, DADOS, MOTOR DE PASOS, …, VISTAS).

- **Estado**: un único objeto global `S = {char, turn, log, view}` que se guarda en `localStorage` con la clave `dwarf-guia-v1` mediante `save()` (que además vuelve a pintar la barra de estado). Llama siempre a `save()` después de modificar `S`. `ch()` devuelve `S.char`; `blankChar()` define la forma completa del personaje: añade ahí los campos nuevos para que las partidas antiguas reciban valores por defecto. Exportar/Importar (en Inicio) serializa `S` como JSON. `log(título, texto)` añade una entrada a la bitácora (máximo 500).
- **Globales transitorias**: `K` = combate en curso, `G` = mazmorra en curso; no se guardan.
- **Motor de pasos**: el flujo es una cadena de funciones con callbacks de continuación que pintan en `#stage`:
  - `ask(o)`: pide el resultado de un dado (`o.dice` como `'1d6'`, `'2d6'`, `'d66'`; `o.onResult(v)`; opcionales `min/max/reroll/hint/extra/cancel`). Incluye «Tirar por mí».
  - `show(o)`: tarjeta con texto y `o.choices[{label, fn, cls, help}]`; `msg(...)` es el atajo de un solo botón.
  - `test(o)`: prueba de atributo (1d20 + `totalMod(attr)` contra `o.dr`); gestiona el gasto de Juerga/Hongo, críticos y pifias, y llama a `o.onDone(ok, tirada, total)`.
  - `askNum` y `askText` para entrada libre. `o.table` es la etiqueta «Tabla consultada»; `tbl(filas, resaltada)` y `nrows(arr)` generan las tablas.
  - Cada paso se registra con `setCur(...)` en `CURS[S.view]` para que `render()` pueda reproducirlo con `replay()` al volver a la vista. Las funciones suelen recibir un callback `next`/`back`/`after` para continuar.
- **Creación de personaje**: `createIntro` → `cBg` → `cName` → `cTrait` → `cAttr(i)` → `cHP` → `cGold` → `cArmor` → `cWeapon` → `cSupplies` → `cSeason` → `cTerrain` → `cDone`.
- **Turno diario**: `dayStart` → `dWeather` → `dRation` (o `dCave`) → `dPlan` → por hexágono `dHex` → `dPOI` → `dEnc` → `finishHex` → `forageMenu` (forrajear/cazar/pescar) → `campMenu` → `nightEnc` → `sleep`/`dawn`. `S.turn.phase` guarda el progreso; `resumeTurn()` traduce la fase a su función tras recargar.
- **Encuentros → combate**: `encounterResult`/`poiResult` analizan el texto con `detect()`, que usa `KW` (palabra clave en español → clave de `BEAST`) para ofrecer combate vía `combatSetup`/`startCombat`. Al añadir una criatura, añade tanto la entrada en `BEAST` como su palabra clave en `KW` (el orden importa: primero las frases más específicas).
- **Herramientas** (`TOOLS` + `toolsHTML()`): oráculo, pruebas, reacción y generador de PNJ, rumor, combate, mazmorra, tesoro y asentamiento; disponibles en cualquier momento desde la vista de turnos.
- **Vistas**: `render()` decide según `S.view` (`home`, `create`, `turn`, `sheet`, `log`); `go(vista)` cambia de vista. `body[data-view]` y `body[data-season]` controlan en el CSS las imágenes de fondo de `imagenes/` (fondos por estación en turnos, herrería en la creación).
