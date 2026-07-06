# Bloco D — Deploy Portainer (Traefik + PostgreSQL) (Implementation Plan)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Empacotar o HeliosGen (Next.js SSR) em imagem Docker e publicá-lo no Portainer atrás do Traefik, com um serviço PostgreSQL no mesmo stack, seguindo o padrão de `speedsales_landing/deploy`.

**Architecture:** `next.config.ts` com `output: "standalone"` gera um servidor Node mínimo. Um `Dockerfile` multi-stage (Node 22 Alpine, pnpm) roda `node server.js` na porta 3000. `stack.yml` define dois serviços — `heliosgen` (imagem GHCR, labels Traefik na rede `traefik_public`) e `postgres:16-alpine` (rede interna, volume nomeado). Migrations Drizzle aplicadas no boot.

**Tech Stack:** Docker, Docker Swarm/Compose (Portainer), Traefik, PostgreSQL 16, Node 22, pnpm.

**Pré-requisito:** Blocos A–C concluídos (app roda 100% em Postgres, sem Supabase).

## Global Constraints

- pnpm com `pnpm-lock.yaml` (usar `--frozen-lockfile`).
- Rede externa `traefik_public` já existe no servidor (igual à referência).
- Certresolver Traefik chamado `le`, entrypoint `websecure`, TLS on (igual referência).
- App escuta na porta **3000** (não 80 — não é nginx estático).
- `CALLBACK_BASE_URL` = domínio público do Traefik (webhook de vídeo precisa ser alcançável pela internet).
- R2 permanece obrigatório para mídia; suas envs entram no stack.
- Segredos (SESSION_SECRET, ADMIN_*, POSTGRES_PASSWORD, DATABASE_URL, R2_*) via env do Portainer — nunca commitados.

---

## File Structure

- Modify: `next.config.ts` — `output: "standalone"`.
- Create: `deploy/Dockerfile` — build multi-stage Node.
- Create: `deploy/.dockerignore`.
- Create: `deploy/stack.yml` — serviços heliosgen + postgres.
- Create: `deploy/entrypoint.sh` — aplica migrations e sobe o server.
- Create: `deploy/README.md` — passos no Portainer.
- Modify: `.env.example` — consolidar todas as envs de produção.

---

## Task 1: Next standalone output

**Files:** Modify `next.config.ts`.

- [ ] **Step 1: Ler o guia** `node_modules/next/dist/docs/` sobre `output: "standalone"` nesta versão do Next (16.x) — confirmar nome/posição da opção.

- [ ] **Step 2: Editar `next.config.ts`** — adicionar dentro do objeto de config:

```ts
output: "standalone",
```

- [ ] **Step 3: Build local** — Run: `pnpm build` — Expected: gera `.next/standalone/server.js` e `.next/static`.

- [ ] **Step 4: Rodar o standalone localmente** — Run: `node .next/standalone/server.js` (com envs mínimas) — Expected: server sobe na porta 3000.

- [ ] **Step 5: Commit**

```bash
git add next.config.ts
git commit -m "chore(deploy): next output standalone"
```

---

## Task 2: Dockerfile

**Files:** Create `deploy/Dockerfile`, `deploy/.dockerignore`.

- [ ] **Step 1: `deploy/.dockerignore`**

```
node_modules
.next
.git
data
public/generated
docs
*.md
.env*
```

- [ ] **Step 2: `deploy/Dockerfile`**

```dockerfile
# ── deps ──────────────────────────────────────────────────────────────────────
FROM node:22-alpine AS deps
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# ── build ─────────────────────────────────────────────────────────────────────
FROM node:22-alpine AS build
WORKDIR /app
RUN corepack enable
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

# ── runner ────────────────────────────────────────────────────────────────────
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000
# sharp precisa de libc; alpine + node:22 já inclui. curl usado por generate (azure) — instalar:
RUN apk add --no-cache curl
# artefatos standalone
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
# migrations + drizzle p/ aplicar no boot
COPY --from=build /app/lib/db/migrations ./lib/db/migrations
COPY --from=build /app/drizzle.config.ts ./drizzle.config.ts
COPY --from=build /app/node_modules/drizzle-kit ./node_modules/drizzle-kit
COPY deploy/entrypoint.sh ./entrypoint.sh
RUN chmod +x ./entrypoint.sh
EXPOSE 3000
CMD ["./entrypoint.sh"]
```

> **Nota:** `sharp` é dependência nativa. Se o build falhar no alpine, trocar a base para `node:22-slim` (Debian) nas três stages. Validar no Step 3.

- [ ] **Step 3: Build da imagem** — Run: `docker build -f deploy/Dockerfile -t heliosgen:local .` — Expected: build OK.

- [ ] **Step 4: Commit**

```bash
git add deploy/Dockerfile deploy/.dockerignore
git commit -m "feat(deploy): Dockerfile node standalone"
```

---

## Task 3: Entrypoint com migrations

**Files:** Create `deploy/entrypoint.sh`.

- [ ] **Step 1: `deploy/entrypoint.sh`**

```sh
#!/bin/sh
set -e
echo "[entrypoint] aplicando migrations..."
# drizzle-kit migrate usa DATABASE_URL do ambiente
node_modules/.bin/drizzle-kit migrate || {
  echo "[entrypoint] migrate falhou (banco indisponível?)"; exit 1;
}
echo "[entrypoint] iniciando server na porta ${PORT:-3000}..."
exec node server.js
```

> **Alternativa se `drizzle-kit` não rodar em produção:** aplicar as migrations via `deploy/db-init/*.sql` montado no Postgres (init-only, roda apenas em banco vazio). Nesse caso, remover drizzle-kit do runner e copiar `lib/db/migrations/*.sql` para `deploy/db-init/`. Escolher UMA estratégia e documentar no README.

- [ ] **Step 2: Testar entrypoint** com um Postgres local:

```bash
docker run --rm -e DATABASE_URL=postgres://... -e SESSION_SECRET=... -p 3000:3000 heliosgen:local
```
Expected: migrations aplicadas, server sobe, `GET /` responde.

- [ ] **Step 3: Commit**

```bash
git add deploy/entrypoint.sh
git commit -m "feat(deploy): entrypoint aplica migrations e sobe server"
```

---

## Task 4: stack.yml (Traefik + Postgres)

**Files:** Create `deploy/stack.yml`.

- [ ] **Step 1: `deploy/stack.yml`**

```yaml
version: "3.8"

services:
  heliosgen:
    image: ghcr.io/devhost-softwares/heliosgen:latest
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      NODE_ENV: production
      PORT: "3000"
      DATABASE_URL: "postgres://heliosgen:${POSTGRES_PASSWORD}@postgres:5432/heliosgen"
      SESSION_SECRET: ${SESSION_SECRET}
      ADMIN_EMAIL: ${ADMIN_EMAIL}
      ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      CALLBACK_BASE_URL: "https://helios.seu-dominio.com"
      R2_ACCOUNT_ID: ${R2_ACCOUNT_ID}
      R2_ACCESS_KEY_ID: ${R2_ACCESS_KEY_ID}
      R2_SECRET_ACCESS_KEY: ${R2_SECRET_ACCESS_KEY}
      R2_BUCKET_NAME: ${R2_BUCKET_NAME}
      R2_PUBLIC_URL: ${R2_PUBLIC_URL}
    networks:
      - traefik_public
      - internal
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.heliosgen.rule=Host(`helios.seu-dominio.com`)"
      - "traefik.http.routers.heliosgen.entrypoints=websecure"
      - "traefik.http.routers.heliosgen.tls=true"
      - "traefik.http.routers.heliosgen.tls.certresolver=le"
      - "traefik.http.services.heliosgen.loadbalancer.server.port=3000"
      - "traefik.docker.network=traefik_public"

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: heliosgen
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: heliosgen
    volumes:
      - heliosgen_pgdata:/var/lib/postgresql/data
    networks:
      - internal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U heliosgen"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  traefik_public:
    external: true
  internal:
    driver: overlay

volumes:
  heliosgen_pgdata:
```

> **Notas:** (1) `driver: overlay` para Swarm; se o Portainer usar Compose standalone, trocar por `driver: bridge`. (2) `Host(...)` e `CALLBACK_BASE_URL` devem usar o domínio real. (3) `depends_on` não espera healthcheck em Swarm — o entrypoint já falha e reinicia até o Postgres subir (retry via `restart: unless-stopped`).

- [ ] **Step 2: Validar sintaxe** — Run: `docker compose -f deploy/stack.yml config` — Expected: sem erros de parsing.

- [ ] **Step 3: Commit**

```bash
git add deploy/stack.yml
git commit -m "feat(deploy): stack Traefik + Postgres"
```

---

## Task 5: .env.example consolidado e README de deploy

**Files:** Modify `.env.example`; Create `deploy/README.md`.

- [ ] **Step 1: Consolidar `.env.example`** — garantir que contém todas as chaves de produção com comentários: `DATABASE_URL`, `SESSION_SECRET`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `POSTGRES_PASSWORD`, `CALLBACK_BASE_URL`, `R2_*`, `GUEST_MODE` (opcional), `KIE_API_KEY` (guest). OpenRouter/kie/azure keys por usuário via UI (documentar isso).

- [ ] **Step 2: `deploy/README.md`** — passos:

```md
# Deploy no Portainer

## Pré-requisitos
- Rede externa `traefik_public` já criada e Traefik ativo (entrypoint `websecure`, certresolver `le`).
- Bucket Cloudflare R2 configurado (mídia).
- DNS do domínio apontando para o servidor.

## Passos
1. Ajustar `deploy/stack.yml`: trocar `helios.seu-dominio.com` pelo domínio real (label Host e CALLBACK_BASE_URL).
2. Build e push da imagem:
   docker build -f deploy/Dockerfile -t ghcr.io/devhost-softwares/heliosgen:latest .
   docker push ghcr.io/devhost-softwares/heliosgen:latest
3. No Portainer → Stacks → Add stack → colar `deploy/stack.yml`.
4. Em "Environment variables", definir: POSTGRES_PASSWORD, SESSION_SECRET (>=32 chars),
   ADMIN_EMAIL, ADMIN_PASSWORD, R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY,
   R2_BUCKET_NAME, R2_PUBLIC_URL.
5. Deploy. No primeiro boot o admin é criado (ADMIN_EMAIL/ADMIN_PASSWORD) e as migrations aplicadas.
6. Acessar https://<dominio>, logar, e em Settings salvar a chave do provider (kie.ai ou OpenRouter).

## Webhook de vídeo
CALLBACK_BASE_URL deve ser o domínio público (Traefik) — providers assíncronos (kie/openrouter)
fazem POST em https://<dominio>/api/callback.
```

- [ ] **Step 3: Commit**

```bash
git add .env.example deploy/README.md
git commit -m "docs(deploy): env consolidado e README do Portainer"
```

---

## Task 6: Verificação end-to-end (staging)

- [ ] **Step 1: Subir o stack** num ambiente de teste (ou `docker compose -f deploy/stack.yml up` com `bridge` + Traefik local/porta).

- [ ] **Step 2: Validar boot** — logs mostram migrations aplicadas e "server na porta 3000". Postgres `healthy`.

- [ ] **Step 3: Login** com ADMIN_EMAIL/ADMIN_PASSWORD → sessão OK.

- [ ] **Step 4: Geração de imagem** (kie ou openrouter conforme chave em Settings) → aparece na galeria (persistido no Postgres).

- [ ] **Step 5: Geração de vídeo** → confirmar que o callback chega em `/api/callback` (checar logs) e o vídeo é armazenado no R2 e marcado `done`.

- [ ] **Step 6: Registrar resultado** — anotar no README qualquer ajuste necessário (base image slim, driver de rede, etc.).

---

## Self-Review

- [ ] **Cobertura:** standalone (T1), Dockerfile (T2), entrypoint/migrations (T3), stack Traefik+Postgres (T4), env+README (T5), verificação (T6). ✅
- [ ] **Placeholder scan:** domínio `helios.seu-dominio.com` é claramente marcado como "trocar pelo real" no README e nas notas — não é um placeholder oculto.
- [ ] **Consistência:** porta 3000 em next.config/Dockerfile/stack; `DATABASE_URL` idêntico entre stack e o que o app/entrypoint esperam; `CALLBACK_BASE_URL` = domínio Traefik.
- [ ] **Segredos** nunca commitados (só em env do Portainer).

## Handoff

Fim dos 4 blocos. App self-hosted no Portainer: Next.js SSR + PostgreSQL, auth single-user, provider kie/openrouter selecionável.
