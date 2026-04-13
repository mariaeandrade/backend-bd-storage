# 🗂️ API Exemplo — Node.js + Express + Prisma + Supabase + Render

> Stack: **Node.js + Express + Prisma + PostgreSQL (Supabase) + Storage (Supabase) + Render**

---

## 📋 Pré-requisitos

- Conta no [GitHub](https://github.com)
- Conta no [Supabase](https://supabase.com)
- Conta no [Render](https://render.com)
- Node.js instalado localmente
- Postman para testes

---

## 📁 Estrutura do Projeto

```
api-exemplo/
├── prisma/
│   ├── migrations/
│   ├── schema.prisma
│   └── seed.js
├── src/
│   ├── controllers/
│   │   ├── exemploController.js
│   │   └── arquivoController.js
│   ├── lib/
│   │   ├── helpers/
│   │   │   └── arquivoHelper.js
│   │   ├── middlewares/
│   │   │   ├── apiKey.js
│   │   │   └── fileGate.js
│   │   └── services/
│   │       ├── prismaClient.js
│   │       └── supabase.js
│   ├── models/
│   │   └── ExemploModel.js
│   ├── routes/
│   │   ├── exemploRoute.js
│   │   └── arquivoRoute.js
│   └── server.js
├── .env                ← nunca suba para o GitHub
├── .gitignore
├── package.json
└── prisma.config.js
```

---

## 🔁 Fluxo da Requisição

```
Cliente (Postman)
   ↓
Render (API Node rodando aqui)
   ↓
apiKey.js (valida X-API-Key)
   ↓
Multer + fileGate.js (recebe arquivo em memória)
   ↓
Validação + regra de negócio
   ↓
Supabase Storage (salva foto ou documento)
   ↓
Prisma + PostgreSQL Supabase (salva URL no banco)
```

---

## 🛣️ Rotas da API

| Método | Rota                          | Descrição                |
| ------ | ----------------------------- | ------------------------ |
| GET    | `/`                           | Status da API            |
| POST   | `/api/exemplos`               | Criar exemplo            |
| GET    | `/api/exemplos`               | Listar exemplos          |
| GET    | `/api/exemplos/:id`           | Buscar exemplo por ID    |
| PUT    | `/api/exemplos/:id`           | Atualizar exemplo        |
| DELETE | `/api/exemplos/:id`           | Deletar exemplo          |
| POST   | `/api/exemplos/:id/foto`      | Upload de foto           |
| GET    | `/api/exemplos/:id/foto`      | Retorna URL da foto      |
| DELETE | `/api/exemplos/:id/foto`      | Remover foto             |
| POST   | `/api/exemplos/:id/documento` | Upload de documento      |
| GET    | `/api/exemplos/:id/documento` | Retorna URL do documento |
| DELETE | `/api/exemplos/:id/documento` | Remover documento        |

---

## Etapa 1 — Desenvolvimento Local

### 1.1 Configurar `.env` local

Crie o arquivo `.env` na raiz do projeto com as credenciais do **PostgreSQL local**:

```env
# Servidor
PORT=3000

# Banco de dados local
DATABASE_URL="postgresql://postgres:amods@localhost:7777/exemplo_db"
```

### 1.2 Schema do banco

`prisma/schema.prisma`:

```prisma
generator client {
    provider = "prisma-client-js"
}

datasource db {
    provider = "postgresql"
}

model Exemplo {
    id        Int       @id @default(autoincrement())
    nome      String
    foto      String?
    documento String?

    @@map("exemplo")
}
```

### 1.3 Testar rotas no Postman

Use `http://localhost:3000` diretamente. Teste cada rota abaixo:

**Criar exemplo:**
```
POST http://localhost:3000/api/exemplos
Content-Type: application/json

{
    "nome": "Omega"
}
```

**Listar exemplos:**
```
GET http://localhost:3000/api/exemplos
```

Teste também os filtros disponíveis via query string:
```
GET http://localhost:3000/api/exemplos?nome=alpha
GET http://localhost:3000/api/exemplos?estado=true
GET http://localhost:3000/api/exemplos?preco=3500
```

**Buscar por ID:**
```
GET http://localhost:3000/api/exemplos/1
```

**Atualizar exemplo:**
```
PUT http://localhost:3000/api/exemplos/1
Content-Type: application/json

{
    "nome": "Exemplo Omega"
}
```

**Deletar exemplo:**
```
DELETE http://localhost:3000/api/exemplos/1
```

> ✅ CRUD funcionando? Avance para a Etapa 2.

---

## Etapa 2 — Banco de Dados no Supabase

### 2.1 Criar projeto no Supabase

1. Acesse [supabase.com](https://supabase.com) e faça login
2. Clique em **New project**
3. Preencha:
   - **Name:** `exemplo-db`
   - **Database Password:** anote essa senha
   - **Region:** `South America (São Paulo)`
4. Em **Security**, configure:
   - **Enable Data API:** ❌ desativado
   - **Enable automatic RLS:** ❌ desativado
5. Clique em **Create new project** e aguarde ~2 minutos

> ⚠️ Ambos desativados pois usamos conexão direta ao PostgreSQL via `DATABASE_URL` com Prisma. O controle de acesso é feito pela API Key da nossa aplicação.

### 2.2 Pegar a Connection String

1. No painel do projeto, clique em **Connect**
2. Selecione **ORM** → **Prisma**
3. Copie as variáveis exibidas em `.env.local` e adicione no seu `.env`
4. Comente a `DATABASE_URL` local e adicione as duas novas:

```env
# Servidor
PORT=3000

# Banco de dados local (comentado)
# DATABASE_URL="postgresql://postgres:senha@localhost:7777/exemplo_db"

# Supabase — conexão via pooler (aplicação)
DATABASE_URL="postgresql://..."

# Supabase — conexão direta (migrations)
DIRECT_URL="postgresql://..."
```

5. Substitua `[YOUR-PASSWORD]` pela senha anotada na etapa 2.1

### 2.3 Ajustar o `prisma.config.js`

O Supabase usa PgBouncer (pooling) por padrão. O Prisma precisa de conexão direta para migrations, por isso usa `DIRECT_URL`:

Ajuste o `prisma.config.js`:

```js
datasource: {
    url: process.env['DIRECT_URL'],
},
```

### 2.4 Rodar a migration

```bash
npx prisma migrate dev --name init
npx prisma generate
```

### 2.5 Rodar o Seed

```bash
npx prisma db seed
```

> Navegue no Supabase: **Table Editor**, **SQL Editor** e **Schema Visualizer** para confirmar os dados.

### 2.6 Testar rotas no Postman

A API ainda roda local — apenas o banco agora é o Supabase. Use `http://localhost:3000` normalmente.

**Criar exemplo:**
```
POST http://localhost:3000/api/exemplos
Content-Type: application/json

{
    "nome": "Omega"
}
```

**Listar exemplos:**
```
GET http://localhost:3000/api/exemplos
```

**Buscar por ID:**
```
GET http://localhost:3000/api/exemplos/1
```

**Atualizar exemplo:**
```
PUT http://localhost:3000/api/exemplos/1
Content-Type: application/json

{
    "nome": "Exemplo Omega"
}
```

**Deletar exemplo:**
```
DELETE http://localhost:3000/api/exemplos/1
```

> ✅ CRUD funcionando com banco no Supabase? Avance para a Etapa 3.

---

## Etapa 3 — Rotas de Arquivo (foto e documento)

### 3.1 Criar `src/lib/middlewares/apiKey.js`

```js
export const apiKey = (req, res, next) => {
    const key = req.headers['x-api-key'];

    if (!key || key !== process.env.API_KEY) {
        return res.status(401).json({ error: 'API Key inválida ou ausente.' });
    }

    next();
};
```

### 3.2 Adicionar no `.env`

```env
# Servidor
PORT=3000
API_KEY="exemplo-backend-2026"
```

### 3.3 Registrar em `src/server.js`

```js
import { apiKey } from './lib/middlewares/apiKey.js';

app.use('/api/exemplos', apiKey, exemplosRoutes);
```

### 3.4 Testar no Postman

**Sem API Key — deve retornar 401**

**Com API Key — deve funcionar normalmente**

### 3.5 Criar `src/lib/middlewares/fileGate.js`

```js
import multer from 'multer';

const storage = multer.memoryStorage();

const tiposPermitidos = [
    'image/jpeg',
    'image/png',
    'image/webp',
    'image/gif',
    'application/pdf',
    'application/msword',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    'application/vnd.ms-excel',
    'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    'text/plain',
    'text/csv',
];

const fileFilter = (req, file, cb) => {
    tiposPermitidos.includes(file.mimetype)
        ? cb(null, true)
        : cb(new Error('Tipo de arquivo não permitido.'));
};

export const upload = multer({
    storage,
    fileFilter,
    limits: { fileSize: 1 * 1024 * 1024 }, // 1MB
});
```

> ⚠️ Arquivos ficam em memória (`memoryStorage`) — não são salvos em disco, vão direto pro Supabase Storage.
> ⚠️ Os tipos permitidos são verificados pelo `mimetype` do arquivo, não pela extensão, para evitar falsificação.

---

## Etapa 4 — Storage no Supabase (foto e documento)

### 4.1 Criar bucket no Supabase

1. No painel do Supabase, acesse **Storage → New bucket**
2. Preencha:
   - **Name:** `arquivos`
   - **Public bucket:** ✅ ativado
   - **Restrict file size:** ❌ desativado
   - **Restrict MIME types:** ❌ desativado
3. Clique em **Save**

> ⚠️ O bucket vai ser configurado para ser público, pois toda segurança e regra de negocios será gerenciada no backend.

### 4.2 Criar `src/lib/services/supabase.js`

```js
import { createClient } from '@supabase/supabase-js';
import 'dotenv/config';

const supabase = createClient(
    process.env.SUPABASE_URL,
    process.env.SUPABASE_SERVICE_ROLE_KEY
);

export default supabase;
```

### 4.3 Pegar credenciais do Supabase

1. Na **Project Overview (home)** do projeto, copie a `SUPABASE_URL` com `Project URL`
2. Em **Project Settings → Configuration → API Keys**:
   - Em **Legacy anon, service_role API keys**, copie a `service_role secret`

### 4.4 Adicionar no `.env`
```env
# Supabase Storage
SUPABASE_URL="https://xxxx.supabase.co"
SUPABASE_SERVICE_ROLE_KEY="eyJ..."
```

> ⚠️ Nunca suba o `.env` para o GitHub. Use `service_role` apenas no backend — ela tem permissão total.

### 4.5 Criar `src/lib/helpers/arquivoHelper.js`

```js
import sharp from 'sharp';
import supabase from '../services/supabase.js';

const BUCKET = 'arquivos';

const prepararFoto = async (buffer) =>
    sharp(buffer)
        .resize({ width: 800, withoutEnlargement: true })
        .webp({ quality: 80 })
        .toBuffer();

export const upload = async (id, file) => {
    const ehFoto = file.mimetype.startsWith('image/');

    const buffer = ehFoto ? await prepararFoto(file.buffer) : file.buffer;
    const path = ehFoto
        ? `${id}/foto.webp`
        : `${id}/documento.${file.originalname.split('.').pop()}`;
    const contentType = ehFoto ? 'image/webp' : file.mimetype;

    const { error } = await supabase.storage
        .from(BUCKET)
        .upload(path, buffer, { contentType, upsert: true });

    if (error) throw new Error(error.message);

    return supabase.storage.from(BUCKET).getPublicUrl(path).data.publicUrl;
};

export const deletar = async (url) => {
    const path = url.split(`${BUCKET}/`)[1];
    const { error } = await supabase.storage.from(BUCKET).remove([path]);
    if (error) throw new Error(error.message);
};
```
> O Sharp converte jpeg, png, webp e gif para `.webp`, padronizando o formato e reduzindo o tamanho.

### 4.6 Criar `src/controllers/arquivoController.js`

```js
import ExemploModel from '../models/ExemploModel.js';
import {
    upload as uploadStorage,
    deletar as deletarStorage,
} from '../lib/helpers/arquivoHelper.js';

const uploadArquivo = (tipo) => async (req, res) => {
    try {
        const { id } = req.params;
        if (isNaN(id)) return res.status(400).json({ error: 'ID inválido.' });

        const exemplo = await ExemploModel.buscarPorId(parseInt(id));
        if (!exemplo) return res.status(404).json({ error: 'Registro não encontrado.' });
        if (!req.file) return res.status(400).json({ error: 'Nenhum arquivo enviado.' });

        if (exemplo[tipo]) await deletarStorage(exemplo[tipo]);
        exemplo[tipo] = await uploadStorage(id, req.file);
        const data = await exemplo.atualizar();

        return res.status(200).json({ message: `${tipo} enviado com sucesso!`, url: data[tipo] });
    } catch (error) {
        return res.status(500).json({ error: `Erro ao fazer upload do ${tipo}.` });
    }
};

const buscarArquivo = (tipo) => async (req, res) => {
    try {
        const { id } = req.params;
        if (isNaN(id)) return res.status(400).json({ error: 'ID inválido.' });

        const exemplo = await ExemploModel.buscarPorId(parseInt(id));
        if (!exemplo) return res.status(404).json({ error: 'Registro não encontrado.' });
        if (!exemplo[tipo]) return res.status(404).json({ error: `Nenhum ${tipo} cadastrado.` });

        return res.status(200).json({ url: exemplo[tipo] });
    } catch (error) {
        return res.status(500).json({ error: `Erro ao buscar ${tipo}.` });
    }
};

const deletarArquivo = (tipo) => async (req, res) => {
    try {
        const { id } = req.params;
        if (isNaN(id)) return res.status(400).json({ error: 'ID inválido.' });

        const exemplo = await ExemploModel.buscarPorId(parseInt(id));
        if (!exemplo) return res.status(404).json({ error: 'Registro não encontrado.' });
        if (!exemplo[tipo]) return res.status(404).json({ error: `Nenhum ${tipo} para remover.` });

        await deletarStorage(exemplo[tipo]);
        exemplo[tipo] = null;
        await exemplo.atualizar();

        return res.status(200).json({ message: `${tipo} removido com sucesso!` });
    } catch (error) {
        return res.status(500).json({ error: `Erro ao remover ${tipo}.` });
    }
};

export const uploadFoto = uploadArquivo('foto');
export const buscarFoto = buscarArquivo('foto');
export const deletarFoto = deletarArquivo('foto');

export const uploadDocumento = uploadArquivo('documento');
export const buscarDocumento = buscarArquivo('documento');
export const deletarDocumento = deletarArquivo('documento');
```

### 4.7 Criar `src/routes/arquivoRoute.js`

```js
import express from 'express';
import * as controller from '../controllers/arquivoController.js';
import { upload } from '../lib/middlewares/fileGate.js';

const router = express.Router();

router.post('/:id/foto', upload.single('foto'), controller.uploadFoto);
router.get('/:id/foto', controller.buscarFoto);
router.delete('/:id/foto', controller.deletarFoto);

router.post('/:id/documento', upload.single('documento'), controller.uploadDocumento);
router.get('/:id/documento', controller.buscarDocumento);
router.delete('/:id/documento', controller.deletarDocumento);

export default router;
```

### 4.8 Registrar em src/server.js

```js
import arquivoRoutes from './routes/arquivoRoute.js';

app.use('/api/exemplos', apiKey, arquivoRoutes);
```
### 4.9 Testar no Postman

**Upload de foto:**
```
POST http://localhost:3000/api/exemplos/1/foto
Body → form-data
  key:  foto  |  type: File  |  value: [selecione uma imagem]
```

**Buscar foto:**
```
GET http://localhost:3000/api/exemplos/1/foto
```

**Deletar foto:**
```
DELETE http://localhost:3000/api/exemplos/1/foto
```

**Upload de documento:**
```
POST http://localhost:3000/api/exemplos/1/documento
Body → form-data
  key:  documento  |  type: File  |  value: [selecione um PDF, Word ou Excel]
```

**Buscar documento:**
```
GET http://localhost:3000/api/exemplos/1/documento
```

**Deletar documento:**
```
DELETE http://localhost:3000/api/exemplos/1/documento
```
> Confirme no Supabase: **Storage → arquivos → pasta `1/`** que os arquivos aparecem após o upload e somem após o delete.

> ✅ Storage funcionando? Avance para a Etapa 5 — Deploy no Render.

---

## Etapa 5 — Deploy no Render

### 5.1 Criar conta e conectar GitHub no Render

1. Log associando sua conta **GitHub** e autorize o acesso

### 5.2 Criar o Web Service

1. No dashboard, clique em **New → Web Service**
2. Selecione o repositório da API
3. Preencha:
   - **Name:** `Backend-Supabase-Render`
   - **Language:** `Node`
   - **Branch:** `main`
   - **Region:** `Oregon (US West) - melhor latencia para o Brasil`
   - **Root Directory:** deixe em branco (a menos que package.json esteja em uma subpasta)
   - **Build Command:** `npm install && npx prisma generate`
   - **Start Command:** `node src/server.js`
   - **Instance Type:** `Free`

### 5.3 Configurar variáveis de ambiente

Em **Environment → Add Environment Variable**, adicione todas as variáveis do `.env`:

| Key | Value |
|-----|-------|
| `PORT` | `3000` |
| `API_KEY` | `exemplo-backend-2026` |
| `DATABASE_URL` | `postgresql://...` (pooler) |
| `DIRECT_URL` | `postgresql://...` (direct) |
| `SUPABASE_URL` | `https://xxxx.supabase.co` |
| `SUPABASE_SERVICE_ROLE_KEY` | `eyJ...` |

### 5.4 Fazer o deploy

Clique em **Deploy Web Service** e acompanhe os logs:

```
==> Cloning from https://github.com/mapcarboni/Backend-Supabase-Render
==> Checking out commit 145debbcd4be6174c218e5ecc7b3a9b7dc2fa758 in branch main
==> Using Node.js version 22.22.0 (default)
==> Docs on specifying a Node.js version: https://render.com/docs/node-version
==> Running build command 'npm install && npx prisma generate'...

    ...

==> Uploading build...
==> Uploaded in 5.0s. Compression took 2.1s
==> Build successful 🎉
==> Deploying...
==> Setting WEB_CONCURRENCY=1 by default, based on available CPUs in the instance
==> Running 'node src/server.js'
🚀 Servidor rodando em http://localhost:3000
==> Your service is live 🎉
==> Available at your primary URL https://[nome-do-seu-projeto].onrender.com
```

### 5.5 Testar no Postman com a URL do Render

Anote a URL fornecida pelo Render e configure uma variável de ambiente no Postman:
- **Variable:** `base_url`
- **Value:** `https://[nome-do-seu-projeto].onrender.com`

**Status da API:**
```
GET {{base_url}}/
```

**Criar exemplo:**
```
POST {{base_url}}/api/exemplos
Header: X-API-Key: exemplo-backend-2026
Content-Type: application/json

{
    "nome": "Deploy Omega"
}
```

**Listar exemplos:**
```
GET {{base_url}}/api/exemplos
Header: X-API-Key: exemplo-backend-2026
```

**Buscar por ID:**
```
GET {{base_url}}/api/exemplos/1
Header: X-API-Key: exemplo-backend-2026
```

**Atualizar exemplo:**
```
PUT {{base_url}}/api/exemplos/1
Header: X-API-Key: exemplo-backend-2026
Content-Type: application/json

{
    "nome": "Exemplo Omega Atualizado"
}
```

**Deletar exemplo:**
```
DELETE {{base_url}}/api/exemplos/1
Header: X-API-Key: exemplo-backend-2026
```

**Upload de foto:**
```
POST {{base_url}}/api/exemplos/1/foto
Header: X-API-Key: exemplo-backend-2026
Body → form-data
  key: foto  |  type: File  |  value: [selecione uma imagem]
```

**Buscar foto:**
```
GET {{base_url}}/api/exemplos/1/foto
Header: X-API-Key: exemplo-backend-2026
```

**Deletar foto:**
```
DELETE {{base_url}}/api/exemplos/1/foto
Header: X-API-Key: exemplo-backend-2026
```

**Upload de documento:**
```
POST {{base_url}}/api/exemplos/1/documento
Header: X-API-Key: exemplo-backend-2026
Body → form-data
  key: documento  |  type: File  |  value: [selecione um PDF, Word ou Excel]
```

**Buscar documento:**
```
GET {{base_url}}/api/exemplos/1/documento
Header: X-API-Key: exemplo-backend-2026
```

**Deletar documento:**
```
DELETE {{base_url}}/api/exemplos/1/documento
Header: X-API-Key: exemplo-backend-2026
```

> ⚠️ No plano free o Render hiberna após 15 minutos de inatividade. A primeira requisição pode levar ~30 segundos.

> ✅ API respondendo pela URL do Render? Deploy concluído!
