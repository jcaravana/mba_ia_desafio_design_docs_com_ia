# GitHub Copilot Custom Instructions

Consulte as regras completas de arquitetura e domínio em AGENTS.md.

# Visão Geral da Arquitetura do Monorepo

O **Babliger Monorepo** é composto por quatro aplicações independentes e pastas de documentação técnica/arquitetura:

## 1. Subprojetos

### 1.1. BACKEND (`BACKEND/`)
- **Stack**: Node.js (`>= 23`), Express 5, Mongoose / MongoDB, Socket.io (`@socket.io/cluster-adapter`, `@socket.io/sticky`), Sucrase (ES modules).
- **Comandos**:
  - `yarn dev`: Executa com nodemon e hot-reload (porta 3333; MongoDB: `mongodb://127.0.0.1:27017/Babliger`).
  - `yarn dev:debug`: Executa com flag `--inspect`.
  - `yarn start`: Executa direto via `sucrase/register`.
  - `yarn build`: Transpila `src/` para `dist/`.
  - `npx eslint src`: Lint manual do código.
- **Estrutura e Padrões**:
  - `src/app.js`: Configura cluster, Express, Mongoose e Socket.io sticky.
  - `src/server.js`: Instancia o singleton `App`.
  - `src/routes.js`: Todas as rotas centralizadas; públicas antes de `authMiddleware`, privadas após.
  - `src/middlewares/autenticacao.js`: Validação de token JWT e checagem de usuário ativo.
  - `src/controllers/*Controller.js`: Singletons exportados com validação Yup.
  - `src/model/*.js`: Schemas Mongoose diretos.

### 1.2. MOBILE (`MOBILE/`)
- **Stack**: Expo SDK 58, React Native 0.88, React 19, TypeScript, `expo-router`, NativeWind, Zustand (MMKV), Drizzle ORM (`expo-sqlite`), axios.
- **Comandos**:
  - `yarn start`: Metro bundler (`npx expo start`).
  - `yarn android` / `yarn ios`: Build e execução nativa.
  - `yarn test --watchAll=false`: Testes unitários Jest (`jest-expo`).
  - `yarn lint`: ESLint (`eslint-config-expo` + prettier).
  - `npx drizzle-kit generate`: Gera migrations do banco SQLite a partir de `src/database/schemas/schema.ts`.
- **Estrutura e Padrões**:
  - `src/app/`: Roteamento por arquivo via `expo-router`.
  - `src/store/`: Stores Zustand por domínio persistidas com MMKV.
  - `src/tasks/backgroundTaskSincronizacaoSync.ts`: Sincronismo offline em background.
  - `src/services/*Service.ts`: Lógica de sincronização e comunicação com a API.
  - `src/lib/api.ts`: Cliente HTTP axios com interceptor para renovação/logout 401.

### 1.3. BACKOFFICE-WEB (`BACKOFFICE-WEB/`)
- **Stack**: Node.js (`>= 22.12.0`), Vite 7, React 19, TypeScript, Material UI (`@mui/material`), Tailwind 4, TanStack React Query, React Hook Form + Zod.
- **Comandos**:
  - `yarn dev` ou `npm run dev`: Servidor Vite de desenvolvimento.
  - `yarn build` ou `npm run build`: Typecheck (`tsc`) e build de produção Vite.
  - `yarn lint` ou `npm run lint`: ESLint (`eslint.config.mjs`).
  - `yarn preview`: Pré-visualização do bundle de produção.
- **Objetivo**: Gestão administrativa geral do sistema Babliger (conteúdo global de cartões/categorias, moderação de contas, métricas e auditoria).

### 1.4. PORTAL-PAIS-WEB (`PORTAL-PAIS-WEB/`)
- **Stack**: Node.js (`>= 22.12.0`), Next.js 16 (App Router), React 19, TypeScript, Material UI, Tailwind 4, NextAuth, TanStack React Query.
- **Comandos**:
  - `yarn dev` ou `npm run dev`: Servidor Next.js em desenvolvimento.
  - `yarn build` ou `npm run build`: Build de produção Next.js.
  - `yarn start` ou `npm run start`: Inicializa servidor de produção Next.js.
  - `yarn lint` ou `npm run lint`: ESLint (`eslint.config.mjs`).
- **Objetivo**: Portal acessível aos pais e responsáveis para personalização das pranchas da criança, upload de imagens/áudios familiares e visualização do progresso de comunicação.

## 2. Isolamento de Subprojetos
Cada aplicação possui `package.json` próprio e dependências isoladas. Não compartilham scripts de build de raiz. Sempre execute comandos dentro da respectiva subpasta (`cd BACKEND`, `cd MOBILE`, `cd BACKOFFICE-WEB`, `cd PORTAL-PAIS-WEB`).

---

# Contexto de Domínio e Produto: Babliger

## 1. Visão do Produto
O **Babliger** é uma solução de Comunicação Alternativa e Aumentativa (CAA / AAC) voltada para crianças no espectro autista (TEA), pais/cuidadores e terapeutas.

O sistema disponibiliza:
- Interface acessível com pranchas e cartões de comunicação visual e auditiva (TTS ou áudios gravados por familiares).
- Sincronização resiliente offline para que a criança nunca fique sem comunicação, mesmo sem sinal de rede.
- Acompanhamento do desenvolvimento e engajamento da comunicação através de métricas acessíveis no portal dos pais e no backoffice.

## 2. Entidades Principais do Domínio
- **Criança**: Usuário final da prancha no aplicativo mobile. Possui perfil com preferências sensoriais, nível de vocabulário e pranchas associadas.
- **Responsável / Cuidador**: Usuário do aplicativo ou portal web que gerencia a rotina da criança, cartões personalizados e pranchas.
- **Categoria**: Agrupamento temático de cartões (ex.: Alimentação, Sentimentos, Brincadeiras, Higiene).
- **Cartão de Comunicação**: Item individual com pictograma/imagem, rótulo textual e retorno auditivo.
- **Prancha**: Arranjo de cartões em grid configurável.
- **Evento de Interação**: Registro de toques, frases formadas e tempo de uso, sincronizado para análise terapêutica.
- **Assinatura**: Gestão de planos, limites de pranchas e recursos premium.

## 3. Estado do Código Legado e Migração (ADR-003)
O código herdado em `BACKEND/src` e `MOBILE/src` continha a estrutura de monitoramento de condutores de trem (`turno`, `trem`, `rodante`, `equipe`).

- **Regra Fundamental (ADR-003)**: As entidades de trem **não devem ser preservadas como modelo de negócio**. Elas estão sendo progressivamente substituídas pelas entidades do Babliger.
- **Padrão a Preservar e Reutilizar**: O **padrão arquitetural de sincronismo offline** (fila local outbox SQLite + background task resiliente a quedas de rede).

---

# Convenções de Código e Diretrizes de Engenharia

## 1. Idioma e Nomenclatura
- **Modelos de Domínio e Banco de Dados**: Nomes de entidades, atributos, tabelas e coleções devem ser mantidos em **Português do Brasil (`pt-BR`)** (ex.: `criança`, `responsável`, `prancha`, `categoria`).
- **Termos Técnicos e Infraestrutura**: Nomes de bibliotecas, design patterns e infraestrutura seguem o padrão em inglês (ex.: `Service`, `Controller`, `Repository`, `Hook`, `Database`, `Store`).
- **Commits e Documentação Técnica**: Redigidos em **Português do Brasil (`pt-BR`)**.

## 2. Segurança e Segredos
- **Nunca comitar segredos**: Chaves de API, senhas, tokens JWT e senhas de keystores nunca devem ser incluídos em commits.
- **Arquivos `.env`**: Sempre mantenha `.env.example` atualizado em cada subprojeto com as variáveis necessárias e documentadas.

## 3. Qualidade e Integridade do Código
- **Preservação de Comentários**: Preserve comentários e docstrings existentes não relacionados à tarefa atual.
- **Tipagem TypeScript Estrita**: Nos subprojetos TypeScript (`MOBILE`, `BACKOFFICE-WEB`, `PORTAL-PAIS-WEB`), mantenha a checagem rigorosa sem introduzir `any` injustificado.
- **Aderência aos Linters**: Respeite as regras configuradas nos arquivos `eslint.config.mjs` de cada subprojeto.

## 4. Gestão de Diretrizes de IA (SSOT)
- **Fonte única da verdade**: Os arquivos-fonte das diretrizes residem em `ai/guidelines/`. Arquivos como `GEMINI.md`, `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `.windsurfrules` e `.github/copilot-instructions.md` são **gerados automaticamente** pelo script `node scripts/sync-ai.mjs` e **não devem ser editados diretamente**.
- **Para alterar diretrizes**: Edite os arquivos em `ai/guidelines/` e execute `node scripts/sync-ai.mjs` para redistribuir.
- **Para importar alterações feitas em um harness**: Execute `node scripts/sync-ai.mjs --pull <harness>` (ex.: `--pull gemini`, `--pull claude`, `--pull cursor`, `--pull copilot`, `--pull windsurf`, `--pull agents`) e depois `node scripts/sync-ai.mjs` para redistribuir.
