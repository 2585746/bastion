# Prompt para Claude Code — Portafolio web de Bastión

> **Cómo usarlo:**
> 1. Crea en GitHub un repositorio **vacío** llamado `bastion-web` y clónalo en tu PC.
> 2. Copia dentro, en una carpeta `fuentes/`, estos archivos del repositorio `bastion`: `docs/00-estado-actual.md`, `docs/03-incidencias.md`, `docs/plan-maestro.md`, `docs/ROADMAP.md`, tu CV en PDF (español e inglés) y una foto tuya si quieres que aparezca.
> 3. Rellena el bloque **MIS DATOS** de abajo.
> 4. Abre Claude Code en esa carpeta y pega todo el texto que hay a partir de la línea `---INICIO DEL PROMPT---`.
> 5. Trabaja por fases: revisa lo que entrega en cada una antes de decirle que continúe.

---INICIO DEL PROMPT---

Vas a construir mi portafolio web personal. Soy Martín De Prada, técnico de soporte IT y estudiante de administración de sistemas (ASIR). La web presenta mi proyecto **Bastión**: un laboratorio de infraestructura y ciberseguridad autoalojado, con IA integrada y nube híbrida. El público son reclutadores y responsables técnicos de empresas en España, Alemania y Países Bajos. Tienen poco tiempo: en 30 segundos deben entender qué sé hacer y poder comprobarlo.

Trabaja por fases y **detente al final de cada una** para que yo revise. No avances a la siguiente fase sin mi confirmación.

## MIS DATOS (rellenados por mí)

- Nombre a mostrar: Martín De Prada
- Titular: [EJEMPLO: IT Support Technician · Systems & Security]
- Correo de contacto: [ESCRIBE AQUÍ O "NO MOSTRAR"]
- LinkedIn: [URL]
- GitHub: https://github.com/2585746
- Ubicación a mostrar: [EJEMPLO: "España · abierto a reubicación en Alemania y Países Bajos"]
- Dominio final: [TODAVÍA NO / tudominio.com]
- ¿Mostrar foto?: [SÍ, está en fuentes/foto.jpg / NO]

Si algún dato está entre corchetes sin rellenar, pregúntame antes de inventarlo.

## FUENTES DE CONTENIDO

Todo el contenido técnico sale de la carpeta `fuentes/`. Léela entera antes de escribir nada.

- `00-estado-actual.md`: qué hay construido y cómo.
- `03-incidencias.md`: 25 incidencias reales con síntoma, diagnóstico, causa, solución y aprendizaje.
- `plan-maestro.md` y `ROADMAP.md`: hacia dónde va el proyecto y el avance.
- CV en PDF: experiencia y formación.

**Reglas de contenido, sin excepciones:**
1. **No inventes nada.** Ni tecnologías, ni cifras, ni experiencia, ni certificaciones. Si falta un dato, deja un marcador `TODO:` visible y dímelo en el resumen de la fase.
2. Distingue lo **hecho** de lo **planificado**. Lo que en el ROADMAP no está marcado como hecho se presenta como "en curso" o "próximamente", nunca como logro.
3. Escribe en primera persona, con frases cortas y concretas. Sin adjetivos vacíos ("apasionado", "innovador", "robusto"). Cada afirmación se apoya en algo verificable.
4. Español **neutro** (no de España: sin "vosotros", "ordenador" ni modismos peninsulares) e inglés natural. Las dos versiones se redactan, no se traducen palabra por palabra.

## PRIVACIDAD Y SEGURIDAD (crítico)

Es la web de un proyecto de seguridad: una fuga aquí lo desacredita. **Nunca publiques:**

- Direcciones IP de ningún tipo (ni privadas `192.168.x.x`, ni de Tailscale `100.x.x.x`), nombres de host internos completos ni dominios `*.home.arpa`. Usa nombres genéricos: "el hipervisor", "la VM de Docker".
- El nombre de la red Tailscale, temas de ntfy, identificadores de webhooks, tokens, claves ni nombres de usuario de sistemas.
- Mi teléfono, dirección postal, localidad exacta, fecha de nacimiento ni el nombre de mi empresa cliente más allá de lo que aparezca en el CV.
- Marca y modelo del router ni el proveedor de internet.

Al terminar cada fase, ejecuta una búsqueda en todo el proyecto de estos patrones y muéstrame el resultado: `192\.168\.`, `100\.\d+\.\d+\.\d+`, `home\.arpa`, `ntfy\.sh/`, `webhook/`, `@gmail`. Solo puede aparecer mi correo de contacto si autoricé mostrarlo.

Los diagramas de arquitectura muestran **capas y flujos**, no direcciones.

## STACK TÉCNICO

- **Astro** (última versión estable), salida **100 % estática**. Sin backend ni base de datos.
- TypeScript estricto. Contenido en **Markdown/MDX** con *content collections* y esquemas validados.
- CSS propio con variables (sin Tailwind ni librerías de componentes). JavaScript del lado del cliente: el mínimo imprescindible.
- Fuente Poppins **autoalojada** (paquete `@fontsource/poppins`), sin llamadas a Google Fonts.
- Sin analítica, sin cookies, sin rastreadores, sin recursos de terceros. Así no hace falta banner de cookies.
- Internacionalización con las rutas de Astro: español en `/`, inglés en `/en/`. Selector de idioma visible y etiquetas `hreflang`.
- Diagramas en SVG propios o Mermaid renderizado en la compilación (no en el navegador).

## SISTEMA DE DISEÑO (obligatorio, es la identidad del proyecto)

Tema oscuro, minimalista y elegante, con un único color de acento rojo.

**Colores** (defínelos como variables CSS):
- Fondo principal `#141414` · fondo secundario para bloques alternos `#1A1A1A`
- Tarjetas `#1F1F1F` · bordes y divisores `#2A2A2A`
- Acento rojo `#E02B2F` · gradiente rojo de `#D4202B` a `#A3121C` a 160deg
- Texto principal `#FFFFFF` · texto secundario `#B5B5B5` · íconos neutros `#6B6B6B`

**Tipografía:** Poppins, pesos 400, 500, 600, 700 y 800.
- Títulos: blancos, peso 700–800, interlineado compacto.
- Etiquetas sobre los títulos: pequeñas, en rojo, en negrita.
- Párrafos: 13–15 px, peso 400, interlineado 1.8, color gris secundario.
- Botones, enlaces de acción y menú: mayúsculas, 11–12 px, peso 600, espaciado de 2–3 px.
- Cifras destacadas: grandes, peso 800, en rojo.

**Forma:** esquinas rectas o casi (radio de 0 a 2 px). Sin sombras pesadas: la profundidad sale de superponer tonos de gris. Mucho espacio entre secciones. Transiciones suaves de 0.3 s.

**Componentes:**
- Botón principal: fondo rojo sólido, texto blanco en mayúsculas, rectangular, relleno amplio; al pasar el cursor, un tono más oscuro.
- Enlace de acción: texto rojo en mayúsculas con una flecha `→` al final.
- Tarjeta: fondo gris oscuro, relleno generoso, ícono de línea fina en rojo arriba, título blanco y texto gris.
- Tarjeta destacada: fondo con el gradiente rojo y todo el contenido en blanco. **Como máximo una por sección.**

El rojo se usa con moderación: botones, cifras, íconos, enlaces y un único elemento resaltado por sección. Si una pantalla tiene mucho rojo, está mal.

Íconos: de línea fina (*outline*), en SVG incrustado. Sin emojis.

## ESTRUCTURA DEL SITIO

1. **Inicio**
   - Encabezado: nombre, titular, una frase que diga qué hago y qué busco, y dos botones: "Ver Bastión" y "Descargar CV".
   - Franja de cifras reales sacadas de las fuentes (por ejemplo: incidencias documentadas, servicios en producción, puertos abiertos en casa: 0). Solo cifras que pueda demostrar.
   - Tres o cuatro tarjetas con lo que sé hacer, cada una enlazada a la evidencia.
   - Últimas entradas del blog y acceso a las incidencias.

2. **Bastión** (la página principal del proyecto)
   - Qué es y por qué existe, en un párrafo.
   - Diagrama de arquitectura por capas (sin direcciones).
   - Capas del proyecto en tarjetas: hipervisor, red y acceso, DNS, monitorización y alertas, copias de seguridad, plataforma Docker, automatización e IA.
   - **Decisiones de diseño**: una lista de decisiones con su porqué (por ejemplo: ningún puerto abierto en casa; credenciales de solo lectura para cada integración; el vigilante no vive dentro de lo que vigila). Es lo que más valora alguien técnico.
   - **Hoja de ruta**: fases con su estado (hecho / en curso / pendiente), generada desde un archivo de datos fácil de actualizar.

3. **Incidencias** (bitácora)
   - Listado filtrable por área (red, DNS, Docker, copias, IA…) con buscador en el cliente.
   - Cada incidencia en su página, con las cinco partes: síntoma, diagnóstico, causa, solución y aprendizaje.
   - Importa las 25 desde `fuentes/03-incidencias.md` a una *content collection*, **aplicando las reglas de privacidad** (sustituye IP y nombres internos por descripciones genéricas).
   - Un bloque inicial con mi método de diagnóstico (las cuatro reglas).

4. **Blog**
   - Listado y página de artículo, con tiempo de lectura, fecha y etiquetas. Canal RSS.
   - Escribe **un solo** artículo de ejemplo a partir de las fuentes: "Cómo construí un agente de IA que vigila mi servidor (y por qué solo puede leer)". Los demás los escribiré yo: déjame una plantilla.

5. **Sobre mí / CV**
   - Resumen profesional, experiencia y formación tomados del CV.
   - Tecnologías agrupadas por área, **sin barras de porcentaje ni niveles inventados**.
   - Botones para descargar el CV en español y en inglés.
   - Contacto: solo los canales que autoricé en MIS DATOS.

6. **Página 404** con el mismo estilo.

Navegación superior fija y sencilla; pie de página con enlaces a GitHub, LinkedIn y RSS, y una línea: "Sitio estático, sin cookies ni rastreadores."

## CALIDAD (requisitos de aceptación)

- **Adaptable:** perfecto en móvil de 360 px, tableta y escritorio. Sin desplazamiento horizontal.
- **Accesibilidad:** HTML semántico, un solo `h1` por página, navegación completa con teclado, foco visible, contraste AA (comprueba el gris secundario sobre los fondos), texto alternativo en imágenes, `lang` correcto en cada versión y respeto de `prefers-reduced-motion`.
- **Rendimiento:** Lighthouse ≥ 95 en rendimiento, accesibilidad, buenas prácticas y SEO. Imágenes optimizadas con el componente de imagen de Astro. Sin JavaScript innecesario.
- **SEO:** título y descripción por página, Open Graph con una imagen propia en el estilo del sitio, `sitemap.xml`, `robots.txt` y datos estructurados `Person` en la página de inicio.
- **Seguridad:** ningún recurso externo. Añade un archivo con las cabeceras recomendadas (CSP estricta, HSTS, `X-Content-Type-Options`, `Referrer-Policy`) listo para mi servidor Caddy.
- **Código:** componentes pequeños y con nombres claros, sin código muerto, comentarios solo donde aporten. `README.md` que explique cómo añadir una incidencia, un artículo y cómo actualizar la hoja de ruta.

## DESPLIEGUE

- Flujo de **GitHub Actions** que compile y publique en **GitHub Pages** en cada `push` a `main`.
- Configura `site` y `base` para que funcione ahora en `https://2585746.github.io/bastion-web/` y, cambiando **una sola variable**, en mi dominio propio más adelante. Documenta ese cambio en el README.
- El resultado de `npm run build` (carpeta `dist/`) debe poder copiarse tal cual a cualquier servidor estático.

## FASES DE TRABAJO

**Fase 1 — Cimientos.** Lee `fuentes/`. Proponme en un mensaje breve: el mapa del sitio, el titular y la frase del encabezado (dos opciones en español y en inglés) y la lista de cifras reales que encontraste. Crea el proyecto Astro, las variables de diseño, la tipografía, el *layout* base, la navegación, el pie y una página de **guía de estilo** en `/estilo` con todos los componentes. Detente.

**Fase 2 — Inicio y Bastión.** Las dos páginas completas en español, con el diagrama y la hoja de ruta. Detente.

**Fase 3 — Incidencias.** La colección, la importación de las 25 con la limpieza de privacidad, el listado con filtros y las páginas individuales. Muéstrame tres incidencias antes y después de la limpieza para que valide el criterio. Detente.

**Fase 4 — Blog y Sobre mí.** Con el artículo de ejemplo, la plantilla y el RSS. Detente.

**Fase 5 — Inglés.** Versión completa en `/en/`, selector de idioma y `hreflang`. Detente.

**Fase 6 — Calidad y despliegue.** Accesibilidad, rendimiento, SEO, cabeceras, flujo de GitHub Actions y README. Ejecuta la compilación, la búsqueda de patrones de privacidad y dame un informe final con: resultados de Lighthouse, lista de `TODO:` pendientes y cualquier dato que no pudiste verificar en las fuentes.

En cada fase: ejecuta `npm run build` y corrige los errores antes de detenerte, haz *commits* pequeños con mensajes claros, y dime con honestidad qué quedó sin terminar.

Empieza por la Fase 1.

---FIN DEL PROMPT---

## Después de que Claude Code termine

1. Revisa tú mismo el sitio en el móvil y en el PC antes de publicarlo.
2. Lee **cada incidencia publicada** buscando datos internos que se hayan colado.
3. Activa GitHub Pages: repositorio → **Settings → Pages → Source: GitHub Actions**.
4. Cuando tengas el dominio y la VM, sigue `runbooks/05-web-publica-google-cloud.md`.
5. Enlaza la web desde tu perfil de GitHub, LinkedIn y el CV.
