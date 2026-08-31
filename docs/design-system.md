# Design system

One person building 32 screens will produce 32 slightly different screens unless something stops them. `constants/theme.ts` is that thing — a single token file, 107 lines, that every screen reads from.

Its penetration is the only interesting measure of whether a solo design system is real:

| | |
|---|---|
| Files consuming the theme context | **38** |
| Files using the `makeStyles(C, isDark)` memo pattern | **36** |
| References to the live text scale | **321** |
| Files honouring reduce-motion | **12** |

A theme file nobody imports is a mood board. These numbers are the difference.

---

## Tokens

**Colour** — 10 role-named tokens, defined twice for light and dark:

`background` · `surface` · `primary` · `subtext` · `border` · `fillLight` · `fillMid` · `success` · `danger` · `warning`

Named by **role**, not by hue. There is no `grey3` or `blue500`, so a screen asks for `border` and gets the right value in both schemes without knowing which scheme it is in. Dark mode is a second set of values, not a second set of screens.

**Typography** — Outfit at five weights (400/500/600/700/800), on a six-step scale: `h1` 28 · `h2` 24 · `h3` 20 · `body` 15 · `label` 13 · `caption` 11.

**Spacing** — nine values, four of them named for their job rather than their size: `screenPadding` 24, `cardPadding` 16, `cardGap` 12, `sectionGap` 20, plus `xs`–`xl`. Semantic names are what stop the gap between two cards drifting from 12 to 10 to 14 across a codebase.

**Radius** — six values, again semantic: `button` 100 (pill), `card` 18, `input` 14, `bottomSheet` 24, `iconBox` 12, `sm` 8.

**Elevation** — two shadows, `card` and `elevated`, each carrying its Android `elevation` alongside the iOS shadow so the two platforms agree.

---

## The appearance model

The theme context resolves six user settings into one object:

| Setting | Options |
|---|---|
| Colour scheme | Light · Dark |
| Follow system | On · Off |
| Text size | Small · Medium · Large |
| Bold text | On · Off |
| High contrast | On · Off |
| Reduce motion | On · Off |

All six persist to device storage and reset on sign-out, so a shared device does not carry one technician's settings into the next one's session.

**The mechanism is the design decision.** Text scale and bold-text preference are not read separately by each screen — they are folded into the resolved colour object `C` that every screen already depends on:

```ts
type ThemeColors = BaseColors & {
  textScale:  number;   // from the Small / Medium / Large setting
  boldFamily: string;   // font family respecting the bold-text preference
};
```

Because `C` is the dependency of every `makeStyles(C, isDark)` memo in the app, changing text size invalidates the same memos that changing to dark mode invalidates. One mechanism, four settings, no per-screen wiring, and no way for a screen to accidentally opt out. High contrast works the same way — it rewrites `border` and `subtext` before `C` is assembled, so contrast is applied at the token layer rather than screen by screen.

That is why the text scale can be referenced 321 times without becoming 321 places to forget.

---

## Motion

Motion is a system decision here, not a per-screen improvisation, and it has an off switch that works.

The shared `BottomSheet` is the clearest example. It springs up with no overshoot, and the backdrop fades in **progressively as the sheet rises** — barely visible at the start of the lift, reaching about 45% only once the sheet has nearly landed. The naive version fades the backdrop on open, which reads as a dark flash before anything appears. On close, sheet and backdrop move in tandem and the modal unmounts only after the animation finishes.

With reduce-motion on, the same sheet snaps to its final position instead. The setting is honoured in 12 files, including every animation a user meets in the main flow — not declared and then ignored.

---

## Accessibility: what is done, and what is not

**Done.** Six appearance settings covering scheme, text size, weight, contrast and motion — all persisted, all applied through the token layer. Font scaling and bold text are threaded through the type system rather than approximated. High contrast changes real values. Reduce motion actually stops animations.

**Not done, and worth stating plainly: there are no screen-reader labels.** There is not one `accessibilityLabel`, `accessibilityRole` or `accessibilityHint` in the codebase. TalkBack and VoiceOver will announce what React Native infers, which for an icon-only button is nothing useful.

**And one deliberate trade.** The app sets `maxFontSizeMultiplier: 1` globally, overriding the OS text-size setting. Android allows 200% scaling, which breaks the fixed-height, tabular layouts that dense nameplate data needs. The app substitutes its own three-step control rather than honouring the system one — a complete legibility model it controls, in place of a partial one it would break. That is defensible, and it is still a trade: a user who has set their phone to 200% for a reason will not get 200% here.

The honest summary is that FieldSnap is designed for **low-vision legibility and motion sensitivity**, and not yet for **non-visual use**. Labelling the icon buttons is the next milestone, not a solved problem.

---

## Component vocabulary

The system is token-heavy and component-light — one shared `BottomSheet` and one shared auth sheet body, with most screens composing directly from tokens. That is a real limitation compared with a shared `ScreenHeader` / `EmptyState` / `StatusBadge` vocabulary, and it is why the 36-file `makeStyles(C, isDark)` convention carries so much of the consistency load: the pattern is repeated deliberately, and it is what a component library would be extracted from next.

Where a pattern did repeat, it was extracted. `GetStartedSheetContent` exists because the auth sheet appears on both the onboarding screen and the settings screen, and two hand-maintained copies would have drifted within a week.

---

## Icons

The tab bar icons are not an icon-font import. They are the exact SVG path data from the design files, pasted into typed components that take a `color` prop — drawn as vectors, exported, and shipped without a translation step in between. The file says so where the paths begin: *"exact paths from design SVG files."*

---

## The PDF is part of the design system

The exported report is the artifact that leaves the app and reaches a client, so it is designed rather than dumped: page margins set in inches, a running footer that repeats correctly across page breaks, a branded header, and data, OCR and notes presented as distinct card types. It uses its own print palette rather than the app's, because a screen palette printed on white paper is not the same design problem.
