# NEXO Vendas v9

Versão preparada para publicação usando GitHub + Render pelo celular.

## O que existe

- Loja responsiva
- Produtos com imagens incorporadas
- Carrinho
- Pagamento somente via PIX
- Chave PIX configurada: `02326770692`
- Pedido salvo em PostgreSQL
- Painel administrativo
- Login administrativo por sessão HTTP-only
- Status: Pendente → PIX confirmado → Entregue
- Exclusão de pedidos
- Health check em `/health`

## Publicação pelo celular

### 1. GitHub

Crie um repositório privado e envie **todos os arquivos desta pasta**, mantendo:
- `package.json`
- `server.js`
- `render.yaml`
- `.gitignore`
- `.env.example`
- `README.md`
- `public/index.html`

O GitHub permite carregar arquivos pelo navegador; cada arquivo enviado pelo navegador deve ter no máximo 25 MiB.

### 2. Render

No Render:
1. New → Blueprint
2. Conecte o repositório do GitHub.
3. O `render.yaml` cria o Web Service e o PostgreSQL.
4. Quando o Render pedir os valores `ADMIN_USER` e `ADMIN_PASSWORD`, defina os seus.
5. `JWT_SECRET` é gerado automaticamente.
6. `PIX_KEY` já está configurada.

Depois do deploy, o Render fornecerá uma URL `https://...onrender.com`.

### 3. Primeiro acesso

Abra a URL pública e use:
- usuário: o `ADMIN_USER` que você definiu
- senha: o `ADMIN_PASSWORD` que você definiu

Não use a senha de demonstração das versões antigas.

## Importante

A v9 usa PostgreSQL em vez de SQLite porque o Render recomenda um datastore gerenciado para dados relacionais. O armazenamento local dos serviços Render é efêmero por padrão; por isso, o banco gerenciado evita perder os pedidos em reinícios/deploys.

Nunca coloque senhas, JWT_SECRET ou DATABASE_URL dentro do GitHub.
