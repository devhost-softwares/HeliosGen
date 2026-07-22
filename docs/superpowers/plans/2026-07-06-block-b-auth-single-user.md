# Bloco B — Autenticação single-user (Implementation Plan)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Substituir Supabase Auth por autenticação própria single-user (um admin), com sessão em cookie httpOnly assinado, e remover completamente a dependência de `@supabase/*`.

**Architecture:** Uma tabela `users` (do Bloco A) guarda o admin. Login por email+senha (hash argon2) emite uma sessão `iron-session` em cookie httpOnly. `resolveUserId`/`requireUser` leem o cookie. `middleware.ts` protege rotas. A UI troca `lib/supabase/client` por rotas `/api/auth/*`. GUEST_MODE continua bypassando auth.

**Tech Stack:** Next.js 16, `iron-session`, `@node-rs/argon2` (hash), Vitest.

**Pré-requisito:** Bloco A concluído (Repository + tabela `users` existente).

## Global Constraints

- Node 22, pnpm.
- Ler `node_modules/next/dist/docs/` para o padrão de cookies/middleware desta versão do Next antes de codar.
- `GUEST_MODE=true` continua sem exigir login (single-user local).
- Cookie de sessão: httpOnly, `secure` em produção, `sameSite: "lax"`.
- Sem OAuth, sem magic link, sem reset por email. Troca de senha só autenticado.
- Segredo em `SESSION_SECRET` (≥ 32 chars). Falha explícita se ausente fora de guest.

---

## File Structure

- Create: `lib/auth/session.ts` — config iron-session, `getSession`, `requireUser`, `SessionData`.
- Create: `lib/auth/password.ts` — `hashPassword`, `verifyPassword`.
- Create: `lib/auth/ensureAdmin.ts` — cria admin a partir de `ADMIN_EMAIL`/`ADMIN_PASSWORD` se ausente.
- Create: `app/api/auth/login/route.ts`, `app/api/auth/logout/route.ts`, `app/api/auth/me/route.ts`.
- Modify: `lib/guestMode.ts` — `resolveUserId` lê a sessão.
- Modify: `middleware.ts` — valida cookie de sessão (remove Supabase).
- Modify: `lib/getKieToken.ts`, `lib/getAzureKey.ts` — `getKieToken(req)`/`getAzureToken(req)` usam sessão.
- Modify: UI — `components/AuthModal.tsx`, `AuthButton.tsx`, `ResetPasswordModal.tsx`, `SettingsModal.tsx`, `CreditBalance.tsx`, `QuickAssist.tsx`, `app/chat/page.tsx`, `app/gallery/page.tsx`, e nós que importam `lib/supabase/client`.
- Delete (final): `lib/supabase/client.ts`, `lib/supabase/server.ts`, `lib/supabase/admin.ts`, `app/api/auth/callback/route.ts` (callback OAuth Supabase).
- Modify: `package.json` (remover `@supabase/*`, add `iron-session`, `@node-rs/argon2`), `.env.example`.

---

## Task 1: Dependências e env

**Files:** Modify `package.json`, `.env.example`.

- [ ] **Step 1: Instalar/remover deps**

```bash
pnpm add iron-session @node-rs/argon2
```
(Remover `@supabase/ssr` e `@supabase/supabase-js` só na Task 8, quando não houver mais imports.)

- [ ] **Step 2: `.env.example`** — adicionar seção:

```
# ── Auth (single-user) ────────────────────────────────────────────────────────
SESSION_SECRET=            # >= 32 chars aleatórios
ADMIN_EMAIL=              # email do admin criado no primeiro boot
ADMIN_PASSWORD=           # senha inicial do admin
```

- [ ] **Step 3: Commit**

```bash
git add package.json pnpm-lock.yaml .env.example
git commit -m "chore(auth): deps iron-session e argon2"
```

---

## Task 2: Hash de senha (TDD)

**Files:** Create `lib/auth/password.ts`, `lib/auth/password.test.ts`.

**Interfaces:** Produces `hashPassword(pw: string): Promise<string>`, `verifyPassword(hash: string, pw: string): Promise<boolean>`.

- [ ] **Step 1: Teste que falha** — `lib/auth/password.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { hashPassword, verifyPassword } from "./password";

describe("password", () => {
  it("verifica senha correta e rejeita errada", async () => {
    const h = await hashPassword("s3nha-forte");
    expect(await verifyPassword(h, "s3nha-forte")).toBe(true);
    expect(await verifyPassword(h, "errada")).toBe(false);
  });
});
```

- [ ] **Step 2: Rodar e ver falhar** — Run: `pnpm test password` — Expected: FAIL (módulo inexistente).

- [ ] **Step 3: Implementar `lib/auth/password.ts`**

```ts
import { hash, verify } from "@node-rs/argon2";

export function hashPassword(pw: string): Promise<string> {
  return hash(pw);
}
export function verifyPassword(h: string, pw: string): Promise<boolean> {
  return verify(h, pw).catch(() => false);
}
```

- [ ] **Step 4: Rodar e ver passar** — Run: `pnpm test password` — Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/auth/password.ts lib/auth/password.test.ts
git commit -m "feat(auth): hash de senha argon2 com teste"
```

---

## Task 3: Sessão iron-session

**Files:** Create `lib/auth/session.ts`.

**Interfaces:** Produces `SessionData { userId?: string; email?: string }`, `getSession(): Promise<IronSession<SessionData>>`, `requireUser(): Promise<string>` (lança/redireciona se ausente).

- [ ] **Step 1: Criar `lib/auth/session.ts`**

```ts
import { getIronSession, type IronSession } from "iron-session";
import { cookies } from "next/headers";
import { GUEST_MODE, GUEST_USER_ID } from "@/lib/guestMode";

export interface SessionData { userId?: string; email?: string; }

const options = {
  password: process.env.SESSION_SECRET ?? "",
  cookieName: "heliosgen_session",
  cookieOptions: {
    httpOnly: true,
    sameSite: "lax" as const,
    secure: process.env.NODE_ENV === "production",
    path: "/",
  },
};

export async function getSession(): Promise<IronSession<SessionData>> {
  const store = await cookies();
  return getIronSession<SessionData>(store, options);
}

export async function getUserId(): Promise<string | null> {
  if (GUEST_MODE) return GUEST_USER_ID;
  const s = await getSession();
  return s.userId ?? null;
}
```

> **Nota:** Se `SESSION_SECRET` estiver vazio fora do GUEST_MODE, `iron-session` lança. Adicionar checagem explícita no boot (Task 5) com mensagem clara.

- [ ] **Step 2: Compilar** — Run: `pnpm exec tsc --noEmit` — Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add lib/auth/session.ts
git commit -m "feat(auth): sessao iron-session"
```

---

## Task 4: Rotas de auth (login/logout/me)

**Files:** Create `app/api/auth/login/route.ts`, `logout/route.ts`, `me/route.ts`.

**Interfaces:** Consumes `getSession`, `getRepo`, `verifyPassword`.

- [ ] **Step 1: `login/route.ts`**

```ts
import { NextRequest, NextResponse } from "next/server";
import { getSession } from "@/lib/auth/session";
import { getRepo } from "@/lib/repo";
import { verifyPassword } from "@/lib/auth/password";

export async function POST(req: NextRequest) {
  const { email, password } = await req.json();
  if (!email || !password) return NextResponse.json({ error: "email e senha obrigatórios" }, { status: 400 });
  const user = await getRepo().users.getByEmail(email);
  if (!user || !(await verifyPassword(user.passwordHash, password))) {
    return NextResponse.json({ error: "Credenciais inválidas" }, { status: 401 });
  }
  const session = await getSession();
  session.userId = user.id;
  session.email = user.email;
  await session.save();
  return NextResponse.json({ ok: true, email: user.email });
}
```

> Requer adicionar `users.getByEmail(email)` e `users.getById(id)` à interface `Repo` (Bloco A `lib/repo/types.ts` + `pg.ts`). Adicioná-los aqui se ainda não existirem:
> ```ts
> users: {
>   getByEmail(email: string): Promise<{ id: string; email: string; passwordHash: string } | null>;
>   getById(id: string): Promise<{ id: string; email: string } | null>;
>   create(u: { email: string; passwordHash: string; isAdmin?: boolean }): Promise<{ id: string }>;
> }
> ```

- [ ] **Step 2: `logout/route.ts`**

```ts
import { NextResponse } from "next/server";
import { getSession } from "@/lib/auth/session";
export async function POST() {
  const s = await getSession();
  s.destroy();
  return NextResponse.json({ ok: true });
}
```

- [ ] **Step 3: `me/route.ts`**

```ts
import { NextResponse } from "next/server";
import { getUserId, getSession } from "@/lib/auth/session";
export async function GET() {
  const uid = await getUserId();
  if (!uid) return NextResponse.json({ user: null });
  const s = await getSession();
  return NextResponse.json({ user: { id: uid, email: s.email ?? null } });
}
```

- [ ] **Step 4: Compilar** — Run: `pnpm exec tsc --noEmit` — Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/api/auth/login app/api/auth/logout app/api/auth/me lib/repo
git commit -m "feat(auth): rotas login/logout/me + users no repo"
```

---

## Task 5: Bootstrap do admin

**Files:** Create `lib/auth/ensureAdmin.ts`; Modify um ponto de boot (ex.: `middleware.ts` ou instrumentation) para chamar idempotente.

**Interfaces:** Produces `ensureAdmin(): Promise<void>`.

- [ ] **Step 1: `lib/auth/ensureAdmin.ts`**

```ts
import { getRepo } from "@/lib/repo";
import { hashPassword } from "./password";

let done = false;
export async function ensureAdmin(): Promise<void> {
  if (done || process.env.GUEST_MODE === "true") return;
  const email = process.env.ADMIN_EMAIL;
  const password = process.env.ADMIN_PASSWORD;
  if (!email || !password) return; // configuração incompleta — não cria
  const existing = await getRepo().users.getByEmail(email);
  if (!existing) {
    await getRepo().users.create({ email, passwordHash: await hashPassword(password), isAdmin: true });
  }
  done = true;
}
```

- [ ] **Step 2: Invocar no boot** — usar `instrumentation.ts` do Next (ler `node_modules/next/dist/docs/` para o hook correto nesta versão) chamando `ensureAdmin()` em `register()`. Se `instrumentation` não estiver disponível, chamar no início de `login` e `me`.

- [ ] **Step 3: Teste manual** — subir com `ADMIN_EMAIL`/`ADMIN_PASSWORD` e `POST /api/auth/login` — Expected: 200 e cookie setado.

- [ ] **Step 4: Commit**

```bash
git add lib/auth/ensureAdmin.ts instrumentation.ts
git commit -m "feat(auth): bootstrap idempotente do admin"
```

---

## Task 6: `resolveUserId`, helpers de token e middleware

**Files:** Modify `lib/guestMode.ts`, `lib/getKieToken.ts`, `lib/getAzureKey.ts`, `middleware.ts`.

- [ ] **Step 1: `lib/guestMode.ts`** — trocar `resolveUserId` para ler a sessão:

```ts
import { GUEST_MODE, GUEST_USER_ID } from "./guestMode"; // (mesmo módulo — manter consts)
export async function resolveUserId(): Promise<string | null> {
  if (GUEST_MODE) return GUEST_USER_ID;
  const { getUserId } = await import("./auth/session");
  return getUserId();
}
```

> **Breaking:** a assinatura muda de `resolveUserId(req)` para `resolveUserId()` (a sessão vem de cookies via `next/headers`). Atualizar TODAS as chamadas em `app/api/**` (grep `resolveUserId(`) para remover o argumento `req`.

- [ ] **Step 2: `getKieToken(req)` / `getAzureToken(req)`** — remover leitura do header Bearer e `supabaseAdmin.auth.getUser`; passar a resolver userId via sessão:

```ts
import { getUserId } from "@/lib/auth/session";
export async function getKieToken(): Promise<string | null> {
  const uid = await getUserId();
  return uid ? getKieTokenForUser(uid) : null;
}
```
Atualizar chamadas correspondentes nas rotas (`assistant`, etc.) para a nova assinatura sem `req`.

- [ ] **Step 3: `middleware.ts`** — remover Supabase; validar sessão:

```ts
import { NextResponse, type NextRequest } from "next/server";
import { GUEST_MODE } from "@/lib/guestMode";

export async function middleware(request: NextRequest) {
  if (GUEST_MODE) return NextResponse.next();
  const hasSession = request.cookies.has("heliosgen_session");
  const isAuthRoute = request.nextUrl.pathname.startsWith("/api/auth");
  if (!hasSession && !isAuthRoute) {
    // páginas: deixar a UI abrir o AuthModal; APIs: 401 vem das próprias rotas via requireUser
    return NextResponse.next();
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|api/callback|api/upload-to-r2).*)"],
};
```

> **Nota:** O middleware não valida a assinatura do cookie (só presença). A validação real acontece em `getUserId()` dentro das rotas. Rotas de dados que exigem usuário devem chamar `getUserId()` e retornar 401 se nulo (padrão já usado hoje com `resolveUserId`).

- [ ] **Step 4: Compilar** — Run: `pnpm exec tsc --noEmit` — Expected: PASS (corrigir todas as chamadas com `req` removido).

- [ ] **Step 5: Commit**

```bash
git add lib/guestMode.ts lib/getKieToken.ts lib/getAzureKey.ts middleware.ts app/api
git commit -m "refactor(auth): sessao por cookie substitui token supabase"
```

---

## Task 7: UI de autenticação

**Files:** Modify `components/AuthModal.tsx`, `AuthButton.tsx`, `ResetPasswordModal.tsx`, `SettingsModal.tsx`, `CreditBalance.tsx`, `QuickAssist.tsx`, `app/chat/page.tsx`, `app/gallery/page.tsx`, nós em `components/nodes/*` que importam `lib/supabase/client`.

- [ ] **Step 1: Substituir client Supabase por chamadas às rotas** — em cada componente:
  - Onde havia `supabase.auth.signInWithPassword(...)` → `fetch("/api/auth/login", { method: "POST", body: JSON.stringify({ email, password }) })`.
  - `supabase.auth.signOut()` → `fetch("/api/auth/logout", { method: "POST" })`.
  - `supabase.auth.getUser()` / `getSession()` → `fetch("/api/auth/me").then(r => r.json())`.
  - Remover envio de header `Authorization: Bearer <token>` nas chamadas de API — a sessão vai por cookie automaticamente (`credentials: "same-origin"` é o default).

- [ ] **Step 2: `AuthModal`** — reduzir a **apenas login** (single-user). Remover a aba de signup e o fluxo OAuth. `ResetPasswordModal` → troca de senha autenticada (chamar nova rota `POST /api/auth/change-password` se desejado; caso contrário remover o modal). Se `NEXT_PUBLIC_DEMO_MODE`, manter o comportamento de demo existente.

> **Decisão registrada no spec:** single-user, sem signup aberto. Não recriar cadastro.

- [ ] **Step 3: Remover envio de Bearer nas chamadas fetch** dos nós (`GenerateNode`, `VideoGeneratorNode`, etc.) que hoje anexam o token Supabase. As rotas passam a usar o cookie.

- [ ] **Step 4: Rodar** — Run: `pnpm dev` (com admin configurado); logar via UI, gerar imagem, deslogar.
Expected: login funciona, sessão persiste, logout limpa.

- [ ] **Step 5: Commit**

```bash
git add components app
git commit -m "refactor(auth): UI usa rotas /api/auth (remove supabase client)"
```

---

## Task 8: Remoção do Supabase

**Files:** Delete `lib/supabase/client.ts`, `lib/supabase/server.ts`, `lib/supabase/admin.ts`, `app/api/auth/callback/route.ts`; Modify `package.json`.

- [ ] **Step 1: Grep final**

Run: buscar `@/lib/supabase`, `@supabase/`, `supabaseAdmin`, `createBrowserClient`, `createServerClient` em todo o repo.
Expected: nenhum resultado.

- [ ] **Step 2: Deletar arquivos supabase e a rota de callback OAuth.**

```bash
git rm lib/supabase/client.ts lib/supabase/server.ts lib/supabase/admin.ts app/api/auth/callback/route.ts
```

- [ ] **Step 3: Remover deps**

```bash
pnpm remove @supabase/ssr @supabase/supabase-js
```

- [ ] **Step 4: Build completo**

Run: `pnpm build` e `GUEST_MODE=true pnpm build`
Expected: ambos OK.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore(auth): remove supabase por completo"
```

---

## Self-Review

- [ ] **Cobertura:** tabela users (Bloco A) + login/logout/me (Task 4) + bootstrap admin (Task 5) + sessão em cookie (Task 3/6) + UI (Task 7) + remoção Supabase (Task 8). ✅
- [ ] **Placeholder scan:** sem TODO/TBD; `ResetPasswordModal` resolvido (troca de senha ou removido) explicitamente.
- [ ] **Consistência de tipos:** `getSession`, `getUserId`, `users.getByEmail/getById/create`, `resolveUserId()` (sem arg) usados de forma consistente entre tasks.
- [ ] **Grep:** zero referências a `@supabase` e `supabaseAdmin` no repo.
- [ ] **GUEST_MODE** continua sem exigir login.

## Handoff

Após Bloco B, o app não depende mais de Supabase (nem dados nem auth). Próximo: **Bloco C** (provider selecionável).
