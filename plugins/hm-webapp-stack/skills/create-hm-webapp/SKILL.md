---
name: create-hm-webapp
description: Erstellt eine opinionated hellomirrors web-app unter Nutzung von NextJS, shadcn/ui, Tailwind CSS 4, Ultracite/Biome, Zod, TanStack Query/Form, Zustand
allowed-tools: Bash(bunx *), Bash(bun *), Bash(cd *), Bash(mkdir *), Bash(rm *), Read, Write, Edit, Glob, Grep
---

# HM Web-App Skill

Dieser Skill erstellt eine neue Web-Applikation nach dem hellomirrors-Standard-Stack.

Frage den User **vor dem Start** nach:
1. **Projektname** — wird als Verzeichnisname und in package.json verwendet
2. **Kurze Beschreibung** der App

---

## Phase 1: Projekt-Scaffolding (via Tools)

### 1.1 Next.js App erstellen

Führe folgenden Befehl aus, um das Next.js-Projekt zu erstellen:

```bash
bunx create-next-app@latest <projektname> \
  --typescript \
  --tailwind \
  --biome \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --turbopack \
  --use-bun \
  --yes
```

> `--biome` sorgt dafür, dass Biome statt ESLint eingerichtet wird — kein manuelles Aufräumen von ESLint-Dateien nötig. `--use-bun` stellt sicher, dass bun als Package Manager genutzt wird.

### 1.2 Dependencies installieren

Wechsle ins Projektverzeichnis und installiere die zusätzlichen Pakete:

```bash
cd <projektname>

# Core UI & Styling
bun add clsx tailwind-merge class-variance-authority tw-animate-css lucide-react next-themes sonner

# Data Fetching & Forms
bun add @tanstack/react-query @tanstack/react-form zod

# State Management (optional — nur installieren wenn Client-State benötigt wird)
bun add zustand

# Dev Dependencies (ultracite erweitert die von create-next-app installierte Biome-Config)
bun add -D ultracite vitest lefthook @types/node @types/react
```

> **Zustand** ist optional. Installiere es nur, wenn die App Client-State hat, der nicht vom Server kommt (z.B. Sidebar offen/zu, aktiver Tab, User-Präferenzen). Viele Apps kommen mit React Query allein aus.

### 1.3 shadcn/ui initialisieren

```bash
bunx shadcn@latest init --style base-mira --base-color mist --css-variables --rsc --tsx
```

Danach installiere die Standard-Komponenten. Nutze bevorzugt den **shadcn MCP Server** (Tool `add_component`) um Komponenten hinzuzufügen, da dieser die beste Kompatibilität mit dem gewählten Preset sicherstellt. Fallback auf CLI:

```bash
bunx shadcn@latest add input button label alert-dialog select checkbox radio-group badge card scroll-area switch textarea tooltip dialog tabs separator sheet
```

### 1.4 MCP Server einrichten

Erstelle `.mcp.json` im Projektroot mit den MCP Servern für shadcn, Next.js DevTools und Ultracite:

```json
{
  "mcpServers": {
    "shadcn": {
      "command": "bunx",
      "args": ["shadcn@latest", "mcp"]
    },
    "next-devtools": {
      "command": "bunx",
      "args": ["-y", "next-devtools-mcp@latest"]
    },
    "ultracite": {
      "command": "bunx",
      "args": ["-y", "mcp-remote", "https://docs.ultracite.ai/mcp"]
    }
  }
}
```

**shadcn MCP** stellt Tools bereit um Komponenten zu suchen, hinzuzufügen und Presets abzufragen. Nutze diese Tools bevorzugt gegenüber CLI-Befehlen für shadcn-Operationen.

**Next.js DevTools MCP** (erfordert Next.js 16+) verbindet sich automatisch mit dem laufenden Dev-Server und bietet:
- `get_errors` — Build-, Runtime- und Type-Errors vom Dev-Server abrufen
- `get_logs` — Browser-Console-Logs und Server-Output lesen
- `get_routes` — Alle App-Router und Pages-Router Routes auflisten
- `get_page_metadata` — Metadata zu Seiten (Routes, Komponenten, Rendering)
- `get_project_metadata` — Projektstruktur, Config, Dev-Server-URL
- `get_server_action_by_id` — Server Actions nach ID nachschlagen

**Ultracite MCP** bietet Zugriff auf die Ultracite Knowledge Base:
- `SearchUltracite` — Suche nach Ultracite-Docs, Code-Beispielen, API-Referenzen und Guides
- Nutze diesen MCP Server bei Fragen zu Biome-Konfiguration, Ultracite-Regeln oder Formatting-Problemen

### 1.5 Claude Code Projekt-Settings einrichten

Erstelle `.claude/settings.json` im Projektroot mit einem PostToolUse Hook, der nach jeder Dateiänderung automatisch `ultracite fix` ausführt:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "command": "bun fix --skip=correctness/noUnusedImports",
            "type": "command"
          }
        ],
        "matcher": "Write|Edit"
      }
    ]
  }
}
```

> Der Hook überspringt `noUnusedImports` bewusst, da während der Entwicklung Imports oft temporär ungenutzt sind und erst im nächsten Schritt verwendet werden.

---

## Phase 2: Konfigurationsdateien

### 2.1 biome.jsonc

Erstelle `biome.jsonc` im Projektroot. **Kritisch:** shadcn/ui-Komponenten werden von Linting, Formatting UND Assist ausgeschlossen. `noChildrenProp` ist global deaktiviert weil TanStack Form das render-prop Pattern nutzt:

```jsonc
{
  "$schema": "./node_modules/@biomejs/biome/configuration_schema.json",
  "extends": [
    "ultracite/biome/core",
    "ultracite/biome/react",
    "ultracite/biome/next",
    "ultracite/biome/vitest"
  ],
  "linter": {
    "rules": {
      "correctness": {
        // TanStack Form uses render-prop pattern: <form.Field children={(field) => ...} />
        "noChildrenProp": "off"
      }
    }
  },
  "overrides": [
    {
      // shadcn/ui components — keep in default state, do not lint or format
      "includes": ["src/components/ui/**"],
      "formatter": {
        "enabled": false
      },
      "linter": {
        "enabled": false
      },
      "assist": {
        "enabled": false
      }
    }
  ]
}
```

### 2.2 tsconfig.json

Ersetze die generierte `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "lib": ["DOM", "DOM.Iterable", "ES2023"],
    "strict": true,
    "jsx": "react-jsx",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "paths": { "@/*": ["./src/*"] },
    "allowJs": true,
    "skipLibCheck": true,
    "noEmit": true,
    "esModuleInterop": true,
    "isolatedModules": true,
    "incremental": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "plugins": [{ "name": "next" }]
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts",
    ".next/dev/types/**/*.ts"
  ],
  "exclude": ["node_modules"]
}
```

### 2.3 next.config.ts

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    typedEnv: true,
  },
};

export default nextConfig;
```

### 2.4 postcss.config.mjs

```javascript
export default {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};
```

### 2.5 vitest.config.ts

```typescript
import path from "node:path";
import { defineConfig } from "vitest/config";

export default defineConfig({
  resolve: {
    alias: {
      "@": path.resolve(import.meta.dirname, "./src"),
    },
  },
  test: {
    globals: true,
    environment: "node",
  },
});
```

### 2.6 package.json Scripts

Ersetze die `scripts`-Sektion in `package.json`. Stelle sicher, dass `"type": "module"` gesetzt ist:

```json
{
  "type": "module",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "test": "vitest run",
    "test:watch": "vitest",
    "check": "ultracite check",
    "fix": "ultracite fix",
    "prepare": "lefthook install",
    "outdated": "bun outdated"
  }
}
```

### 2.7 lefthook.yml (Git Hooks)

Erstelle `lefthook.yml` im Projektroot. Der Hook führt `ultracite fix` aus (nicht nur check), stagt die gefixten Dateien automatisch und filtert auf relevante Dateitypen:

```yaml
pre-commit:
  jobs:
    - run: bun x ultracite fix
      glob:
        - "**/*.js"
        - "**/*.jsx"
        - "**/*.ts"
        - "**/*.tsx"
        - "**/*.json"
        - "**/*.jsonc"
        - "**/*.css"
      stage_fixed: true
```

---

## Phase 3: Projektstruktur anlegen

Erstelle folgende Verzeichnisstruktur unter `src/`:

```
src/
├── app/
│   ├── api/                    # API Route Handlers
│   ├── globals.css             # Tailwind 4 Theme + Imports (von shadcn generiert)
│   ├── layout.tsx              # Root Layout mit Fonts + Providers
│   ├── providers.tsx           # React Query + Theme Provider
│   ├── page.tsx                # Startseite
│   └── header.tsx              # Navigation Header
├── components/
│   └── ui/                     # shadcn/ui Komponenten (NICHT manuell editieren)
├── domain/                     # Pure Business Logic (kein React)
│   ├── types.ts                # TypeScript Types & Interfaces
│   └── schema.ts               # Zod Validation Schemas
├── features/                   # Feature-Module (gruppierte UI + Logik)
├── hooks/                      # Custom React Hooks
├── lib/
│   ├── api-client.ts           # API Fetch-Funktionen
│   └── utils.ts                # cn() Utility
└── tests/                      # Vitest Tests
```

> `src/lib/store.ts` (Zustand) wird nur erstellt, wenn Zustand als Dependency installiert wurde.

### 3.1 globals.css (Tailwind 4 + shadcn Theme)

Die `globals.css` wird von `shadcn init` generiert. Stelle sicher, dass sie folgende Struktur hat:

```css
@import "tailwindcss";
@import "tw-animate-css";
@import "shadcn/tailwind.css";

@custom-variant dark (&:is(.dark *));

@theme {
  /* shadcn generiert hier die Theme-Variablen (OKLch-Farbsystem) */
}

@layer base {
  * {
    @apply border-border outline-ring/50;
  }
  body {
    margin: 0;
    font-feature-settings: "rlig" 1, "calt" 1;
    @apply bg-background text-foreground;
  }
  html {
    scrollbar-gutter: stable;
    @apply font-sans;
  }
}

:root {
  /* shadcn generiert hier Light-Mode Farben */
}

.dark {
  /* shadcn generiert hier Dark-Mode Farben */
}

@theme inline {
  /* shadcn generiert hier die CSS-Variable-zu-Theme-Mappings */
}
```

**Wichtig:** Nach `shadcn init` manuell hinzufügen falls nicht vorhanden:
- `@custom-variant dark (&:is(.dark *));` — Dark-Mode Variant für Tailwind 4
- `scrollbar-gutter: stable;` auf `html` — verhindert Layout-Shift bei Scrollbar-Erscheinen
- `font-feature-settings: "rlig" 1, "calt" 1;` auf `body` — aktiviert Ligatures

Nutze Geist als Standard-Font (via `next/font/google`) mit `variable: "--font-sans"`.

### 3.2 lib/utils.ts

```typescript
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### 3.3 providers.tsx

```typescript
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ThemeProvider } from "next-themes";
import { useState } from "react";
import { TooltipProvider } from "@/components/ui/tooltip";

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 5 * 1000,
            refetchOnWindowFocus: false,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider
        attribute="class"
        defaultTheme="dark"
        disableTransitionOnChange
      >
        <TooltipProvider>{children}</TooltipProvider>
      </ThemeProvider>
    </QueryClientProvider>
  );
}
```

> **TooltipProvider** wrapping ist erforderlich damit shadcn Tooltips funktionieren. `defaultTheme="dark"` statt `"system"` für konsistentere initiale Darstellung.

### 3.4 layout.tsx

```typescript
import type { Metadata } from "next";
import { Geist } from "next/font/google";
import "./globals.css";
import { Providers } from "./providers";
import { cn } from "@/lib/utils";

const geist = Geist({ subsets: ["latin"], variable: "--font-sans" });

export const metadata: Metadata = {
  title: "<App Name>",
  description: "<App Description>",
  icons: { icon: "/favicon.svg" },
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html
      className={cn("font-sans", geist.variable)}
      lang="de"
      suppressHydrationWarning
    >
      <body>
        <Providers>
          <div className="flex min-h-screen flex-col">
            <main className="flex-1">{children}</main>
          </div>
        </Providers>
      </body>
    </html>
  );
}
```

> **Font-Pattern:** `variable: "--font-sans"` registriert die CSS-Variable, `cn("font-sans", geist.variable)` auf `<html>` aktiviert sie. Nicht `geist.className` verwenden — das funktioniert nicht mit Tailwind 4 `@theme inline`.

### 3.5 Zustand Store Template (optional)

Nur erstellen wenn Zustand installiert wurde. Erstelle `src/lib/store.ts` als Basis-Store:

```typescript
import { create } from "zustand";

interface AppState {
  // App-spezifischer State hier
}

export const useAppStore = create<AppState>()((set, get) => ({
  // State + Actions
}));
```

---

## Phase 4: AGENTS.md erstellen

Erstelle `AGENTS.md` im Projektroot mit den Ultracite Code Standards. Diese Datei wird von Claude Code und anderen AI-Agenten gelesen:

```markdown
# Ultracite Code Standards

This project uses **Ultracite**, a zero-config preset that enforces strict code quality standards through automated formatting and linting.

## Quick Reference

- **Format code**: `bun x ultracite fix`
- **Check for issues**: `bun x ultracite check`
- **Diagnose setup**: `bun x ultracite doctor`

Biome (the underlying engine) provides robust linting and formatting. Most issues are automatically fixable.

---

## Core Principles

Write code that is **accessible, performant, type-safe, and maintainable**. Focus on clarity and explicit intent over brevity.

### Type Safety & Explicitness

- Use explicit types for function parameters and return values when they enhance clarity
- Prefer `unknown` over `any` when the type is genuinely unknown
- Use const assertions (`as const`) for immutable values and literal types
- Leverage TypeScript's type narrowing instead of type assertions
- Use meaningful variable names instead of magic numbers - extract constants with descriptive names

### Modern JavaScript/TypeScript

- Use arrow functions for callbacks and short functions
- Prefer `for...of` loops over `.forEach()` and indexed `for` loops
- Use optional chaining (`?.`) and nullish coalescing (`??`) for safer property access
- Prefer template literals over string concatenation
- Use destructuring for object and array assignments
- Use `const` by default, `let` only when reassignment is needed, never `var`

### Async & Promises

- Always `await` promises in async functions - don't forget to use the return value
- Use `async/await` syntax instead of promise chains for better readability
- Handle errors appropriately in async code with try-catch blocks
- Don't use async functions as Promise executors

### React & JSX

- Use function components over class components
- Call hooks at the top level only, never conditionally
- Specify all dependencies in hook dependency arrays correctly
- Use the `key` prop for elements in iterables (prefer unique IDs over array indices)
- Nest children between opening and closing tags instead of passing as props
- Don't define components inside other components
- Use semantic HTML and ARIA attributes for accessibility:
  - Provide meaningful alt text for images
  - Use proper heading hierarchy
  - Add labels for form inputs
  - Include keyboard event handlers alongside mouse events
  - Use semantic elements (`<button>`, `<nav>`, etc.) instead of divs with roles

### Error Handling & Debugging

- Remove `console.log`, `debugger`, and `alert` statements from production code
- Throw `Error` objects with descriptive messages, not strings or other values
- Use `try-catch` blocks meaningfully - don't catch errors just to rethrow them
- Prefer early returns over nested conditionals for error cases

### Code Organization

- Keep functions focused and under reasonable cognitive complexity limits
- Extract complex conditions into well-named boolean variables
- Use early returns to reduce nesting
- Prefer simple conditionals over nested ternary operators
- Group related code together and separate concerns

### Security

- Add `rel="noopener"` when using `target="_blank"` on links
- Avoid `dangerouslySetInnerHTML` unless absolutely necessary
- Don't use `eval()` or assign directly to `document.cookie`
- Validate and sanitize user input

### Performance

- Avoid spread syntax in accumulators within loops
- Use top-level regex literals instead of creating them in loops
- Prefer specific imports over namespace imports
- Avoid barrel files (index files that re-export everything)
- Use proper image components (e.g., Next.js `<Image>`) over `<img>` tags

### Framework-Specific Guidance

**Next.js:**
- Use Next.js `<Image>` component for images
- Use App Router metadata API for head elements
- Use Server Components for async data fetching instead of async Client Components

**React 19+:**
- Use ref as a prop instead of `React.forwardRef`

---

## Testing

- Write assertions inside `it()` or `test()` blocks
- Avoid done callbacks in async tests - use async/await instead
- Don't use `.only` or `.skip` in committed code
- Keep test suites reasonably flat - avoid excessive `describe` nesting

---

Most formatting and common issues are automatically fixed by Biome. Run `bun x ultracite fix` before committing to ensure compliance.
```

---

## Phase 5: Ultracite Check ausführen

Nach der vollständigen Einrichtung, führe den Formatter/Linter aus um sicherzustellen, dass alles den Konventionen entspricht:

```bash
bun run fix
```

Prüfe danach:

```bash
bun run check
```

Behebe alle gemeldeten Fehler **außer** in `src/components/ui/` — diese Dateien sind von Biome ausgeschlossen und dürfen NICHT manuell reformatiert werden.

---

## Architektur-Prinzipien

Diese Prinzipien MÜSSEN bei allen nachfolgenden Erweiterungen der App eingehalten werden:

### Separation of Concerns

| Layer | Pfad | Verantwortung | React-Abhängig? |
|-------|------|---------------|-----------------|
| **Domain** | `src/domain/` | Types, Schemas, Business Logic, Engine | Nein |
| **Features** | `src/features/` | Feature-UI-Module, feature-spezifische Komponenten | Ja |
| **Hooks** | `src/hooks/` | React Query Wrapper, State-Hooks, Custom Hooks | Ja |
| **Lib** | `src/lib/` | Utilities, API-Client, Zustand Stores, Storage | Minimal |
| **App** | `src/app/` | Routing, Layouts, API Routes, Page-Komposition | Ja |
| **Components** | `src/components/` | Shared UI-Komponenten (shadcn + custom) | Ja |

### Datenfluss

```
API Routes (app/api/) <-> API Client (lib/api-client.ts)
                              |
                     React Query Hooks (hooks/)
                              |
                     Zustand Store (lib/store.ts)  <->  Domain Logic (domain/)
                              |
                     Feature Components (features/)
```

### Kernregeln

1. **Domain ist pure** — `src/domain/` darf kein React importieren. Types, Zod-Schemas und Business Logic leben dort und sind unabhängig testbar.

2. **Zod für Runtime-Validierung** — Jede externe Schnittstelle (API Input, Form-Daten, Storage) wird mit Zod-Schemas validiert. TypeScript-Types werden aus den Schemas abgeleitet wo sinnvoll (`z.infer<typeof schema>`).

3. **TanStack Query für Server-State** — Alle API-Calls laufen über React Query. Mutations nutzen Cache-Invalidierung via `queryClient.setQueryData()` im `onSuccess` Callback.

4. **TanStack Form für Formulare** — Formulare nutzen das render-prop Pattern: `<form.Field children={(field) => ...} />`. Feld-Werte werden via `useStore(form.store, (s) => s.values.fieldName)` gelesen. Validierung passiert auf Feld- und Formular-Ebene.

5. **Zustand für Client-State** — UI-State, der nicht vom Server kommt (z.B. Sidebar offen/zu, aktiver Tab, User-Präferenzen) lebt in Zustand-Stores. Viele Apps brauchen keinen Zustand Store — React Query reicht oft aus.

6. **shadcn/ui-Komponenten sind unantastbar** — Dateien in `src/components/ui/` werden NIEMALS manuell editiert oder von Biome formatiert. Neue Komponenten kommen via `bunx shadcn@latest add <name>`.

7. **Feature-Module** — Zusammengehörige UI-Logik wird in `src/features/<feature-name>/` gruppiert. Jedes Feature-Modul enthält seine eigenen Komponenten.

8. **API Routes** — Server-seitige Logik nutzt Next.js Route Handlers in `src/app/api/`. Jeder Endpoint validiert Input mit Zod.

9. **Ultracite first** — Jede neue Datei muss `ultracite check` bestehen. Bei Konflikten zwischen Ultracite-Regeln und manueller Formatierung gewinnt Ultracite.

10. **Kein ESLint** — Biome (via Ultracite) ersetzt ESLint komplett.

### Code-Patterns aus der Praxis

#### API Client Pattern

```typescript
const BASE = "/api";

async function handleResponse<T>(res: Response): Promise<T> {
  if (!res.ok) {
    const text = await res.text().catch(() => "Unknown error");
    throw new Error(text || `HTTP ${res.status}`);
  }
  return res.json();
}

export async function fetchItems(): Promise<Item[]> {
  const res = await fetch(`${BASE}/items`);
  return handleResponse<Item[]>(res);
}

export async function saveItem(item: Item): Promise<Item> {
  const res = await fetch(`${BASE}/items`, {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(item),
  });
  return handleResponse<Item>(res);
}
```

#### React Query Hook Pattern

```typescript
// Simple Query
export function useItems() {
  return useQuery({
    queryKey: ["items"] as const,
    queryFn: fetchItems,
  });
}

// Mutation with Cache Update
export function useSaveItem() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: saveItem,
    onSuccess: (data) => {
      queryClient.setQueryData(["items"], data);
    },
  });
}

// Higher-Order Mutation (for CRUD-heavy domains)
function useModelMutation<TArgs>(
  updater: (current: Model, args: TArgs) => Model
) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (args: TArgs): Promise<Model> => {
      const current = queryClient.getQueryData<Model>(["model"]);
      if (!current) throw new Error("No model loaded");
      return saveModel(updater(current, args));
    },
    onSuccess: (data) => {
      queryClient.setQueryData(["model"], data);
    },
  });
}
```

#### TanStack Form Pattern

```typescript
const form = useForm({
  defaultValues: {
    name: initial?.name ?? "",
    type: (initial?.type ?? "string") as ItemType,
  },
  onSubmit: ({ value }) => {
    onSave(value);
  },
});

// Subscribe to field values outside of form.Field
const selectedType = useStore(form.store, (s) => s.values.type);

return (
  <form onSubmit={(e) => { e.preventDefault(); form.handleSubmit(); }}>
    <form.Field
      name="name"
      children={(field) => (
        <div>
          <Label>Name</Label>
          <Input
            value={field.state.value}
            onChange={(e) => field.handleChange(e.target.value)}
          />
        </div>
      )}
    />
  </form>
);
```

#### Custom Component Pattern (neben shadcn)

```typescript
interface ConfirmDialogProps {
  children: React.ReactElement;
  onConfirm: () => void;
  title?: string;
  description?: string;
  confirmLabel?: string;
  variant?: "destructive" | "default";
}

export function ConfirmDialog({
  children,
  onConfirm,
  title = "Are you sure?",
  description = "This action cannot be undone.",
  confirmLabel = "Continue",
  variant = "destructive",
}: ConfirmDialogProps) {
  return (
    <AlertDialog>
      <AlertDialogTrigger render={children} />
      <AlertDialogContent>
        {/* ... */}
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

### Konventionen

- **Dateinamen:** kebab-case für alle Dateien (`use-domain-model.ts`, `category-panel.tsx`)
- **Komponenten:** PascalCase Exports (`export function CategoryPanel()`)
- **Hooks:** camelCase mit `use`-Prefix (`useDomainModel`, `useConfigurator`)
- **Types:** In `domain/types.ts` sammeln, PascalCase
- **Schemas:** In `domain/schema.ts` sammeln, camelCase mit `Schema`-Suffix
- **Path Alias:** Immer `@/` nutzen statt relativer Imports (`import { cn } from "@/lib/utils"`)
- **Module Type:** ESM (`"type": "module"` in package.json)
- **Sprache:** Code, Variablen und Kommentare in Englisch
- **Props:** Interface `Props` oder beschreibend (z.B. `ConfirmDialogProps`), nie inline-Typen
- **No barrel files:** Keine `index.ts` Re-Exports, spezifische Imports bevorzugen
- **forwardRef:** Nicht verwenden — React 19+ nutzt ref als normales Prop

### Test-Patterns

```typescript
// Setup: Shared test data
const testModel = { /* ... */ };

function makeEngine(model?: Model) {
  return createEngine(model ?? testModel);
}

// Tests: Flat describe/it structure
describe("Validation rules", () => {
  it("passes when all constraints are met", () => {
    const engine = makeEngine();
    const result = engine.validate(validConfig);
    expect(result.issues).toHaveLength(0);
  });

  it("reports error when constraint is violated", () => {
    const engine = makeEngine();
    const result = engine.validate(invalidConfig);
    const issues = result.issues.filter((i) => i.ruleId === "rule-id");
    expect(issues).toHaveLength(1);
  });
});
```

- Domain-Tests in `src/tests/` — testen pure Logik ohne React
- `vitest` mit globals (`describe`, `it`, `expect` ohne Import)
- Flat `describe` Struktur, kein exzessives Nesting
- Factory-Funktionen für Test-Setup

---

## Checkliste nach Erstellung

Stelle sicher, dass folgende Punkte erfüllt sind, bevor du dem User die App übergibst:

- [ ] `bun run build` läuft fehlerfrei
- [ ] `bun run check` (ultracite) hat keine Fehler
- [ ] `src/components/ui/` ist von Biome ausgeschlossen (biome.jsonc overrides inkl. `assist`)
- [ ] Providers (QueryClient + ThemeProvider + TooltipProvider) sind im Root-Layout eingebunden
- [ ] `cn()` Utility existiert in `src/lib/utils.ts`
- [ ] Zod ist eingerichtet und ein Beispiel-Schema in `src/domain/schema.ts` vorhanden
- [ ] `vitest.config.ts` ist konfiguriert mit `@/` Alias (via `import.meta.dirname`)
- [ ] Kein ESLint — nur Biome/Ultracite
- [ ] Git Hooks via lefthook konfiguriert (fix + stage_fixed)
- [ ] `AGENTS.md` mit Ultracite Code Standards erstellt
- [ ] `.claude/settings.json` mit PostToolUse Hook erstellt
- [ ] `globals.css` enthält `scrollbar-gutter: stable` und `@custom-variant dark`
- [ ] Font via `variable: "--font-sans"` Pattern eingebunden
- [ ] `"type": "module"` in package.json gesetzt
