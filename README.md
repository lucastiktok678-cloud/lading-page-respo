# Diário de Trade — Projeto Completo (Front-end + Back-end + Banco de Dados)

## 1. Arquitetura

```
trade-journal/
├── backend/                   ← API Node.js/Express + PostgreSQL
│   ├── src/
│   │   ├── config/            db.js, mercadopago.js
│   │   ├── controllers/       auth, trade, premarket, settings, stats,
│   │   │                      plan, webhook, admin, upload, migration
│   │   ├── middleware/        auth.js (JWT), subscription.js, errorHandler.js
│   │   ├── routes/            um ficheiro de rotas por recurso
│   │   ├── services/          storageService.js (upload de imagens)
│   │   ├── jobs/               expireSubscriptions.js (cron diário)
│   │   ├── db/                 schema.sql, migrate.js
│   │   └── server.js           ponto de entrada
│   ├── public/
│   │   ├── admin/               painel administrativo estático
│   │   └── uploads/             imagens enviadas pelos utilizadores
│   ├── package.json
│   ├── .env.example
│   └── API.md                   documentação de todos os endpoints
└── frontend/
    ├── index.html                a aplicação (visual 100% preservado)
    ├── reset-password.html       página do link de recuperação de senha
    └── pagamento.html            retorno do checkout (próxima fase)
```

**Por que Node/Express/PostgreSQL:** é o que você pediu e é uma escolha sólida
para este caso — API REST simples, dados relacionais (utilizador → trades →
planos), e ecossistema maduro. Não há motivo técnico para trocar.

---

## 2. Pré-requisitos

- Node.js 18 ou superior
- PostgreSQL 13 ou superior (local, Docker, ou um serviço gerido como Railway/Render/Supabase)

---

## 3. Configurar o banco de dados

**Opção A — PostgreSQL local já instalado:**
```bash
createdb diario_trade
```

**Opção B — Docker (mais rápido se não tiver Postgres instalado):**
```bash
docker run --name diario-trade-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=diario_trade -p 5432:5432 -d postgres:16
```

**Opção C — Serviço gerido (Railway, Render, Supabase, Neon):**
Crie o banco pelo painel deles e copie a "Connection String" — vai ser o valor de `DATABASE_URL`.

---

## 4. Configurar e correr o back-end

```bash
cd backend
cp .env.example .env
```

Abra o `.env` e preencha:

| Variável | Onde obter / o que colocar |
|---|---|
| `DATABASE_URL` | `postgres://postgres:postgres@localhost:5432/diario_trade` (ajuste utilizador/senha/porta conforme o seu Postgres) |
| `JWT_SECRET` | Qualquer string longa e aleatória — ex: gere com `openssl rand -hex 32` no terminal |
| `PORT` | `3000` (ou outra porta livre) |
| `APP_URL` | `http://localhost:3000` (endereço público do próprio back-end) |
| `FRONTEND_URL` | `http://localhost:5173` ou a porta onde for servir o `frontend/` |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | credenciais do 1º utilizador administrador, criado automaticamente na migração |
| `SMTP_*` | opcional nesta fase — sem isto, o link de recuperação de senha aparece no terminal em vez de ser enviado por e-mail |
| `MP_ACCESS_TOKEN` | deixe como está — só será usado na próxima fase (Mercado Pago) |

Instale as dependências, crie as tabelas e arranque o servidor:
```bash
npm install
npm run migrate    # cria as tabelas no PostgreSQL + o utilizador admin
npm run dev         # inicia com reinício automático (nodemon)
# ou: npm start
```

Se tudo correr bem, verá no terminal:
```
Diário de Trade API a correr em http://localhost:3000
```

Teste rapidamente:
```bash
curl http://localhost:3000/api/health
# { "ok": true, "time": "..." }
```

---

## 5. Servir o front-end

O `frontend/index.html` chama a API em `http://localhost:3000/api` sempre que
estiver a correr em `localhost`. Basta servir a pasta com qualquer servidor
estático — por exemplo:

```bash
cd frontend
npx serve .          # ou: python3 -m http.server 5173
```

Abra o endereço que aparecer (ex: `http://localhost:5173`) no navegador.

> Se o front-end for publicado num domínio diferente de `localhost` mais
> tarde, edite a constante `API_BASE` no topo do `<script>` de `index.html`
> (ou defina `window.DIARIO_API_BASE_URL` antes do script carregar).

---

## 6. Testar o fluxo completo (os 11 passos pedidos)

1. Abra `http://localhost:5173` → verá o ecrã de login (visual preservado).
2. Clique em **"Criar conta"**, preencha nome/e-mail/senha/confirmação → **Criar Conta**.
3. É autenticado automaticamente e cai direto no Painel de Controlo.
4. Vá a **Diário de Negociação → Nova Transação**, preencha e **Guardar**.
5. Esse pedido já foi para `POST /api/trades` e gravado no PostgreSQL (confirme com o passo 8).
6. Atualize a página (F5).
7. A sessão continua ativa (token JWT em `localStorage`) e a transação continua lá — porque agora vem de `GET /api/trades`, não do `localStorage` de dados.
8. Confirme direto no banco, se quiser:
   ```bash
   psql "$DATABASE_URL" -c "SELECT date, instrument, net_pnl FROM trades;"
   ```
9. No painel **Sistema**, clique em **Terminar Sessão**.
10. Faça login novamente com o mesmo e-mail/senha.
11. As suas transações continuam lá.

**Teste de isolamento entre utilizadores:** crie uma segunda conta com outro
e-mail, registe uma transação nela, e confirme que ela **não aparece** para o
primeiro utilizador (nem vice-versa) — toda consulta ao banco é filtrada por
`WHERE user_id = $1`, usando o utilizador do token JWT, nunca um valor vindo
do cliente.

---

## 7. Migração de dados antigos (quem já usava a versão só-no-navegador)

Se o navegador ainda tiver a chave `trading_journal_db_v1` no `localStorage`
(da versão anterior, 100% local), a aplicação deteta isso automaticamente
depois do login/registo e mostra um cartão perguntando se quer importar. Ao
confirmar, os dados (incluindo imagens em base64) são enviados para
`POST /api/migration/import`, gravados no PostgreSQL, e só depois disso a
cópia local é apagada.

---

## 8. Armazenamento de imagens

Screenshots de trades e capturas do plano pré-mercado **não** são guardados
como base64 no banco. O front-end envia o ficheiro para `POST /api/uploads`
(multipart), o back-end grava em `backend/public/uploads/` e devolve uma URL
pública (`http://.../uploads/arquivo.png`), que é o único valor guardado nas
colunas `screenshot`, `daily_chart`, `intraday_chart`.

Isto funciona imediatamente em desenvolvimento e num único servidor. Para
produção com múltiplas instâncias/deploys sem disco persistente, troque a
implementação em `backend/src/services/storageService.js` por S3, Cloudflare
R2 ou Cloudinary — as instruções detalhadas de cada opção estão comentadas
no topo desse ficheiro.

---

## 9. Segurança já implementada

- Senhas com `bcrypt` (nunca em texto puro)
- JWT assinado com `JWT_SECRET` (nunca no front-end)
- Todas as queries usam parâmetros (`$1, $2...`) — sem concatenação de string → protegido contra SQL Injection
- Toda rota de dados filtra por `user_id` do token, nunca por um valor enviado pelo cliente
- `express-rate-limit` nas rotas de login/registo/recuperação de senha
- CORS restrito a `FRONTEND_URL`
- Segredos apenas em `.env` (nunca commitado — ver `.gitignore`)
- Tratamento de erros centralizado (`middleware/errorHandler.js`) sem vazar detalhes internos ao cliente

---

## 10. Erros comuns e como resolver

| Sintoma | Causa provável | Solução |
|---|---|---|
| `ECONNREFUSED` ao arrancar | Postgres não está a correr, ou `DATABASE_URL` errado | Confirme que o Postgres está ativo e a string de ligação bate certo |
| `relation "users" does not exist` | Esqueceu de correr a migração | `npm run migrate` |
| Front-end mostra "Não foi possível ligar ao servidor" | Back-end não está a correr, ou porta/URL diferente | Confirme `npm run dev` a correr e `API_BASE` no front-end apontando para a porta certa |
| CORS bloqueado no browser | `FRONTEND_URL` no `.env` não bate com o endereço real do front-end | Ajuste `FRONTEND_URL` e reinicie o back-end |
| Upload de imagem falha com 401/402 | Token expirado, ou conta desativada | Faça login novamente |

---

## 11. O que fica para a próxima etapa (não implementado agora, por pedido explícito)

- Planos Mensal/Anual, checkout Mercado Pago, webhook de confirmação automática, bloqueio por assinatura vencida — o código já existe (`plans`, `webhooks`, `subscription.js`), mas está **inativo**: toda conta nasce com `status='active'` e `plan='gratuito'` sem data de expiração, então nada bloqueia ninguém por enquanto.
