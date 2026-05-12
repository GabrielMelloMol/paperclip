# AIOS — Setup Local

Fork de [paperclipai/paperclip](https://github.com/paperclipai/paperclip) adaptado para o ecossistema Gabriel Mol (GMOL).

## Pré-requisitos

- Docker Desktop 24+  
- `docker compose` v2+  
- Node.js 20+ e `openssl` (para gerar o secret)

## Setup em 3 passos

### 1. Configure as variáveis de ambiente

```bash
cd docker/
cp .env.aios.example .env
# Edite BETTER_AUTH_SECRET com um valor seguro:
openssl rand -hex 32
```

Conteúdo mínimo do `docker/.env`:

```env
BETTER_AUTH_SECRET=<saída do openssl acima>
PAPERCLIP_PUBLIC_URL=http://localhost:3101
AIOS_PORT=3101
```

> O AIOS usa a porta **3101** para não conflitar com uma instância Paperclip local na 3100.

### 2. Suba o stack

```bash
cd docker/
docker compose --env-file .env up -d
```

Isso cria:
- **db** — PostgreSQL 17 com volume persistente `docker_pgdata`
- **server** — API + UI Paperclip na porta 3101

Aguarde o container estar saudável:

```bash
curl http://localhost:3101/api/health
# → {"status":"ok","bootstrapStatus":"bootstrap_pending",...}
```

### 3. Bootstrap — criar o primeiro usuário

O primeiro deploy precisa de um convite de bootstrap. Execute o script:

```bash
./scripts/aios-bootstrap.sh
```

Ou manualmente:

```bash
# 1. Gerar token e hash
TOKEN="pcp_bootstrap_$(openssl rand -hex 24)"
TOKEN_HASH=$(echo -n "$TOKEN" | openssl dgst -sha256 | awk '{print $2}')
EXPIRES=$(date -u -v+72H '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d '+72 hours' '+%Y-%m-%dT%H:%M:%SZ')

# 2. Inserir no PostgreSQL
cd docker/
docker compose exec db psql -U paperclip -d paperclip -c "
INSERT INTO invites (invite_type, token_hash, allowed_join_types, expires_at, invited_by_user_id, created_at, updated_at)
VALUES ('bootstrap_ceo', '$TOKEN_HASH', 'human', '$EXPIRES', 'system', NOW(), NOW());
"

# 3. Criar conta na UI ou via curl
AIOS_BASE="http://localhost:3101"
curl -s -c /tmp/aios-cookies.txt -X POST "$AIOS_BASE/api/auth/sign-up/email" \
  -H "Content-Type: application/json" \
  -d '{"email":"seu@email.com","password":"sua-senha","name":"Seu Nome"}'

# 4. Aceitar o convite de bootstrap
curl -s -b /tmp/aios-cookies.txt -X POST "$AIOS_BASE/api/invites/$TOKEN/accept" \
  -H "Content-Type: application/json" \
  -H "Origin: http://localhost:3101" \
  -d '{"requestType":"human"}'

# 5. Criar a empresa
curl -s -b /tmp/aios-cookies.txt -X POST "$AIOS_BASE/api/companies" \
  -H "Content-Type: application/json" \
  -H "Origin: http://localhost:3101" \
  -d '{"name":"GMOL AIOS","slug":"gmol"}'
```

## Validação

```bash
# Saúde do servidor
curl http://localhost:3101/api/health
# → {"status":"ok","bootstrapStatus":"ready",...}

# UI
open http://localhost:3101
```

## Comandos úteis

```bash
# Ver logs em tempo real
cd docker/ && docker compose logs -f server

# Parar o stack
cd docker/ && docker compose down

# Parar e apagar volumes (reset completo)
cd docker/ && docker compose down -v

# Rebuild após mudanças no código
cd docker/ && docker compose build server && docker compose up -d server
```

## Próximos passos

| Issue | Descrição |
|-------|-----------|
| GMO-69 | Deploy do AIOS em VPS (configurar DNS + HTTPS) |
| GMO-71 | Agente Telegram pessoal — MVP |
| GMO-74 | Agente WhatsApp via Evolution API — MVP |
| GMO-76 | Dashboard React MVP — substituir UI do Paperclip |

## Estrutura do repositório

```
docker/
  docker-compose.yml       # Stack: PostgreSQL + servidor AIOS
  docker-compose.quickstart.yml  # Versão single-container
  .env                     # Não commitado — copie de .env.aios.example
  .env.aios.example        # Template de variáveis de ambiente
Dockerfile                 # Build do servidor Node.js
README.aios.md             # Este arquivo
```
