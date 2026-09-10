# Control Horario

Aplicación de control horario / fichaje de trabajo, pensada para el móvil, **100% local y sin backend**. Es un único archivo `index.html` autocontenido (HTML + CSS + JavaScript), sin dependencias externas ni servidor: todos los datos se guardan en el propio dispositivo (`localStorage`).

## Funciones principales

- **Fichaje** de entrada/salida, descansos y comidas con un toque.
- **Horas extra ligadas a días de viaje**: solo se contabilizan (y suman al banco de horas) los días marcados explícitamente como "día de viaje" (con lugar y cliente).
- **Banco de horas**: horas extra generadas menos horas ya compensadas, con saldo disponible.
- **Calendario laboral**: marca días como festivo, vacaciones o compensación de horas (horas fijas: 8,5 h de lunes a jueves, 6 h los viernes) directamente sobre el calendario del banco de horas.
- **Histórico** filtrable y buscable, con edición manual de cualquier fichaje, descanso, comida o marca de viaje.
- **Informe mensual coloreado**: exporta un `.xlsx` día a día (viaje en verde, vacaciones en azul, festivo en amarillo, compensación en naranja) con el saldo del banco de horas al inicio y al final del mes.
- **Exportación** genérica a Excel/CSV con columnas configurables, para cualquier periodo.
- **Perfil directivo**: importa los informes mensuales de cada trabajador y descárgalos todos juntos en un único Excel (una hoja por informe) para reenviarlos.
- **Modo claro/oscuro/automático**, copia de seguridad (exportar/importar JSON) y aviso configurable si se te olvida fichar.
- Funciona **sin conexión** y puede instalarse como app (PWA) en la pantalla de inicio.

## Cómo usarlo

No requiere instalación ni build. Basta con abrir `index.html`:

- **En el móvil u ordenador, directamente**: doble clic / abrir con el navegador.
- **Publicado con GitHub Pages** (recomendado para usarlo cómodamente desde el móvil):
  1. Sube este repositorio a GitHub.
  2. En el repositorio: `Settings → Pages → Deploy from a branch → main / (root)`.
  3. Entra en la URL que te da GitHub Pages desde el móvil y usa "Añadir a pantalla de inicio" para tenerlo como una app.

## Estructura del repositorio

```
index.html          La aplicación completa (HTML + CSS + JS)
manifest.json        Manifiesto PWA (icono, accesos directos de fichar entrada/salida)
service-worker.js     Cache para que funcione sin conexión
docs/                 Notas y especificación original del proyecto
```

## Privacidad

Ningún dato (horarios, viajes, clientes, histórico) sale del dispositivo. No hay analítica ni publicidad ni llamadas a servicios externos.

## Aviso

Esta app no es un sistema de fichaje legalmente homologado; es una herramienta personal de control de horas y organización del banco de horas.
