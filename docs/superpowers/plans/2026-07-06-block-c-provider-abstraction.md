# Bloco C — Abstração de Provider (kie.ai ⇄ openrouter.ai) (Implementation Plan)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Permitir que o usuário escolha entre **kie.ai** e **openrouter.ai** como provider de LLM, imagem e vídeo, extraindo os branches inline das rotas para uma camada `lib/providers/` com implementações intercambiáveis.

**Architecture:** Uma interface `GenProvider` (`chat`, `submitImage`, `submitVideo`, `normalizeCallback`). Implementações `kie`, `openrouter`, `azure`. O provider ativo é resolvido por usuário (coluna `provider` + `openrouter_api_token` em `user_settings`), com override por request. As rotas `generate`, `generate-video`, `assistant`, `callback` delegam ao provider. Imagem no OpenRouter é síncrona (base64→R2→done); vídeo é assíncrono via `callback_url` reaproveitando `/api/callback`.

**Tech Stack:** Next.js 16, TypeScript. Sem novas deps de runtime (usa `fetch` + `lib/r2.ts` + `lib/jobStore.ts` + `getRepo()`).

**Pré-requisito:** Blocos A e B concluídos (Repository + auth por sessão).

## Global Constraints

- Node 22, pnpm. Ler `node_modules/next/dist/docs/` antes de mexer em rotas.
- Não quebrar o fluxo kie.ai atual: default do provider = `kie`.
- Imagem OpenRouter é **síncrona** (`POST /api/v1/images`, resposta base64) → seguir o padrão do branch Azure existente (upload R2 + `jobStore.set(taskId,{status:"done",imageUrl})`).
- Vídeo OpenRouter é **assíncrono** (`POST /api/v1/videos` + `callback_url`) → normalizado em `/api/callback`.
- Chat: OpenAI-compat streaming em ambos.
- Toda credencial de provider vem de `user_settings` via `getRepo()`; nunca expor ao browser.

---

## File Structure

- Create: `lib/providers/types.ts` — interface `GenProvider` e tipos de params/resultado.
- Create: `lib/providers/kie.ts` — implementação extraída do código atual.
- Create: `lib/providers/openrouter.ts` — nova implementação.
- Create: `lib/providers/azure.ts` — extrai branch Azure de `generate/route.ts` e `assistant/route.ts`.
- Create: `lib/providers/index.ts` — `getProviderForUser(userId)`.
- Create: `lib/providers/openrouter.test.ts` — testa `normalizeCallback` e montagem de payload.
- Modify: `lib/repo/types.ts` + `lib/repo/pg.ts` + `lib/repo/guest.ts` — `userSettings` ganha `provider` e `openrouterApiToken`.
- Modify: `lib/db/schema.ts` + nova migration — colunas `provider`, `openrouter_api_token`.
- Modify: `app/api/generate/route.ts`, `app/api/generate-video/route.ts`, `app/api/assistant/route.ts`, `app/api/callback/route.ts`.
- Modify: `lib/modelConfig.ts`, `lib/models.ts` — campo `provider` + modelos OpenRouter.
- Modify: `components/SettingsModal.tsx` — seletor de provider + campo OpenRouter key.
- Create: `app/api/settings/openrouter-key/route.ts`, `app/api/settings/provider/route.ts`.

---

## Task 1: Schema — provider e chave OpenRouter em user_settings

**Files:** Modify `lib/db/schema.ts`, `lib/repo/types.ts`, `lib/repo/pg.ts`, `lib/repo/guest.ts`; new migration.

**Interfaces:** Produces em `userSettings`: `provider: "kie"|"openrouter"`, `openrouterApiToken: string|null`, e métodos `setProvider(userId, p)`, `setOpenrouterToken(userId, t)`.

- [ ] **Step 1: `lib/db/schema.ts`** — adicionar colunas em `userSettings`:

```ts
provider: text("provider").notNull().default("kie"),
openrouterApiToken: text("openrouter_api_token"),
```

- [ ] **Step 2: Gerar migration** — Run: `DATABASE_URL=... pnpm db:generate` — Expected: nova migration com `ALTER TABLE user_settings ADD COLUMN ...`.

- [ ] **Step 3: `lib/repo/types.ts`** — estender `userSettings`:

```ts
userSettings: {
  get(userId: string): Promise<{ kieApiToken: string | null; azureApiKey: string | null; provider: string; openrouterApiToken: string | null } | null>;
  setKieToken(userId: string, token: string): Promise<void>;
  setAzureKey(userId: string, key: string): Promise<void>;
  setOpenrouterToken(userId: string, token: string): Promise<void>;
  setProvider(userId: string, provider: string): Promise<void>;
};
```

- [ ] **Step 4: Implementar** em `pg.ts` (upsert com `onConflictDoUpdate`) e `guest.ts` (persistir no JSON). Espelhar o padrão de `setKieToken`.

- [ ] **Step 5: Compilar + migrate teste** — Run: `pnpm exec tsc --noEmit && DATABASE_URL=.../heliosgen_test pnpm db:migrate` — Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add lib/db lib/repo
git commit -m "feat(providers): colunas provider e openrouter em user_settings"
```

---

## Task 2: Interface `GenProvider`

**Files:** Create `lib/providers/types.ts`.

**Interfaces:** Produces os tipos abaixo (contrato para Tasks 3–5).

- [ ] **Step 1: Criar `lib/providers/types.ts`**

```ts
export interface ChatMessage { role: "user" | "assistant" | "system"; content: string; }
export interface ChatParams {
  model: string; messages: ChatMessage[];
  azure?: { endpoint: string; deployment: string; modelName: string };
}
export interface ImageParams {
  model: string; prompt: string; imageUrls: string[];
  aspectRatio: string; quality: string; userId: string | null;
  azure?: { baseUrl: string; deployment: string; resolution?: string; qualityHint?: string };
}
export interface VideoParams {
  model: string; prompt: string; imageUrls: string[];
  aspectRatio: string; duration?: number; klingMode?: string; sound?: boolean;
  userId: string | null;
}
export interface SubmitResult { taskId: string; referenceImageUrls?: string[]; }

export interface CallbackResult {
  taskId: string;
  state: "success" | "fail" | "pending";
  urls: string[];
  error?: string;
  type?: "image" | "video";
}

export interface GenProvider {
  id: "kie" | "openrouter" | "azure";
  chat(params: ChatParams): Promise<Response>;              // streaming OpenAI-compat
  submitImage(params: ImageParams): Promise<SubmitResult>;  // sync (openrouter/azure) OU async (kie)
  submitVideo(params: VideoParams): Promise<SubmitResult>;
  normalizeCallback(body: unknown): CallbackResult | null;  // async providers; null se não reconhece
}
```

- [ ] **Step 2: Compilar** — Run: `pnpm exec tsc --noEmit` — Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add lib/providers/types.ts
git commit -m "feat(providers): interface GenProvider"
```

---

## Task 3: Provider `kie` (extração)

**Files:** Create `lib/providers/kie.ts`. Reaproveita helpers de `app/api/generate/route.ts` (mover `httpsPost`/`fetchBuffer`/`resolveImages` para `lib/providers/_http.ts` se compartilhados).

**Interfaces:** Consumes `types.ts`, `getKieTokenForUser`, `jobStore`, `getRepo`, `lib/r2`. Produces `export const kieProvider: GenProvider`.

- [ ] **Step 1: Mover helpers compartilhados** — extrair `httpsPost`, `fetchBuffer`, `resolveImages` de `generate/route.ts` para `lib/providers/_http.ts` e exportá-los. Atualizar imports em `generate/route.ts` (temporariamente ainda com branch inline; migrado na Task 6).

- [ ] **Step 2: Implementar `kie.ts`** — copiar a lógica do branch kie.ai atual:
  - `submitImage` → monta `input` de `IMAGE_MODELS`, `POST https://api.kie.ai/api/v1/jobs/createTask` com `callBackUrl`, cria `jobStore.set(taskId,{status:"pending"})`, insere em `getRepo().generations`, retorna `{ taskId }`.
  - `submitVideo` → equivalente ao `generate-video/route.ts` atual (mesmo endpoint de task + callback; `jobStore.set(taskId,{status:"pending",type:"video"})`).
  - `chat` → `POST` no endpoint OpenAI-compat de `OPENAI_COMPAT_ENDPOINTS` (mover o mapa de `assistant/route.ts` para cá), streaming.
  - `normalizeCallback(body)` → extrair a lógica de `extractUrls` + parsing de `state`/`taskId` de `callback/route.ts`, retornando `CallbackResult` (ou `null` se não parece payload kie).

```ts
export const kieProvider: GenProvider = {
  id: "kie",
  async chat(p) { /* fetch OpenAI-compat kie, retorna Response streaming */ },
  async submitImage(p) { /* createTask + jobStore + repo insert */ },
  async submitVideo(p) { /* createTask video */ },
  normalizeCallback(body) {
    const data = (body as any).data ?? body;
    const taskId = data.taskId ?? data.id ?? (body as any).taskId ?? (body as any).id;
    if (!taskId) return null;
    const state = String(data.state ?? data.status ?? "").toLowerCase();
    // ...reaproveitar extractUrls; mapear "success"/"fail"/intermediário
  },
};
```

- [ ] **Step 3: Compilar** — Run: `pnpm exec tsc --noEmit` — Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add lib/providers/kie.ts lib/providers/_http.ts app/api/generate/route.ts
git commit -m "feat(providers): extrai provider kie"
```

---

## Task 4: Provider `azure` (extração)

**Files:** Create `lib/providers/azure.ts`.

**Interfaces:** Consumes `types.ts`, `getAzureKeyForUser`, `curlMultipartPost`/`httpsPost`, `jobStore`, `getRepo`, `lib/r2`. Produces `export const azureProvider: GenProvider`.

- [ ] **Step 1: Mover `curlMultipartPost`** de `generate/route.ts` para `lib/providers/_http.ts`.

- [ ] **Step 2: Implementar `azure.ts`** — copiar o branch Azure inteiro de `generate/route.ts` (`submitImage`: text-to-image JSON `/images/generations` e image-to-image multipart `/images/edits`; upload base64→R2; `jobStore.set(...,"done")`; insere em `getRepo().generations`). `chat` → o branch `azure-auto` de `assistant/route.ts`. `submitVideo` → não suportado: lançar erro claro `"Azure não suporta geração de vídeo"`. `normalizeCallback` → `() => null` (Azure é síncrono).

- [ ] **Step 3: Compilar** — Run: `pnpm exec tsc --noEmit` — Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add lib/providers/azure.ts lib/providers/_http.ts
git commit -m "feat(providers): extrai provider azure"
```

---

## Task 5: Provider `openrouter` (TDD no callback e payload)

**Files:** Create `lib/providers/openrouter.ts`, `lib/providers/openrouter.test.ts`.

**Interfaces:** Consumes `types.ts`, `getRepo().userSettings` (openrouterApiToken), `jobStore`, `lib/r2`. Produces `export const openrouterProvider: GenProvider`.

Referência da API (doc atual):
- Chat: `POST https://openrouter.ai/api/v1/chat/completions` (OpenAI-compat, `stream: true`).
- Imagem: `POST https://openrouter.ai/api/v1/images` — **síncrono**, resposta com base64.
- Vídeo: `POST https://openrouter.ai/api/v1/videos` — async, aceita `callback_url`; status `pending|in_progress|completed|failed`; resultado em `unsigned_urls` quando `completed`.

- [ ] **Step 1: Teste que falha** — `lib/providers/openrouter.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { openrouterProvider } from "./openrouter";

describe("openrouter.normalizeCallback", () => {
  it("mapeia completed → success com urls", () => {
    const r = openrouterProvider.normalizeCallback({
      id: "vid_123", status: "completed", unsigned_urls: ["https://or/v.mp4"],
    });
    expect(r).toEqual({ taskId: "vid_123", state: "success", urls: ["https://or/v.mp4"], type: "video" });
  });
  it("mapeia failed → fail com erro", () => {
    const r = openrouterProvider.normalizeCallback({ id: "vid_9", status: "failed", error: "boom" });
    expect(r?.state).toBe("fail");
    expect(r?.error).toBe("boom");
  });
  it("retorna null para payload desconhecido", () => {
    expect(openrouterProvider.normalizeCallback({ foo: 1 })).toBeNull();
  });
});
```

- [ ] **Step 2: Rodar e ver falhar** — Run: `pnpm test openrouter` — Expected: FAIL.

- [ ] **Step 3: Implementar `openrouter.ts`**

```ts
import type { GenProvider, CallbackResult } from "./types";
import { jobStore } from "@/lib/jobStore";
import { uploadBuffer } from "@/lib/r2";
import { getRepo } from "@/lib/repo";

const BASE = "https://openrouter.ai/api/v1";

async function key(userId: string | null): Promise<string> {
  const s = userId ? await getRepo().userSettings.get(userId) : null;
  if (!s?.openrouterApiToken) throw new Error("Sem OpenRouter API key. Configure em Settings.");
  return s.openrouterApiToken;
}

export const openrouterProvider: GenProvider = {
  id: "openrouter",

  async chat(p) {
    const auth = await key(null); // userId vem do params na integração real — ver Task 6
    return fetch(`${BASE}/chat/completions`, {
      method: "POST",
      headers: { Authorization: `Bearer ${auth}`, "Content-Type": "application/json" },
      body: JSON.stringify({ model: p.model, messages: p.messages, stream: true }),
    });
  },

  async submitImage(p) {
    const auth = await key(p.userId);
    const taskId = `or-img-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
    jobStore.set(taskId, { status: "pending", type: "image", userId: p.userId ?? undefined });
    await getRepo().generations.insert({
      taskId, userId: p.userId, generationType: "image", status: "pending",
      prompt: p.prompt, model: p.model, aspectRatio: p.aspectRatio, quality: p.quality,
      referenceImageUrls: p.imageUrls,
    });
    (async () => {
      try {
        const res = await fetch(`${BASE}/images`, {
          method: "POST",
          headers: { Authorization: `Bearer ${auth}`, "Content-Type": "application/json" },
          body: JSON.stringify({
            model: p.model, prompt: p.prompt, aspect_ratio: p.aspectRatio,
            ...(p.imageUrls.length ? { input_references: p.imageUrls } : {}),
          }),
        });
        const json = await res.json();
        if (!res.ok) throw new Error(json?.error?.message ?? `OpenRouter ${res.status}`);
        const b64 = json?.data?.[0]?.b64_json ?? json?.images?.[0]?.b64_json;
        if (!b64) throw new Error("OpenRouter não retornou imagem");
        const url = await uploadBuffer(Buffer.from(b64, "base64"), "image/png", "generated");
        jobStore.set(taskId, { status: "done", imageUrl: url });
        await getRepo().generations.updateByTaskId(taskId, { status: "done", imageUrl: url, imageUrls: [url] });
      } catch (e) {
        const msg = e instanceof Error ? e.message : String(e);
        jobStore.set(taskId, { status: "error", error: msg });
        await getRepo().generations.updateByTaskId(taskId, { status: "error", errorMsg: msg });
      }
    })();
    return { taskId };
  },

  async submitVideo(p) {
    const auth = await key(p.userId);
    const callbackBase = process.env.CALLBACK_BASE_URL;
    if (!callbackBase) throw new Error("CALLBACK_BASE_URL não configurado");
    const res = await fetch(`${BASE}/videos`, {
      method: "POST",
      headers: { Authorization: `Bearer ${auth}`, "Content-Type": "application/json" },
      body: JSON.stringify({
        model: p.model, prompt: p.prompt, aspect_ratio: p.aspectRatio,
        ...(p.duration ? { duration: p.duration } : {}),
        ...(p.imageUrls.length ? { frame_images: p.imageUrls } : {}),
        callback_url: `${callbackBase.replace(/\/$/, "")}/api/callback`,
      }),
    });
    const json = await res.json();
    if (!res.ok) throw new Error(json?.error?.message ?? `OpenRouter ${res.status}`);
    const taskId = json?.id ?? json?.job?.id;
    if (!taskId) throw new Error("OpenRouter não retornou id do job");
    jobStore.set(taskId, { status: "pending", type: "video", userId: p.userId ?? undefined });
    await getRepo().generations.insert({
      taskId, userId: p.userId, generationType: "video", status: "pending",
      prompt: p.prompt, model: p.model, aspectRatio: p.aspectRatio,
      duration: p.duration, klingMode: p.klingMode, sound: p.sound, referenceImageUrls: p.imageUrls,
    });
    return { taskId };
  },

  normalizeCallback(body: unknown): CallbackResult | null {
    const b = body as Record<string, any>;
    const taskId = b?.id ?? b?.job?.id;
    const status = b?.status ?? b?.job?.status;
    if (!taskId || !status) return null;
    if (status === "completed") {
      const urls = b.unsigned_urls ?? b.urls ?? (b.output ? [b.output] : []);
      return { taskId, state: "success", urls, type: "video" };
    }
    if (status === "failed") return { taskId, state: "fail", urls: [], error: b.error ?? "Falhou" };
    return { taskId, state: "pending", urls: [] };
  },
};
```

> **Nota de integração:** `chat` precisa do `userId` para buscar a chave. Ajustar `ChatParams` para incluir `userId` (atualizar `types.ts` e o provider `kie`/`azure` juntos) OU resolver a chave fora e passar via params. Escolher uma abordagem e aplicar consistentemente nas Tasks 2–6.

- [ ] **Step 4: Rodar e ver passar** — Run: `pnpm test openrouter` — Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/providers/openrouter.ts lib/providers/openrouter.test.ts
git commit -m "feat(providers): implementa openrouter (img sync, video async) com teste"
```

---

## Task 6: Seletor + integração nas rotas

**Files:** Create `lib/providers/index.ts`; Modify `app/api/generate/route.ts`, `generate-video/route.ts`, `assistant/route.ts`, `callback/route.ts`.

**Interfaces:** Produces `getProviderForUser(userId): Promise<GenProvider>`.

- [ ] **Step 1: `lib/providers/index.ts`**

```ts
import type { GenProvider } from "./types";
import { kieProvider } from "./kie";
import { openrouterProvider } from "./openrouter";
import { azureProvider } from "./azure";
import { getRepo } from "@/lib/repo";

const REGISTRY: Record<string, GenProvider> = {
  kie: kieProvider, openrouter: openrouterProvider, azure: azureProvider,
};

export async function getProviderForUser(userId: string | null, override?: string): Promise<GenProvider> {
  if (override && REGISTRY[override]) return REGISTRY[override];
  const s = userId ? await getRepo().userSettings.get(userId) : null;
  return REGISTRY[s?.provider ?? "kie"] ?? kieProvider;
}
export const ALL_PROVIDERS = REGISTRY;
```

- [ ] **Step 2: `generate/route.ts`** — remover os dois branches inline (Azure + kie) e delegar:

```ts
const userId = await resolveUserId();
// Azure só quando params azureBaseUrl/azureDeployment presentes (mantém compat):
const provider = azureBaseUrl && azureDeployment
  ? ALL_PROVIDERS.azure
  : await getProviderForUser(userId);
const { taskId, referenceImageUrls } = await provider.submitImage({
  model, prompt: prompt!, imageUrls: r2ImageUrls, aspectRatio, quality, userId,
  azure: azureBaseUrl ? { baseUrl: azureBaseUrl, deployment: azureDeployment!, resolution: azureResolution, qualityHint: azureQuality } : undefined,
});
return NextResponse.json({ taskId, referenceImageUrls });
```

- [ ] **Step 3: `generate-video/route.ts`** — delegar a `provider.submitVideo(...)`.

- [ ] **Step 4: `assistant/route.ts`** — resolver provider por modelo/usuário e chamar `provider.chat(...)`, retornando o `Response` de streaming. Manter o branch `azure-auto` via `ALL_PROVIDERS.azure`.

- [ ] **Step 5: `callback/route.ts`** — tornar provider-agnóstico:

```ts
const body = await req.json();
// tenta cada provider até um reconhecer o payload
let cb = null;
for (const p of Object.values(ALL_PROVIDERS)) { cb = p.normalizeCallback(body); if (cb) break; }
if (!cb) return NextResponse.json({ received: true });
// aplica cb.state → jobStore/settle + getRepo().generations.updateByTaskId (reaproveitar mirrorToR2 + isVideo via jobStore)
```
Preservar a lógica existente de `mirrorToR2` e detecção `isVideo` a partir do `jobStore`.

- [ ] **Step 6: Compilar + build** — Run: `pnpm exec tsc --noEmit && pnpm build` — Expected: PASS.

- [ ] **Step 7: Teste manual guest** — `GUEST_MODE=true` com provider default kie: gerar imagem/vídeo → ainda funciona (sem regressão).

- [ ] **Step 8: Commit**

```bash
git add lib/providers/index.ts app/api
git commit -m "refactor(providers): rotas delegam ao provider selecionado"
```

---

## Task 7: Catálogo de modelos por provider

**Files:** Modify `lib/modelConfig.ts`, `lib/models.ts`.

- [ ] **Step 1: Adicionar campo `provider`** a `ImageModel`/`VideoModel` (`lib/modelConfig.ts`) e ao `Model` LLM (`lib/models.ts`). Default `"kie"` para os existentes (não quebrar).

- [ ] **Step 2: Adicionar entradas OpenRouter** — modelos reais (validar slugs na doc): ex. imagem `seedream-4.5`, vídeo `google/veo-3.1`, chat (ex. `anthropic/claude-...`). Preencher `apiInput` conforme a forma da API OpenRouter (aspect_ratio, input_references). Não inventar slugs — marcar para validação na execução com `// TODO(validar-slug)` **substituído por slug real antes do commit**.

- [ ] **Step 3: Compilar** — Run: `pnpm exec tsc --noEmit` — Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add lib/modelConfig.ts lib/models.ts
git commit -m "feat(providers): catalogo de modelos por provider"
```

---

## Task 8: Settings UI (provider + chave OpenRouter)

**Files:** Create `app/api/settings/openrouter-key/route.ts`, `app/api/settings/provider/route.ts`; Modify `components/SettingsModal.tsx`.

- [ ] **Step 1: Rotas de settings** — espelhar `settings/kie-key/route.ts` para `openrouter-key` (`GET hasToken`, `POST` salva via `getRepo().userSettings.setOpenrouterToken`) e `provider` (`GET`/`POST` via `setProvider`, validando `"kie"|"openrouter"`). Usar `getUserId()` (Bloco B) para auth.

- [ ] **Step 2: `SettingsModal.tsx`** — adicionar:
  - Um `<select>`/toggle de provider (kie/openrouter) que faz `POST /api/settings/provider`.
  - Um campo de chave OpenRouter (padrão idêntico ao campo kie/azure existente) → `POST /api/settings/openrouter-key`.
  - Estado inicial via `GET` das rotas.

- [ ] **Step 3: Dropdown de modelos** — filtrar/agrupar por `provider` selecionado (usar o campo adicionado na Task 7). Se o app mantém o provider por-modelo, garantir que a seleção de um modelo OpenRouter roteie corretamente (override no request ou provider do usuário).

- [ ] **Step 4: Rodar** — `pnpm dev`: salvar chave OpenRouter, trocar provider, gerar imagem com um modelo OpenRouter (se chave real disponível) — validar fim-a-fim; senão validar que a request sai com o payload/endpoint corretos via logs.

- [ ] **Step 5: Commit**

```bash
git add app/api/settings components/SettingsModal.tsx
git commit -m "feat(providers): UI de provider e chave OpenRouter"
```

---

## Self-Review

- [ ] **Cobertura:** schema (T1), interface (T2), kie/azure/openrouter (T3–T5), seletor + rotas (T6), catálogo (T7), UI (T8). ✅
- [ ] **Placeholder scan:** slugs de modelo OpenRouter devem ser reais antes do commit (T7) — sem `TODO` remanescente.
- [ ] **Consistência de tipos:** `GenProvider` (`chat/submitImage/submitVideo/normalizeCallback`), `CallbackResult`, `getProviderForUser`, `userSettings.get(...).provider/openrouterApiToken` usados igualmente entre tasks. Decisão sobre `userId` em `ChatParams` aplicada de forma consistente (T5 nota).
- [ ] **Sem regressão kie:** default `kie`, fluxo guest de imagem/vídeo intacto.

## Handoff

Após Bloco C, o usuário escolhe provider em Settings. Próximo: **Bloco D** (deploy Portainer).
