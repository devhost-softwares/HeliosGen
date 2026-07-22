# HeliosGen — Provider selecionável, PostgreSQL e Deploy Portainer

**Data:** 2026-07-06
**Status:** Aprovado para planejamento
**Autor:** Alexandre + Claude

## Objetivo

Três mudanças na aplicação HeliosGen (Next.js 16, SSR):

1. **Provider selecionável** — permitir escolher entre **kie.ai** e **openrouter.ai** para geração (LLM/chat, imagem e vídeo).
2. **Banco PostgreSQL** — substituir Supabase (Postgres gerenciado + Auth + RLS) por **PostgreSQL próprio** com **Drizzle ORM** e **autenticação simples single-user**.
3. **Deploy Portainer** — arquivos de deploy (Dockerfile + stack.yml) no padrão Traefik da referência `speedsales_landing/deploy`, com Postgres no mesmo stack.

## Decisões tomadas

| Tema | Decisão |
|---|---|
| Autenticação | **Single-user (admin único)** definido por env/setup. Sem OAuth, sem signup aberto. |
| Driver de banco | **Drizzle ORM** (tipagem + migrations versionadas). |
| Escopo OpenRouter | Substitui kie.ai em **tudo**: LLM, imagem e vídeo. |
| Deploy | **Traefik + Postgres no mesmo stack** (auto-contido). |

## Contexto do código atual (achados)

- **Dupla implementação já existe:** quase toda rota faz `if (GUEST_MODE) guestDb.x() else supabaseAdmin.from(...)`. ~47 arquivos tocam `supabase`.
- **Provider é branch inline:** `app/api/generate/route.ts` tem branch "Azure" e branch "Kie.ai" no mesmo arquivo; `app/api/assistant/route.ts` mapeia endpoints kie.ai OpenAI-compat + Azure.
- **Fluxo de jobs:** kie.ai = task assíncrona (`POST /api/v1/jobs/createTask`) + webhook em `/api/callback`. Estado in-memory em `lib/jobStore.ts`, eventos SSE em `lib/jobEvents.ts` (`/api/job-stream`).
- **Mídia:** sempre no Cloudflare R2 (`lib/r2.ts`) — **permanece**, não faz parte desta mudança.
- **Auth:** `middleware.ts` + `supabaseAdmin.auth.getUser(token)` + RLS com `auth.uid()`. Schema em `supabase-setup.sql` referencia `auth.users`.

### API OpenRouter (confirmado na doc atual)

| Papel | kie.ai | OpenRouter |
|---|---|---|
| LLM/chat | `chat/completions` OpenAI-compat | `POST /api/v1/chat/completions` OpenAI-compat |
| Imagem | job assíncrono + callback | `POST /api/v1/images` **síncrono**, retorna base64 |
| Vídeo | job assíncrono + callback | `POST /api/v1/videos` assíncrono + `callback_url` + polling |

---

## Bloco A — Camada de dados (Repository) sobre PostgreSQL com Drizzle

### Arquitetura

Nova pasta `lib/db/`:
- `schema.ts` — schema Drizzle (tabelas: `users`, `generations`, `spaces`, `folders`, `user_uploads`, `user_settings`, `asset_cache`).
- `client.ts` — conexão Drizzle (`drizzle-orm/postgres-js` + `postgres`), lendo `DATABASE_URL`.
- `repo.ts` — interface `Repo` com submódulos: `generations`, `spaces`, `folders`, `uploads`, `userSettings`, `assetCache`, `users`.
- Implementações: `repoPg.ts` (Drizzle, padrão) e `repoGuest.ts` (adapta `lib/guest/db.ts`, mantido para `GUEST_MODE`).
- `index.ts` exporta `getRepo()` → escolhe Pg ou Guest conforme `GUEST_MODE`.

### Migração das rotas

Cada rota que hoje faz `if (GUEST_MODE) ... else supabaseAdmin.from(...)` passa a chamar `repo.<entidade>.<metodo>(...)`. O `if (GUEST_MODE)` sai das rotas e vive só dentro de `getRepo()`. Rotas afetadas incluem: `generate`, `generate-video`, `callback`, `job-status`, `job-stream`, `gallery`, `folders`, `folder-items`, `spaces/publish`, `public/space/[id]`, `settings/*`, `upload-*`, `credit`.

### Schema (mudanças vs Supabase)

- `auth.users` → tabela própria `users (id uuid pk, email text unique, password_hash text, is_admin boolean, created_at)`.
- Remover RLS/policies — **autorização passa a ser na aplicação**: toda query filtra por `user_id` do usuário da sessão.
- Trigger `touch_updated_at` → replicado no Postgres ou tratado em código (Drizzle `$onUpdate`).
- Manter tipos: `jsonb` para `data`/`image_urls`, `text[]` para `reference_image_urls`.
- Migrations geridas por `drizzle-kit` (pasta `lib/db/migrations/`), aplicadas no boot/deploy.

### Fora de escopo

- `lib/supabase/*` e `@supabase/*` são removidos ao final, quando nenhuma rota os referenciar.

---

## Bloco B — Autenticação simples single-user

### Modelo

- **Um usuário admin.** Credenciais iniciais definidas por env (`ADMIN_EMAIL`, `ADMIN_PASSWORD`) e/ou fluxo de setup no primeiro boot que cria o registro em `users` com `password_hash` (argon2/bcrypt).
- Sessão em **cookie httpOnly assinado** via `iron-session` (segredo `SESSION_SECRET`). Sem OAuth, sem magic link, sem reset por email (troca de senha via UI autenticada).

### Rotas/arquivos

- `app/api/auth/login/route.ts`, `logout/route.ts` (novas). `signup` **não** exposto (single-user); opcionalmente um `setup` idempotente só enquanto não existe admin.
- `lib/auth/session.ts` — helpers `getSession(req)`, `requireUser(req)` → substituem `supabaseAdmin.auth.getUser(token)`.
- `lib/guestMode.ts::resolveUserId` passa a ler a sessão (cookie) em vez do token Supabase.
- `middleware.ts` — valida cookie de sessão; redireciona não autenticados.
- UI: `AuthModal`, `AuthButton`, `SettingsModal`, `ResetPasswordModal`, `CreditBalance`, `QuickAssist` deixam de importar `lib/supabase/client` e passam a usar as rotas `/api/auth/*` e `fetch` autenticado por cookie.
- `lib/getKieToken.ts` / `lib/getAzureKey.ts` — trocam leitura Supabase por `repo.userSettings` + sessão.

### GUEST_MODE

Permanece: quando `GUEST_MODE=true`, auth é bypassada (single-user local em JSON), como já é hoje.

---

## Bloco C — Abstração de Provider (kie.ai ⇄ openrouter.ai)

### Interface

Nova pasta `lib/providers/`:

```ts
interface GenProvider {
  id: "kie" | "openrouter" | "azure";
  chat(params: ChatParams): Promise<Response>;            // OpenAI-compat, streaming
  submitImage(params: ImageParams): Promise<SubmitResult>; // { taskId } — sync ou async
  submitVideo(params: VideoParams): Promise<SubmitResult>;
  normalizeCallback?(body: unknown): CallbackResult | null; // p/ providers assíncronos
}
type SubmitResult = { taskId: string };
```

Implementações:
- `kie.ts` — extrai a lógica atual de `generate`/`generate-video`/`assistant` (task assíncrona + callback).
- `openrouter.ts` — novo:
  - `chat` → `POST https://openrouter.ai/api/v1/chat/completions`.
  - `submitImage` → `POST /api/v1/images` (síncrono): gera `taskId` local, decodifica base64, faz upload R2, marca `done` no `jobStore` e no repo — **mesmo padrão do branch Azure atual**.
  - `submitVideo` → `POST /api/v1/videos` com `callback_url = CALLBACK_BASE_URL/api/callback` → assíncrono; `normalizeCallback` mapeia o payload OpenRouter para `{ taskId, urls, state }`.
- `azure.ts` — extrai o branch Azure já existente.

### Seleção de provider

- Nova coluna em `user_settings`: `provider text default 'kie'`, `openrouter_api_token text`.
- Regra de resolução: provider explícito do request > provider do usuário > `kie` (default).
- `lib/getProvider.ts` → `getProviderForUser(userId)` retorna a implementação certa + credencial.
- `app/api/callback/route.ts` chama `provider.normalizeCallback(body)` para suportar formatos kie **e** openrouter no mesmo endpoint.

### Catálogo de modelos

- `lib/modelConfig.ts` (imagem/vídeo) e `lib/models.ts` (LLM) ganham campo `provider`.
- Adicionar entradas de modelos OpenRouter (ex.: `google/veo-3.1` vídeo, `seedream-4.5` imagem, modelos de chat) com o mapeamento `apiInput` equivalente.

### UI

- `SettingsModal` ganha: seletor de provider (kie/openrouter) e campo para **OpenRouter API key** (padrão igual ao campo kie/azure já existente).
- Dropdown de modelos passa a filtrar/agrupar por provider selecionado.

---

## Bloco D — Deploy Portainer (Traefik + Postgres no stack)

Nova pasta `deploy/` (padrão adaptado da referência `speedsales_landing/deploy`):

### `next.config.ts`
- Habilitar `output: "standalone"` para imagem enxuta.

### `deploy/Dockerfile`
- Multi-stage Node 22 Alpine:
  - stage `deps` → `pnpm install --frozen-lockfile`.
  - stage `build` → `pnpm build` (usa standalone).
  - stage `runner` → copia `.next/standalone`, `.next/static`, `public`; roda `node server.js`; `EXPOSE 3000`.

### `deploy/stack.yml`
- Serviço **`heliosgen`**: imagem `ghcr.io/devhost-softwares/heliosgen:...`, `restart: unless-stopped`, rede `traefik_public`, labels Traefik (Host, websecure, TLS certresolver `le`, `loadbalancer.server.port=3000`), env vars, `depends_on: postgres` com healthcheck, também numa rede interna `heliosgen_internal`.
- Serviço **`postgres`**: `postgres:16-alpine`, volume nomeado `heliosgen_pgdata`, env `POSTGRES_*`, healthcheck `pg_isready`, apenas na rede interna. Init SQL/migrations montados de `deploy/db-init/` (ou migração aplicada pelo app no boot via drizzle-kit/migrate).
- Redes: `traefik_public` (external), `heliosgen_internal` (interna).

### Variáveis de ambiente (documentar em `.env.example`)
`DATABASE_URL`, `SESSION_SECRET`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `CALLBACK_BASE_URL`, `KIE_API_KEY` (guest), R2_* , OpenRouter (por usuário via UI). `GUEST_MODE` opcional.

### Arquivos auxiliares
- `deploy/.dockerignore`.
- `deploy/README.md` com passos no Portainer (criar stack, setar envs, rede traefik_public).

### Atenção
- `CALLBACK_BASE_URL` deve ser o domínio público do Traefik (webhook de vídeo precisa ser alcançável pela internet).
- R2 permanece obrigatório para mídia.

---

## Ordem de implementação sugerida

1. **Bloco A** (Repository + Drizzle + schema) — base de tudo; mantém Supabase Auth temporariamente.
2. **Bloco B** (Auth single-user) — remove dependência de Supabase Auth; remove `lib/supabase/*`.
3. **Bloco C** (Provider abstraction + OpenRouter) — independente, mas mais limpo após A/B.
4. **Bloco D** (Deploy) — por último, quando o app roda 100% em Postgres.

Cada bloco é entregável e testável isoladamente.

## Riscos / pontos abertos

- **Migração de dados existentes** do Supabase para o novo Postgres: fora de escopo por padrão (assume base nova). Confirmar se há dados em produção a migrar.
- **Isolamento multiusuário** foi descartado (single-user) — se no futuro virar multiusuário, a autorização por `user_id` na aplicação já prepara o terreno.
- Modelos OpenRouter específicos e seus `apiInput` precisam ser validados contra a doc/testes reais na fase de implementação.
