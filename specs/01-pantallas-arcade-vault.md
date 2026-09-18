# SPEC 01 — Pantallas visuales del MVP de Arcade Vault

> **Status:** Aprovado
> **Depends on:** (ninguno)
> **Date:** 2026-09-18
> **Objective:** Portar a Next.js (App Router) las 5 pantallas visuales del prototipo de Arcade Vault (biblioteca, detalle, reproductor, salón de la fama y autenticación) con datos y sesión simulados, sin implementar lógica de juego real.

---

## Por qué existe este spec

El diseño visual ya está resuelto al 100% en `references/resources/templates/` (HTML + React vía CDN, sin build). El trabajo de este spec no es diseñar, sino portar ese prototipo a la arquitectura real del proyecto (Next.js 16 / App Router / React 19 / TypeScript / Tailwind 4), decidiendo cómo se traducen sus patrones (routing por hash, componentes `.jsx` sueltos, estado de sesión en el componente raíz) a convenciones idiomáticas de Next.js. El tema visual (`app/globals.css`, fuentes en `app/layout.tsx`) ya fue portado en una rama anterior y no se toca aquí.

---

## Scope

**In:**

- 5 rutas de App Router, todas en español:
  - `/` → Biblioteca
  - `/juego/[id]` → Detalle del juego
  - `/juego/[id]/jugar` → Reproductor (pantalla de partida)
  - `/salon-de-fama` → Salón de la Fama
  - `/auth` → Autenticación
- `Nav` (barra desktop + panel móvil con hamburguesa) y `footer` viviendo en `app/layout.tsx`, visibles en las 5 rutas.
- Sesión de usuario simulada vía `AuthProvider` (React Context) que persiste en `localStorage` bajo la clave `av_user`, expuesta a través de `useAuth()`.
- Guardado de puntuaciones simuladas en `localStorage` bajo la clave `av_scores` al terminar una partida.
- Datos mock tipados en `app/data.ts`: juegos (`GAMES`), categorías (`CATS`), jugadores (`PLAYERS`) y el generador determinístico de tablas de puntuación (`seededScores`), migrados 1:1 desde `data.jsx`.
- Componentes compartidos en `components/`: `nav.tsx` y `game-card.tsx` (con el efecto *tilt* al pasar el mouse).
- Réplica fiel del diseño, textos en español, clases CSS (`app/globals.css`, ya existentes) e interacciones visuales de las 5 pantallas originales:
  - Biblioteca: buscador, chips de categoría, grilla de tarjetas.
  - Detalle: portada, tags, estadísticas, tabla de mejores puntuaciones, botón "Jugar ahora".
  - Reproductor: HUD (jugador/puntaje/vidas/nivel), pantalla "CRT" con arena de placeholders, pausa, modal de fin de partida con formulario para guardar puntuación.
  - Salón de la Fama: tabs por juego, podio (oro/plata/bronce), tabla completa, fila "tu mejor marca" cuando hay sesión iniciada.
  - Auth: tabs iniciar sesión / crear cuenta, formulario, botón "jugar como invitado", botones sociales decorativos (sin integración real).
- La simulación de partida del Reproductor se mantiene idéntica a la plantilla: puntaje que sube solo cada 220 ms y arena visual con enemigos/nave estáticos vía CSS. Es una animación de estado, no un juego jugable.

**Out of scope (para futuros specs):**

- Cualquier lógica de juego real: colisiones, físicas, control por teclado/táctil que afecte el juego.
- Backend, base de datos o autenticación real (validación de credenciales, hashing, sesiones de servidor).
- Persistencia distinta a `localStorage`.
- Metadata SEO por ruta, Open Graph, sitemap.
- Tests automatizados (el proyecto no tiene test runner configurado).
- Temas alternativos (claro/oscuro) — el prototipo usa un único tema oscuro neón.
- Internacionalización — todo el contenido permanece en español, igual que las plantillas.
- Integración real con Google/GitHub en el login (quedan como botones decorativos).

---

## Data model

```ts
// app/data.ts
export interface Game {
  id: string;
  title: string;
  short: string;
  long: string;
  cat: "ARCADE" | "PUZZLE" | "SHOOTER" | "VERSUS";
  cover: string; // clase CSS, ej. "cover-bricks"
  color: "cyan" | "magenta" | "yellow" | "green";
  best: number;
  plays: string;
}

export interface ScoreRow {
  rank: number;
  name: string;
  score: number;
  date: string; // "DD/MM/YYYY"
}

export const GAMES: Game[]; // 8 juegos, migrados de data.jsx
export const CATS: readonly string[]; // ["TODOS", "ARCADE", "PUZZLE", "SHOOTER", "VERSUS"]
export function seededScores(seed: number, count?: number): ScoreRow[];
```

```ts
// lib/auth-context.tsx
export interface AuthUser {
  name: string;
}
// Contexto: { user: AuthUser | null; login: (u: AuthUser | null) => void; logout: () => void }
// Persistencia: localStorage["av_user"]
```

Claves de `localStorage` (idénticas al prototipo):

- `av_user`: `AuthUser | null` serializado.
- `av_scores`: arreglo de `{ game: string; score: number; name: string; at: number }`.

---

## Implementation plan

1. Crear `app/data.ts` con las interfaces `Game`/`ScoreRow` y los datos migrados de `data.jsx` (`GAMES`, `CATS`, `PLAYERS`, `seededScores`), tipados. Sistema sigue compilando (archivo sin consumidores aún).
2. Crear `lib/auth-context.tsx` con `AuthProvider`/`useAuth()` (`user`, `login`, `logout`), leyendo/escribiendo `av_user` en `localStorage` dentro de un `useEffect` (evita mismatch de hidratación). Envolver `{children}` con `<AuthProvider>` en `app/layout.tsx`.
3. Crear `components/nav.tsx`, puerto de `nav.jsx`: usar `next/link` para navegar, `usePathname()` para el estado activo (en vez de comparar `route.name`), y `useAuth()` para mostrar "Iniciar sesión" o el nombre del usuario. Renderizarlo en `app/layout.tsx` junto al `footer` (también migrado desde `app.jsx`), antes de `{children}`.
4. Crear `components/game-card.tsx` (tarjeta con tilt, puerto de `GameCard` en `biblioteca.jsx`) y reescribir `app/page.tsx` como la pantalla Biblioteca: estado de búsqueda y categoría, filtrado sobre `GAMES`/`CATS` importados de `app/data.ts`, enlaces a `/juego/[id]` con `next/link`.
5. Crear `app/juego/[id]/page.tsx` (Detalle), puerto de `detalle.jsx`: buscar el juego en `GAMES` por `params.id`, llamar `notFound()` si no existe, generar la tabla de mejores puntuaciones con `seededScores`, enlazar a `/juego/[id]/jugar` y de vuelta a `/`.
6. Crear `app/juego/[id]/jugar/page.tsx` (Reproductor) como client component (`"use client"`), puerto de `reproductor.jsx`: HUD, arena CRT, intervalo de puntaje simulado, pausa, modal de fin de partida con input de nombre, guardado en `localStorage["av_scores"]` vía `useAuth()` para prellenar el nombre.
7. Crear `app/salon-de-fama/page.tsx` (Salón de la Fama), puerto de `salon.jsx`: tabs por juego (`GAMES`), podio top 3, tabla completa con `seededScores`, fila "tu mejor marca" condicionada a `useAuth().user`.
8. Crear `app/auth/page.tsx` (Auth), puerto de `auth.jsx`: tabs iniciar sesión/crear cuenta, formulario controlado, botón "jugar como invitado", y tras enviar el formulario llamar `useAuth().login(...)` y redirigir a `/` con `useRouter().push("/")`.
9. Eliminar el scaffold sin uso: `public/next.svg`, `public/vercel.svg` (ya no quedan referenciados tras el paso 4).
10. Recorrido final manual: `npm run dev` dentro de `05-arcade-vault/`, navegar las 5 pantallas y sus enlaces cruzados; correr `npm run lint` y `npm run build` y confirmar que ambos terminan sin errores.

---

## Acceptance criteria

- [ ] `npm run dev` sirve `/` y muestra la Biblioteca con las 8 tarjetas de juego, buscador y chips de categoría funcionando (filtran la grilla en tiempo real).
- [ ] Al hacer clic en una tarjeta o en su botón "JUGAR", se navega a `/juego/<id>` y se ve la portada, tags, estadísticas y tabla de mejores puntuaciones de ese juego.
- [ ] En `/juego/<id>`, el botón "JUGAR AHORA" navega a `/juego/<id>/jugar` y muestra el HUD con puntaje incrementando automáticamente cada ~220ms.
- [ ] En el Reproductor, "PAUSA" detiene el incremento de puntaje y muestra el overlay "EN PAUSA"; "FIN" abre el modal de fin de partida con el puntaje final.
- [ ] Guardar la puntuación en el modal de fin de partida agrega una entrada a `localStorage["av_scores"]` y muestra el mensaje de confirmación.
- [ ] `/salon-de-fama` muestra tabs por cada juego; cambiar de tab actualiza el podio (oro/plata/bronce) y la tabla completa.
- [ ] Con sesión iniciada, `/salon-de-fama` muestra la fila "tu mejor marca"; sin sesión, esa fila no aparece.
- [ ] `/auth` permite alternar entre "Iniciar sesión" y "Crear cuenta"; enviar cualquiera de los dos formularios (o "jugar como invitado") guarda la sesión en `localStorage["av_user"]` y redirige a `/`.
- [ ] Con sesión iniciada, el `Nav` muestra el nombre de usuario en vez del botón "Iniciar Sesión"; hacer clic cierra sesión y limpia `localStorage["av_user"]`.
- [ ] El menú móvil (icono hamburguesa) abre y cierra el panel lateral en viewports angostos, con los mismos enlaces que el `Nav` de escritorio.
- [ ] `npm run lint` y `npm run build` terminan sin errores.

---

## Decisions

- **Sí:** rutas nativas de App Router (`/`, `/juego/[id]`, `/juego/[id]/jugar`, `/salon-de-fama`, `/auth`) en vez de una sola ruta con estado tipo hash-routing. Da URLs reales y compartibles, y es el patrón idiomático de Next.js 16.
- **No:** replicar el enrutamiento por hash de `app.jsx` (`location.hash` con JSON serializado). Tenía sentido en un prototipo sin build, no en Next.js.
- **Sí:** `Nav` y `footer` viven una sola vez en `app/layout.tsx`. Evita duplicarlos en cada `page.tsx` y es el patrón estándar de layouts compartidos.
- **Sí:** sesión de usuario vía `AuthProvider` (Context) en vez de leer `localStorage` de forma independiente en cada pantalla. El `Nav` vive en el layout y necesita conocer al usuario sin prop-drilling entre rutas que no tienen relación padre-hijo entre sí.
- **Sí:** mantener la simulación de puntaje automático y la arena de placeholders del Reproductor tal cual la plantilla. Ya es "solo visual" (no hay juego real detrás), y quitarla dejaría la pantalla sin ningún feedback dinámico que mostrar.
- **Sí:** componentes compartidos en `components/` (raíz del proyecto) y datos mock en `app/data.ts`. Separa UI reusable de las rutas sin alejar los datos de `app/`.
- **Sí:** slugs de ruta anidados `/juego/[id]` y `/juego/[id]/jugar`, reflejando que el reproductor es una sub-vista del detalle del juego.
- **Sí:** eliminar `public/next.svg` y `public/vercel.svg` del scaffold de `create-next-app`. Ya no los referencia ninguna pantalla tras reemplazar `app/page.tsx`.
- **No:** tocar `app/favicon.ico`. Ninguna plantilla define un ícono nuevo; queda fuera de este spec.
- **No:** usar el skill `/frontend-design` para esta implementación. El diseño ya está completamente resuelto en las plantillas; el trabajo es de portado, no de diseño.

---

## Risks

| Riesgo | Mitigación |
| --- | --- |
| Mismatch de hidratación al leer `localStorage` (`av_user`, `av_scores`) durante el render inicial en el servidor | Leer `localStorage` solo dentro de `useEffect` en `AuthProvider` y en el Reproductor; renderizar el estado "sin sesión" hasta que el efecto corra en cliente. |
| El intervalo de puntaje simulado del Reproductor sigue corriendo si el usuario navega fuera sin desmontar correctamente | Limpiar el `setInterval` en el `return` del `useEffect`, igual que en `reproductor.jsx`. |
| `localStorage` deshabilitado (modo privado) | Los `try/catch` ya presentes en el prototipo se mantienen: si falla, la sesión/puntuación simplemente no persiste, sin romper la pantalla. |

---

## What is **not** in this spec

- Lógica de juego real para cualquiera de los 8 juegos.
- Backend, autenticación real o base de datos.
- Tests automatizados.
- SEO, Open Graph o metadata avanzada por ruta.
- Temas alternativos o internacionalización.

Cada uno de estos, si se necesita más adelante, va en su propio spec.
