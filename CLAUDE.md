# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es

App de control horario / fichaje, móvil primero, **100% local y sin backend**. Toda la aplicación (HTML + CSS + JS) vive en un único `index.html` autocontenido: sin build, sin dependencias, sin librerías de CDN. Los datos se guardan solo en `localStorage` del navegador. Interfaz, identificadores y comentarios en español; mantener ese idioma.

Restricciones de diseño que hay que respetar (vienen de la especificación original, `docs/Promt.txt`):
- No dividir la app en varios archivos ni añadir dependencias externas.
- Ningún dato sale del dispositivo (sin analítica ni llamadas a servicios).

`docs/Promt.txt` es la especificación **inicial**; la app ha evolucionado desde entonces (p. ej. la función de "trayectos"/Google Maps se eliminó). Ante discrepancias manda el código actual.

## Ejecutar y probar

No hay build, lint ni tests automatizados. Para probar hay que abrir la app en un navegador.

- Chrome (y la extensión Claude in Chrome) no deja automatizar `file://`, así que hay que servir la carpeta por HTTP en `localhost`.
- En esta máquina **no hay Node ni Python**. Sirve con un pequeño servidor estático en PowerShell (`System.Net.HttpListener` en `http://localhost:<puerto>/` que devuelva los archivos de la carpeta), lanzado en segundo plano.
- Cada origen (`localhost:<puerto>`) tiene su propio `localStorage`: probar en un puerto dedicado no toca los datos reales del usuario. Usar el mismo puerto conserva los datos de prueba entre sesiones.
- Las funciones de la app son globales (script clásico, no módulo), así que desde la consola/`javascript_tool` se puede llamar directamente a `verDia(fecha)`, `calcDia(fecha)`, `DATA`, `save()`, etc., y simular clics con `document.querySelector('[data-action="..."]').click()`.
- Sin Node no se puede hacer `node --check`; la forma de verificar sintaxis es cargar la página y revisar la consola.
- Para probar descargas: Chrome bloquea en silencio las descargas automáticas repetidas. Se sustituye `window.descargarBlob` en la pestaña por un `fetch(..., {method:'PUT'})` contra el servidor de pruebas (que guarda el archivo); el XLSX lo sigue generando el código real. En esta máquina hay Excel: por COM (`Excel.Application`) se puede abrir el archivo para validarlo y exportarlo a PDF para revisarlo.
- La confirmación de sobrescribir al importar en Equipo es un modal propio de la app (no un `confirm()` del navegador): `confirmarImportarEquipo()` espera a que se pulse «Sobrescribir», así que no hay que hacerle `await` desde `javascript_tool` (se queda colgado).

Despliegue: GitHub Pages desde la raíz de `main` (ver README).

## Arquitectura de `index.html`

El `<script>` está organizado en secciones marcadas con `/* ----- Nombre ----- */` (Utilidades, Modelo de datos, Estado UI, Días de viaje, Calendario laboral, Bajas, Acciones, Render por pestaña, Exportación, Generador XLSX, Informe mensual, Ajustes, Backup, Navegación, Delegación de eventos, Arranque). Buscar por esos títulos para orientarse.

### Estado
- `DATA`: todo el estado persistente, serializado entero en `localStorage['controlHorario_data_v1']` con `save()`. Forma en `defaultData()`.
- **Al añadir un campo nuevo a `DATA` hay que tocarlo en tres sitios**: `defaultData()`, `loadData()` (fusión defensiva campo a campo) e `importarBackupSeleccionado()` (restaurar copia). Si no, se pierde al recargar o al restaurar un backup.
- `UI`: estado efímero de la interfaz (pestaña, filtros, mes/semana de referencia). No se persiste, salvo la última pestaña (clave aparte `controlHorario_ultimaPestana_v1`).

### Render y eventos
- Render imperativo con template strings: cada pestaña tiene su `renderX()` que reescribe su panel (`renderHoy`, `renderHistorico`, `renderResumen`, `renderEquipo`, `renderAjustes`). `renderAll()` repinta banner + pestaña actual. Tras mutar `DATA`: `save()` y luego `renderAll()` (o el `render`/`verDia` concreto).
- Interpolar texto de usuario siempre con `esc()`.
- Modales con `openModal(html)` / `closeModal()` sobre `#modalRoot`.
- **Todos los clics pasan por un único listener delegado** (`document.addEventListener('click', ...)`, sección "Delegación de eventos") que hace `switch` sobre `data-action`; los parámetros van en otros `data-*` y llegan como `t.dataset`. Un botón nuevo = `data-action="..."` en el HTML + un `case` en ese switch. Ojo: `dataset` pasa los nombres a minúsculas/camelCase según la regla de HTML, usar `data-` en minúsculas.
- Un `setInterval` de 1 s (`tickLive`) actualiza los contadores en vivo del fichaje abierto (con `textContent`) y llama a `renderBanner()`.
- **No reescribir cada segundo el `innerHTML` de nada que tenga botones**: si el botón se sustituye entre que el dedo baja y sube, el toque no llega a ser `click` (en el móvil fallaba casi siempre el «Fichar entrada ahora» del aviso). `renderBanner()` solo repinta si cambia el HTML (`_bannerHtml`) y la hora del aviso va aparte en `#avisoFichajeHora`.

### Cálculo de horas (lo más delicado)
- `calcJornada(j)` calcula un **tramo** (un par entrada/salida); `calcDia(fecha)` **agrega todos los tramos del día** y aplica el límite de jornada normal una sola vez. Cualquier cálculo de un día completo (resúmenes, banco de horas, informes) debe usar `calcDia`, nunca sumar `calcJornada`.
- Un tramo pertenece al día de su **entrada** (`j.fecha`), aunque cruce medianoche.
- Fichaje abierto = `salida == null`; mientras está abierto, cuenta hasta `Date.now()`. Solo debe haber uno abierto (lo creado con "Fichar entrada"); las altas manuales exigen entrada y salida.
- Salidas olvidadas: un fichaje abierto de un día anterior **nunca se cierra con la hora actual**; "Fichar salida" y el aviso abren `fichajeAbiertoManual`, que propone `salidaSugerida()` (la hora con la que se cumple la jornada efectiva del día). Cualquier tramo de más de `TRAMO_LARGO_MS` (14 h) pide confirmación (`confirmarTramoLargo`) antes de guardarse, sea al fichar, en alta manual o al editar; un camino nuevo que fije `salida` debe pasar por ahí.
- Jornada exigible por día: `jornadaExigibleMs(fecha)` → 0 en días libres (`tipoDiaLibre`: festivo, vacaciones o día inactivo en Ajustes); si no, según `config.dias` o el tramo de `config.periodos` que cubra la fecha.
- El excedente sobre la jornada **solo es hora extra** si el día está marcado como viaje/reunión/otros (`DATA.viajes`) o, en día libre, si el usuario lo confirmó al fichar salida (`DATA.extraDiaLibre[fecha] === true`). Si no, son horas trabajadas no imputables.
- «Fuera de la oficina» (`DATA.fueraOficina[fecha] === true`, botón dentro del recuadro del contador en Hoy) es **solo informativo**: deja constancia de que los fichajes pueden caer fuera del horario normal, pero no convierte nada en hora extra. Es independiente de la marca de viaje.
- «Jornada prevista» en Hoy = `calc.normalMax` = `jornadaExigibleMs(fecha)`: horas de trabajo efectivo (sin descansos ni comida) exigidas ese día.
- Descanso de 12 h entre jornadas (sección «Descanso de 12 h entre jornadas»). No hay hora fija de entrada por día: la empresa tiene una **franja flexible** común (`config.entradaFlexDesde`–`entradaFlexHasta`, 07:30–08:30, editable en Ajustes).
  - Se aplica solo si el día anterior **imputa extra** (`diaImputaExtra`: viaje/reunión/otros o día libre confirmado) y su salida + 12 h cae después del inicio de la franja (salida > 19:30).
  - Al día siguiente se puede entrar de salida + 12 h a salida + 12 h + duración de la franja (1 h).
  - Si la primera entrada cae en ese margen, se compensa **solo el retraso respecto al fin de la franja normal** (08:30), y solo en la parte de jornada que falte: `min(retraso, jornada − trabajado)`, con el día cerrado (`compensacionDescansoMs`). Si entra más tarde del margen, solo se compensa de 08:30 al final del margen (salida + 12 h + 1 h) y el resto lo recupera él. Salir antes no se compensa, y entrar dentro de la franja normal o antes de cumplir las 12 h tampoco. Hacer la jornada completa conserva las extra.
  - `jornadaExigibleBaseMs` = la de Ajustes; `jornadaExigibleMs` = base − compensación (la que usan los cálculos).
  - Aviso (`avisarDescansoTrasSalida`) al fichar la salida de un día que imputa extra, o al confirmar después que imputa: «Para cumplir con el descanso de 12 h, mañana podrás entrar de X a Y compensándote las horas».
- Banco de horas = extra generada (`calcDia(...).extra` por día) − días de compensación del calendario laboral − horas compensadas por el descanso de 12 h (solo hasta hoy). En los informes estas últimas salen en «Horas compensadas» del día. Todo día con entrada retrasada por el descanso (`descansoEntreJornadas` no nulo, aunque no se descuente nada por haber completado la jornada) lleva en Comentario «Entrada tardía por descanso de 12 h» (`notaEntradaTardiaDescanso`, también en la exportación genérica en días de viaje). `bancoHorasSaldoMs(hastaFecha)` permite el saldo a una fecha pasada.
- Las bajas no suman horas trabajadas: se contabilizan aparte (`horasBajaMsFor`). Son rangos `{inicio, fin}` en `DATA.bajas`; la baja en curso (`fin == null`) se gestiona desde Hoy, y las pasadas se marcan/quitan desde el modal del día del calendario (`abrirFormBajaPasada`, `quitarDiaBaja`). Invariantes: las bajas no se solapan, y un día de baja no puede tener fichajes (se bloquea en ambos sentidos).

### Informes y Excel
- XLSX generado a mano (ZIP sin compresión, método STORE, + Open XML) en "Generador XLSX autocontenido". El lector de XLSX (perfil directivo, pestaña Equipo) **solo entiende los archivos generados por la propia app**; si se reabren y guardan en Excel/LibreOffice quedan comprimidos con DEFLATE y no se pueden importar.
- Informe mensual: el saldo del banco se arrastra de un mes al siguiente (saldo inicial = saldo a fin del mes anterior). Al descargar se guarda una huella del contenido en `DATA.informesDescargados`; si un mes ya descargado cambia, la app avisa de que hay que reenviar ese mes **y todos los posteriores** ya descargados.
  - La huella (`huellaInformeMes`) se calcula sobre las filas de `construirFilasInformeMensual`: un campo nuevo por fila hace que todos los meses descargados parezcan cambiados. Si es opcional, añadirlo solo cuando tenga valor (como `fueraOficina`).
- Columnas del informe mensual (trabajador y equipo, misma función `contextoHojaInformeMensual`): Fecha, Día, Hora inicio, Hora fin, Horas efectivas, Horas extra, Horas compensadas, Horas de baja, Fuera de oficina, Comentario, Tipo. El importador (`parseHojaInformeMensual`) localiza las columnas **por el texto de la cabecera**, así que admite columnas nuevas y archivos antiguos sin ellas. Para añadir una columna con datos hay que tocar: `construirFilasInformeMensual`, `contextoHojaInformeMensual` + `construirBloqueMesInforme`, `parseHojaInformeMensual`, el mapeo de filas en `exportarInformesEquipoCombinado` y `CAMPOS_DIF_DIA` (aviso de cambios al reimportar).

### Perfiles
- Pantalla de identificación con rol `trabajador` o `directivo` (este último añade la pestaña Equipo para importar y fusionar informes de trabajadores). La contraseña se guarda como SHA-256 pero no es seguridad real: es solo para identificar a quién pertenece el dispositivo.

## PWA / service worker

- `service-worker.js` usa **red primero, caché como respaldo**: los cambios publicados se ven sin reinstalar.
- Si se añade o renombra un archivo que deba funcionar sin conexión, añadirlo a `APP_SHELL` y subir `CACHE_NAME` (`control-horario-vN`).
- `manifest.json` define un único acceso directo `index.html?accion=fichar` que ficha entrada o salida según el estado; lo procesa `procesarAccesoDirecto()` (también acepta `?accion=entrada|salida` de instalaciones antiguas).
