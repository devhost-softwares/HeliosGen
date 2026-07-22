# Bloco A — PostgreSQL + Drizzle + Repository (Implementation Plan)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Substituir o acesso a dados Supabase por PostgreSQL via Drizzle ORM, atrás de uma camada Repository única que colapsa os branches `if (GUEST_MODE) … else supabaseAdmin …`.

**Architecture:** Introduzir `lib/db/` (schema Drizzle + client + repositório Pg) e `lib/repo/` (interface `Repo` + implementações `pg` e `guest`). Todas as rotas passam a chamar `getRepo()` em vez de `supabaseAdmin.from(...)` ou `guestDb.*`. Supabase Auth permanece intacto neste bloco (removido no Bloco B).

**Tech Stack:** Next.js 16, TypeScript, Drizzle ORM (`drizzle-orm`, `drizzle-kit`), `postgres` (postgres.js driver), Vitest (novo, para testes de unidade da camada de dados).

## Global Constraints

- Node 22, pnpm (lockfile `pnpm-lock.yaml` — usar `pnpm add`, nunca npm/yarn).
- Antes de escrever qualquer código, ler os guias em `node_modules/next/dist/docs/` (AGENTS.md: este Next.js tem breaking changes).
- `GUEST_MODE=true` deve continuar 100% funcional após cada tarefa.
- Nenhuma rota pode importar `@/lib/supabase/admin` ao final do bloco (exceto código de auth, tratado no Bloco B).
- Tipos JSON: `jsonb` para `spaces.data` e `generations.image_urls`; `text[]` para `generations.reference_image_urls`.
- Autorização por aplicação: toda query de leitura/escrita por usuário filtra por `user_id`. Sem RLS.

---

## File Structure

- Create: `lib/db/client.ts` — conexão postgres.js + instância Drizzle.
- Create: `lib/db/schema.ts` — tabelas Drizzle (`users`, `generations`, `spaces`, `folders`, `user_uploads`, `user_settings`, `asset_cache`).
- Create: `lib/db/migrations/` — output do `drizzle-kit generate`.
- Create: `drizzle.config.ts` — config do drizzle-kit.
- Create: `lib/repo/types.ts` — interface `Repo` e DTOs.
- Create: `lib/repo/pg.ts` — implementação Drizzle da `Repo`.
- Create: `lib/repo/guest.ts` — adapta `lib/guest/db.ts` à interface `Repo`.
- Create: `lib/repo/index.ts` — `getRepo()` (escolhe pg/guest por `GUEST_MODE`).
- Create: `vitest.config.ts`, `lib/repo/repo.test.ts`.
- Modify: rotas em `app/api/**` que usam `supabaseAdmin.from(...)` (enumeradas na Task 8).
- Modify: `lib/assetCache.ts`, `lib/folderStore.ts`, `lib/galleryUtils.ts`, `lib/chatSessionStore.ts`, `lib/useSpaceSync.ts` (se acessam supabase server-side).
- Modify: `package.json` (deps + scripts `db:generate`, `db:migrate`, `test`).
- Modify: `.env.example` (adicionar `DATABASE_URL`).

---

## Task 1: Dependências, Vitest e config Drizzle

**Files:**
- Modify: `package.json`
- Create: `vitest.config.ts`
- Create: `drizzle.config.ts`
- Modify: `.env.example`

**Interfaces:**
- Produces: scripts `pnpm test`, `pnpm db:generate`, `pnpm db:migrate`; env `DATABASE_URL`.

- [ ] **Step 1: Instalar dependências**

```bash
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit vitest
```

- [ ] **Step 2: Adicionar scripts ao `package.json`**

No bloco `"scripts"`, adicionar:

```json
"test": "vitest run",
"test:watch": "vitest",
"db:generate": "drizzle-kit generate",
"db:migrate": "drizzle-kit migrate"
```

- [ ] **Step 3: Criar `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: { environment: "node", include: ["**/*.test.ts"] },
});
```

Instalar o plugin de paths:

```bash
pnpm add -D vite-tsconfig-paths
```

- [ ] **Step 4: Criar `drizzle.config.ts`**

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./lib/db/schema.ts",
  out: "./lib/db/migrations",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

- [ ] **Step 5: Adicionar `DATABASE_URL` ao `.env.example`**

Adicionar sob nova seção:

```
# ── PostgreSQL ────────────────────────────────────────────────────────────────
# Ex: postgres://heliosgen:senha@localhost:5432/heliosgen
DATABASE_URL=
```

- [ ] **Step 6: Verificar que o projeto compila**

Run: `pnpm exec tsc --noEmit`
Expected: sem novos erros de tipo relacionados às deps.

- [ ] **Step 7: Commit**

```bash
git add package.json pnpm-lock.yaml vitest.config.ts drizzle.config.ts .env.example
git commit -m "chore: adicionar drizzle, postgres.js e vitest"
```

---

## Task 2: Schema Drizzle

**Files:**
- Create: `lib/db/schema.ts`
- Create: `lib/db/client.ts`

**Interfaces:**
- Produces: tabelas `users, generations, spaces, folders, userUploads, userSettings, assetCache`; `export const db` (instância Drizzle).

- [ ] **Step 1: Criar `lib/db/schema.ts`**

Reproduzir o schema de `supabase-setup.sql` em Drizzle, trocando `references auth.users` por FK à nova tabela `users`. Copiar exatamente estes campos (fonte: `supabase-setup.sql`):

```ts
import { pgTable, uuid, text, timestamp, boolean, integer, jsonb, bigint } from "drizzle-orm/pg-core";
import { sql } from "drizzle-orm";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  email: text("email").notNull().unique(),
  passwordHash: text("password_hash").notNull(),
  isAdmin: boolean("is_admin").notNull().default(false),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const userUploads = pgTable("user_uploads", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  userId: uuid("user_id").references(() => users.id, { onDelete: "set null" }),
  r2Url: text("r2_url").notNull(),
  mimeType: text("mime_type"),
  source: text("source").notNull().default("user_upload"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const spaces = pgTable("spaces", {
  id: text("id").primaryKey(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  data: jsonb("data").notNull().default(sql`'{}'::jsonb`),
  isPublic: boolean("is_public").notNull().default(false),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const generations = pgTable("generations", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  userId: uuid("user_id").references(() => users.id, { onDelete: "set null" }),
  taskId: text("task_id").notNull().unique(),
  generationType: text("generation_type").notNull(),
  status: text("status").notNull().default("pending"),
  prompt: text("prompt"),
  model: text("model"),
  aspectRatio: text("aspect_ratio"),
  quality: text("quality"),
  azureResolution: text("azure_resolution"),
  duration: integer("duration"),
  klingMode: text("kling_mode"),
  sound: boolean("sound"),
  referenceImageUrls: text("reference_image_urls").array().default(sql`'{}'`),
  imageUrl: text("image_url"),
  imageUrls: jsonb("image_urls"),
  videoUrl: text("video_url"),
  errorMsg: text("error_msg"),
});

// folders: derivar de lib/folderStore.ts + supabase-folders.sql
export const folders = pgTable("folders", {
  id: uuid("id").primaryKey().default(sql`gen_random_uuid()`),
  userId: uuid("user_id").references(() => users.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  parentId: uuid("parent_id"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const userSettings = pgTable("user_settings", {
  userId: uuid("user_id").primaryKey().references(() => users.id, { onDelete: "cascade" }),
  kieApiToken: text("kie_api_token"),
  azureApiKey: text("azure_api_key"),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const assetCache = pgTable("asset_cache", {
  hash: text("hash").primaryKey(),
  cdnUrl: text("cdn_url").notNull(),
  mimeType: text("mime_type"),
  byteSize: bigint("byte_size", { mode: "number" }),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
});
```

> **Nota:** Antes de finalizar `folders`, ler `supabase-folders.sql` e `lib/folderStore.ts` e ajustar colunas para bater exatamente com o uso real (nome de coluna de parent, ordem, etc.).

- [ ] **Step 2: Criar `lib/db/client.ts`**

```ts
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import * as schema from "./schema";

const globalForDb = globalThis as unknown as { _pg?: ReturnType<typeof postgres> };
const client = globalForDb._pg ?? postgres(process.env.DATABASE_URL!, { max: 10 });
if (process.env.NODE_ENV !== "production") globalForDb._pg = client;

export const db = drizzle(client, { schema });
```

- [ ] **Step 3: Gerar a migration**

Run: `DATABASE_URL=postgres://x pnpm db:generate`
Expected: cria `lib/db/migrations/0000_*.sql`.

- [ ] **Step 4: Conferir o SQL gerado** contém as 7 tabelas e a função de updated_at (adicionar trigger manual no SQL se o drizzle não gerar — copiar `touch_updated_at` de `supabase-setup.sql`).

- [ ] **Step 5: Commit**

```bash
git add lib/db drizzle.config.ts
git commit -m "feat(db): schema drizzle e client postgres"
```

---

## Task 3: Interface `Repo` e DTOs

**Files:**
- Create: `lib/repo/types.ts`

**Interfaces:**
- Produces: interface `Repo` com submódulos e os métodos exatos consumidos pelas rotas. Estes nomes são o contrato para as Tasks 4–8.

- [ ] **Step 1: Criar `lib/repo/types.ts`**

Definir a interface derivada do uso real. Métodos mínimos (derivados das rotas em `app/api/**` e de `lib/guest/db.ts`):

```ts
export interface GenerationInsert {
  taskId: string; userId: string | null; generationType: "image" | "video";
  status: string; prompt?: string; model?: string; aspectRatio?: string;
  quality?: string; azureResolution?: string; duration?: number;
  klingMode?: string; sound?: boolean; referenceImageUrls?: string[];
  imageUrl?: string; imageUrls?: string[]; videoUrl?: string;
}
export interface GenerationUpdate {
  status?: string; imageUrl?: string; imageUrls?: string[];
  videoUrl?: string; errorMsg?: string;
}
export interface GenerationRow extends GenerationInsert { id: string; createdAt: string; updatedAt: string; }

export interface Repo {
  generations: {
    insert(g: GenerationInsert): Promise<void>;
    updateByTaskId(taskId: string, patch: GenerationUpdate): Promise<void>;
    getByTaskId(taskId: string): Promise<GenerationRow | null>;
    listByUser(userId: string, opts?: { limit?: number; type?: "image" | "video" }): Promise<GenerationRow[]>;
  };
  spaces: {
    upsert(s: { id: string; userId: string; name: string; data: unknown; isPublic?: boolean }): Promise<void>;
    getById(id: string): Promise<{ id: string; userId: string; name: string; data: unknown; isPublic: boolean } | null>;
    listByUser(userId: string): Promise<Array<{ id: string; name: string; updatedAt: string }>>;
    setPublic(id: string, userId: string, isPublic: boolean): Promise<void>;
  };
  folders: {
    list(userId: string): Promise<Array<{ id: string; name: string; parentId: string | null }>>;
    create(userId: string, name: string, parentId: string | null): Promise<{ id: string }>;
    rename(id: string, userId: string, name: string): Promise<void>;
    remove(id: string, userId: string): Promise<void>;
  };
  uploads: {
    insert(u: { userId: string | null; r2Url: string; mimeType?: string; source?: string }): Promise<void>;
    listByUser(userId: string): Promise<Array<{ id: string; r2Url: string; mimeType: string | null }>>;
  };
  userSettings: {
    get(userId: string): Promise<{ kieApiToken: string | null; azureApiKey: string | null } | null>;
    setKieToken(userId: string, token: string): Promise<void>;
    setAzureKey(userId: string, key: string): Promise<void>;
  };
  assetCache: {
    get(hash: string): Promise<{ cdnUrl: string; mimeType: string | null } | null>;
    put(a: { hash: string; cdnUrl: string; mimeType?: string; byteSize?: number }): Promise<void>;
  };
}
```

> **Nota:** Antes de commitar, fazer um grep por `supabaseAdmin.from(` e `guestDb.` em `app/` e `lib/` e confirmar que todos os acessos reais têm método correspondente. Ajustar a interface para cobrir 100% dos usos.

- [ ] **Step 2: Verificar compilação**

Run: `pnpm exec tsc --noEmit`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add lib/repo/types.ts
git commit -m "feat(repo): interface Repo e DTOs"
```

---

## Task 4: Implementação `pg` da Repo (TDD)

**Files:**
- Create: `lib/repo/pg.ts`
- Create: `lib/repo/repo.test.ts`

**Interfaces:**
- Consumes: `db` de `lib/db/client.ts`, tabelas de `lib/db/schema.ts`, tipos de `lib/repo/types.ts`.
- Produces: `export const pgRepo: Repo`.

> **Setup de teste:** Os testes rodam contra um Postgres real de teste. Definir `DATABASE_URL` de teste (ex.: container efêmero ou banco `heliosgen_test`). Se não houver Postgres disponível no ambiente de execução do agente, marcar o teste com `describe.skipIf(!process.env.DATABASE_URL)` e validar manualmente conforme Step 4.

- [ ] **Step 1: Escrever o teste que falha** — `lib/repo/repo.test.ts`

```ts
import { describe, it, expect, beforeAll } from "vitest";
import { pgRepo } from "./pg";

const hasDb = !!process.env.DATABASE_URL;

describe.skipIf(!hasDb)("pgRepo.generations", () => {
  it("insere e recupera por taskId", async () => {
    const taskId = `test-${Date.now()}`;
    await pgRepo.generations.insert({
      taskId, userId: null, generationType: "image",
      status: "pending", prompt: "hello", model: "nano-banana-2",
    });
    const row = await pgRepo.generations.getByTaskId(taskId);
    expect(row?.status).toBe("pending");
    expect(row?.prompt).toBe("hello");
  });

  it("atualiza status por taskId", async () => {
    const taskId = `test-upd-${Date.now()}`;
    await pgRepo.generations.insert({ taskId, userId: null, generationType: "image", status: "pending" });
    await pgRepo.generations.updateByTaskId(taskId, { status: "done", imageUrl: "https://cdn/x.png" });
    const row = await pgRepo.generations.getByTaskId(taskId);
    expect(row?.status).toBe("done");
    expect(row?.imageUrl).toBe("https://cdn/x.png");
  });
});
```

- [ ] **Step 2: Rodar o teste e ver falhar**

Run: `pnpm test`
Expected: FAIL (`pgRepo` não existe / método ausente), ou SKIP se sem `DATABASE_URL`.

- [ ] **Step 3: Implementar `lib/repo/pg.ts`**

Implementar cada método com Drizzle. Exemplo do submódulo `generations` (replicar padrão para os demais, mapeando camelCase↔snake do schema):

```ts
import { and, desc, eq } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { generations, spaces, folders, userUploads, userSettings, assetCache } from "@/lib/db/schema";
import type { Repo, GenerationInsert, GenerationUpdate, GenerationRow } from "./types";

export const pgRepo: Repo = {
  generations: {
    async insert(g: GenerationInsert) {
      await db.insert(generations).values({
        taskId: g.taskId, userId: g.userId, generationType: g.generationType,
        status: g.status, prompt: g.prompt, model: g.model, aspectRatio: g.aspectRatio,
        quality: g.quality, azureResolution: g.azureResolution, duration: g.duration,
        klingMode: g.klingMode, sound: g.sound, referenceImageUrls: g.referenceImageUrls ?? [],
        imageUrl: g.imageUrl, imageUrls: g.imageUrls, videoUrl: g.videoUrl,
      });
    },
    async updateByTaskId(taskId, patch: GenerationUpdate) {
      await db.update(generations)
        .set({ ...patch, updatedAt: new Date() })
        .where(eq(generations.taskId, taskId));
    },
    async getByTaskId(taskId) {
      const [r] = await db.select().from(generations).where(eq(generations.taskId, taskId)).limit(1);
      return (r as GenerationRow) ?? null;
    },
    async listByUser(userId, opts) {
      const conds = [eq(generations.userId, userId)];
      if (opts?.type) conds.push(eq(generations.generationType, opts.type));
      return db.select().from(generations).where(and(...conds))
        .orderBy(desc(generations.createdAt)).limit(opts?.limit ?? 100) as Promise<GenerationRow[]>;
    },
  },
  spaces: { /* upsert/getById/listByUser/setPublic — Drizzle equivalente */ },
  folders: { /* list/create/rename/remove */ },
  uploads: { /* insert/listByUser */ },
  userSettings: {
    async get(userId) {
      const [r] = await db.select().from(userSettings).where(eq(userSettings.userId, userId)).limit(1);
      return r ? { kieApiToken: r.kieApiToken, azureApiKey: r.azureApiKey } : null;
    },
    async setKieToken(userId, token) {
      await db.insert(userSettings).values({ userId, kieApiToken: token })
        .onConflictDoUpdate({ target: userSettings.userId, set: { kieApiToken: token, updatedAt: new Date() } });
    },
    async setAzureKey(userId, key) {
      await db.insert(userSettings).values({ userId, azureApiKey: key })
        .onConflictDoUpdate({ target: userSettings.userId, set: { azureApiKey: key, updatedAt: new Date() } });
    },
  },
  assetCache: {
    async get(hash) {
      const [r] = await db.select().from(assetCache).where(eq(assetCache.hash, hash)).limit(1);
      return r ? { cdnUrl: r.cdnUrl, mimeType: r.mimeType } : null;
    },
    async put(a) {
      await db.insert(assetCache).values(a).onConflictDoNothing();
    },
  },
};
```

Preencher `spaces`, `folders`, `uploads` seguindo o mesmo padrão (não deixar stubs vazios — implementar todos os métodos da interface).

- [ ] **Step 4: Aplicar migration no banco de teste e rodar**

```bash
DATABASE_URL=postgres://.../heliosgen_test pnpm db:migrate
DATABASE_URL=postgres://.../heliosgen_test pnpm test
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/repo/pg.ts lib/repo/repo.test.ts
git commit -m "feat(repo): implementacao postgres com testes"
```

---

## Task 5: Implementação `guest` da Repo

**Files:**
- Create: `lib/repo/guest.ts`
- Modify: `lib/guest/db.ts` (apenas se algum método faltar)

**Interfaces:**
- Consumes: funções existentes de `lib/guest/db.ts`.
- Produces: `export const guestRepo: Repo`.

- [ ] **Step 1: Criar `lib/repo/guest.ts`** — adaptar cada método da `Repo` a chamadas de `lib/guest/db.ts`.

```ts
import type { Repo } from "./types";
import * as guestDb from "@/lib/guest/db";

export const guestRepo: Repo = {
  generations: {
    async insert(g) { guestDb.insertGeneration(g as never); },
    async updateByTaskId(taskId, patch) {
      guestDb.updateGeneration(taskId, {
        status: patch.status, image_url: patch.imageUrl, image_urls: patch.imageUrls,
        video_url: patch.videoUrl, error_msg: patch.errorMsg,
      } as never);
    },
    async getByTaskId(taskId) { return (guestDb.getGeneration?.(taskId) as never) ?? null; },
    async listByUser() { return (guestDb.listGenerations?.() as never) ?? []; },
  },
  // ...demais submódulos mapeando 1:1 para guestDb.*
};
```

> **Nota:** Onde `lib/guest/db.ts` não tiver a função equivalente (ex.: `getGeneration`, `listGenerations`), adicioná-la seguindo o padrão de I/O em JSON já existente no arquivo. Não inventar comportamento — espelhar o que a versão Supabase faz.

- [ ] **Step 2: Compilar**

Run: `pnpm exec tsc --noEmit`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add lib/repo/guest.ts lib/guest/db.ts
git commit -m "feat(repo): implementacao guest sobre json local"
```

---

## Task 6: `getRepo()` seletor

**Files:**
- Create: `lib/repo/index.ts`

**Interfaces:**
- Produces: `export function getRepo(): Repo`.

- [ ] **Step 1: Criar `lib/repo/index.ts`**

```ts
import { GUEST_MODE } from "@/lib/guestMode";
import type { Repo } from "./types";
import { pgRepo } from "./pg";
import { guestRepo } from "./guest";

export function getRepo(): Repo {
  return GUEST_MODE ? guestRepo : pgRepo;
}
export type { Repo } from "./types";
```

- [ ] **Step 2: Compilar** — Run: `pnpm exec tsc --noEmit` — Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add lib/repo/index.ts
git commit -m "feat(repo): seletor getRepo por GUEST_MODE"
```

---

## Task 7: Migrar `user_settings` e helpers de token para a Repo

**Files:**
- Modify: `lib/getKieToken.ts`, `lib/getAzureKey.ts`
- Modify: `app/api/settings/kie-key/route.ts`, `app/api/settings/azure-key/route.ts`
- Modify: `lib/assetCache.ts`

**Interfaces:**
- Consumes: `getRepo()`.

- [ ] **Step 1: Reescrever `getKieTokenForUser`** em `lib/getKieToken.ts` para usar `getRepo().userSettings.get(userId)` no lugar de `supabaseAdmin.from("user_settings")`. Manter o branch `GUEST_MODE` que já usa `guest/db` — ou substituí-lo por `getRepo()` (que já resolve guest). Preferir `getRepo()` para eliminar o branch.

```ts
import { getRepo } from "@/lib/repo";
export async function getKieTokenForUser(userId: string): Promise<string | null> {
  const s = await getRepo().userSettings.get(userId);
  return s?.kieApiToken ?? null;
}
```

> A função `getKieToken(req)` que hoje faz `supabaseAdmin.auth.getUser(token)` permanece por enquanto (auth ainda é Supabase; tratada no Bloco B). Trocar apenas o acesso à tabela.

- [ ] **Step 2: Repetir para `lib/getAzureKey.ts`** (usar `s?.azureApiKey`).

- [ ] **Step 3: Migrar as rotas de settings** para `getRepo().userSettings.setKieToken(...)` / `setAzureKey(...)` e `.get(...)` no GET.

- [ ] **Step 4: Migrar `lib/assetCache.ts`** para `getRepo().assetCache.get/put`.

- [ ] **Step 5: Rodar em GUEST_MODE**

Run: `GUEST_MODE=true pnpm dev` e exercitar Settings (salvar/ler chave) na UI.
Expected: chave salva e lida sem erro.

- [ ] **Step 6: Commit**

```bash
git add lib/getKieToken.ts lib/getAzureKey.ts app/api/settings lib/assetCache.ts
git commit -m "refactor: settings/tokens/assetCache via repo"
```

---

## Task 8: Migrar rotas de geração e leitura

**Files (Modify):**
- `app/api/generate/route.ts`
- `app/api/generate-video/route.ts`
- `app/api/callback/route.ts`
- `app/api/job-status/route.ts`
- `app/api/job-stream/route.ts`
- `app/api/gallery/route.ts`
- `app/api/folders/route.ts`, `app/api/folders/[id]/route.ts`, `app/api/folder-items/route.ts`
- `app/api/spaces/publish/route.ts`, `app/api/public/space/[id]/route.ts`
- `app/api/upload-asset/route.ts`, `app/api/upload-video/route.ts`, `app/api/upload-to-r2/route.ts`
- `app/api/fetch-url/route.ts`, `app/api/credit/route.ts`

**Interfaces:**
- Consumes: `getRepo()`.

- [ ] **Step 1: Para cada arquivo, aplicar a transformação mecânica:**
  - Remover `import { supabaseAdmin } from "@/lib/supabase/admin"` e o branch `if (GUEST_MODE) guestDb.x() else supabaseAdmin...`.
  - Substituir por uma única chamada `getRepo().<entidade>.<metodo>(...)`.
  - Padrão de exemplo (em `generate/route.ts`, o insert do kie.ai branch):

```ts
// ANTES
if (GUEST_MODE) {
  guestDb.insertGeneration({ task_id: taskId, user_id: currentUserId, generation_type: "image", status: "pending", prompt, model, aspect_ratio: aspectRatio, quality, reference_image_urls: r2ImageUrls });
} else {
  supabaseAdmin.from("generations").insert({ task_id: taskId, /* ... */ }).then(({ error }) => { if (error) console.error(error.message); });
}
// DEPOIS
await getRepo().generations.insert({
  taskId, userId: currentUserId, generationType: "image", status: "pending",
  prompt, model, aspectRatio, quality, referenceImageUrls: r2ImageUrls,
});
```

  - No `callback/route.ts`, trocar os `supabaseAdmin.from("generations").update(...).eq("task_id", taskId)` por `await getRepo().generations.updateByTaskId(taskId, {...})`.

- [ ] **Step 2: Após migrar cada arquivo, buscar resíduos**

Run: `pnpm exec grep -rn "supabaseAdmin.from" app/`
(ou usar o grep do editor)
Expected: nenhum resultado em rotas de dados (auth ainda pode referenciar em `resolveUserId`, tratado no Bloco B).

- [ ] **Step 3: Compilar**

Run: `pnpm exec tsc --noEmit`
Expected: PASS.

- [ ] **Step 4: Teste manual guest end-to-end**

Run: `GUEST_MODE=true pnpm dev`; gerar uma imagem (com `KIE_API_KEY` no env guest) e confirmar que aparece na galeria.
Expected: geração persistida e listada.

- [ ] **Step 5: Commit**

```bash
git add app/api
git commit -m "refactor: rotas de dados via repo (remove branch supabase/guest)"
```

---

## Task 9: Limpeza de acessos residuais em `lib/` e componentes server-side

**Files:**
- Modify: `lib/folderStore.ts`, `lib/galleryUtils.ts`, `lib/chatSessionStore.ts`, `lib/useSpaceSync.ts` (apenas os que acessam supabase **server-side**; acessos client-side de auth ficam para o Bloco B).

- [ ] **Step 1: Para cada arquivo, identificar** se o acesso é a dados (migrar para `getRepo()`) ou a auth/cliente-browser (deixar para Bloco B). Migrar os de dados.

- [ ] **Step 2: Compilar e rodar guest** — Run: `pnpm exec tsc --noEmit && GUEST_MODE=true pnpm build` — Expected: build OK.

- [ ] **Step 3: Commit**

```bash
git add lib
git commit -m "refactor: acessos de dados restantes via repo"
```

---

## Self-Review (executar ao final)

- [ ] **Cobertura do spec (Bloco A):** schema (Task 2), repo pg (Task 4), repo guest (Task 5), seletor (Task 6), migração de todas as rotas de dados (Tasks 7–9). ✅
- [ ] **Grep final:** `supabaseAdmin.from(` só pode aparecer, se ainda, em código de auth (removido no Bloco B). Nenhum acesso de dados restante.
- [ ] **Consistência de tipos:** nomes de método usados nas rotas (`generations.insert`, `updateByTaskId`, `userSettings.setKieToken`, `assetCache.put`) batem com `lib/repo/types.ts`.
- [ ] **GUEST_MODE:** `pnpm build` e fluxo de geração guest funcionam.

## Notas de handoff

- Ao final do Bloco A, o app roda com PostgreSQL para **dados**, mas ainda depende de **Supabase Auth** (`supabaseAdmin.auth.getUser`, `lib/supabase/client.ts` na UI). Isso é removido no **Bloco B**.
- `lib/supabase/admin.ts` só pode ser removido após o Bloco B.
