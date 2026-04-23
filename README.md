# 🗄️ BackupCloud API — Arquitetura Completa

> Padrões: Clean Architecture · SOLID · Object Calisthenics · DDD · Big Tech Go Patterns  
> Stack: Go 1.23+ · PostgreSQL · Redis · Backblaze B2 · JWT RS256 · SMTP · Docker  

---

## 🧭 Decisão de Escala: Monolito Modular

```
Monolito Modular → depois Microserviços (se necessário)
```

---

## 📁 Estrutura Completa de Diretórios

```
backup-api/
│
├── cmd/
│   └── api/
│       └── main.go                    # Entrypoint: wiring de dependências (DI manual ou wire)
│
├── internal/                          # Código privado da aplicação (não importável externamente)
│   │
│   ├── domain/                        # Camada de Domínio — zero dependências externas
│   │   ├── backup/
│   │   │   ├── entity.go              # Entidade Backup (regras de negócio puras)
│   │   │   ├── status.go              # Value Object: BackupStatus (enum seguro)
│   │   │   ├── repository.go          # Interface do repositório (Dependency Inversion)
│   │   │   └── service.go             # Domain Service: lógica que não pertence a 1 entidade
│   │   │
│   │   ├── company/
│   │   │   ├── entity.go              # Entidade Company com CNPJ validado
│   │   │   ├── cnpj.go                # Value Object: CNPJ (imutável, auto-validado)
│   │   │   ├── repository.go          # Interface do repositório
│   │   │   └── errors.go              # Erros de domínio específicos
│   │   │
│   │   ├── schedule/
│   │   │   ├── entity.go              # Entidade Schedule (dias + horário + timezone)
│   │   │   ├── recurrence.go          # Value Object: Recurrence (regra de repetição)
│   │   │   ├── repository.go          # Interface do repositório
│   │   │   └── evaluator.go           # Domain Service: calcula próxima execução
│   │   │
│   │   ├── user/
│   │   │   ├── entity.go              # Entidade User
│   │   │   ├── password.go            # Value Object: HashedPassword
│   │   │   ├── role.go                # Value Object: Role (admin, operator, viewer)
│   │   │   └── repository.go          # Interface do repositório
│   │   │
│   │   └── notification/
│   │       ├── event.go               # Domain Events: BackupDone, BackupFailed, etc.
│   │       └── dispatcher.go          # Interface do dispatcher de eventos
│   │
│   ├── application/                   # Camada de Aplicação — orquestra domínio + infra
│   │   ├── usecase/
│   │   │   ├── backup/
│   │   │   │   ├── presign.go         # UC: Gera URL pré-assinada para upload direto ao B2
│   │   │   │   ├── multipart_init.go  # UC: Inicia upload multipart (>5GB)
│   │   │   │   ├── multipart_part.go  # UC: Obtém URL para uma parte
│   │   │   │   ├── complete.go        # UC: Confirma upload e verifica integridade
│   │   │   │   ├── abort.go           # UC: Aborta upload em andamento
│   │   │   │   ├── list.go            # UC: Lista backups por CNPJ + filtros
│   │   │   │   └── get_status.go      # UC: Retorna status atual do backup
│   │   │   │
│   │   │   ├── schedule/
│   │   │   │   ├── upsert.go          # UC: Cria ou atualiza agendamento de CNPJ
│   │   │   │   ├── delete.go          # UC: Remove agendamento
│   │   │   │   ├── get.go             # UC: Retorna agendamento atual
│   │   │   │   └── check_overdue.go   # UC: Verifica CNPJs com backup atrasado
│   │   │   │
│   │   │   ├── auth/
│   │   │   │   ├── login.go           # UC: Login → JWT access + refresh token
│   │   │   │   ├── refresh.go         # UC: Renova access token via refresh (rotation)
│   │   │   │   ├── logout.go          # UC: Invalida tokens (blacklist no Redis)
│   │   │   │   └── register.go        # UC: Registra novo usuário/operador
│   │   │   │
│   │   │   └── company/
│   │   │       ├── register.go        # UC: Cadastra novo CNPJ → dispara email
│   │   │       ├── update.go          # UC: Atualiza dados da empresa
│   │   │       └── list.go            # UC: Lista empresas com paginação
│   │   │
│   │   ├── authz/                     # Autorização por recurso
│   │   │   └── checker.go             # AssertBackupAccess, AssertScheduleAccess
│   │   │
│   │   └── port/                      # Ports (interfaces que a aplicação precisa da infra)
│   │       ├── storage.go             # Interface: StorageProvider (B2, S3)
│   │       ├── mailer.go              # Interface: Mailer
│   │       ├── cache.go               # Interface: Cache
│   │       ├── token.go               # Interface: TokenProvider
│   │       └── clock.go               # Interface: Clock (testabilidade)
│   │
│   ├── infrastructure/                # Camada de Infraestrutura — implementa as interfaces
│   │   ├── persistence/
│   │   │   ├── postgres/
│   │   │   │   ├── db.go              # Conexão, pool, health check
│   │   │   │   ├── backup_repo.go     # Implementa domain/backup/repository.go
│   │   │   │   ├── company_repo.go    # Implementa domain/company/repository.go
│   │   │   │   ├── schedule_repo.go   # Implementa domain/schedule/repository.go
│   │   │   │   └── user_repo.go       # Implementa domain/user/repository.go
│   │   │   └── migrations/            # Arquivos SQL versionados (goose)
│   │   │       ├── 001_create_users.sql
│   │   │       ├── 002_create_companies.sql
│   │   │       ├── 003_create_backups.sql
│   │   │       ├── 004_create_schedules.sql
│   │   │       └── 005_create_sessions.sql
│   │   │
│   │   ├── storage/
│   │   │   └── b2/
│   │   │       ├── client.go          # Implementa port/storage.go via SDK do B2
│   │   │       ├── multipart.go       # Lógica de Large File API (>5GB em partes)
│   │   │       └── presigned.go       # URLs pré-assinadas para download seguro
│   │   │
│   │   ├── cache/
│   │   │   └── redis/
│   │   │       ├── client.go          # Conexão Redis com retry e circuit breaker
│   │   │       ├── token_blacklist.go # Blacklist de tokens JWT invalidados
│   │   │       └── rate_limiter.go    # Rate limiting por IP/usuário
│   │   │
│   │   ├── mailer/
│   │   │   ├── smtp/
│   │   │   │   └── client.go          # Implementa port/mailer.go via SMTP
│   │   │   └── templates/
│   │   │       ├── backup_done.html   # Template: backup concluído
│   │   │       ├── backup_failed.html # Template: backup falhou/atrasado
│   │   │       ├── company_added.html # Template: novo CNPJ cadastrado
│   │   │       ├── config_confirm.html# Template: confirmação de configuração
│   │   │       └── base.html          # Layout base dos emails
│   │   │
│   │   ├── scheduler/
│   │   │   ├── engine.go              # Scheduler interno (gocron)
│   │   │   ├── backup_job.go          # Job: verifica e dispara alertas de overdue
│   │   │   └── cleanup_job.go         # Job: limpa uploads abandonados
│   │   │
│   │   └── token/
│   │       └── jwt/
│   │           └── provider.go        # Implementa port/token.go com JWT RS256
│   │
│   └── interfaces/                    # Camada de Interface — adaptadores HTTP
│       └── http/
│           ├── server.go              # Configuração do servidor (timeouts, TLS)
│           ├── router.go              # Definição de rotas agrupadas por domínio
│           │
│           ├── handler/
│           │   ├── auth_handler.go    # POST /auth/login, /refresh, /logout, etc.
│           │   ├── backup_handler.go  # POST /backups/presign, /multipart, /confirm
│           │   ├── company_handler.go # CRUD /companies
│           │   └── schedule_handler.go# CRUD /schedules
│           │
│           ├── middleware/
│           │   ├── auth.go            # Valida JWT + injeta claims no contexto
│           │   ├── rate_limit.go      # Rate limiting via Redis
│           │   ├── security_headers.go# Cabecalhos de segurança (CSP, X-Content-Type-Options)
│           │   ├── sanitize.go        # Limita body size, valida Content-Type
│           │   ├── cors.go            # CORS para o front-end
│           │   ├── request_id.go      # Injeta X-Request-ID para rastreabilidade
│           │   ├── recover.go         # Recupera panics e loga
│           │   └── logger.go          # Structured logging por request (zerolog)
│           │
│           └── dto/
│               ├── backup_dto.go      # Request/Response structs com validação
│               ├── auth_dto.go        # LoginRequest, TokenResponse, etc.
│               ├── company_dto.go     # CompanyRequest, CompanyResponse
│               └── schedule_dto.go    # ScheduleRequest, ScheduleResponse
│
├── pkg/                               # Código reutilizável (pode ser importado externamente)
│   ├── apperrors/
│   │   ├── errors.go                  # Tipos de erro: NotFound, Conflict, Unauthorized...
│   │   └── codes.go                   # Mapeamento erro → HTTP status code
│   ├── validator/
│   │   └── validator.go               # Wrapper do go-playground/validator com msgs PT-BR
│   ├── logger/
│   │   └── logger.go                  # Logger global (zerolog) com campos padrão
│   ├── crypto/
│   │   └── hash.go                    # bcrypt wrapper, geração de tokens aleatórios
│   ├── pagination/
│   │   └── pagination.go              # Struct Cursor/Offset pagination reutilizável
│   ├── cnpj/
│   │   └── cnpj.go                    # Validação e formatação de CNPJ
│   └── sanitize/
│       └── sanitize.go                # Sanitização HTML com bluemonday
│
├── config/
│   ├── config.go                      # Lê variáveis de ambiente com viper/envconfig
│   └── config.example.env             # Exemplo comentado de todas as variáveis
│
├── docker/
│   ├── Dockerfile                     # Multi-stage build (builder + distroless)
│   ├── docker-compose.yml             # Local: API + Postgres + Redis + MailHog
│   └── docker-compose.prod.yml        # Produção com secrets e healthchecks
│
├── scripts/
│   ├── migrate.sh                     # Roda migrations com goose
│   └── seed.sh                        # Popula dados de desenvolvimento
│
├── docs/
│   └── openapi.yaml                   # Spec OpenAPI 3.0 gerada (swaggo ou manual)
│
├── .github/
│   └── workflows/
│       ├── ci.yml                     # Test + lint + build em cada PR
│       └── cd.yml                     # Deploy automático em merge na main
│
├── go.mod
├── go.sum
├── Makefile                           # Atalhos: make run, make test, make migrate
└── README.md
```

---

## 🏗️ Camadas e Responsabilidades (Clean Architecture)

```
┌─────────────────────────────────────────────┐
│              interfaces/http                 │  ← Fala com o mundo externo (HTTP)
│         handlers · middleware · dto          │
├─────────────────────────────────────────────┤
│              application/usecase             │  ← Orquestra casos de uso
│              application/authz              │  ← Verificações de acesso
│     Não tem regra de negócio, só fluxo       │
├─────────────────────────────────────────────┤
│                   domain/                    │  ← Regras de negócio puras
│        entities · value objects · services  │  ← ZERO dependências externas
├─────────────────────────────────────────────┤
│               infrastructure/                │  ← Detalhes técnicos
│        postgres · redis · b2 · smtp          │
└─────────────────────────────────────────────┘

Regra de dependência: as setas apontam SEMPRE para dentro.
Domain não conhece ninguém. Infrastructure implementa interfaces do Domain/Application.
```

---

## 🚨 Upload de Arquivos — Direto ao B2

```
API apenas coordena, B2 recebe direto
[App Desktop] ──"quero fazer upload"──► [API Go]
                                            │
                                    Valida permissão
                                    Cria registro no DB
                                    Pede URL assinada ao B2
                                            │
[App Desktop] ◄── { uploadUrl, authToken } ─┘
      │
      └──── arquivo direto ──────────────────► [B2]
                                                 │
[App Desktop] ──"concluí, confirma pra mim"──► [API Go]
                                                    │
                                            Verifica no B2 se o
                                            arquivo existe mesmo
                                            Atualiza DB: status=done
                                            Dispara email de confirmação
```

## 📤 Fluxo de Upload Direto ao B2 Detalhado

### Upload de arquivo único (< 5GB)

```
POST /api/v1/backups/presign
Body: { cnpj, filename, size_bytes, checksum_sha256 }

API faz:
  1. Verifica se usuário tem permissão para esse CNPJ
  2. Chama B2: b2_get_upload_url (gera URL temporária de upload)
  3. Cria registro no DB: status = 'authorized'
  4. Retorna ao app: { backup_id, upload_url, upload_auth_token, expires_at }

App faz:
  PUT {upload_url}
  Headers: Authorization: {upload_auth_token}
           X-Bz-File-Name: {filename}
           X-Bz-Content-Sha1: {checksum}
  Body: arquivo binário diretamente ao B2

POST /api/v1/backups/{backup_id}/confirm
Body: { b2_file_id, b2_file_name, checksum_sha256 }

API faz:
  1. Chama B2: b2_get_file_info para verificar que o arquivo existe
  2. Confere checksum recebido vs o que o B2 reporta
  3. Atualiza DB: status = 'done', completed_at = NOW()
  4. Dispara evento BackupCompleted → email
```

### Upload multipart (> 5GB — Large File API do B2)

```
POST /api/v1/backups/multipart/initiate
Body: { cnpj, filename, size_bytes }
→ API cria registro no DB + chama b2_start_large_file
← { backup_id, b2_file_id }

POST /api/v1/backups/{backup_id}/multipart/part-url
Body: { part_number }
→ API chama b2_get_upload_part_url
← { part_upload_url, part_auth_token }  (cada parte tem URL própria)

[App envia parte diretamente ao B2 usando part_upload_url]

POST /api/v1/backups/{backup_id}/multipart/complete
Body: { parts: [{ part_number, sha1 }] }
→ API chama b2_finish_large_file
→ Verifica integridade
→ Atualiza DB: status = 'done'
← { backup_id, filename, size_bytes, completed_at }
```

### Estados do upload no banco

```sql
ALTER TYPE upload_status ADD VALUE 'authorized' BEFORE 'initiated';
-- 'authorized': API autorizou, app ainda não fez upload
-- 'initiated': Upload iniciado, aguardando partes (multipart)
-- 'uploading': Primeira parte recebida pelo B2
-- 'completing': app enviou /confirm, aguardando verificação
-- 'done': verificado e confirmado
-- 'failed': erro
-- 'aborted': cancelado
```

---

## 👤 Auto-cadastro (sem admin)

### Fluxo de registro público

```
POST /api/v1/auth/register   ← rota PÚBLICA (sem JWT)
Body: {
  name, email, password,
  cnpj,            ← CNPJ da empresa vinculado ao cadastro
  company_name,
  contact_phone
}
```

### O que acontece internamente

1. Valida formato do CNPJ (pkg/cnpj)
2. Verifica se CNPJ já existe → 409 Conflict
3. Verifica se email já existe → 409 Conflict
4. Valida força da senha (min 8 chars, 1 maiúscula, 1 número, 1 especial)
5. Cria Company + User na mesma transação Postgres (tudo ou nada)
6. Role padrão: 'owner' do CNPJ (pode ver/editar apenas o próprio CNPJ)
7. Envia email de boas-vindas + confirmação de email
8. Retorna 201 Created + { access_token, refresh_token } (já loga automaticamente)

### Roles e o que cada uma pode fazer

```
owner      → CRUD completo apenas do próprio CNPJ + seus usuários
operator   → Upload de backups + leitura (sem delete, sem configuração)
viewer     → Apenas leitura (relatórios, status)
admin      → Acesso total (suporte / plataforma) — criado apenas via CLI/seed
```

### Registro de operador adicional (feito pelo owner)

```
POST /api/v1/companies/{cnpj}/users        ← protegido: só owner do CNPJ
Body: { name, email, role: "operator"|"viewer" }

→ Cria usuário com senha temporária aleatória
→ Envia email com link de definição de senha (token com TTL 48h)
```

---

### Camadas adicionais de proteção

```go
// 1. Value Object CNPJ — rejeita qualquer coisa que não seja 14 dígitos
type CNPJ string
func NewCNPJ(raw string) (CNPJ, error) {
    digits := regexp.MustCompile(`\D`).ReplaceAllString(raw, "")
    if len(digits) != 14 { return "", ErrInvalidCNPJ }
    if !validateCNPJDigits(digits) { return "", ErrInvalidCNPJ }
    return CNPJ(digits), nil
}

// 2. UUID para IDs — impossível de manipular
// Todos os IDs são UUID v4, nunca inteiros sequenciais

// 3. Validação de entrada no DTO antes de qualquer lógica
type InitiateBackupRequest struct {
    CNPJ     string `json:"cnpj"     validate:"required,len=14,numeric"`
    Filename string `json:"filename" validate:"required,min=1,max=255,filepath"`
    SizeBytes int64 `json:"size_bytes" validate:"required,min=1,max=107374182400"`
}
```

### Middleware de sanitização de input

```go
func Sanitize(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Limita tamanho do body (evita memory bomb)
        r.Body = http.MaxBytesReader(w, r.Body, 1*1024*1024) // 1MB para JSON

        // Rejeita Content-Type incomum em rotas que esperam JSON
        if r.Method != "GET" && !strings.HasPrefix(r.Header.Get("Content-Type"), "application/json") {
            http.Error(w, "unsupported content type", http.StatusUnsupportedMediaType)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

---

## 🚫 Proteção XSS

```go
func SecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        h := w.Header()
        h.Set("Content-Type", "application/json; charset=utf-8")
        h.Set("Cache-Control", "no-store")
        h.Set("X-Content-Type-Options", "nosniff")
        h.Set("X-XSS-Protection", "1; mode=block")
        h.Set("X-Frame-Options", "DENY")
        h.Set("Referrer-Policy", "no-referrer")
        h.Set("Content-Security-Policy", "default-src 'none'")
        h.Del("X-Powered-By")
        next.ServeHTTP(w, r)
    })
}
```

### Sanitização de strings que vão para o banco

```go
import "github.com/microcosm-cc/bluemonday"

var strictPolicy = bluemonday.StrictPolicy() // remove TODO HTML

func StripHTML(input string) string {
    return strictPolicy.Sanitize(strings.TrimSpace(input))
}
// <script>alert(1)</script> vira string vazia
```

---

## 🔑 Autorização por Recurso (Prevenção de IDOR)

```
✅ Proteção implementada em TODO endpoint:
GET /api/v1/backups/abc-123-uuid
→ Valida JWT → extrai { user_id, company_id, role }
→ Busca backup pelo ID
→ Verifica: backup.company_id == claims.company_id
→ Se não bater → 404 (não 403! não confirma que o recurso existe)
```

### Implementação: AuthorizationChecker

```go
type Checker struct {
    backupOwnership   ResourceOwnership
    scheduleOwnership ResourceOwnership
}

func (c *Checker) AssertBackupAccess(ctx context.Context, backupID uuid.UUID) error {
    claims := auth.ClaimsFromContext(ctx)
    if claims.Role == RoleAdmin { return nil }
    ok, err := c.backupOwnership.BelongsToCompany(ctx, backupID, claims.CompanyID)
    if err != nil { return err }
    if !ok { return apperrors.ErrNotFound } // 404
    return nil
}
```

### Tabela completa de permissões por endpoint

```
Endpoint                              owner   operator  viewer   admin
─────────────────────────────────────────────────────────────────────
POST /auth/register                   ✅pub   ✅pub     ✅pub    ✅pub
POST /auth/login                      ✅pub   ✅pub     ✅pub    ✅pub
GET  /companies/{cnpj}                ✅own   ✅own     ✅own    ✅all
PUT  /companies/{cnpj}                ✅own   ❌        ❌       ✅all
DELETE /companies/{cnpj}              ✅own   ❌        ❌       ✅all
POST /companies/{cnpj}/users          ✅own   ❌        ❌       ✅all
GET  /schedules/{cnpj}                ✅own   ✅own     ✅own    ✅all
PUT  /schedules/{cnpj}                ✅own   ❌        ❌       ✅all
POST /backups/presign                 ✅own   ✅own     ❌       ✅all
POST /backups/{id}/confirm            ✅own   ✅own     ❌       ✅all
GET  /backups/{id}/status             ✅own   ✅own     ✅own    ✅all
GET  /backups/{id}/download           ✅own   ✅own     ✅own    ✅all
GET  /schedules/overdue               ❌      ❌        ❌       ✅all
```

---

## 🔄 Controle de Sessões e Refresh Token Rotation

### O que o projeto implementa

1. Access Token → JWT RS256, TTL 15 minutos, stateless
2. Refresh Token → opaque token (UUID aleatório), TTL 7 dias, stateful no DB
3. Sessões → tabela no Postgres (uma linha por dispositivo logado)
4. Blacklist → Redis para tokens invalidados antes do vencimento

### Tabela de sessões

```sql
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    refresh_token   TEXT NOT NULL UNIQUE,          -- hash bcrypt do token
    device_info     TEXT,                           -- "Windows 11 / BackupApp 2.1"
    ip_address      INET,
    last_used_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Fluxo de sessão com Refresh Token Rotation

```
Login:
  → Gera access_token (JWT, 15min)
  → Gera refresh_token (UUID aleatório, 64 bytes)
  → Salva hash(refresh_token) na tabela sessions
  → Retorna os dois ao app

Uso normal:
  → App usa access_token em cada request
  → Quando access_token expira (401):
      POST /auth/refresh { refresh_token }
      → API busca session pelo hash do refresh_token
      → Verifica se não está revogado, não expirou
      → ROTATION: invalida o refresh_token antigo
      → Gera novo par (access + refresh)
      → Salva nova session, retorna novos tokens
      → (se alguém tentar reusar o refresh antigo → detecta comprometimento)

Logout:
  → Pega jti do access_token → coloca no Redis com TTL até expirar
  → Marca session como revoked_at = NOW() no Postgres
  → App deleta tokens do storage local

"Logout de todos os dispositivos":
  → Revoga TODAS as sessions do user_id
  → Coloca todos access tokens ativos na blacklist
```

### Token Reuse Detection

```go
func (uc *RefreshUseCase) Execute(ctx context.Context, rawToken string) (*TokenPair, error) {
    tokenHash := crypto.Hash(rawToken)
    session, err := uc.sessionRepo.FindByTokenHash(ctx, tokenHash)
    // ...
    if session.RevokedAt != nil {
        // Token já rotacionado → possível roubo → revoga TUDO do usuário
        uc.sessionRepo.RevokeAllForUser(ctx, session.UserID)
        uc.logger.Warn().Msg("refresh token reuse detected — all sessions revoked")
        return nil, apperrors.ErrUnauthorized
    }
    // Rotation: invalida o atual, cria um novo
    return uc.rotateSession(ctx, session)
}
```

---

## 📧 Domain Events

| Evento | Template | Gatilho |
|--------|----------|---------|
| Novo CNPJ cadastrado | `company_added.html` | `company/register.go` |
| Confirmação de configuração | `config_confirm.html` | `schedule/upsert.go` |
| Backup concluído | `backup_done.html` | Evento `BackupCompleted` |
| Backup falhou | `backup_failed.html` | Evento `BackupFailed` |
| Backup atrasado | `backup_overdue.html` | Job `backup_job.go` |
| Boas-vindas | `welcome.html` | `auth/register.go` |

---

## 📅 Sistema de Agendamento e Verificação de Overdue

### Como funciona

O **app desktop envia** o agendamento para a API. A API **não agenda backups**, ela apenas:
1. Armazena o agendamento (dias da semana + horário + timezone)
2. Verifica periodicamente se o backup foi feito
3. Dispara alertas se não foi feito no prazo

### Job de verificação (roda a cada 5 minutos)

```go
func (uc *CheckOverdueUseCase) isBackupOverdue(company Company, schedule Schedule) OverdueStatus {
    expectedWindow := schedule.ExpectedWindowForToday()
    if expectedWindow == nil {
        return OverdueStatusNotScheduled
    }
    lastBackup := company.LastBackup()

    // Backup ainda dentro da janela → aguardar
    if time.Now().Before(expectedWindow.DeadlineAt) {
        return OverdueStatusPending
    }
    // Backup em andamento → NÃO é overdue, está processando
    if lastBackup != nil && lastBackup.IsInProgress() && lastBackup.StartedAt.After(expectedWindow.StartsAt) {
        return OverdueStatusProcessing
    }
    // Backup concluído dentro da janela → OK
    if lastBackup != nil && lastBackup.IsSuccessful() && lastBackup.CompletedAt.After(expectedWindow.StartsAt) {
        return OverdueStatusDone
    }
    // Passou do prazo sem backup válido → OVERDUE
    return OverdueStatusOverdue
}
```

---

## 🔐 Diagrama de Segurança em Camadas

```
Request HTTP
     │
     ▼
[1] SecurityHeaders middleware
     CSP, X-Content-Type-Options, etc.
     │
     ▼
[2] RequestID middleware
     X-Request-ID único (rastreabilidade)
     │
     ▼
[3] Logger middleware
     método, path, IP, request_id
     │
     ▼
[4] RateLimit middleware (Redis)
     100 req/min por IP (global)
     20 req/min por user (autenticado)
     5 req/min para /auth/login (anti brute-force)
     │
     ▼
[5] Recover middleware
     Captura panic, loga stack trace, retorna 500
     │
     ▼
[6] Sanitize middleware
     Limita body a 1MB, valida Content-Type
     │
     ▼
[7] Auth middleware (rotas protegidas)
     Valida JWT RS256
     Verifica blacklist no Redis
     Injeta claims no context
     │
     ▼
[8] Handler
     Valida DTO (go-playground/validator)
     Chama use case
     │
     ▼
[9] Use Case
     Chama authz.AssertXxxAccess() ← verifica ownership do recurso
     Executa lógica de negócio
     │
     ▼
[10] Repository (pgx)
     Queries parametrizadas — SQL Injection impossível
     Conexão TLS com Postgres
```

---

## 📡 API Endpoints

```
AUTH — Auto-cadastro público + gestão de sessões
─────────────────────────────────────────────────
POST   /api/v1/auth/register            ← SEM JWT, qualquer pessoa cadastra
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
POST   /api/v1/auth/logout-all          ← revoga todas as sessões do usuário
GET    /api/v1/auth/sessions            ← lista dispositivos logados
DELETE /api/v1/auth/sessions/{id}       ← revoga sessão específica (logout remoto)
POST   /api/v1/auth/forgot-password     ← envia email com link de reset
POST   /api/v1/auth/reset-password      ← consome token do email, define nova senha
POST   /api/v1/auth/confirm-email       ← confirma email pelo token recebido

COMPANIES — Gerenciamento do CNPJ
──────────────────────────────────
GET    /api/v1/companies               Lista (paginado)
POST   /api/v1/companies               Cadastra novo CNPJ (admin only após auto-cadastro)
GET    /api/v1/companies/{cnpj}        Detalhes
PUT    /api/v1/companies/{cnpj}        Atualiza
DELETE /api/v1/companies/{cnpj}        Desativa

SCHEDULES — Agendamento de backups
───────────────────────────────────
GET    /api/v1/schedules/{cnpj}        Agendamento atual
PUT    /api/v1/schedules/{cnpj}        Cria/atualiza agendamento
DELETE /api/v1/schedules/{cnpj}        Remove agendamento
GET    /api/v1/schedules/overdue       Lista CNPJs com backup atrasado (admin only)

BACKUPS — Upload direto ao B2
───────────────────────────────
POST   /api/v1/backups/presign
       Body: { cnpj, filename, size_bytes, checksum_sha256 }
       → Retorna { backup_id, upload_url, upload_auth_token, expires_in_seconds }

POST   /api/v1/backups/{id}/confirm
       Body: { b2_file_id, checksum_sha256 }
       → API verifica no B2, atualiza status, dispara email

POST   /api/v1/backups/multipart/initiate
       Body: { cnpj, filename, size_bytes, total_parts }
       → { backup_id, b2_file_id }

POST   /api/v1/backups/{id}/multipart/part-url
       Body: { part_number }
       → { part_upload_url, part_auth_token }

POST   /api/v1/backups/{id}/multipart/complete
       Body: { parts: [{ number, sha1 }] }
       → Finaliza no B2, confirma, atualiza DB

POST   /api/v1/backups/{id}/multipart/abort

GET    /api/v1/backups?cnpj=&status=&from=&to=&page=
GET    /api/v1/backups/{id}/status
GET    /api/v1/backups/{id}/download    → URL pré-assinada de download (TTL 1h)

HEALTH
GET    /health                         { status, db, redis, b2 }
GET    /metrics                        Prometheus metrics
```

---

## 🗄️ Schema PostgreSQL Completo (com Sessões)

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Usuários
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    email           TEXT NOT NULL UNIQUE,
    password_hash   TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'owner' CHECK (role IN ('admin','owner','operator','viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Empresas / CNPJs
CREATE TABLE companies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cnpj            CHAR(14) NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    email           TEXT NOT NULL,
    contact_name    TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Sessões (refresh tokens)
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    refresh_token   TEXT NOT NULL UNIQUE,
    device_info     TEXT,
    ip_address      INET,
    last_used_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_refresh_token ON sessions(refresh_token);

-- Agendamentos
CREATE TABLE schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL UNIQUE REFERENCES companies(id) ON DELETE CASCADE,
    days_of_week    SMALLINT[] NOT NULL,
    time_of_day     TIME NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'America/Sao_Paulo',
    tolerance_min   INT NOT NULL DEFAULT 60,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_backup_at  TIMESTAMPTZ,
    alert_sent_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Backups (com status 'authorized')
CREATE TYPE upload_status AS ENUM (
    'authorized', 'initiated', 'uploading', 'completing', 'done', 'failed', 'aborted'
);

CREATE TABLE backups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(id),
    b2_file_id      TEXT,
    b2_bucket       TEXT,
    filename        TEXT NOT NULL,
    original_path   TEXT,
    size_bytes      BIGINT,
    parts_count     INT DEFAULT 1,
    checksum_sha256 TEXT,
    status          upload_status NOT NULL DEFAULT 'authorized',
    error_message   TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '24 hours',
    created_by      UUID REFERENCES users(id)
);

-- Índices
CREATE INDEX idx_backups_company_id ON backups(company_id);
CREATE INDEX idx_backups_status ON backups(status);
CREATE INDEX idx_backups_started_at ON backups(started_at DESC);
CREATE INDEX idx_schedules_active ON schedules(is_active) WHERE is_active = true;

-- Audit log (imutável)
CREATE TABLE audit_logs (
    id          BIGSERIAL PRIMARY KEY,
    user_id     UUID REFERENCES users(id),
    action      TEXT NOT NULL,
    entity      TEXT NOT NULL,
    entity_id   UUID,
    details     JSONB,
    ip_address  INET,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## ⚡ Redis — O que Cachear

| Chave | TTL | Uso |
|-------|-----|-----|
| `jwt:blacklist:{jti}` | 7 dias | Tokens invalidados no logout |
| `rate:ip:{ip}` | 1 min | Rate limiting por IP |
| `rate:user:{user_id}` | 1 min | Rate limiting por usuário autenticado |
| `company:{cnpj}` | 5 min | Cache de dados da empresa (evita DB) |
| `schedule:{company_id}` | 2 min | Cache de agendamento |
| `backup:status:{upload_id}` | 10 min | Status de upload em andamento |
| `overdue:alert:{company_id}:{date}` | 24h | Evita alertas duplicados |

---

## 🔌 Principais Dependências Go

```go
// HTTP
github.com/go-chi/chi/v5           // Router leve, padrão net/http

// PostgreSQL
github.com/jackc/pgx/v5            // Driver PG rápido e seguro (queries parametrizadas)
github.com/pressly/goose/v3        // Migrations versionadas

// Redis
github.com/redis/go-redis/v9       // Cliente oficial Redis

// Auth
github.com/golang-jwt/jwt/v5       // JWT com RS256
golang.org/x/crypto                // bcrypt

// Validação e Sanitização
github.com/go-playground/validator/v10
github.com/microcosm-cc/bluemonday // Sanitização HTML

// Logging
github.com/rs/zerolog              // JSON structured logging

// Config
github.com/spf13/viper             // Lê env vars + arquivos

// Email
github.com/wneessen/go-mail        // SMTP moderno

// B2
github.com/Backblaze/blazer        // SDK oficial B2

// Scheduler interno
github.com/go-co-op/gocron/v2     // Scheduler goroutine-safe

// UUID
github.com/google/uuid

// Testes
github.com/stretchr/testify        // Assert + mock
github.com/testcontainers/testcontainers-go // Testes com infra real em Docker
```

---

## 🧪 Estratégia de Testes

```
internal/
├── domain/backup/
│   └── entity_test.go             # Testes unitários puros (sem mock, sem DB)
├── application/usecase/backup/
│   └── presign_test.go            # Testa use case com mocks das interfaces
└── infrastructure/persistence/
    └── postgres/
        └── backup_repo_test.go    # Integration test com testcontainers (Postgres real)
```

### Regra: pirâmide de testes
- **70% unitários** — domínio e use cases com mocks
- **20% integração** — repositórios com banco real (testcontainers)
- **10% E2E** — fluxos críticos completos via HTTP

---

## 🐳 Docker Multi-Stage Build

```dockerfile
# Stage 1: Build
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /backup-api ./cmd/api

# Stage 2: Runtime mínimo (~10MB)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /backup-api /backup-api
COPY --from=builder /app/internal/mailer/templates /templates
EXPOSE 8080
ENTRYPOINT ["/backup-api"]
```

---

## 📊 Observabilidade

```go
// 3 pilares: Logs + Métricas + Traces

// 1. Logs estruturados (zerolog)
log.Info().
    Str("request_id", requestID).
    Str("cnpj", cnpj).
    Str("upload_id", uploadID).
    Int64("size_bytes", size).
    Msg("backup upload initiated")

// 2. Métricas Prometheus (expostas em /metrics)
backup_uploads_total{status="done|failed|aborted"}
backup_upload_duration_seconds
schedule_overdue_total

// 3. Traces distribuídos (OpenTelemetry)
```

---

## ⚙️ Variáveis de Ambiente Necessárias

```env
# Servidor
APP_PORT=8080
APP_ENV=production
APP_BASE_URL=https://api.seudominio.com

# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_NAME=backupcloud
DB_USER=backupcloud_api
DB_PASSWORD=senha_forte_aqui
DB_SSL_MODE=require        # nunca 'disable' em produção
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5

# Redis
REDIS_URL=redis://:senha@localhost:6379/0
REDIS_TLS=true

# JWT (RS256)
JWT_PRIVATE_KEY_PATH=/secrets/jwt_private.pem
JWT_PUBLIC_KEY_PATH=/secrets/jwt_public.pem
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=168h       # 7 dias

# Backblaze B2
B2_KEY_ID=xxxxxxxxxxxx
B2_APPLICATION_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
B2_BUCKET_ID=xxxxxxxxxxxxxxxxxxxx
B2_BUCKET_NAME=backups-prod
B2_DOWNLOAD_URL_TTL=3600

# SMTP
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASSWORD=SG.xxxxxxxxxxxx
SMTP_FROM=noreply@seudominio.com
SMTP_FROM_NAME=BackupCloud

# Segurança
BCRYPT_COST=12
RATE_LIMIT_GLOBAL=100
RATE_LIMIT_AUTH=5
PASSWORD_RESET_TTL=2h
EMAIL_CONFIRM_TTL=48h
```
