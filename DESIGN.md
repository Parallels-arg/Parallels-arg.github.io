---
name: Bruno Echagüe
description: Blog personal de writeups, reversing y notas de seguridad, con estética de terminal en ámbar.
colors:
  amber-phosphor: "#ffb547"
  amber-ember: "#9a4f00"
  amber-glow: "rgb(255 181 71 / 12%)"
  amber-glow-light: "rgb(154 79 0 / 12%)"
  ink-black: "#101113"
  sidebar-black: "#141518"
  card-black: "#17181b"
  raised-black: "#1c1d21"
  rule-dark: "#23252a"
  control-border-dark: "#2a2c31"
  bone-heading: "#ecebe6"
  bone-text: "#c8c6c0"
  ash-label: "#a3a29c"
  ash-muted: "#8e8d88"
  paper: "#fcfcfa"
  paper-sidebar: "#f3f2ee"
  card-white: "#ffffff"
  rule-light: "#e6e4de"
  control-border-light: "#dedcd5"
  graphite-heading: "#161512"
  graphite-text: "#2d2c29"
  stone-label: "#4f4d48"
  stone-muted: "#6b6963"
typography:
  display:
    fontFamily: "JetBrains Mono, ui-monospace, SFMono-Regular, Menlo, Consolas, monospace"
    fontSize: "clamp(2rem, 5vw, 3rem)"
    fontWeight: 700
    lineHeight: 1.1
    letterSpacing: "-0.03em"
  headline:
    fontFamily: "JetBrains Mono, ui-monospace, SFMono-Regular, Menlo, Consolas, monospace"
    fontSize: "2rem"
    fontWeight: 700
    lineHeight: 1.2
    letterSpacing: "-0.02em"
  title:
    fontFamily: "JetBrains Mono, ui-monospace, SFMono-Regular, Menlo, Consolas, monospace"
    fontSize: "1.54rem"
    fontWeight: 600
    lineHeight: 1.2
    letterSpacing: "-0.01em"
  card-title:
    fontFamily: "JetBrains Mono, ui-monospace, SFMono-Regular, Menlo, Consolas, monospace"
    fontSize: "1.25rem"
    fontWeight: 700
    lineHeight: 1.2
  body:
    fontFamily: "Source Sans Pro, Microsoft Yahei, sans-serif"
    fontSize: "1.08rem"
    fontWeight: 400
    lineHeight: 1.75
  lead:
    fontFamily: "Source Sans Pro, Microsoft Yahei, sans-serif"
    fontSize: "1.15rem"
    fontWeight: 400
    lineHeight: 1.65
  label:
    fontFamily: "JetBrains Mono, ui-monospace, SFMono-Regular, Menlo, Consolas, monospace"
    fontSize: "0.8rem"
    fontWeight: 600
    letterSpacing: "0.06em"
  meta:
    fontFamily: "JetBrains Mono, ui-monospace, SFMono-Regular, Menlo, Consolas, monospace"
    fontSize: "0.85rem"
    fontWeight: 400
    lineHeight: 1.75
rounded:
  sm: "4px"
  md: "6px"
  pill: "1rem"
  full: "50%"
spacing:
  xs: "0.25rem"
  sm: "0.5rem"
  md: "0.75rem"
  base: "1rem"
  lg: "1.5rem"
  xl: "2.5rem"
  xxl: "3.5rem"
components:
  button-primary:
    backgroundColor: "{colors.amber-phosphor}"
    textColor: "{colors.ink-black}"
    typography: "{typography.label}"
    rounded: "{rounded.md}"
    padding: "0 1.1rem"
    height: "44px"
  button-primary-hover:
    backgroundColor: "{colors.amber-glow}"
    textColor: "{colors.amber-phosphor}"
  button-outline:
    backgroundColor: "{colors.ink-black}"
    textColor: "{colors.amber-phosphor}"
    typography: "{typography.label}"
    rounded: "{rounded.md}"
    padding: "0 1.1rem"
    height: "44px"
  button-outline-hover:
    backgroundColor: "{colors.amber-glow}"
  card-post:
    backgroundColor: "{colors.card-black}"
    textColor: "{colors.bone-text}"
    rounded: "{rounded.md}"
    padding: "1rem"
  lang-switch-option:
    textColor: "{colors.ash-muted}"
    typography: "{typography.label}"
    rounded: "{rounded.sm}"
    padding: "0 0.65rem"
  lang-switch-option-active:
    backgroundColor: "{colors.amber-glow}"
    textColor: "{colors.amber-phosphor}"
  input-search:
    backgroundColor: "{colors.ink-black}"
    textColor: "{colors.bone-text}"
    rounded: "{rounded.pill}"
    height: "2rem"
  nav-item:
    textColor: "{colors.ash-label}"
    typography: "{typography.label}"
    padding: "0.6rem 1.5rem"
  nav-item-active:
    backgroundColor: "{colors.raised-black}"
    textColor: "{colors.amber-phosphor}"
  skip-link:
    backgroundColor: "{colors.amber-phosphor}"
    textColor: "{colors.ink-black}"
    rounded: "{rounded.md}"
    padding: "0.6rem 1rem"
---

# Design System: Bruno Echagüe

<!--
  Fuente de verdad del código: assets/css/jekyll-theme-chirpy.scss (tokens y estilos propios),
  _includes/metadata-hook.html (switch de idioma) y los overrides en _layouts/ y _includes/.
  El resto viene del tema Chirpy v7.6.0. Los tokens de componentes usan los valores del modo oscuro;
  la sección Colors indica el par de cada uno en modo claro.
-->

## Overview

**Creative North Star: "El cuaderno del reverser"**

El sitio es la libreta de laboratorio de alguien que desarma binarios: preciso, metódico y escrito para entender. Los datos, las rutas, las fechas, los handles y los títulos se anotan en monoespaciada, como salida de una herramienta; el razonamiento largo se escribe en una sans legible, porque un writeup se lee de punta a punta. El ámbar es la tinta del cuaderno: ambienta, marca lo importante y le da calidez a un fondo casi negro.

La densidad es de lectura, no de panel de control. Hay pocas superficies, separadas por tono más que por sombra, y una sola cosa se mueve en todo el sitio: el cursor de bloque al lado del nombre en la home. La identidad se gana con detalles exactos (el foco ámbar, la selección de texto ámbar, los metadatos en mono) y no con decoración.

El verde fósforo quedó descartado de forma explícita: es el cliché del género. El sistema es terminal, pero en ámbar.

**Key Characteristics:**
- Ámbar como único acento, en dos versiones calibradas por tema.
- JetBrains Mono para títulos, etiquetas y datos; Source Sans Pro para la prosa.
- Fondos casi negros con matiz neutro-cálido en oscuro; papel apenas cálido en claro.
- Profundidad por tono, sombras mínimas.
- Esquinas de 6px y bordes de 1px: herramienta bien hecha, sin redondeo blando.
- Bilingüe (EN en la raíz, ES en /es) con la misma identidad.

## Colors

Un acento ámbar sobre neutros cálidos casi negros (oscuro) o papel (claro); cada rol tiene un par por tema.

### Primary
- **Ámbar fósforo** (amber-phosphor): el acento en modo oscuro. Links, estado activo del menú, subtítulo con el handle, cursor de bloque, botón principal, anillo de foco, selección de texto. Contraste 10.7:1 sobre ink-black.
- **Ámbar brasa** (amber-ember): el mismo rol en modo claro, oscurecido para texto: 5.9:1 sobre paper. Nunca usar amber-phosphor como texto en modo claro.
- **Resplandor ámbar** (amber-glow / amber-glow-light): fondos suaves al 12% para estado activo (opción de idioma), hover de botones y realces de bloque.

### Neutral
- **Tinta** (ink-black) / **Papel** (paper): fondo principal.
- **Barra lateral** (sidebar-black) / **Papel lateral** (paper-sidebar): fondo del sidebar, un paso más alto que el principal.
- **Tarjeta** (card-black) / **Tarjeta blanca** (card-white): tarjetas de posts y botones de fondo.
- **Elevado** (raised-black): encabezados de tarjetas, hover del menú, fondo del ítem activo.
- **Línea** (rule-dark / rule-light): divisores y bordes de sección.
- **Borde de control** (control-border-dark / control-border-light): borde del buscador y del switch de idioma.
- **Hueso títulos** (bone-heading) / **Grafito títulos** (graphite-heading): títulos y nombre del sitio.
- **Hueso texto** (bone-text) / **Grafito texto** (graphite-text): texto corrido.
- **Ceniza etiqueta** (ash-label) / **Piedra etiqueta** (stone-label): ítems del menú, labels secundarios. 7.1:1 y 7.5:1.
- **Ceniza apagada** (ash-muted) / **Piedra apagada** (stone-muted): metadatos, fechas, pie. 5.7:1 y 5.3:1; es el piso de contraste del sistema.

### Named Rules
**The Tinta Ámbar Rule.** El ámbar ambienta: puede ir en títulos destacados, bordes, fondos suaves (amber-glow) y estados, además de links y foco. Nunca como color de un bloque de texto corrido, y nunca como texto sobre fondo claro en su versión fósforo.

**The No Green Rule.** Ningún verde como color de identidad. El verde solo aparece donde el tema lo usa con significado (avisos de tipo "tip", resaltado de sintaxis).

**The 4.5 Floor Rule.** Ningún texto del sistema baja de 4.5:1 en ninguno de los dos temas. Los grises "apagados" ya están en el piso; no aclararlos más.

## Typography

**Display Font:** JetBrains Mono (con ui-monospace, SFMono-Regular, Menlo, Consolas)
**Body Font:** Source Sans Pro (con Microsoft Yahei, sans-serif)
**Label/Mono Font:** JetBrains Mono

**Character:** la mono anota y la sans explica. JetBrains Mono le da a los títulos y metadatos el tono de salida de herramienta; Source Sans Pro mantiene cómodos los párrafos largos de un writeup.

### Hierarchy
- **Display** (700, clamp de 2rem a 3rem, 1.1): solo el nombre en la home, seguido del cursor de bloque.
- **Headline** (700, 2rem, 1.2): título de post y de página.
- **Title** (600, 1.54rem, 1.2): secciones dentro de un post (H2); H3 y H4 bajan de tamaño con el mismo peso.
- **Card title** (700, 1.25rem, 1.2): títulos en las tarjetas de la home.
- **Lead** (400, 1.15rem, 1.65): la presentación de la home y la descripción bajo el título de un post, en color apagado. Máximo 60ch.
- **Body** (400, 1.08rem, 1.75): prosa de posts y páginas. Párrafos, listas y citas limitados a 72ch; código, tablas e imágenes usan el ancho completo.
- **Label** (600, 0.8rem, 0.06em, mayúsculas en el menú): menú lateral, botones, switch de idioma, encabezado "Últimos posts".
- **Meta** (400, 0.85rem): fechas, categorías, breadcrumb, licencia, pie.

### Named Rules
**The Mono Annotates Rule.** La mono es para lo que una herramienta escribiría: títulos, etiquetas, fechas, rutas, handles, código. Un párrafo de explicación nunca va en mono.

**The Balanced Heading Rule.** Títulos con `text-wrap: balance`, prosa con `text-wrap: pretty`.

## Layout

Layout de Chirpy: sidebar fijo de 260px a la izquierda (300px desde 1650px), barra superior de 3rem y columna de contenido con un panel lateral a partir de xl. En pantallas chicas el sidebar se esconde detrás del botón de menú y la barra superior muestra título, switch de idioma y buscador.

La home abre con el bloque de presentación (2.5rem arriba, 3.5rem desde 768px, separado por una línea de 1px), luego el encabezado "Últimos posts" y la lista de tarjetas. El ritmo usa la escala de espaciado del frontmatter: pasos chicos (0.25 a 0.75rem) dentro de un grupo, 1.5rem o más entre grupos.

En pantallas táctiles todos los controles miden al menos 44px: botones de la home, opciones EN/ES, menú, buscador, íconos de contacto y de compartir.

## Elevation & Depth

Plano por defecto, profundidad por tono. Cada superficie es un paso de luminosidad (ink-black, sidebar-black, card-black, raised-black) y los divisores son líneas de 1px. La única sombra es la de las tarjetas de posts, suave y con desplazamiento, más un borde de 1px en modo oscuro.

### Shadow Vocabulary
- **Tarjeta oscura** (`box-shadow: rgb(0 0 0 / 45%) 0 6px 18px 0, rgb(255 255 255 / 6%) 0 0 0 1px`): tarjetas de posts en modo oscuro.
- **Tarjeta clara** (`box-shadow: rgb(22 21 18 / 6%) 0 2px 8px 0, rgb(22 21 18 / 9%) 0 0 0 1px`): tarjetas de posts en modo claro.

### Named Rules
**The Tone Before Shadow Rule.** Para separar una superficie, primero subir un paso de tono o agregar una línea de 1px. Una sombra nueva tiene que justificar por qué el tono no alcanza.

## Shapes

Esquinas firmes y chicas: 6px en botones, tarjetas, switch de idioma y skip link; 4px en elementos internos (opciones del switch). Los círculos se reservan para el avatar y los íconos del sidebar. El buscador es la única pastilla (1rem) y viene del tema. Los bordes son de 1px; el foco es un contorno de 2px con 2px de separación.

## Components

### Buttons
Precisos y sobrios: esquinas de 6px, borde de 1px, sin sombra.
- **Shape:** esquinas firmes (6px), alto mínimo de 44px.
- **Primary:** fondo ámbar, texto tinta, mono 600. Uno solo por vista (en la home, "Sobre mí").
- **Outline:** fondo transparente, borde y texto ámbar (en la home, "GitHub").
- **Hover / Focus:** el fondo pasa a amber-glow y el texto a ámbar (200ms, ease-out). El foco es un contorno ámbar de 2px con 2px de separación, igual en todo el sitio.

### Cards / Containers
- **Corner Style:** 6px.
- **Background:** card-black / card-white.
- **Shadow Strategy:** la sombra de tarjeta descrita en Elevation & Depth.
- **Internal Padding:** 1rem.
- **Contenido:** título en mono 700, descripción del post (el campo `description` del front matter) en sans apagada, metadatos en mono.

### Inputs / Fields
- **Style:** buscador en pastilla (1rem), borde de 1px en control-border, 2rem de alto.
- **Focus:** el borde pasa a ámbar con un anillo de 1px; el cursor de escritura es ámbar.
- **Sin resultados:** mensaje en lenguaje claro que sugiere otra palabra o las categorías, anunciado a lectores de pantalla.

### Navigation
- **Sidebar:** avatar (enlaza a "Sobre mí"), nombre en mono 700, handle en ámbar mono, menú en mono 600 mayúsculas con 0.06em de tracking, en ash-label. Ítem activo en ámbar sobre raised-black. Íconos de tema y contacto abajo, en círculos.
- **Barra superior:** breadcrumb en mono, switch EN | ES y buscador a la derecha.
- **Mobile:** sidebar detrás del botón de menú; la barra superior conserva el título, el switch y el buscador.

### Language Switch
Grupo de dos opciones EN | ES en mono 600 de 0.75rem, dentro de un borde de 1px y esquinas de 6px. La opción actual lleva fondo amber-glow y texto ámbar (`aria-pressed`). Mientras verifica que la traducción exista, la opción elegida parpadea con el mismo paso del cursor. En pantallas táctiles el borde desaparece y cada opción mide 44px.

### Block Cursor
Bloque ámbar de 0.55em x 0.95em pegado al nombre en la home, parpadeo en pasos de 1.1s. Es el único elemento animado del sitio y se detiene con `prefers-reduced-motion`.

## Do's and Don'ts

### Do:
- **Do** usar amber-ember (#9a4f00) para todo texto ámbar en modo claro, y amber-phosphor (#ffb547) solo en oscuro.
- **Do** escribir títulos, etiquetas, fechas y datos en JetBrains Mono, y la prosa en Source Sans Pro.
- **Do** separar superficies con un paso de tono o una línea de 1px antes de pensar en una sombra.
- **Do** mantener 6px de radio en controles y tarjetas, y 44px de área táctil en pantallas táctiles.
- **Do** darle a cada post un `description` corto: es el texto de la tarjeta y de la vista previa al compartir.
- **Do** usar el mismo foco ámbar de 2px en todo lo que el teclado pueda alcanzar.

### Don't:
- **Don't** usar verde como color de identidad.
- **Don't** escribir párrafos en monoespaciada.
- **Don't** agregar un segundo color de acento.
- **Don't** animar nada más que el cursor de la home y transiciones de estado de 200ms.
- **Don't** bajar ningún texto de 4.5:1 de contraste en ninguno de los dos temas.
- **Don't** poner etiquetas pequeñas tipo "kicker" encima de los títulos.
