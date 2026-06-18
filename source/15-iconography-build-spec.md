# Iconography — Build Spec

> A meditative “illuminate the Word” app. The user slowly colors sacred line art —
> like a medieval scribe illuminating a manuscript — while sitting with a verse.
> Calm, offline, single-player, no accounts, no ads.

-----

## 0. How to use this document with Claude Code

1. Put this file in the repo root as `SPEC.md` (or `CLAUDE.md` so Claude Code reads it automatically every session).
1. Build **one phase at a time** (Section 9). Don’t ask Claude Code to “build the whole app” — give it one phase, test on your phone, commit, then move on.
1. After each phase, check it against that phase’s **Acceptance criteria** before continuing.
1. When you hit the spots marked ⚠️ **Gotcha**, slow down — those are where non-coders lose days.

-----

## 1. Product in one paragraph

Iconography is a tap-to-fill coloring app for sacred art. Each artwork is vector line art divided into regions; tapping a region fills it with the selected color from a curated palette. Each piece pairs with a short Scripture verse shown beside the canvas. Finishing a piece triggers a gentle “illumination” (gold-shimmer) moment, after which the user can save the image to their photos or share it. Everything runs locally on the device. There is no login, no server, no user-generated content, and no in-app purchases in v1.

**Why this scope:** zero backend, zero moderation, zero auth → fastest possible path to a shippable native app, and the calmest possible App Store review.

-----

## 2. Explicitly OUT of scope for v1

Do not build these. They are deliberately deferred so v1 ships.

- ❌ User accounts / login / cloud sync
- ❌ Any backend or database server
- ❌ User-generated content, sharing feeds, comments, social features
- ❌ In-app purchases / subscriptions / paywall
- ❌ Freehand drawing or raster “paint bucket” flood-fill (see §6 — v1 is **region fill only**)
- ❌ Audio/soundscapes (nice v1.5 idea; not now)
- ❌ Analytics SDKs (add later if ever)

-----

## 3. Tech stack (use exactly these)

|Concern        |Choice                                                                    |Notes                                                                       |
|---------------|--------------------------------------------------------------------------|----------------------------------------------------------------------------|
|Framework      |**Expo (managed) + React Native + TypeScript**                            |Most AI-codeable native path.                                               |
|Navigation     |**expo-router**                                                           |File-based routing.                                                         |
|Coloring canvas|**react-native-svg**                                                      |Regions = tappable SVG `<Path>`s. The core of the app.                      |
|Animation      |**react-native-reanimated** + **moti**                                    |For the completion shimmer + gentle transitions.                            |
|Local storage  |**@react-native-async-storage/async-storage**                             |Stores coloring progress as JSON.                                           |
|Save image     |**react-native-view-shot** (or `expo` captureRef) + **expo-media-library**|Capture canvas → save to camera roll.                                       |
|Share          |**expo-sharing**                                                          |Native share sheet.                                                         |
|Haptics        |**expo-haptics**                                                          |Subtle tap feedback on fill.                                                |
|Fonts          |**expo-font**                                                             |See §7 for specific fonts — do NOT use Inter/Roboto/system.                 |
|Build + ship   |**EAS Build** + **EAS Submit**                                            |Cloud-compiles native binaries; submits to both stores. No Mac/Xcode needed.|

⚠️ **Gotcha:** Always install native libraries with `npx expo install <pkg>`, never plain `npm install`. Expo pins compatible versions; mixing raw npm versions of react-native-svg / reanimated is the #1 cause of white-screen crashes.

-----

## 4. Project structure

```
iconography/
├── app/                      # expo-router screens
│   ├── _layout.tsx           # root layout, font loading, theme provider
│   ├── index.tsx             # Gallery (home)
│   ├── color/[id].tsx        # Coloring canvas for one artwork
│   └── about.tsx             # About + privacy policy link
├── src/
│   ├── components/
│   │   ├── ArtworkCard.tsx    # gallery tile (shows progress)
│   │   ├── ColorCanvas.tsx    # THE core: renders SVG + handles fills
│   │   ├── PaletteDock.tsx    # color swatches selector
│   │   └── Completion.tsx     # shimmer + save/share
│   ├── artworks/              # the content library
│   │   ├── catalog.ts         # array of artwork metadata
│   │   └── svg/               # one component per artwork (see §6)
│   ├── theme/
│   │   ├── colors.ts          # palette tokens (§7)
│   │   └── typography.ts
│   ├── state/
│   │   └── progress.ts        # load/save coloring progress (AsyncStorage)
│   └── lib/
│       └── palettes.ts        # curated color sets
├── assets/                    # fonts, icon, splash
├── app.json                   # Expo config (name, icon, splash, bundle ids)
└── SPEC.md                    # this file
```

-----

## 5. Data model (all local)

### 5.1 Artwork (static, bundled — never changes at runtime)

```ts
type Artwork = {
  id: string;            // "psalm-23-shepherd"
  title: string;         // "The Good Shepherd"
  verse: string;         // "The Lord is my shepherd; I shall not want."
  reference: string;     // "Psalm 23:1"
  category: "psalms" | "gospels" | "creation" | "feasts";
  regionIds: string[];   // every fillable region's id, e.g. ["sky","robe","halo",...]
  suggestedPalette: string; // palette id from palettes.ts
};
```

### 5.2 Progress (persisted in AsyncStorage, key = `progress:<artworkId>`)

```ts
type ArtworkProgress = {
  fills: Record<string, string>; // regionId -> hex color, e.g. { "halo": "#C9A227" }
  completedAt: string | null;    // ISO date when all regions filled
  lastOpenedAt: string;          // ISO date
};
```

“Completed” = every `regionId` in the artwork has an entry in `fills`.

### 5.3 Palette

```ts
type Palette = { id: string; name: string; colors: string[] }; // colors are hex
```

-----

## 6. The core mechanic (read this carefully — it’s the whole app)

**Each artwork is an SVG made of closed `<Path>` regions.** Coloring = tap a region → it fills with the selected color. This is the single most important design decision: **region-fill, not pixel flood-fill.** Region-fill is crisp, performant, and easy to implement. Raster flood-fill is a swamp — do not attempt it in v1.

### How a region fills

Each `<Path>` has a stable `id` (the region id). The canvas keeps `fills` state mapping `regionId -> color`. On tap:

1. Identify which region was tapped (each Path has its own `onPress`).
1. Set `fills[regionId] = selectedColor`.
1. Fire a light haptic.
1. Persist progress (debounced).
1. If all regions now filled → trigger Completion.

### Reference shape of an artwork SVG component

```tsx
// src/artworks/svg/PsalmShepherd.tsx
import Svg, { Path } from "react-native-svg";

export function PsalmShepherd({ fills, lineColor, onRegionPress }) {
  const fill = (id: string) => fills[id] ?? "#F4ECD8"; // default = unpainted parchment
  return (
    <Svg viewBox="0 0 400 400">
      <Path id="sky"  d="..." fill={fill("sky")}  stroke={lineColor} strokeWidth={2}
            onPress={() => onRegionPress("sky")} />
      <Path id="hill" d="..." fill={fill("hill")} stroke={lineColor} strokeWidth={2}
            onPress={() => onRegionPress("hill")} />
      {/* ...one <Path> per region, ids must match Artwork.regionIds... */}
    </Svg>
  );
}
```

`ColorCanvas.tsx` owns the `fills` state, the selected color, undo (keep a small history stack), and renders the right artwork component by `id`.

### ⚠️ Gotcha — the real work is the ART, not the code

The code for filling is simple. The bottleneck is producing **clean SVGs where every fillable area is a separate closed path with an id.** Pipeline:

1. Generate or source line art (AI image-gen line art, or open-license sacred line drawings).
1. Vectorize raster → SVG (Illustrator Image Trace, Inkscape trace, or vectorizer.ai).
1. **Clean up by hand** so regions are closed, non-overlapping, and each gets a meaningful `id`.
1. Drop in as a component matching the shape above.

**For v1, prepare just 5–8 artworks this way.** Don’t try to ship 50. Quality of the first handful sells the app. Treat new artworks as content you add over time.

-----

## 7. Design direction (this app lives or dies on feel)

Aesthetic: **illuminated manuscript / sacred reverence.** Calm, warm, unhurried. The opposite of a loud kids’ coloring app. NO generic-AI look — no purple-on-white gradients, no Inter/Roboto/system fonts.

### Color tokens (`src/theme/colors.ts`)

```ts
export const colors = {
  parchment:  "#F4ECD8",  // app background + unpainted regions
  parchmentDeep: "#E8DCC0",
  ink:        "#2B2118",  // line work + primary text
  gold:       "#C9A227",  // illumination accent (completion shimmer)
  lapis:      "#1F3A5F",  // deep blue
  vermillion: "#9E2B25",  // red
  forest:     "#3E5C3A",
  ochre:      "#B07A2C",
};
```

Default “unpainted” region = `parchment`, so an untouched piece looks like blank vellum.

### Painting palettes (`src/lib/palettes.ts`)

Curate 3–4 small palettes (8 colors each) so choices feel intentional, not infinite:

- **Illumination** — gold, lapis, vermillion, forest, ochre, ink, cream, rose.
- **Dawn** — soft warm pastels.
- **Vespers** — muted dusk tones.

### Typography (`src/theme/typography.ts`)

- **Display / titles / the verse:** a characterful serif — **Cormorant Garamond** or **EB Garamond** (load via expo-font). Reverent, manuscript-like.
- **Body / UI:** a clean humanist serif or quiet sans with warmth — e.g. **Spectral** or **Newsreader**. Avoid Inter/Roboto.

### Motion

- Page transitions: slow gentle fades (~300–400ms), never bouncy.
- Fill: region eases into color over ~150ms, plus a light haptic.
- **Completion = the delight moment:** a soft gold shimmer sweeps across the finished piece, the verse fades up, then Save/Share appear. Spend your animation budget here.

### Whole-app vibe

Generous negative space, parchment texture on backgrounds (subtle), thin rules, no harsh shadows. It should feel like a quiet chapel, not a toy store.

-----

## 8. Screens

### 8.1 Gallery — `app/index.tsx`

- Grid of `ArtworkCard`s. Each card shows a thumbnail that **reflects the user’s current coloring** (partially painted pieces look partially painted).
- Card states: untouched / in-progress / completed (completed gets a small gold mark).
- Tap → open `color/[id]`.
- Calm header with app title in the display serif. No bottom tab bar needed in v1.

### 8.2 Coloring canvas — `app/color/[id].tsx`

- Top: the verse + reference, set in the display serif.
- Center: the `ColorCanvas` (the SVG), comfortably sized, pinch-zoom optional (v1.5).
- Bottom: `PaletteDock` (swatches) + Undo button.
- Auto-saves continuously. Back returns to gallery with progress kept.
- When the final region fills → `Completion` overlay.

### 8.3 Completion — `Completion.tsx` (overlay/modal)

- Gold shimmer over the finished art, verse fades up.
- Buttons: **Save to Photos** (expo-media-library), **Share** (expo-sharing), **Done** (back to gallery).

### 8.4 About — `app/about.tsx`

- App description, a line of intent (“a quiet space to sit with Scripture”), version, and a **Privacy Policy link** (required by both stores even though the app collects nothing).

-----

## 9. Build sequence (phases — do in order)

### Phase 0 — Environment ⚠️ (the non-coder danger zone)

- Install Node LTS. Confirm `node -v`.
- `npx create-expo-app@latest iconography -t expo-router` (TypeScript template).
- `npm i -g eas-cli`, then `eas login` (create a free Expo account first).
- Get the app running in **Expo Go** on your physical phone (`npx expo start`, scan QR).
- **Acceptance:** the starter app loads on your phone over the QR code.

### Phase 1 — Skeleton + design system

- Load fonts (expo-font), add `colors.ts` / `typography.ts`, build a theme.
- Stub three routes (index, color/[id], about) with placeholder content using the real fonts/colors.
- **Acceptance:** you can navigate between three screens; they look like the §7 direction, not default RN.

### Phase 2 — Gallery with static catalog

- Create `catalog.ts` with 2–3 placeholder artworks (simple SVGs are fine for now).
- Build `ArtworkCard` + the grid.
- **Acceptance:** gallery renders cards; tapping one routes to `color/[id]` with the right id.

### Phase 3 — Coloring canvas core (the heart)

- Build `ColorCanvas` + `PaletteDock`. Implement tap-to-fill, selected color, undo, haptics.
- Wire in 1 real hand-prepped artwork SVG (§6).
- **Acceptance:** you can tap regions and they fill with the chosen color; undo works; no lag.

### Phase 4 — Persistence

- `state/progress.ts`: save `fills` to AsyncStorage (debounced), load on open.
- Gallery thumbnails reflect saved progress.
- **Acceptance:** color a piece, kill the app, reopen → your coloring is still there and shows on the card.

### Phase 5 — Completion + save/share

- Detect “all regions filled” → `Completion` overlay with gold shimmer + verse.
- Save to Photos and Share working.
- **Acceptance:** finishing a piece celebrates, and the saved image appears in your phone’s photo library.

### Phase 6 — Content + polish

- Add the full v1 set of **5–8 prepared artworks**.
- About screen + privacy policy link, app icon + splash (Canva), empty/edge states, final motion tuning.
- **Acceptance:** app feels complete and calm end-to-end with real content.

### Phase 7 — Build + ship

- `eas build -p ios` and `eas build -p android` (let EAS manage signing — see gotcha).
- Test the real builds (TestFlight / internal testing).
- Prepare store listings (screenshots from Canva, description, privacy policy URL).
- `eas submit` to App Store Connect and Google Play.
- **Acceptance:** builds pass, listings complete, submitted for review.

-----

## 10. Shipping notes & gotchas

- ⚠️ **Signing/credentials:** when `eas build` asks, let **EAS manage credentials automatically** for both platforms. Do not hand-manage certificates/provisioning profiles — that’s the classic time-sink.
- ⚠️ **Use `npx expo install`** for every native package (repeat of §3 because it bites people twice).
- **Apple/Google accounts** must exist before Phase 7: Apple Developer (~$99/yr), Google Play (~$25 one-time). Identity verification can take a day or two — start this early, in parallel with Phase 1.
- **Privacy policy** is required even though the app collects nothing. A simple hosted page stating “this app collects no personal data and stores everything locally on your device” is enough. Put the URL in About and in both store listings.
- **App Store review:** because there’s no UGC, no login, and no IAP, this is the gentlest review path. Just make sure it never crashes, screenshots match the app, and the metadata is honest.
- **Data safety forms:** declare “no data collected” on both stores (true if you skip analytics).

-----

## 11. v1.5+ backlog (note, don’t build)

- Optional ambient soundscapes (great fit — could use original tracks).
- Daily verse / a new artwork on a cadence.
- Pinch-zoom on the canvas for fine detail.
- iCloud/Drive backup of progress.
- Gentle streaks (“days illuminated”).
