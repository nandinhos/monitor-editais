# Changelog & Aprendizados Arquiteturais do Starter Kit

> **Origem**: Análise e síntese do histórico do repositório base (`nunomaduro/laravel-starter-kit-inertia-react`).  
> **Propósito**: Preservar o histórico de decisões técnicas, boas práticas, lições aprendidas e evolução arquitetural do starter kit antes da inicialização do repositório local do **Monitor de Editais**.

---

## Índice

1. [Visão Geral & Filosofia de Design](#1-visão-geral--filosofia-de-design)
2. [Padrões Arquiteturais e Decisões de Backend](#2-padrões-arquiteturais-e-decisões-de-backend)
   - [2.1 Arquitetura Action-First (`app/Actions`)](#21-arquitetura-action-first-appactions)
   - [2.2 Identificadores UUID por Padrão](#22-identificadores-uuid-por-padrão)
   - [2.3 Segurança, Autenticação e 2FA](#23-segurança-autenticação-e-2fa)
   - [2.4 Transações de Banco de Dados (`DB::transaction`)](#24-transações-de-banco-de-dados-dbtransaction)
3. [Armadilhas Críticas & Lições Superadas](#3-armadilhas-críticas--lições-superadas)
   - [3.1 Double Hashing com Casts do Laravel](#31-double-hashing-com-casts-do-laravel)
   - [3.2 Rate Limiting em Rotas de Dois Fatores (2FA)](#32-rate-limiting-em-rotas-de-dois-fatores-2fa)
   - [3.3 Cache Relativo em Ambientes CI / Docker (PHPStan e Rector)](#33-cache-relativo-em-ambientes-ci--docker-phpstan-e-rector)
4. [Evolução do Frontend (React + Inertia v3 + Vite)](#4-evolução-do-frontend-react--inertia-v3--vite)
   - [4.1 Layouts Persistentes](#41-layouts-persistentes)
   - [4.2 Transição do Compilador (Rolldown no lugar de Babel)](#42-transição-do-compilador-rolldown-no-lugar-de-babel)
   - [4.3 Sistema de Notificações / Toasts](#43-sistema-de-notificações--toasts)
5. [Estratégia de Testes e Qualidade Contínua](#5-estratégia-de-testes-e-qualidade-contínua)
   - [5.1 Pest 5 & Testes de Arquitetura (`pest arch`)](#51-pest-5--testes-de-arquitetura-pest-arch)
   - [5.2 Testes End-to-End com Playwright](#52-testes-end-to-end-com-playwright)
6. [Timeline Completa de Commits Indexada](#6-timeline-completa-de-commits-indexada)

---

## 1. Visão Geral & Filosofia de Design

O starter kit original foi concebido para fornecer uma fundação de nível empresarial baseada em:
- **PHP 8.4+ / Laravel 13**
- **Inertia.js v3** com **React**
- **Tailwind CSS** e **Vite**
- **Laravel Fortify** para backend de autenticação desacoplado
- **Pest v5** com regras arquiteturais rígidas

A evolução do repositório demonstra um movimento claro de:
1. Desacoplamento de controllers em favor de Actions reutilizáveis.
2. Adoção de convenções de segurança modernas (UUIDs, Passkeys, 2FA protegido).
3. Maximização de performance no pipeline de compilação frontend (Bun, Rolldown, Vite Plus).

---

## 2. Padrões Arquiteturais e Decisões de Backend

### 2.1 Arquitetura Action-First (`app/Actions`)
- **Problema**: Lógica de negócios e regras de autenticação misturadas em Controllers ou Models geram acoplamento e dificultam testes unitários e reuso em comandos CLI / APIs.
- **Solução adotada**: Refatoração de toda a autenticação e operações de usuário para classes de ação dedicadas (`app/Actions/`):
  - `CreateUser`, `CreateUserPassword`, `UpdateUserPassword`, `DeleteUser`, `UpdateUserProfileInformation`.
  - Cada Action possui injeção de dependência via construtor e um único método público `handle()`.

### 2.2 Identificadores UUID por Padrão
- **Commit**: `56fcc35` (*feat: uses uuids by default*)
- **Prática**: Substituição de auto-incrementos numéricos por UUIDs nas tabelas centrais:
  - Na migração: `$table->uuid('id')->primary();`
  - No Model: Uso do trait `Illuminate\Database\Eloquent\Concerns\HasUuids`.
- **Benefício**: Evita enumeração sequencial de IDs, simplifica migrações distribuídas e melhora a segurança.

### 2.3 Segurança, Autenticação e 2FA
- Suporte a múltiplos métodos de autenticação:
  - Senha com regras estritas (`Password::defaults()`).
  - Autenticação de dois fatores (TOTP / QR code / Códigos de recuperação).
  - Passkeys / WebAuthn para logins biométricos modernos.
- Desacoplamento da interface através do Laravel Fortify, permitindo que o frontend em React/Inertia controle 100% da UX sem Blade views.

### 2.4 Transações de Banco de Dados (`DB::transaction`)
- **Commit**: `8a96ef6` (*fix: missing transaction*)
- **Lição**: Operações compostas que criam ou atualizam mais de um registro (por exemplo, criar usuário e gerar tokens/dados padrão) devem sempre ser envelopadas em `DB::transaction(fn () => ...)`.

---

## 3. Armadilhas Críticas & Lições Superadas

### 3.1 Double Hashing com Casts do Laravel
- **Commits**: `122486c` (*fix: double hash*) e `a4e0e7d` (*fix: user factory un-required hashing due to casts*)
- **Causa Raiz**: O model `User` já define o cast nativo:
  ```php
  protected function casts(): array
  {
      return [
          'password' => 'hashed',
      ];
  }
  ```
  Ao passar `Hash::make('minha-senha')` no factory ou numa Action, o Laravel aplicava hash sobre o hash gerado, tornando a senha irrecuperável e falhando autenticações.
- **Solução**: Passar a senha em texto plano para o model; o cast `'hashed'` cuidará da criptografia Bcrypt/Argon2 automaticamente.

### 3.2 Rate Limiting em Rotas de Dois Fatores (2FA)
- **Commit**: `81f7488` (*fix: added missing Ratelimiter for 2FA route*)
- **Lição**: Rotas de confirmação de 2FA são alvos de força bruta rápida. Foi inserido limitador de requisições (`RateLimiter`) explícito no `FortifyServiceProvider` para proteger a validação do código de 6 dígitos.

### 3.3 Cache Relativo em Ambientes CI / Docker (PHPStan e Rector)
- **Commits**: `302c362` e `bb32dfc` (*make phpstan, rector and ci cache dirs project-relative*)
- **Problema**: O cache padrão do PHPStan e do Rector em `/tmp` causa perda de cache entre execuções de containers Docker e conflitos de permissões multi-usuário.
- **Solução**: Configurar os caches para pastas relativas dentro do projeto (ex: `.cache/phpstan`, `.cache/rector`), ignoradas no `.gitignore`.

---

## 4. Evolução do Frontend (React + Inertia v3 + Vite)

### 4.1 Layouts Persistentes
- **Commit**: `8d6bf44` (*feat: uses persistent layouts*)
- **Prática**: Ao invés de embrulhar cada página manualmente com `<AppLayout>`, adotar a propriedade de layout persistente do Inertia:
  ```tsx
  Page.layout = (page: React.ReactNode) => <AppLayout children={page} />;
  ```
- **Ganho**: O estado de componentes globais (menus, inputs de busca, modais) e a posição do scroll não são destruídos nem recalculados durante a navegação interna.

### 4.2 Transição do Compilador (Rolldown no lugar de Babel)
- **Commit**: `cf21bff` (*fix: add new rolldown replacement for babel for vite-react v6*)
- **Lição**: A transição para o novo compilador Rolldown reduziu drasticamente o tempo de build do Vite para componentes React modernos em comparação ao pipeline legado com Babel.

### 4.3 Sistema de Notificações / Toasts
- **Commit**: `cf9c0d2` (*feat: add toasts from laravel starter kit*) e `98242e0`
- **Padrão**: Flash messages transmitidas pelo Inertia através de middleware (`HandleInertiaRequests`) são interceptadas no frontend por um hook de Toasts, fornecendo feedback unificado para ações assíncronas.

---

## 5. Estratégia de Testes e Qualidade Contínua

### 5.1 Pest 5 & Testes de Arquitetura (`pest arch`)
- **Commit**: `5630f64` (*feat: add the Laravel preset to pest arch tests*)
- **Prática**: Uso do preset nativo do Laravel no Pest para assegurar integridade arquitetural:
  ```php
  arch('app')
      ->preset()
      ->laravel();
  ```
  Isso valida automaticamente: ausência de chamadas `dd()` ou `dump()`, tipagem consistente, controllers que não acessam models diretamente sem abstração, etc.

### 5.2 Testes End-to-End com Playwright
- **Commit**: `04c3cd7` (*feat: caches playwright installation*) e `50a2866` (*test: adds browser test for resetting the password*)
- **Lição**: Testes de ponta a ponta em navegador garantem fluxos críticos de autenticação (reset de senha, exclusão de conta, 2FA) que não podem ser completamente cobertos por testes HTTP puros.

---

## 6. Timeline Completa de Commits Indexada

| Hash | Categoria | Descrição |
| :--- | :--- | :--- |
| `70defd0` | `init` | Commit inicial do starter kit |
| `1c7828b` | `env` | Exigência mínima de PHP 8.4 |
| `d4bea01` | `model` | Ajustes no modelo User |
| `f83ec64` | `model` | Remoção de `$fillable` |
| `ec6881a` | `model` | `Model::unguard()` global |
| `8792acb` | `refactor` | CRUD e Actions para usuários |
| `4f3bb82` | `refactor` | Refatoração de gerenciamento de senha para Actions |
| `9ae8171` | `refactor` | Refatoração de Two-Factor Authentication para Actions |
| `4946734` | `refactor` | Refatoração de código de e-mail para Actions |
| `e8a6a77` | `refactor` | Mais rotinas de autenticação migradas para Actions |
| `56fcc35` | `feat` | Adoção de UUIDs como chave primária padrão |
| `122486c` | `fix` | Correção de double hash de senhas |
| `8a96ef6` | `fix` | Adição de transações atômicas (`DB::transaction`) |
| `81f7488` | `security`| Rate limiting adicionado às rotas de 2FA |
| `9d75807` | `tooling` | Adoção de Bun para gerenciamento rápido de dependências |
| `52fe5db` | `upgrade` | Upgrade estrutural para Laravel 13 e Inertia v3 |
| `5630f64` | `tests` | Inclusão do preset Laravel nos testes de arquitetura do Pest |
| `e1a82a7` | `upgrade` | Adoção do Pest 5 |
| `cf21bff` | `frontend`| Integração de Rolldown para Vite React v6 |
| `cf9c0d2` | `frontend`| Sistema de Toasts reativos via Inertia |
| `03961b9` | `security`| Adição de Passkeys (WebAuthn) e regras de senha |
| `8d6bf44` | `frontend`| Adoção de layouts persistentes no React |
| `302c362` | `tooling` | Diretório de cache relativo ao projeto para Rector e PHPStan |
| `50a2866` | `tests` | Teste E2E de redefinição de senha com navegador |
