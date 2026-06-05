# Plan de equipo — Evolve (90 min)

**Equipo:** Álvaro · Fernando · Mateo
**Tiempo:** 1 h 30 min
**Stack:** Next.js (App Router) + ElevenLabs Conversational AI Agents + `localStorage`
**Formato:** **web mobile-first / PWA** — diseñada como una app de móvil, demostrable en el navegador.
**Créditos ElevenLabs:** 130 000

> Roles: **Fernando** → diseño (mobile-first, con IA) en paralelo. **Álvaro** y **Mateo** → los 3 bloques funcionales. Todos trabajan a la vez desde el minuto 0.

> Regla de oro: nadie programa el pipeline de audio. La voz la resuelven los **Agents** de ElevenLabs (widget/SDK). Cada uno crea su agente en el dashboard y lo embebe en su página.

---

## 1. Arquitectura (cómo encaja todo)

```
                    Next.js app (un repo, una app)
   ┌──────────────────────────────────────────────────────────┐
   │  /            → Home + selector de bloque                  │
   │  /coach       → BLOQUE 1  (Álvaro)                         │
   │  /personajes  → BLOQUE 2  (Álvaro)                         │
   │  /vocacional  → BLOQUE 3  (Mateo)                          │
   │  diseño/UI    → Fernando (componentes mobile-first)        │
   └──────────────────────────────────────────────────────────┘
                              │  leen / escriben
                              ▼
                ┌──────────────────────────────┐
                │   StudentProfile (JSON)        │
                │   en localStorage              │
                │   → contrato compartido        │
                └──────────────────────────────┘
                              │
                              ▼
                ElevenLabs Conversational AI Agents
                (1 agente por bloque, voz en tiempo real)
```

**La clave de la integración = el contrato `StudentProfile`.** Los tres trabajan contra el mismo objeto. Mientras respeten el contrato, las páginas se integran sin tocar el código del otro.

---

## 2. Contrato compartido — `lib/profile.ts`

Esto lo escribe **una persona en los primeros 10 min** y se sube a `main` antes de que nadie empiece su bloque. Es lo único que NO puede cambiar sin avisar a todos.

```ts
// lib/profile.ts
export type Interest = {
  topic: string;          // "baloncesto", "astronomía", "rap"
  category: "deporte" | "ciencia" | "arte" | "música" | "lectura" | "otro";
  strength: number;       // 1-5, cuánto le interesa
  source: "coach" | "personaje";
};

export type CharacterChat = {
  character: string;      // "Kobe Bryant", "Marie Curie"
  topic: string;
  summary: string;        // resumen corto de lo que habló
};

export type StudentProfile = {
  name: string;
  interests: Interest[];
  chats: CharacterChat[];
  vocationalNotes: string[];   // lo que rellena el bloque 3
  updatedAt: string;
};

const KEY = "evolve_profile";

export const emptyProfile = (): StudentProfile => ({
  name: "",
  interests: [],
  chats: [],
  vocationalNotes: [],
  updatedAt: new Date().toISOString(),
});

export function loadProfile(): StudentProfile {
  if (typeof window === "undefined") return emptyProfile();
  const raw = localStorage.getItem(KEY);
  return raw ? JSON.parse(raw) : emptyProfile();
}

export function saveProfile(p: StudentProfile) {
  p.updatedAt = new Date().toISOString();
  localStorage.setItem(KEY, JSON.stringify(p));
}

export function addInterest(i: Interest) {
  const p = loadProfile();
  p.interests.push(i);
  saveProfile(p);
}

export function addChat(c: CharacterChat) {
  const p = loadProfile();
  p.chats.push(c);
  saveProfile(p);
}
```

> Para la demo, si no os da tiempo a que la IA detecte intereses sola, añadid **botones manuales** ("Me interesa el baloncesto") que llamen a `addInterest`. Lo importante es que el perfil se llene y fluya entre bloques.

---

## 3. Reparto por persona (todos en paralelo desde el minuto 0)

### 🎨 Fernando — Diseño mobile-first / PWA (con IA)
**El look de "app" de todo el producto.** Trabaja en paralelo y va entregando piezas que Álvaro y Mateo enchufan.
- **Primeros 10 min (prioridad máxima, desbloquea a todos):** define el **tema en Tailwind** (paleta, tipografía, radios, sombras) en `app/globals.css` / `tailwind.config` y un **layout shell mobile-first** (`components/AppShell.tsx`: marco tipo móvil, header, bottom-nav entre los 3 bloques). Lo mergea a `main` cuanto antes.
- Genera con IA (v0 / Claude) **componentes React + Tailwind reales** listos para copiar:
  - `components/Card.tsx` (tarjetas de personaje/recomendación)
  - `components/ChatLayout.tsx` (contenedor donde va el widget de voz, con avatar + estado "hablando")
  - `components/ProfileSummary.tsx` (chips de intereses)
- Branding: logo/nombre **Evolve**, splash, icono PWA (`manifest.json`).
- Entrega: que las 3 rutas se vean como **una app de móvil coherente**, no como 3 páginas sueltas.

### 🟦🟩 Álvaro — Bloques 1 y 2: Coach personal + Personajes (`/coach`, `/personajes`)
**Entrada del estudiante + conversaciones inmersivas.** Mismo patrón "hablar con un Agent", por eso van juntos.
- `/coach`: widget del **Agent 1 "Coach"** (voz cálida, motivadora; prompt de coach para adolescentes). Tras la charla → `addInterest(...)` (con botones manuales de respaldo).
- `/personajes`: lee `loadProfile().interests`, muestra **tarjetas de personajes** según esos intereses (baloncesto→Kobe, ciencia→Marie Curie...). Al elegir, abre el widget del **Agent 2 "Personaje"**. Tras la charla → `addChat({character, topic, summary})`.
- Reutiliza el `ChatLayout` y `Card` de Fernando.
- Entrega: el usuario habla con el coach, salen intereses, y desde un interés conversa con un referente.

### 🟥 Mateo — Bloque 3 + esqueleto + integración (`/vocacional`)
**Cierra el recorrido y es el "pegamento" del proyecto.**
- Bloque 3: `/vocacional` lee TODO el `StudentProfile` (intereses + chats), muestra el **perfil evolutivo** (usa `ProfileSummary` de Fernando) y abre el **Agent 3 "Coach vocacional"** con el perfil como contexto → recomienda carreras / universidades / áreas. Voz serena y orientadora.
- **Además, dueño de la base técnica:** crea el repo + `npx create-next-app`, escribe `lib/profile.ts`, crea las 3 rutas vacías + Home, y coordina los merges/integración final.
- Entrega: a partir del contexto acumulado, orientación profesional personalizada + todo integrado.

### Arranque (primeros ~10 min)
- **Mateo:** repo + Next.js + `lib/profile.ts` + 3 rutas vacías + Home → push a `main`.
- **Fernando:** tema Tailwind + `AppShell` mobile-first → push a `main` (desbloquea el look).
- **Álvaro:** crea sus 2 Agents en ElevenLabs y monta `/coach` en cuanto haya rutas.
> Mateo y Fernando empujan a `main` lo antes posible: ese es el cimiento del que todos tiran.

---

## 4. ElevenLabs — agentes y voces

Cada dev crea **sus Agents** en el dashboard de ElevenLabs (Conversational AI) → así no se bloquean entre sí. Fernando no necesita agentes (hace diseño).

| Bloque | Dueño | Agente | Voz sugerida | Prompt (idea) |
|---|---|---|---|---|
| 1 Coach | Álvaro | "Coach Evolve" | cálida, joven, motivadora | "Eres un coach cercano para adolescentes. Pregunta por sus gustos, deporte, hábitos. Anímale a expresarse." |
| 2 Personaje | Álvaro | "Personaje" | según figura | "Eres {personaje}. Habla en primera persona, comparte tu pasión y responde sus curiosidades." |
| 3 Vocacional | Mateo | "Orientador" | serena, confiable | "Eres un orientador vocacional. Con estos intereses {perfil}, sugiere carreras y universidades." |

**Integración del widget:** `<elevenlabs-convai agent-id="..."></elevenlabs-convai>` + el script del embed, o el SDK `@elevenlabs/react`. Cada uno mete su `agent-id`.

### Presupuesto de créditos (130 000)
- Conversational AI consume créditos por minuto de conversación.
- **Repartid ~30k por persona para desarrollo/pruebas** y **dejad ~40k de reserva para la demo final**.
- No dejéis conversaciones abiertas de fondo: cortad el agente al terminar cada prueba.
- Para probar prompts sin gastar voz, testead primero en modo texto en el dashboard.

---

## 5. Timeline (90 min)

| Min | Mateo | Álvaro | Fernando |
|---|---|---|---|
| 0–10 | Repo + Next.js + `lib/profile.ts` + rutas + Home → `main` | Crea Agents en ElevenLabs | Tema Tailwind + `AppShell` mobile-first → `main` |
| 10–65 | Bloque 3 `/vocacional` + coordina merges | Bloques 1 y 2 (`/coach`, `/personajes`) | Componentes (Card, ChatLayout, ProfileSummary) + branding/PWA |
| 65–80 | **Integración:** mergear todo, probar flujo Home → Coach → Personajes → Vocacional con el MISMO perfil | | aplica diseño final a las 3 rutas |
| 80–90 | **Pulir demo:** datos de ejemplo, flujo de principio a fin, ensayar pitch de 2 min | | |

---

## 6. Reglas para que la integración no falle

1. **Nadie toca `lib/profile.ts`** sin avisar a los otros dos.
2. Cada uno trabaja en **su ruta** (`/coach`, `/personajes`, `/vocacional`) → cero conflictos de merge.
3. Componentes compartidos (header, estilos) → carpeta `components/`, pero evitad editarlos a la vez.
4. **Mergead a `main` cada 15-20 min**, no al final.
5. Si algo se atasca, **degradad a manual** (botones que rellenan el perfil) en vez de bloquear la demo.
6. El flujo de la demo manda: lo importante es que se vea **Bloque 1 → 2 → 3 con el perfil pasando entre ellos**.
