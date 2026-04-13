---
name: create-hm-webapp
description: Erstellt eine opinionated hellomirrors web-app unter Nutzung von NextJS, shadcn/ui, Tailwind CSS 4, Ultracite/Biome, Zod, TanStack Query/Form, Zustand
allowed-tools: Bash(pnpm *), Bash(pnpx *), Bash(npx *), Bash(cd *), Bash(mkdir *), Bash(rm *), Bash(ls *), Read, Write, Edit, Glob, Grep
---

# HM Web-App Skill

Dieser Skill erstellt eine neue Web-Applikation nach dem hellomirrors-Standard-Stack.

Frage den User **vor dem Start** nach:
1. **Projektname** — wird als Verzeichnisname und in package.json verwendet
2. **Kurze Beschreibung** der App
3. **Authentik Login?** — Soll die App über authentik.hellomirrors.com abgesichert werden? (Standard: ja)

---

## Phase 1: Projekt-Scaffolding (via Tools)

### 1.1 Next.js App erstellen

Führe folgenden Befehl aus, um das Next.js-Projekt zu erstellen:

```bash
pnpx create-next-app@latest <projektname> \
  --typescript \
  --tailwind \
  --biome \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --turbopack \
  --use-pnpm \
  --yes
```

> `--biome` sorgt dafür, dass Biome statt ESLint eingerichtet wird — kein manuelles Aufräumen von ESLint-Dateien nötig. `--use-pnpm` stellt sicher, dass pnpm als Package Manager genutzt wird.

### 1.2 Dependencies installieren

Wechsle ins Projektverzeichnis und installiere die zusätzlichen Pakete:

```bash
cd <projektname>

# Core UI & Styling
pnpm add clsx tailwind-merge class-variance-authority tw-animate-css lucide-react next-themes sonner

# Data Fetching & Forms
pnpm add @tanstack/react-query @tanstack/react-form zod

# State Management (optional — nur installieren wenn Client-State benötigt wird)
pnpm add zustand

# Dev Dependencies (ultracite erweitert die von create-next-app installierte Biome-Config)
pnpm add -D ultracite vitest lefthook @types/node @types/react
```

> **Zustand** ist optional. Installiere es nur, wenn die App Client-State hat, der nicht vom Server kommt (z.B. Sidebar offen/zu, aktiver Tab, User-Präferenzen). Viele Apps kommen mit React Query allein aus.

### 1.2.1 Biome-Version pinnen

**Kritisch:** Ultracite deklariert die exakte Biome-Version die es benötigt als `peerDependency`. Die von `create-next-app` installierte Biome-Version kann inkompatibel sein. Prüfe welche Version Ultracite erwartet und installiere diese:

```bash
# Prüfe welche Biome-Version ultracite erwartet
grep biome node_modules/ultracite/package.json
# Installiere die exakte Version (Beispiel: 2.4.11)
pnpm add -D @biomejs/biome@2.4.11
```

Wenn `create-next-app` eine `biome.json` erstellt hat, **lösche sie** — wir nutzen `biome.jsonc` (siehe Phase 2.1):

```bash
rm biome.json
```

### 1.2.2 pnpm onlyBuiltDependencies

Füge in `package.json` die `pnpm`-Sektion hinzu, damit native Dependencies kompiliert werden:

```json
{
  "pnpm": {
    "onlyBuiltDependencies": [
      "sharp",
      "unrs-resolver",
      "lefthook",
      "@biomejs/biome"
    ]
  }
}
```

Falls `better-sqlite3` oder andere native Pakete verwendet werden, diese ebenfalls hier hinzufügen.

### 1.3 shadcn/ui initialisieren

```bash
pnpx shadcn@latest init --preset mira --css-variables -y
```

> **Hinweis:** Die CLI-Optionen ändern sich gelegentlich. Falls `--preset mira` nicht funktioniert, nutze `pnpx shadcn@latest init --help` um die aktuellen Optionen zu prüfen.

Danach installiere die Standard-Komponenten. Nutze bevorzugt den **shadcn MCP Server** (Tool `add_component`) um Komponenten hinzuzufügen, da dieser die beste Kompatibilität mit dem gewählten Preset sicherstellt. Fallback auf CLI:

```bash
pnpx shadcn@latest add input button label alert-dialog select checkbox radio-group badge card scroll-area switch textarea tooltip dialog tabs separator sheet
```

### 1.4 MCP Server einrichten

Erstelle `.mcp.json` im Projektroot mit den MCP Servern für shadcn, Next.js DevTools und Ultracite:

```json
{
  "mcpServers": {
    "shadcn": {
      "command": "pnpm",
      "args": ["dlx", "shadcn@latest", "mcp"]
    },
    "next-devtools": {
      "command": "pnpm",
      "args": ["dlx", "next-devtools-mcp@latest"]
    },
    "ultracite": {
      "command": "pnpm",
      "args": ["dlx", "mcp-remote", "https://docs.ultracite.ai/mcp"]
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
            "command": "pnpm fix --skip=correctness/noUnusedImports",
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
    },
    {
      // HTML files — template files, do not lint or format
      "includes": ["*.html"],
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

Ersetze die `scripts`-Sektion in `package.json`. Stelle sicher, dass `"type": "module"` und `"packageManager"` gesetzt sind:

```json
{
  "type": "module",
  "packageManager": "pnpm@10.26.2",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "test": "vitest run",
    "test:watch": "vitest",
    "check": "ultracite check",
    "fix": "ultracite fix",
    "install:lefthook": "lefthook install",
    "outdated": "pnpm outdated"
  }
}
```

> **Wichtig:** `install:lefthook` statt `prepare` verwenden! Das `prepare`-Script wird bei jedem `pnpm install` ausgeführt — auch in Docker-Builds, wo `git` nicht verfügbar ist. `lefthook install` braucht `git` und bricht den Docker-Build ab. Nach dem lokalen `pnpm install` einmalig `pnpm run install:lefthook` ausführen.

### 2.7 lefthook.yml (Git Hooks)

Erstelle `lefthook.yml` im Projektroot. Der Hook führt `ultracite fix` aus (nicht nur check), stagt die gefixten Dateien automatisch und filtert auf relevante Dateitypen:

```yaml
pre-commit:
  jobs:
    - run: pnpm exec ultracite fix
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

Falls Authentik-Auth aktiviert ist, wird die `providers.tsx` um `SessionProvider` erweitert — siehe Phase 6.

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

- **Format code**: `pnpm exec ultracite fix`
- **Check for issues**: `pnpm exec ultracite check`
- **Diagnose setup**: `pnpm exec ultracite doctor`

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

Most formatting and common issues are automatically fixed by Biome. Run `pnpm exec ultracite fix` before committing to ensure compliance.
```

---

## Phase 5: Ultracite Check ausführen

Nach der vollständigen Einrichtung, führe den Formatter/Linter aus um sicherzustellen, dass alles den Konventionen entspricht:

```bash
pnpm run fix
```

Prüfe danach:

```bash
pnpm run check
```

Behebe alle gemeldeten Fehler **außer** in `src/components/ui/` — diese Dateien sind von Biome ausgeschlossen und dürfen NICHT manuell reformatiert werden.

---

## Phase 6: Authentik-Authentifizierung (optional)

Nur ausführen wenn der User Authentik-Login gewünscht hat.

### 6.1 Dependencies

```bash
pnpm add next-auth@beta
```

### 6.2 Auth-Konfiguration

Erstelle `src/lib/auth.ts`:

```typescript
import NextAuth from "next-auth";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    {
      id: "authentik",
      name: "Authentik",
      type: "oidc",
      issuer: process.env.AUTHENTIK_ISSUER,
      clientId: process.env.AUTHENTIK_CLIENT_ID,
      clientSecret: process.env.AUTHENTIK_CLIENT_SECRET,
      client: {
        token_endpoint_auth_method: "client_secret_post",
      },
      profile(profile) {
        return {
          id: profile.sub,
          name: profile.name ?? profile.preferred_username,
          email: profile.email,
          image: profile.picture ?? null,
        };
      },
    },
  ],
  pages: {
    signIn: "/login",
    error: "/login",
  },
  callbacks: {
    authorized: async ({ auth: session }) => !!session,
  },
  trustHost: true,
});
```

### 6.3 Auth API Route

Erstelle `src/app/api/auth/[...nextauth]/route.ts`:

```typescript
import { handlers } from "@/lib/auth";

export const { GET, POST } = handlers;
```

### 6.4 Middleware

Erstelle `src/middleware.ts`:

```typescript
import { auth } from "@/lib/auth";

export const middleware = auth;

export const config = {
  matcher: [
    "/((?!api/auth|login|_next/static|_next/image|favicon\\.svg|manifest\\.json|icons/).*)",
  ],
};
```

> **Nicht** `export { auth as middleware }` verwenden — Biome meldet das als Barrel-File (`noBarrelFile`).

### 6.5 Login-Seite

Erstelle `src/app/login/page.tsx`:

```typescript
import { signIn } from "@/lib/auth";

export default function LoginPage() {
  return (
    <div className="flex min-h-screen items-center justify-center p-4">
      <div className="w-full max-w-sm space-y-6 text-center">
        <div>
          <h1 className="font-bold text-2xl tracking-tight">App Name</h1>
          <p className="mt-1 text-muted-foreground text-sm">hellomirrors</p>
        </div>
        <form
          action={async () => {
            "use server";
            await signIn("authentik", { redirectTo: "/" });
          }}
        >
          <button
            type="submit"
            className="inline-flex h-10 w-full items-center justify-center rounded-md bg-primary px-4 font-medium text-primary-foreground text-sm transition-colors hover:bg-primary/90"
          >
            Mit Authentik anmelden
          </button>
        </form>
      </div>
    </div>
  );
}
```

### 6.6 Providers erweitern

Wenn Auth aktiv ist, wrapp die `providers.tsx` zusätzlich mit `SessionProvider`:

```typescript
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { SessionProvider } from "next-auth/react";
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
    <SessionProvider>
      <QueryClientProvider client={queryClient}>
        <ThemeProvider
          attribute="class"
          defaultTheme="dark"
          disableTransitionOnChange
        >
          <TooltipProvider>{children}</TooltipProvider>
        </ThemeProvider>
      </QueryClientProvider>
    </SessionProvider>
  );
}
```

### 6.7 ENV Variablen

Erstelle `.env.example`:

```env
# Auth (Authentik OIDC)
AUTH_URL=https://<app-name>.hellomirrors.com
AUTH_SECRET=change-me-generate-with-openssl-rand-base64-32
AUTHENTIK_ISSUER=https://authentik.hellomirrors.com/application/o/<app-slug>/
AUTHENTIK_CLIENT_ID=
AUTHENTIK_CLIENT_SECRET=
AUTH_TRUST_HOST=true
```

### 6.8 Authentik-Einrichtung (Anleitung für den User)

Erstelle `docs/authentik-setup.md` mit Schritt-für-Schritt Anleitung:

1. **Provider erstellen** in Authentik: Applications → Providers → Create → OAuth2/OpenID Provider
   - Name: `<app-slug>`
   - Authorization flow: `default-provider-authorization-explicit-consent`
   - Client type: `Confidential`
   - Redirect URIs: `https://<app-url>/api/auth/callback/authentik` (+ `http://localhost:3000/api/auth/callback/authentik` für Dev)
   - Scopes: `openid`, `profile`, `email`
   - **Signing Key:** Einen Key auswählen
   - **⚠️ Encryption Key: LEER LASSEN!** NextAuth unterstützt kein JWE. Wenn ein Encryption Key gesetzt ist, sendet Authentik verschlüsselte id_tokens die NextAuth nicht entschlüsseln kann → `JWE decryption is not configured` Fehler.

2. **Application erstellen**: Applications → Applications → Create
   - Name: `<App Name>`
   - Slug: `<app-slug>`
   - Provider: den eben erstellten Provider auswählen

3. **Zugriff einschränken (optional)**: Policy / Group / User Bindings

4. **ENV Variablen** in Coolify setzen (inkl. `AUTH_URL` mit der öffentlichen App-URL)

5. **Issuer URL testen**: `https://authentik.hellomirrors.com/application/o/<app-slug>/.well-known/openid-configuration` muss ein JSON liefern

### 6.9 Bekannte Fallstricke bei Authentik + NextAuth

| Problem | Ursache | Lösung |
|---------|---------|--------|
| `JWE decryption is not configured` | Encryption Key ist im Authentik Provider gesetzt | In Authentik den **Encryption Key entfernen** (Signing Key behalten) |
| Redirect zu `0.0.0.0:3000` nach Login | `AUTH_URL` ENV Variable fehlt | `AUTH_URL=https://<app>.hellomirrors.com` setzen |
| `error=Configuration` ohne Details | `AUTHENTIK_CLIENT_ID` oder `AUTHENTIK_CLIENT_SECRET` leer | In Coolify prüfen ob die ENV Variablen befüllt sind |
| OIDC Discovery schlägt fehl | Container kann `authentik.hellomirrors.com` nicht per DNS auflösen | Netzwerk-Konfiguration prüfen |
| `noBarrelFile` Lint-Fehler bei Middleware | `export { auth as middleware }` Pattern | Stattdessen `export const middleware = auth;` verwenden |

---

## Phase 7: Docker & Coolify Deployment

### 7.1 next.config.ts für Docker

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  experimental: {
    typedEnv: true,
  },
  // Falls native Pakete verwendet werden (z.B. better-sqlite3):
  // serverExternalPackages: ["better-sqlite3"],
};

export default nextConfig;
```

### 7.2 Dockerfile

```dockerfile
FROM node:22-slim AS base
RUN corepack enable && corepack prepare pnpm@10.26.2 --activate

# Install dependencies only when needed
FROM base AS deps
# Build tools für native Dependencies (z.B. better-sqlite3) — nur wenn benötigt
RUN apt-get update && apt-get install -y --no-install-recommends python3 make g++ && rm -rf /var/lib/apt/lists/*
WORKDIR /app

COPY package.json pnpm-lock.yaml ./
# --ignore-scripts verhindert dass "prepare"/"install:lefthook" läuft (braucht git, nicht in Docker)
# Danach native deps explizit rebuilden
RUN pnpm install --frozen-lockfile --shamefully-hoist --ignore-scripts && \
    pnpm rebuild esbuild
# Falls better-sqlite3 verwendet wird:
# RUN pnpm install --frozen-lockfile --shamefully-hoist --ignore-scripts && \
#     pnpm rebuild better-sqlite3 esbuild

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NEXT_TELEMETRY_DISABLED=1
RUN pnpm run build

# Production image
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*

RUN groupadd --system --gid 1001 nodejs
RUN useradd --system --uid 1001 --gid nodejs nextjs

COPY --from=builder /app/public ./public

RUN mkdir .next
RUN chown nextjs:nodejs .next

COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Full node_modules für server-external packages (native bindings)
COPY --from=deps /app/node_modules ./node_modules

# Default ENV config
COPY --from=builder /app/.env.example ./.env

USER nextjs

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

### 7.3 docker-compose.yaml

**Wichtig für Coolify:** Kein `ports` verwenden — nur `expose`. Coolify managt Port-Binding über seinen Reverse Proxy. Hardcoded `ports: "3000:3000"` kollidiert mit anderen Services auf dem Server.

```yaml
services:
  <app-name>:
    build:
      context: .
      dockerfile: Dockerfile
    expose:
      - "3000"
    environment:
      - AUTH_URL=${AUTH_URL:-https://<app-name>.hellomirrors.com}
      - AUTH_SECRET=${AUTH_SECRET}
      - AUTHENTIK_ISSUER=${AUTHENTIK_ISSUER:-https://authentik.hellomirrors.com/application/o/<app-slug>/}
      - AUTHENTIK_CLIENT_ID=${AUTHENTIK_CLIENT_ID}
      - AUTHENTIK_CLIENT_SECRET=${AUTHENTIK_CLIENT_SECRET}
      - AUTH_TRUST_HOST=true
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s

# Falls persistenter Speicher benötigt wird (z.B. SQLite):
#   volumes:
#     - app-data:/app/data
#
# volumes:
#   app-data:
```

### 7.4 .dockerignore

```
node_modules
.next
.git
.gitignore
data
drizzle
*.md
!README.md
.env
.env.local
.env.*.local
!.env.example
.claude
.mcp.json
docker-compose.yaml
```

### 7.5 Coolify Deployment Checkliste

1. **Build Pack:** Docker Compose
2. **Docker Compose Location:** `/docker-compose.yaml`
3. **"Load Compose File"** klicken um die Datei aus dem Repo zu laden
4. **Environment Variables** in Coolify setzen (alle `${VAR}` Variablen aus dem Compose)
5. **Persistent Storage:** Falls Volumes definiert sind, erscheinen sie automatisch nach Compose-Load

### 7.6 Bekannte Docker/Coolify Fallstricke

| Problem | Ursache | Lösung |
|---------|---------|--------|
| `pnpm install` schlägt fehl mit `exec: "git": not found` | `prepare`/`lefthook install` Script läuft bei `pnpm install` | Script in `install:lefthook` umbenennen + `--ignore-scripts` im Dockerfile |
| `Bind for 0.0.0.0:3000 failed: port already allocated` | `ports: "3000:3000"` im Compose kollidiert mit anderem Service | `expose: "3000"` statt `ports` verwenden (Coolify managt Ports) |
| `.env.example` nicht im Container | `.env.*` in `.dockerignore` schließt `.env.example` aus | Explizit `!.env.example` in `.dockerignore` ausschließen |
| Build fehlerfrei, Container startet nicht | `output: "standalone"` fehlt in `next.config.ts` | `output: "standalone"` hinzufügen |
| Native Deps fehlen zur Laufzeit | `serverExternalPackages` nicht konfiguriert | Native Pakete in `serverExternalPackages` auflisten + `COPY --from=deps node_modules` |

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

6. **shadcn/ui-Komponenten sind unantastbar** — Dateien in `src/components/ui/` werden NIEMALS manuell editiert oder von Biome formatiert. Neue Komponenten kommen via `pnpx shadcn@latest add <name>`.

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

- [ ] `pnpm run build` läuft fehlerfrei
- [ ] `pnpm run check` (ultracite) hat keine Fehler
- [ ] `src/components/ui/` ist von Biome ausgeschlossen (biome.jsonc overrides inkl. `assist`)
- [ ] Providers (QueryClient + ThemeProvider + TooltipProvider) sind im Root-Layout eingebunden
- [ ] `cn()` Utility existiert in `src/lib/utils.ts`
- [ ] Zod ist eingerichtet und ein Beispiel-Schema in `src/domain/schema.ts` vorhanden
- [ ] `vitest.config.ts` ist konfiguriert mit `@/` Alias (via `import.meta.dirname`)
- [ ] Kein ESLint — nur Biome/Ultracite
- [ ] Biome-Version ist auf die von Ultracite benötigte Version gepinnt
- [ ] Git Hooks via lefthook konfiguriert (fix + stage_fixed)
- [ ] lefthook Script heißt `install:lefthook` (nicht `prepare`!)
- [ ] `AGENTS.md` mit Ultracite Code Standards erstellt
- [ ] `.claude/settings.json` mit PostToolUse Hook erstellt
- [ ] `globals.css` enthält `scrollbar-gutter: stable` und `@custom-variant dark`
- [ ] Font via `variable: "--font-sans"` Pattern eingebunden
- [ ] `"type": "module"` und `"packageManager"` in package.json gesetzt
- [ ] Falls Auth: Authentik OIDC funktioniert, kein Encryption Key in Authentik gesetzt
- [ ] Falls Docker: `output: "standalone"` in next.config.ts, Dockerfile mit `--ignore-scripts`
- [ ] Falls Coolify: `expose` statt `ports` im docker-compose, `.env.example` nicht in .dockerignore
