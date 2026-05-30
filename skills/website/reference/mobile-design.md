# Mobile design

Mobile-first **siempre**: diseña para el viewport angosto primero y expande en breakpoints (`sm:`/`md:`/`lg:`). Lo que se ve bien en desktop puede aplastarse en un teléfono — valida el caso angosto, no lo asumas.

## Tabs / pills / chips que no caben → una sola línea con scroll horizontal

Cuando una fila de tabs (nav de un admin), pills o chips no cabe en el ancho del móvil, **NO la apiles en varias filas con `flex-wrap`** — eso desperdicia alto vertical y se ve desordenado. Usa **una sola línea con scroll horizontal** (el patrón canónico: Gmail, Instagram, Twitter):

- **Contenedor:** `flex flex-nowrap overflow-x-auto` + **`min-w-0`** (la clave). Opcional `md:overflow-x-visible` para revertir el scroll en desktop, donde sí sobra ancho.
- **Cada item:** `shrink-0 whitespace-nowrap` — para que no se compriman ni se partan en dos líneas.

```tsx
<nav className="flex min-w-0 items-center gap-1 overflow-x-auto md:overflow-x-visible">
  {tabs.map((t) => (
    <Link key={t.href} href={t.href} className="shrink-0 whitespace-nowrap rounded-lg px-3 py-1.5 text-sm font-medium">
      {t.label}
    </Link>
  ))}
</nav>
```

**Por qué `min-w-0`:** un flex item tiene `min-width: auto`, así que se niega a encoger por debajo de su contenido. Sin `min-w-0`, el contenedor de tabs **empuja a sus vecinos** (logo, menú) fuera del viewport en vez de hacer scroll. Con `min-w-0` sí encoge, el contenido se desborda, y `overflow-x-auto` lo vuelve scrolleable. En touch la barra de scroll es overlay (se auto-oculta), así que se ve limpio.

**Patrón validado** — para una barra de tabs de admin que puede crecer, pasarla de wrap → scroll horizontal de una sola línea (en vez de envolver). Es el **default** para cualquier barra de tabs / segmented control que pueda crecer (admins con muchas secciones, filtros, categorías).

## Grid items que se recortan en tablet/móvil

Un grid item también tiene `min-width: auto` y se niega a encoger bajo su contenido → recorta contenido **en silencio** (sin scroll, sin aviso) cuando el contenedor tiene `overflow: hidden`. Fix:

- Columnas con **`minmax(0, 1fr)`** en vez de `1fr`.
- **`min-width: 0`** en los items.
- Red de seguridad global: `html, body { overflow-x: clip }` — un overflow accidental se ve como recorte limpio en el borde del viewport, no como un scroll horizontal fantasma de toda la página ("¿dónde quedó la mitad?").

## Otros básicos mobile-first

- **Toca-objetivos ≥ 44px.** Botones e items tocables con padding suficiente (`py-2`+) para el dedo, no solo para el cursor.
- **Densidad por breakpoint:** compacta padding/tipografía en móvil (`p-2 sm:p-4`, `text-sm sm:text-base`) en vez de meter el layout de desktop a la fuerza.
- **El POS/admin en tablet** suele ser el caso real (no el teléfono): valida en ~800px portrait, no solo en 375px.
