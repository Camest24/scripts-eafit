# Contexto del proyecto: Userscripts para D2L de EAFIT

## Qué es esto

Colección de userscripts (Tampermonkey) para `interactivavirtual.eafit.edu.co`, la plataforma
D2L Brightspace de EAFIT. Cada script es independiente, se instala por separado, y todos corren
a la vez sin pisarse (salvo un par de conflictos conocidos ya resueltos, ver más abajo).

El dueño del proyecto es un estudiante de Ingeniería de Sistemas de EAFIT que no tiene experiencia
programando — todo el trabajo se hizo iterando por chat, con capturas de pantalla y logs de
consola como principal forma de debugging (no hay acceso directo al DOM en vivo desde el asistente
que arma esto).

## Los scripts

| Archivo | Qué hace |
|---|---|
| `eafit-d2l-descargar-todo.user.js` | Botón "Descargar todo el curso" (zip completo) + un ícono por cada módulo de "Contenidos del curso" para descargar solo ese módulo. Usa la API Valence de D2L. |
| `eafit-d2l-tema-liquid-glass.user.js` | Tema visual: fondo blanco, glassmorphism con `backdrop-filter`, corrector automático de contraste (texto blanco sobre fondos oscuros), foto de perfil circular. |
| `eafit-d2l-utilidades.user.js` | Varios "añadidos" chiquitos en un solo script: esconde el banner promocional de Genially, esconde tabs de semestres viejos en "Mis cursos", esconde "Ver próximos eventos", badge "EAFIT+", y el sistema más grande: un calendario personalizado con eventos propios (click/mantener pulsado en un día, colores por materia, multi-evento por día con gajos de color, panel "Eventos para X" conectado). |
| `eafit-d2l-asistente-ia.user.js` | Botón de chat que resume archivos de una materia con IA, vía OpenRouter (una key, varios modelos en cadena de fallback). Extrae texto de PDFs con pdf.js en el navegador para evitar la restricción de saldo de OpenRouter para "archivos". |
| `eafit-d2l-fondo-vaporwave.user.js` | Pone una imagen de fondo (vaporwave) con blur detrás de todo, más una serie de correcciones para que el contenido (texto, íconos, logo, barra de navegación) se vea bien encima de un fondo transparente. **Es el más frágil/en desarrollo activo ahora mismo** — ver "Estado actual" abajo. |
| `eafit-d2l-tema-cute.user.js` / `-emo.user.js` / `-skibidi.user.js` | Temas visuales alternativos (no se usan a la vez que liquid-glass, son opciones descartadas/for-fun). |
| `geoguessr-5k-hype.user.js`, `5000-score-*.html` | **No relacionados a EAFIT** — son de un proyecto GeoGuessr aparte, ignorar para este contexto. |

## Cosas clave del sitio (D2L Brightspace de EAFIT)

- **API**: `GET /d2l/api/versions/` para saber la versión de LE disponible, después
  `GET /d2l/api/le/{version}/{orgUnitId}/content/root/` (módulos raíz) y
  `GET /d2l/api/le/{version}/{orgUnitId}/content/modules/{moduleId}/structure/` (recursivo,
  trae submódulos y topics). Los topics con `TopicType === 1` son archivos, con `Url` real.
- **orgUnitId**: se saca de la URL, patrón `/d2l/(home|le/content)/(\d+)` o query param `?ou=`.
- **Es un SPA**: casi todo hay que reinyectarlo con `setInterval` (1-1.5s típico) o mejor con
  `MutationObserver`, porque D2L re-renderiza secciones sin recargar la página.
- **Widgets "legacy"** (como Actualizaciones/correos) cargan adentro de un `<iframe>` del MISMO
  dominio (`LegacyWidgetViewer.d2l?...`) — accesible vía `iframe.contentDocument`.
- **El banner promocional del home** ("¿Ya conoces la nueva experiencia de contenidos?") es un
  `<iframe src="...view.genial.ly...">` de un dominio EXTERNO (Genially) — ese sí es inaccesible
  por JS (cross-origin), hay que esconder el `<iframe>` en sí (visible desde afuera aunque no se
  pueda leer adentro), no su contenido.

## Los 3 problemas técnicos que se repitieron una y otra vez (leer esto primero)

### 1. Shadow DOM everywhere
Muchos componentes de D2L (Lit/@brightspace-ui) renderizan su contenido adentro de un
`shadowRoot`. `document.querySelectorAll()` normal NO ve adentro. Solución estándar usada en
todos los scripts:

```js
function deepQueryAll(root, selector) {
  const results = [];
  const stack = [root];
  while (stack.length) {
    const node = stack.pop();
    if (!node || !node.querySelectorAll) continue;
    results.push(...node.querySelectorAll(selector));
    node.querySelectorAll('*').forEach((el) => {
      if (el.shadowRoot) stack.push(el.shadowRoot);
    });
  }
  return results;
}
```

Relacionado: al subir por los padres (`el.parentElement`) para buscar un ancestro, la subida se
corta al llegar a la raíz de un shadow root (`parentElement` da `null` ahí). Hay que cruzar el
límite manualmente:

```js
function getParentAcrossShadow(node) {
  if (node.parentElement) return node.parentElement;
  const root = node.getRootNode();
  if (root && root.host) return root.host;
  return null;
}
```

### 2. Textos partidos con `<br>` real (no wrap por CSS)
Varios títulos/headings de D2L vienen partidos en 2 líneas con un `<br>` literal en el HTML, no
por wrap de CSS. Buscar coincidencia exacta de texto en un nodo "hoja" (`children.length === 0`)
fallaba siempre. Solución: tolerar contenedores que solo tengan hijos de puro formato de texto:

```js
const TEXT_ONLY_TAGS = new Set(['BR', 'B', 'STRONG', 'EM', 'I', 'SPAN', 'SMALL']);
function isPureTextContainer(el) {
  for (const child of el.children) {
    if (!TEXT_ONLY_TAGS.has(child.tagName)) return false;
    if (!isPureTextContainer(child)) return false;
  }
  return true;
}
```

### 3. Guerra de `!important` entre distintos userscripts
Dos scripts separados insertando cada uno su propio `<style>` con reglas `!important` en el mismo
selector (ej. `html, body { background: ... !important }`) — gana el que Tampermonkey cargue
**último** en el documento (empate de especificidad/importancia se rompe por orden de aparición).
Esto rompió el fondo del script de vaporwave contra el tema liquid-glass.

**La solución real**: un estilo puesto **en línea** (`el.style.setProperty(prop, val, 'important')`)
le gana a CUALQUIER `!important` de cualquier hoja de estilos, sin importar el orden de carga. Por
eso el script de vaporwave fuerza `document.documentElement.style` y `document.body.style`
directamente en vez de confiar en su propio `<style>` tag para esa pelea puntual.

## Otros patrones aprendidos

- **API keys de servicios externos**: nunca se pegan en el código del script — van en un campo de
  texto dentro de la UI del panel, se guardan en `localStorage` del navegador del usuario. (Hubo un
  incidente donde el usuario pegó su API key real de OpenRouter en el código y la compartió en una
  captura — se le avisó que la revocara.)
- **OpenRouter** (`https://openrouter.ai/api/v1/chat/completions`, formato tipo OpenAI) se usa para
  el asistente de IA: una sola key, varios modelos en cadena (`OPENROUTER_MODELS` array), si uno
  falla/satura prueba el siguiente. Ojo: **subir un archivo** (`type: "file"`) a través de
  OpenRouter exige que la cuenta tenga como mínimo USD 0.50 de saldo cargado, incluso con el motor
  de lectura gratis (`pdf-text`). Por eso el asistente extrae el texto del PDF él mismo con pdf.js
  en el navegador y lo manda como texto plano en el mensaje — evita esa restricción para la
  mayoría de los PDFs (los que tienen texto real seleccionable, no imágenes escaneadas).
- **pdf.js**: mejor cargarlo con un `<script>` dinámico (`document.createElement('script')`) que
  con `@require` — `@require` da un mensaje de error genérico ("internal error") sin decir por qué
  falló. Usar jsdelivr (sirve directo del paquete real de npm) en vez de cdnjs (purga versiones
  viejas sin avisar). La build "legacy" de `pdfjs-dist` expone la librería como
  `window['pdfjs-dist/build/pdf']`, NO como `window.pdfjsLib`.
- **Nombres de modelos de IA cambian seguido** — verificar contra la documentación oficial antes de
  hardcodear un modelo (pasó con Gemini y con los slugs de Claude en OpenRouter, `claude-3.5-haiku`
  ya no existe, es `claude-haiku-4.5`).
- **Debugging remoto**: como no hay acceso directo al navegador del usuario, el flujo de trabajo es
  agregar `console.log`/`console.warn` bien específicos en los puntos donde algo podría fallar,
  pedirle al usuario que pegue el log completo, y iterar. Evitar tragarse errores en silencio
  (`catch (e) {}` sin loggear) — cuesta rondas enteras de ida y vuelta después.

## Estado actual (al momento de este handoff)

El script más nuevo y menos estable es **`eafit-d2l-fondo-vaporwave.user.js`**. Historial de bugs
en este script, más o menos en orden:

1. El fondo cargaba y desaparecía a los pocos segundos → un wrapper grande de D2L se pintaba de
   blanco encima. Arreglado con un barrido que busca wrappers grandes con fondo opaco y los fuerza
   transparentes (`neutralizeWrapperBackgrounds`).
2. El barrido con `setInterval` era muy lento (el blanco aparecía en milisegundos) → se cambió a
   `MutationObserver` + `requestAnimationFrame` para reaccionar casi al instante.
3. Con el tema liquid-glass activo a la vez, el fondo blanco volvía (pelea de `!important` entre
   hojas de estilo, ver punto 3 de arriba) → arreglado forzando `html`/`body` con estilo en línea.
4. Íconos y texto de la barra superior (correo, campana, nombre del usuario) quedaban oscuros e
   invisibles sobre el fondo → se agregó detección "¿tiene algún ancestro con fondo opaco? si no,
   es mi fondo el que se ve, píntalo blanco" (`whitenIconsOverBackground`,
   `whitenTextOverBackground`), cruzando shadow DOM correctamente.
5. **Bug recién resuelto (última iteración)**: el logo de EAFIT es una `<img>` con
   `alt="Mi página de inicio"` (no dice "EAFIT" en ningún lado), adentro de un shadow root — el
   selector original no lo encontraba. Se corrigió el matching y se reemplaza su `src` por una
   versión blanca real que el usuario encontró.
6. **Bug recién resuelto**: el primer intento de matching por `src` conteniendo "eafit" también
   agarró la foto de perfil del usuario (probablemente el hosting de avatares de esa instancia de
   D2L tiene "eafit" en la URL). Se sacó ese chequeo, quedó solo el match por `alt`.
7. **Bug recién resuelto**: los menús desplegables (notificaciones, perfil) mostraban texto blanco
   sobre fondo blanco (ilegible). Causa real: `neutralizeWrapperBackgrounds` los detectaba como
   "wrapper grande" (son anchos) y los volvía transparentes él mismo, así que
   `whitenTextOverBackground` los veía "sin fondo opaco" y los pintaba blancos — un problema
   causado por el propio script, no por D2L. Se agregó una exclusión: nunca tocar elementos con
   `position: fixed` o `absolute` (los wrappers reales de página son `static`/`relative`; los
   menús flotantes casi siempre son `fixed`/`absolute`).

**Este último fix (punto 7) todavía no fue confirmado por el usuario** — es lo primero a validar
si se retoma este script. Si el problema persiste, el siguiente paso sería pedirle al usuario que
inspeccione el menú desplegable directamente (click derecho → Inspeccionar) para ver su
`position` y `background-color` computados reales, en vez de seguir iterando a ciegas.

## Cómo seguir iterando en este proyecto

1. Cada cambio se valida con `node -c archivo.user.js` antes de entregarlo (chequeo de sintaxis).
2. El usuario reemplaza el contenido completo del script en Tampermonkey (no aplica parches a mano).
3. Cuando algo falla, pedir el log de consola (F12 → Console) completo, no solo la última línea.
4. Los `console.log`/`console.warn` con prefijo `[EAFIT downloader]`, `[EAFIT utilidades]`,
   `[EAFIT IA]` ya están sembrados en varios puntos clave — son la principal herramienta de
   diagnóstico disponible.
5. El usuario prefiere respuestas directas y cortas cuando no está en medio de debugging; durante
   debugging, está dispuesto a iterar rápido con capturas y logs.
