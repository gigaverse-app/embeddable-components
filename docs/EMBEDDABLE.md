# Embeddable — uzupełnienie do `CLAUDE.md`

> **Najpierw przeczytaj [`CLAUDE.md`](../CLAUDE.md) w roocie.** To oficjalny przewodnik od Embeddable (jak działa platforma, workspaces, lifecycle pushu, reguły confirmation przed komendami). Ten plik jest **uzupełnieniem** — opisuje nasze decyzje (Bun, fix `ct`), gotchas spoza oficjalnej dokumentacji i historię synca z upstream boilerplate.

---

## Status repo (2026-05-27)

Repo zostało **zsyncowane ze świeżym `npx @embeddable.com/init`** (snapshot z 2026-05-27). Struktura, configi, presety, dashboard-as-code skill, npm-audit workflow i `CLAUDE.md` pochodzą bezpośrednio z upstream. Nasze modyfikacje:

| Zmiana | Wartość | Powód |
|---|---|---|
| Package manager | **Bun** (`bun.lock`) zamiast npm (`package-lock.json`) | Spójność ze stackiem TapIn / Gigaverse |
| Skrypt `ct` | `tsc --noEmit` | Upstream ma literówkę `tsconfig --noEmit` (komenda nieistniejąca) |
| Skrypt `embeddable:upgrade` | `bunx` zamiast `npx` | Konsekwencja decyzji o Bun |
| Skrypt `reinstall` | `rm -rf node_modules bun.lock && bun install` | Adaptacja upstream'owego `reinstall` do Bun |
| `engines.npm` | Usunięte (zostało `node: ">=22.0.0"`) | Mamy Bun, ograniczenie wersji npm nieistotne |

Wszystko inne (struktura katalogów, configi, dashboard-as-code skill, modele Cube, CLAUDE.md) — **identyczne jak upstream**.

> **Update z upstream:** żeby ponownie zsyncować się ze świeżym boilerplate, odpal `npx @embeddable.com/init` w `/tmp` i zdiffuj. Sekcja 4 niżej opisuje, **czego nie nadpisywać**.

---

## Decyzje, których upstream nie ma

### Bun zamiast npm

Wszystkie komendy z `CLAUDE.md` używające `npm run X` można odpalić jako `bun run X`. Bun czyta `package.json` `scripts` 1:1. Wyjątki gdzie różnica się ujawnia:

- **`bun add <pkg>` usuwa prefix `^`** — instaluje wersję dokładną. Po podbiciu wersji ręcznie dopisz `^` w `package.json` jeśli chcesz.
- **`bunx` zamiast `npx`** — używamy w `embeddable:upgrade`.
- **`bun.lock` jest jedynym lockfile** — jeśli `package-lock.json` się pojawi (np. po `npm install` przez pomyłkę), usuń go.
- **`bun why <pkg>`** zamiast `npm why <pkg>` do inspekcji drzewa zależności.

### Naprawiony skrypt `ct`

Upstream ma `"ct": "tsconfig --noEmit"` (komenda `tsconfig` nie istnieje — to literówka). U nas `"ct": "tsc --noEmit"`. **Po `bun run embeddable:upgrade` upewnij się, że skrypt nie został przywrócony do upstream-owego stanu.**

---

## Gotchas spoza oficjalnej dokumentacji

### Build/CLI

- **`embeddable build` może wywalić OOM** dla większych projektów. Skrypt `dev`/`embeddable:dev` ma już `NODE_OPTIONS='--max-old-space-size=4096'`, ale `embeddable:build` nie. Jeśli wywali, dopisz `cross-env NODE_OPTIONS=... embeddable build` w skrypcie.
- **Safari nie działa z `embeddable:dev`** — security restrictions w Safari blokują dev server. Używaj Chrome/Firefox.
- **`embeddable-entry-point.jsx` w roocie jest auto-generowany przez CLI** podczas build/dev — nie commituj, jest w `.gitignore`.
- **`.embeddable-build/` może urosnąć do setek MB** (po ostatnim runie miało ~380 katalogów cache). To normalne — wszystko gitignored.

### Theming

- **`defineTheme` deep-merguje** — przekazuj tylko zmiany w `dark-theme.ts`, nie całe drzewo. Default theme jest source of truth.
- **`--em-sem-*` to to, co chcesz nadpisywać**, nie `--em-core-*`. Core to primitywy designu (kolory bazowe, spacing scale), semantic to mapowanie do produktu (background, text, chart colors). Override'uj semantic.
- **`as Theme` cast w `embeddable.theme.ts`** — upstream go dodał, bo bez tego TS narzeka po niedawnej zmianie typów w `@embeddable.com/core`. Zostaw.
- **`clientContext` ma typ `any`** w boilerplate. Jeśli używamy w wielu miejscach, zdefiniuj nasz własny typ i wzmocnij sygnaturę `themeProvider`.

### Komponenty (gdy będziemy je dodawać)

- **`meta.name` musi pasować do nazwy pliku przed `.emb.ts`**. Jeśli plik to `Foo.emb.ts`, `meta.name === 'Foo'`. Build się wywali z myląco-brzmiącym błędem przy desynchu.
- **`as const satisfies EmbeddedComponentMeta`** jest ważne — `Inputs<typeof meta>` bez tego nie zinferuje typów inputów.
- **`Value.noFilter()` jako fallback dla pustego eventu** — Embeddable wymaga rozróżnienia "brak wartości" vs "brak filtra".
- **TypeScript w `tsconfig.include` zaczyna się od `"src"`** — pliki w roocie (`embeddable.config.ts`, `embeddable.theme.ts`, `embeddable.lifecycle.ts`) NIE są objęte przez `bun run ct`. CLI je obsługuje, ale `tsc --noEmit` ich nie tyka.

### Wersjonowanie paczek

- **`remarkable-pro` jest w `0.x`** — semver nie chroni przed breaking changes między minor releases. Aktualizuj minor po minor i sprawdzaj `bun run embeddable:build` po każdym kroku.
- **SDK changelog nie jest publiczny** (repo `embeddable-sdk` prywatne na GitHub) — przy większych skokach SDK 4.x → 4.y testuj `bun run embeddable:build` + `bun run embeddable:dev` manualnie.
- **Wszystkie `@embeddable.com/*` aktualizuj razem.** `sdk-core@x` peer-zależy od konkretnej wersji `core`, więc rozjazd wersji = błędy w runtime.
- **`overrides` block w `package.json`** pin'uje `glob ^13.0.0` i `axios 1.13.6` — to security pin z upstream. Nie usuwaj bez sprawdzenia, czy podatność została załatana w nowszych wersjach tranzytywnych.

### Security context — historia migracji schematu

Upstream zmienił schemat `security-contexts.sc.yml`:

**Stary (pre-2026-05):**
```yaml
- name: Jake Sterling
  securityContext:
    artists: [ART_00002]
    language: English
```

**Nowy (obecny, w naszym repo):**
```yaml
- name: Jake Sterling
  securityContext: {}
  filters:
    - member: 'music_artists.artist_id'
      operator: 'equals'
      values: ['ART_00002']
  environment: default
```

`filters` są aplikowane jako Cube member filters bezpośrednio. `securityContext` (jeśli zawiera dane) jest przekazywany do modeli Cube jako `{{ COMPILE_CONTEXT.securityContext.xxx }}`. Możesz używać obu — `filters` dla prostego row-level security, `securityContext` dla bardziej zaawansowanych przypadków.

---

## Co dalej (zaadoptowane, niewdrożone)

### Dashboard-as-code

Mamy w repo:
- `.claude/skills/dashboard-as-code/` — kompletny skill dla agentów Claude (SKILL.md + 4 examples + 5 references)
- `src/embeddable.com/embeddables/` — 2 przykłady (`spotify-artist-dashboard`, `spotify-self-serve-dashboard`)
- Flagę `embeddable:dev --events-file=.embeddable-dev-logs/dev.events.ndjson` — agent może czytać NDJSON log walidacji

**Nie używamy tego jeszcze**, ale infrastruktura jest gotowa. Gdy zdecydujemy się na migrację z buildera no-code na dashboard-as-code, agent z włączonym skill'em `dashboard-as-code` będzie wiedział co robić.

### npm audit workflow

`.github/workflows/npm-audit.yml` — cotygodniowy `npm audit` z notyfikacją na Slack. Wymaga sekretów `SLACK_WEBHOOK_URL` i zmiennych `BUILD_NODE_VERSION`, `NPM_AUDIT_LEVEL` w GH Actions. **Nie jest obecnie skonfigurowany** — jeśli włączymy, ustaw je w GH Settings.

**Uwaga:** workflow używa `npm ci`, nie `bun install`. Jeśli włączymy, trzeba albo:
- Wygenerować lokalnie `package-lock.json` z `npm i --package-lock-only` na potrzeby auditu (nie commituj go), albo
- Przerobić workflow na `bun audit` (Bun ma własny audit od v1.1).

---

## Linki

### Oficjalna dokumentacja
- [Quick-start guide](https://docs.embeddable.com/getting-started/quick-start-guide)
- [Set up your Workspace](https://docs.embeddable.com/getting-started/set-up-your-workspace)
- [Defining Components](https://docs.embeddable.com/development/defining-components)
- [Local Development](https://docs.embeddable.com/development/local-environment)
- [Pushing Code to Workspace](https://docs.embeddable.com/development/pushing-code)
- [Client Context](https://docs.embeddable.com/development/client-context)
- [Remarkable Pro Theming](https://docs.embeddable.com/component-libraries/remarkable-pro/theming)
- [CSS Tokens — Core](https://docs.embeddable.com/component-libraries/remarkable-pro/styling/core-tokens)

### Repozytoria publiczne
- [remarkable-pro releases](https://github.com/embeddable-hq/remarkable-pro/releases) — changelog Remarkable Pro
- [embeddable-boilerplate](https://github.com/embeddable-hq/embeddable-boilerplate) — vanilla components (referencja)

### Community
- [community.embeddable.com](https://community.embeddable.com)
