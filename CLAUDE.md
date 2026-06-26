# CLAUDE.md — Juridico Watcher

> **Este arquivo é a spec viva do projeto.** Todo modelo de IA que tocar neste repositório DEVE lê-lo antes de qualquer ação. Ele define o que o projeto é, o que não é, o que já foi decidido, o que não pode mudar, e o que está pendente. Não é documentação histórica — é o contrato de desenvolvimento.
>
> Inspirado na metodologia do artigo de Fabio Akita: [Do Zero à Pós-Produção em 1 Semana](https://akitaonrails.com/2026/02/20/do-zero-a-pos-producao-em-1-semana-como-usar-ia-em-projetos-de-verdade-bastidores-do-the-m-akita-chronicles/)

---

## 1. O QUE É ESTE PROJETO

**Juridico Watcher** é um programa desktop leve, multiplataforma (Windows + Linux), escrito em Go, que monitora pastas do sistema de arquivos e processa automaticamente documentos PDF que possuam camada de texto. O processamento é baseado em **regras configuráveis pelo usuário**: cada regra define quais pastas observar, quais variáveis extrair do texto do documento, sob quais condições executar, e quais ações tomar (criar pasta, mover, renomear) usando as variáveis extraídas.

**Cenário de uso principal:** documentos jurídicos/notariais como procurações são digitalizados para PDF. O programa identifica automaticamente dados do documento (ex: nome do outorgante) e organiza os arquivos em pastas com nomes padronizados, sem intervenção humana.

**O que este programa NÃO é:**
- Não é um sistema de OCR (não processa PDFs sem camada de texto)
- Não é um sistema de gestão documental completo (GED)
- Não é um serviço web ou API
- Não é um sistema multi-usuário ou multi-tenant
- Não tem IA/ML embutida para classificação automática (é baseado em regras simples)

---

## 2. STACK TECNOLÓGICA — DECISÕES FINAIS E IMUTÁVEIS

Estas decisões foram tomadas deliberadamente. **Não proponha alternativas sem razão técnica crítica.**

| Componente | Tecnologia | Motivo da escolha |
|---|---|---|
| Linguagem | Go 1.21+ | Binário único, cross-compile, stdlib robusta, performance |
| Module path | `github.com/F3rreir4L19/juridico-watcher` | Já usado em go.mod e todos os imports |
| GUI | `fyne.io/fyne/v2` | Nativa, binário único, sem deps em runtime, OpenGL, multiplataforma |
| Banco de dados | SQLite via `modernc.org/sqlite` | Driver Go puro, **SEM CGO**, cross-compile funciona sem toolchain C |
| Filesystem watcher | `github.com/fsnotify/fsnotify` | Padrão Go, suporta inotify (Linux) e ReadDirectoryChangesW (Windows) |
| Extração de texto PDF | `github.com/ledongthuc/pdf` | Go puro, para PDFs com camada de texto; **sem OCR** |
| Geração de PDFs de teste | `github.com/jung-kurt/gofpdf` | Usado APENAS em test helpers, não em produção |
| Testes | stdlib `testing` + `github.com/stretchr/testify` | Padrão consolidado |
| Migrations | Sistema interno via `embed.FS` | Sem dependência externa de ferramentas de migration |
| Logs | `log/slog` (stdlib Go 1.21+) | Sem dependência externa |

**Atenção Fyne + CGO:** Fyne requer CGO para compilar a GUI. A build cross-platform para Linux/Windows precisa de toolchain C disponível. O SQLite usa `modernc.org/sqlite` (CGO-free) para não criar conflito de toolchain duplo. Estas duas decisões são dependentes.

---

## 3. ESTRUTURA DE PASTAS — DEFINITIVA

```
juridico-watcher/
├── CLAUDE.md                            # Este arquivo — leia antes de qualquer coisa
├── README.md                            # Porta de entrada para humanos
├── Makefile                             # Comandos de build/test/run
├── go.mod                               # github.com/F3rreir4L19/juridico-watcher
├── go.sum
├── .gitignore
├── cmd/
│   └── juridico-watcher/
│       └── main.go                      # Entry point — mínimo, apenas inicializa app Fyne
├── internal/
│   ├── domain/                          # Tipos puros, SEM dependências externas
│   │   ├── watch.go
│   │   ├── rule.go
│   │   ├── document.go
│   │   └── errors.go
│   ├── storage/
│   │   ├── sqlite.go                    # Open(), pragma WAL + FK
│   │   ├── migrations.go
│   │   ├── migrations/
│   │   │   └── 001_initial.sql
│   │   ├── watch_repo.go
│   │   ├── rule_repo.go
│   │   └── processed_repo.go
│   ├── pdf/
│   │   └── extractor.go                 # ExtractText(path) → string
│   ├── engine/                          # Motor de regras — sem IO de filesystem
│   │   ├── extractor.go
│   │   ├── evaluator.go
│   │   ├── interpolator.go
│   │   ├── actions.go
│   │   └── pipeline.go
│   ├── watcher/
│   │   ├── watcher.go
│   │   ├── stabilizer.go
│   │   └── scanner.go
│   ├── service/
│   │   ├── monitor_service.go           # StartMonitoring, StopAll
│   │   ├── watch_service.go
│   │   ├── rule_service.go
│   │   └── scan_service.go
│   └── ui/                              # GUI Fyne — Sprint 8-10
│       ├── app.go
│       ├── tab_watches.go
│       ├── tab_rules.go
│       ├── dialog_watch.go
│       ├── dialog_rule.go
│       └── components/
├── test/
│   ├── testhelpers/
│   │   ├── helpers.go                   # TempDB, AssertFileExists, etc.
│   │   └── pdfgen.go                    # WritePDF, WriteEmptyPDF, WriteCorruptPDF
│   ├── fixtures/
│   │   └── procuracao_sample.pdf        # PDF de exemplo para E2E
│   └── integration/
│       ├── e2e_test.go
│       ├── rule_lifecycle_test.go
│       ├── watch_lifecycle_test.go
│       └── watch_runtime_test.go
└── bin/                                 # Binários compilados (gitignored)
```

**Regra de dependência entre pacotes (nunca viole):**
```
domain ← storage ← service ← ui
domain ← pdf     ← engine  ← service
domain ← watcher ← service
```
- `domain` não importa nada do projeto
- `engine` não faz IO de filesystem (isso é responsabilidade de `service`)
- `ui` nunca acessa `storage` ou `engine` diretamente — sempre via `service`

---

## 4. MODELO DE DADOS

### Watch
```go
type Watch struct {
    ID        int64
    Name      string    // único, não vazio
    Path      string    // absoluto, existe no filesystem
    Active    bool
    Recursive bool      // monitora subpastas
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

### Rule
```go
type Rule struct {
    ID          int64
    Name        string       // único, não vazio
    Priority    int          // menor = executa primeiro; padrão 0
    Active      bool
    WatchIDs    []int64      // N:N com Watch via tabela rule_watches
    Extractions []Extraction
    Conditions  []Condition
    Actions     []Action
}
```

### Extraction
```go
type Extraction struct {
    ID           int64
    RuleID       int64
    VariableName string  // identificador usado em {var}
    StartDelim   string  // delimitador inicial (vazio = início do texto)
    EndDelim     string  // delimitador final (vazio = fim do texto)
    Order        int
}
```

### Condition
```go
type Condition struct {
    ID           int64
    RuleID       int64
    VariableName string
    Operator     ConditionOperator  // "eq", "neq", "contains", "not_contains"
    Value        string
    Order        int
}
```

### Action
```go
type Action struct {
    ID     int64
    RuleID int64
    Type   ActionType  // "create_folder", "move", "rename"
    Target string      // pode conter {variavel}
    Order  int
}
```

### ProcessedDoc
```go
type ProcessedDoc struct {
    ID           int64
    FileHash     string            // SHA-256 do conteúdo do arquivo
    OriginalPath string
    RuleID       int64
    Status       ProcessingStatus  // "success","no_match","failed","skipped_moved","no_text"
    ErrorMsg     string
    ProcessedAt  time.Time
}
// UNIQUE(file_hash, rule_id) — mesmo arquivo não é reprocessado pela mesma regra
```

---

## 5. REGRAS DE NEGÓCIO — NÃO NEGOCIÁVEIS

Estas regras foram decididas e implementadas. Qualquer alteração exige discussão explícita.

### RN-01: Aplicabilidade vs Match são separados
A regra aplica extração a **todo PDF** nas pastas listadas. As ações só executam se as **condições** derem match. Regra sem condições executa sempre após extração.

### RN-02: Múltiplas regras em cascata
Todas as regras ativas aplicáveis a uma pasta executam, na ordem de `priority` (menor = primeiro). Se uma regra **moveu** o arquivo (action `move`), as regras seguintes são puladas — o arquivo original não existe mais. Status registrado: `skipped_moved`.

### RN-03: Política de colisão de nome
Ao criar pasta ou mover/renomear arquivo, se o destino já existe, usa sufixo numérico: `arquivo (2).pdf`, `arquivo (3).pdf`. Nunca sobrescreve silenciosamente.

### RN-04: Estabilização antes de processar
No modo watch contínuo, ao detectar `Create` no fsnotify, aguarda o tamanho do arquivo parar de mudar por **2 ciclos consecutivos de 500ms**. Timeout máximo de **30 segundos**. Evita processar arquivo ainda sendo escrito.

### RN-05: PDF sem texto extraível
Marcado como `no_text`. O sistema **não tenta OCR**. Não é erro — é um status esperado.

### RN-06: Delimitadores case-insensitive, primeira ocorrência
Delimitadores são literais, sem regex (no v1). Busca case-insensitive. Usa a primeira ocorrência. Delimitador vazio: início ou fim do texto.

### RN-07: Condições combinadas por AND
Não há OR no v1. Todas as condições da regra devem ser verdadeiras para executar ações.

### RN-08: Deleção de pasta monitorada bloqueada
`WatchRepo.Delete` falha com `ErrWatchInUse` se há regras que referenciam o watch. Usuário deve editar/deletar as regras primeiro. Garantido via `ON DELETE RESTRICT` no SQLite + verificação explícita no repo.

### RN-09: Watch recursivo por padrão
`Recursive: true` por padrão ao criar novo watch. Usuário pode desativar via checkbox.

### RN-10: Interpolação de variável inexistente
Retorna string vazia + loga `slog.Warn`. Nunca deixa o placeholder literal `{variavel}` no nome do arquivo ou pasta.

### RN-11: Deduplicação por hash
Um arquivo com o mesmo SHA-256 não é reprocessado pela mesma regra duas vezes. O campo `UNIQUE(file_hash, rule_id)` em `processed_documents` garante isso no banco.

### RN-12: Sem system tray no MVP
`internal/ui` não implementa system tray. A janela fica aberta enquanto o monitoramento está ativo. System tray é feature de v2.

### RN-13: Condições case-insensitive
Comparações de condições (`eq`, `neq`, `contains`, `not_contains`) são case-insensitive, consistentes com a busca de delimitadores na RN-06. Implementação em `internal/engine/evaluator.go` faz `strings.ToLower` em ambos os lados antes de comparar.

### RN-14: Target vazio em ações é erro
Se uma ação (`create_folder`, `move`, `rename`) recebe target interpolado vazio ou só com espaços, retorna `ErrEmptyTarget`. O pipeline marca o documento como `StatusFailed` com a mensagem do erro, dando feedback explícito ao usuário em vez de comportamento silencioso indesejado (mover para a própria pasta, criar pasta com nome vazio, etc).
---

## 6. FLUXO DO MOTOR DE REGRAS (PIPELINE)

```
PDF detectado (scan, watch ou botão manual)
    │
    ▼
[Estabilizar tamanho] ← apenas no watch contínuo
    │
    ▼
[Calcular SHA-256]
    │
    ├─ já processado por todas regras? → SKIP
    │
    ▼
[ExtractText(path)] ← internal/pdf
    │
    ├─ sem texto? → registra no_text, STOP
    │
    ▼
[Listar regras ativas para esta pasta, ordenadas por priority]
    │
    ▼ para cada regra:
    │
    ├─ [Extrações] → mapa {variavel: valor}
    │
    ├─ [Avaliar condições] → AND de todas
    │       │
    │       ├─ false → registra no_match, próxima regra
    │       │
    │       └─ true → executa ações em sequência
    │               │
    │               └─ action move executou? → registra skipped_moved
    │                   próximas regras PULADAS
    │
    └─ registra resultado em processed_documents
```

---

## 7. PADRÕES DE CÓDIGO

### 7.1 Nomenclatura
- Arquivos de teste: `_test.go` ao lado do arquivo fonte, mesmo pacote com sufixo `_test` (ex: `package storage_test`)
- Nomes de teste: `Test<Tipo>_<Cenario>` em português (ex: `TestExtractor_DelimitadorVazio`)
- Erros de domínio: variáveis sentinela em `internal/domain/errors.go`, ex: `ErrNotFound`, `ErrDuplicateName`, `ErrWatchInUse`
- Funções de helper de teste: em `test/testhelpers/`, exportadas, recebem `t *testing.T`

### 7.2 Padrões de repositório Storage
```go
// Assinatura padrão
func (r *FooRepo) Create(foo *domain.Foo) error           // preenche foo.ID
func (r *FooRepo) GetByID(id int64) (*domain.Foo, error)  // retorna ErrNotFound
func (r *FooRepo) List() ([]*domain.Foo, error)
func (r *FooRepo) Update(foo *domain.Foo) error           // retorna ErrNotFound
func (r *FooRepo) Delete(id int64) error                  // retorna ErrNotFound ou ErrInUse
```

### 7.3 SQLite — Configuração obrigatória
```go
// Sempre abrir com estes pragmas
db, err := sql.Open("sqlite", path+"?_pragma=foreign_keys(1)&_pragma=journal_mode(WAL)")
```

### 7.4 Testes — Helpers obrigatórios
```go
// SEMPRE use TempDB(t) em testes de storage — nunca crie banco em path fixo
db := testhelpers.TempDB(t)

// SEMPRE use Eventually para eventos assíncronos — nunca time.Sleep fixo
require.Eventually(t, func() bool { ... }, 5*time.Second, 200*time.Millisecond)
```

### 7.4.1 — Limitações de fixtures PDF

PDFs gerados via `testhelpers.WritePDF` (gofpdf) **não preservam quebras de linha (`\n`)** de forma confiável quando extraídos por `ledongthuc/pdf`. O texto vira uma sequência contínua separada por espaços. Em testes, **não use `\n` como delimitador**. Use texto literal que aparece no documento, como `"Nome: "` e `" Tipo:"`.

Em produção isto não é problema: o usuário escolhe os delimitadores observando o PDF real, e PDFs reais (digitalizadores corporativos) preservam estrutura textual de forma diferente. Esta limitação é específica do par gofpdf+ledongthuc usado para gerar fixtures sintéticas.

### 7.5 Logs
```go
slog.Info("arquivo processado", "path", path, "rule", rule.Name, "status", status)
slog.Warn("variável não encontrada", "var", name, "rule", rule.Name)
slog.Error("falha ao mover arquivo", "err", err, "src", src, "dst", dst)
```

### 7.6 Proibições
- **Nunca** use `time.Sleep` fixo em testes — use `require.Eventually`
- **Nunca** acesse `storage` diretamente da `ui` — sempre via `service`
- **Nunca** faça IO de filesystem no `engine` — apenas strings/structs
- **Nunca** use CGO explícito além do que Fyne já requer
- **Nunca** importe `internal/ui` de qualquer outro pacote interno
- **Nunca** use `regexp` para delimitadores no v1 — apenas string literals

---

## 8. CONFIGURAÇÃO DO BANCO DE DADOS

### Schema (001_initial.sql)
```sql
CREATE TABLE watches (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    name       TEXT NOT NULL UNIQUE,
    path       TEXT NOT NULL,
    active     INTEGER NOT NULL DEFAULT 1,
    recursive  INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

CREATE TABLE rules (
    id       INTEGER PRIMARY KEY AUTOINCREMENT,
    name     TEXT NOT NULL UNIQUE,
    priority INTEGER NOT NULL DEFAULT 0,
    active   INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE rule_watches (
    rule_id  INTEGER NOT NULL REFERENCES rules(id)  ON DELETE CASCADE,
    watch_id INTEGER NOT NULL REFERENCES watches(id) ON DELETE RESTRICT,
    PRIMARY KEY (rule_id, watch_id)
);

CREATE TABLE extractions (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    rule_id       INTEGER NOT NULL REFERENCES rules(id) ON DELETE CASCADE,
    variable_name TEXT NOT NULL,
    start_delim   TEXT NOT NULL DEFAULT '',
    end_delim     TEXT NOT NULL DEFAULT '',
    "order"       INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE conditions (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    rule_id       INTEGER NOT NULL REFERENCES rules(id) ON DELETE CASCADE,
    variable_name TEXT NOT NULL,
    operator      TEXT NOT NULL,
    value         TEXT NOT NULL,
    "order"       INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE actions (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    rule_id INTEGER NOT NULL REFERENCES rules(id) ON DELETE CASCADE,
    type    TEXT NOT NULL,
    target  TEXT NOT NULL,
    "order" INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE processed_documents (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    file_hash     TEXT NOT NULL,
    original_path TEXT NOT NULL,
    rule_id       INTEGER NOT NULL REFERENCES rules(id) ON DELETE CASCADE,
    status        TEXT NOT NULL,
    error_msg     TEXT NOT NULL DEFAULT '',
    processed_at  TEXT NOT NULL,
    UNIQUE(file_hash, rule_id)
);
```

---

## 9. MAKEFILE — COMANDOS DISPONÍVEIS

```makefile
make test              # test-unit + test-integration
make test-unit         # go test ./internal/...
make test-integration  # go test ./test/integration/... (timeout 60s)
make test-verbose      # todos com -v
make build             # binário para o OS atual em bin/
make build-windows     # cross-compile para Windows (requer toolchain C para Fyne)
make build-linux       # cross-compile para Linux
make run               # go run ./cmd/juridico-watcher
make fmt               # go fmt ./...
make vet               # go vet ./...
make clean             # remove bin/
```

---

## 10. ESTADO ATUAL DO PROJETO

### ✅ Concluído

| Sprint | Descrição | Status |
|--------|-----------|--------|
| 0 | Estrutura de pastas, deps, Makefile, test helpers (TempDB, WritePDF, helpers) | ✅ Completo |
| 1 | Domain (watch, rule, document, errors) + Storage completo (SQLite, migrations, repos CRUD) | ✅ Completo |
| 2 | Extração de texto PDF (`internal/pdf`) + Extractor de variáveis por delimitadores (`internal/engine/extractor.go`) | ✅ Completo |
| 3 | Evaluator (condições AND) + Interpolator ({var}) + Actions (create_folder, move, rename) | ✅ Completo |
| 4 | Pipeline completo que orquestra engine (`internal/engine/pipeline.go`) | ✅ Completo |
| 5 | Watcher fsnotify + Estabilizador de tamanho + Scanner inicial de pasta | ✅ Completo |
| 6 | Camada de Services: MonitorService, WatchService, RuleService, ScanService + testes | ✅ Completo |
| 6.5 | Correções pós-revisão: case-insensitive em condições (RN-13), dedup por hash em runtime, interpolator single-pass, fix recursive watch em subpastas novas, mover monitor_service_test para integração, limpeza de dead code | ✅ Completo |
| 7 | Testes de integração E2E: watch_lifecycle, rule_lifecycle, watch_runtime, scan_service, e2e (cenário procuração completo) | ✅ Completo |
| 8 | UI Fyne — janela principal, aba Monitorar funcional (CRUD com seleção, validação, picker de pasta), MonitorService no startup, placeholder honesto na aba Regras | ✅ Completo |
| 9 | UI Fyne — aba Regras completa: dialog modal único com acordeon de extrações/condições/ações, sub-dialogs para cada item composto, validação em camadas (UI + service), avisos quando variável removida está em uso | ✅ Completo |
| 10 | UI Fyne — finalização: barra de topo com "Atualizar tudo", botão "Atualizar regra" por linha, aba Histórico com filtros, badge "(N falhas)" no título, README com instruções de build/run | ✅ Completo |

### 🔲 Pendente

| Sprint | Descrição | Prioridade |
|--------|-----------|------------|


### Evidência de estado (último `make test-unit` confirmado):
```
ok  github.com/F3rreir4L19/juridico-watcher/internal/engine
ok  github.com/F3rreir4L19/juridico-watcher/internal/pdf
ok  github.com/F3rreir4L19/juridico-watcher/internal/service
ok  github.com/F3rreir4L19/juridico-watcher/internal/storage
ok  github.com/F3rreir4L19/juridico-watcher/internal/watcher
```

---

## 11. O QUE É O MVP (PRODUTO MÍNIMO VIÁVEL)

O MVP está completo quando:

1. **O programa abre** sem crash em Windows e Linux
2. **Aba Monitorar:** usuário consegue adicionar, editar, ativar/desativar e remover pastas monitoradas
3. **Aba Regras:** usuário consegue criar regras com extrações, condições e ações; associar a pastas; definir prioridade
4. **Monitoramento ativo:** ao iniciar o programa, todas as pastas ativas são monitoradas automaticamente
5. **Processamento real:** um PDF com camada de texto colocado em pasta monitorada é extraído, organizado conforme a regra, e o resultado aparece no histórico
6. **Scan manual:** botão "Atualizar" reprocessa PDFs existentes na pasta
7. **Build funcional:** `make build` gera binário que funciona sem instalação adicional em Windows e Linux

O MVP **não precisa ter:**
- System tray (v2)
- Importação/exportação de regras (v2)
- Suporte a múltiplos usuários (fora do escopo)
- Suporte a outros formatos além de PDF (v2)
- OCR (fora do escopo do projeto)
- Logs persistidos em arquivo (v2)
- Auto-update (v2)

**Status:** MVP completo em 2026-05-09 ✅

---

## 12. VISÃO PÓS-MVP (v2)

### v2.1 — UX e Polimento
- **System tray:** programa roda em background na bandeja do sistema, janela opcional
- **Notificações nativas:** notificação de desktop quando arquivo é processado
- **Log persistido:** aba de histórico com filtros por data/status/regra
- **Importação/exportação de regras:** formato JSON para backup e compartilhamento

### v2.2 — Engine
- **Regex nos delimitadores:** usuário pode usar expressões regulares além de literais
- **Operador OR em condições:** combinações mais complexas de condições
- **Action: copiar** (além de mover — mantém original)
- **Action: renomear pasta**
- **Variáveis de sistema:** `{data_atual}`, `{ano}`, `{mes}`, `{dia}` disponíveis automaticamente

### v2.3 — Integração e Distribuição
- **Instalador Windows (NSIS ou Inno Setup)**
- **Pacote .deb/.rpm para Linux**
- **Auto-update integrado** (GitHub Releases)
- **Watch de rede** (pastas mapeadas via SMB/NFS — avaliar limitações do fsnotify)

### v2.4 — Suporte a outros formatos
- **DOCX/ODT** com extração de texto nativa
- **TXT/CSV** como documentos processáveis
- OCR via integração com Tesseract (modo opcional, dependência explícita)

---

## 13. DECISÕES DE ARQUITETURA REGISTRADAS

### D-01: modernc.org/sqlite em vez de mattn/go-sqlite3
**Razão:** `mattn/go-sqlite3` requer CGO. `modernc.org/sqlite` é Go puro. Como o Fyne já obriga CGO para a GUI, adicionar um segundo vetor de CGO para SQLite criaria complexidade de toolchain e potencial de conflito em cross-compile. A decisão foi usar CGO apenas onde obrigatório (Fyne) e evitá-lo onde possível (SQLite).

### D-02: Engine sem IO de filesystem
**Razão:** O `internal/engine` recebe texto extraído e retorna ações a executar, sem tocar em filesystem. Isso permite testes unitários determinísticos do motor de regras sem precisar criar arquivos temporários. O `internal/service` é responsável por orquestrar IO real.

### D-03: SHA-256 para deduplicação, não path
**Razão:** Um arquivo pode ser movido e reaparecer em outra pasta. O path muda, o conteúdo não. Usar hash garante que o mesmo documento não seja reprocessado pela mesma regra, independente de onde esteja.

### D-04: Migrations via embed.FS
**Razão:** Sem ferramenta externa (goose, migrate, etc.), sem binário extra, sem configuração adicional. O programa carrega as migrations do próprio binário. Adequado para um app desktop onde "gerenciar infraestrutura de banco" não é uma preocupação do usuário.

### D-05: Sem OR em condições no v1
**Razão:** AND é suficiente para o cenário principal e mantém a lógica simples e sem ambiguidade de precedência. OR foi explicitamente adiado para v2.

### D-06: Fyne como framework GUI
**Razão:** Avaliamos opções (webview, walk, Qt bindings). Fyne é o único que oferece binário único + visual aceitável + suporte a Windows e Linux sem dependências de runtime. O custo (CGO obrigatório) é aceitável dado o benefício.

### D-07: Lista com nome em destaque + subtítulo em cinza
**Razão:** Comunica hierarquia visual melhor que ícones simples (✓/✗) e é amigável para usuários não-técnicos. Implementado via widget custom em `internal/ui/components/list_item.go` (TwoLineItem). Reutilizado pela aba Regras na Sprint 9.

### D-08: Seleção explícita por clique
**Razão:** Botões de ação operam SEMPRE no item selecionado, nunca em watches[0] / rules[0] como um stub poderia fazer. Botões ficam desabilitados quando nada está selecionado, eliminando ambiguidade. Esta decisão é não-negociável: nunca retornar a um modelo onde a UI age "no primeiro item" sem seleção explícita.

### D-09: MonitorService inicia automaticamente no startup
**Razão:** Para o advogado-leigo, "abrir o programa" significa "ativar o monitoramento". Ter um botão extra "Iniciar Monitoramento" gera confusão ("por que está desligado?"). Trade-off aceito: o programa precisa estar aberto para monitorar. System tray (rodar em background) é v2.

### D-10: Cache local em estruturas de aba, recarga explícita via reload()
**Razão:** Os callbacks do widget.List do Fyne (length e update) são chamados muitas vezes durante render. Fazer Service.List() neles cria N+1 queries no SQLite. Cada aba (watchesTab, rulesTab futura) mantém um slice local atualizado por reload(), e os callbacks só leem dele.

### D-11: Erros traduzidos em humanizeError
**Razão:** Mensagens como "entrada inválida" ou "constraint failed" são inúteis para o usuário final. internal/ui/components/errors.go mapeia ErrNotFound, ErrDuplicateName, ErrWatchInUse, ErrInvalidInput em mensagens amigáveis em português. Erros desconhecidos têm a mensagem do Go limpa (último wrap removido).

### D-12: Dialog modal único com acordeon para criar/editar regras
**Razão:** Atomicidade do agregado Rule. RuleRepo.Update apaga e reinsere todos os filhos numa transação; espelhar isso na UI mantém o modelo mental coerente. Cancelar o dialog = nada vai pro banco. Salvar = tudo vai junto. Coerente com o padrão da aba Monitorar.

### D-13: Sub-dialogs para extrações, condições e ações
**Razão:** Cada item composto tem campos diferentes (extração: 3 strings; condição: variável + operador + valor; ação: tipo polimórfico + target). Inline edit em widget.List ficaria confuso, especialmente para o select de tipo de ação que muda os placeholders/hints. Sub-dialog dedicado dá foco e validação clara. Não-fechar-sub-dialog é descartar a edição daquele item.

### D-14: Feature "caminho entre aspas vira watch automático" adiada para v2
**Razão:** No briefing original, o usuário podia colar texto com caminhos entre aspas e o sistema criava watches automaticamente (path01, path02). Para o MVP, adicionar pasta uma a uma via picker é suficiente e mais didático para usuário leigo. A complexidade da feature (parser robusto, naming sequencial, edge cases de overlap entre paths) não se justifica no escopo atual. Registrada como feature explícita de v2.

### D-15: Aba "Histórico" dedicada como terceira aba
**Razão:** Coerência com o padrão de duas abas existente. Espaço para histórico longo (até 100 itens) com filtros por regra e status. Dimensionada para uso ocasional do tipo "checar o que aconteceu". Indicador "(N falhas)" no título destaca falhas novas sem forçar o usuário a abrir a aba constantemente.

### D-15.1: "Última visita" marcada ao SAIR da aba, não ao entrar
**Razão:** Se o badge zera no clique de entrada, o usuário perde a informação de "essas eu ainda não vi" assim que abre a aba. Marcar ao sair (via tabs.OnChanged) dá tempo de olhar as falhas com calma; só quando ele troca de aba é que consideramos "viu". Persistido em app_state via AppStateRepo.

### D-16: Botão "Atualizar" em dois lugares
**Razão:** "Atualizar tudo" no topo cobre o caso "configurei tudo, processar agora os PDFs já presentes". "Atualizar regra" por linha cobre "criei uma regra nova, quero aplicar só ela aos PDFs existentes sem reprocessar pelas outras". ScanService.ScanRule passa lista contendo apenas a regra alvo, ignorando as outras associadas à mesma pasta. Dedup por hash+rule_id (RN-11) ainda protege contra reprocessamento desnecessário.

### D-17: Build cross-platform via documentação, não cross-compile automatizado
**Razão:** Cross-compile de Fyne (CGO + OpenGL) entre Windows/Linux é desproporcionalmente complicado para o MVP. Build nativo em cada SO funciona com requisitos modestos (Go + toolchain C local). README documenta requisitos e comandos. CI/CD com runners nativos fica para v2.
---

## 14. GUIA PARA MODELOS DE IA TRABALHANDO NESTE PROJETO

### Antes de qualquer ação:
1. Leia este arquivo completo
2. Verifique o estado atual (seção 10) para entender o que está feito
3. Não proponha refatorações de código já funcionando sem razão explícita
4. Não adicione dependências sem consultar a seção 2

### Ao implementar:
- Siga a estrutura de pastas da seção 3 **exatamente**
- Respeite as regras de dependência entre pacotes (domain ← storage ← service ← ui)
- Use os patterns de código da seção 7
- Escreva testes antes ou junto da implementação — nunca depois
- Nomes de teste em português, no formato `Test<Tipo>_<Cenario>`
- Use `require.Eventually` para eventos assíncronos, nunca `time.Sleep`

### Ao terminar uma Sprint:
- Todos os testes devem passar: `make test-unit`
- Atualize a seção 10 deste arquivo (marque a sprint como concluída)
- Faça commit com mensagem descritiva

### O que NÃO fazer:
- Não implemente system tray, OCR, regex ou OR em condições — são v2
- Não acesse storage diretamente da UI
- Não use time.Sleep em testes
- Não crie arquivos de documentação adicionais além deste CLAUDE.md e o README.md
- Não mude o module path do go.mod
- Não troque `modernc.org/sqlite` por outro driver SQLite
- Não introduza novos frameworks ou bibliotecas sem discussão

### Sprint 7 — O que precisa ser feito agora:
Os arquivos `test/integration/e2e_test.go`, `rule_lifecycle_test.go`, `watch_lifecycle_test.go` e `watch_runtime_test.go` são **stubs vazios** (apenas `package integration`). O Sprint 7 precisa implementar:

1. **`test/fixtures/procuracao_sample.pdf`** — PDF com texto de exemplo de procuração
2. **`test/integration/e2e_test.go`** — Teste E2E completo:
   - Cria watch em pasta temporária
   - Cria regra com extração de "Nome: " → "nome", ação create_folder + move para `{nome}`
   - Inicia MonitorService
   - Copia `procuracao_sample.pdf` na pasta monitorada
   - Espera (via `require.Eventually`) arquivo aparecer em subpasta `{nome_extraído}`
   - Verifica registro em `processed_documents` com `status = "success"`
3. **`test/integration/rule_lifecycle_test.go`** — Ciclo de vida completo de regra (criar, associar watch, ativar, processar arquivo, desativar, deletar)
4. **`test/integration/watch_lifecycle_test.go`** — Ciclo de vida de watch + tentativa de deletar watch em uso (deve retornar `ErrWatchInUse`)
5. **`test/integration/watch_runtime_test.go`** — Watch em runtime: adicionar arquivo, modificar, remover — verificar comportamento correto

---

## 15. HISTÓRICO DE MUDANÇAS NESTE ARQUIVO

| Data | Mudança |
|------|---------|
| 2026-05-07 | Criação inicial do CLAUDE.md, consolidando docs/resumo juridico watcher.txt e decisões das conversas de desenvolvimento. Estado: Sprint 6 completo, Sprints 7-10 pendentes. |
| 2026-05-08 | Sprint 6.5 — correções pós-revisão completas. Adicionada RN-13 (condições case-insensitive). Pipeline agora consulta dedup antes de processar (RN-11 enforced em runtime, não só no banco). Pipeline usa caminho real retornado de `executeActions` em vez de simulação. Watcher detecta subpastas novas em modo recursivo. Interpolator faz single-pass independente da ordem do map. Teste de MonitorService movido para `test/integration/` por ser end-to-end. Stubs de integração unificados em `package integration_test`. |
| 2026-05-08 | Sprint 7 — testes de integração E2E completos. Cinco arquivos em `test/integration/` cobrem: ciclo de vida de watch (CRUD + RN-08 ErrWatchInUse), ciclo de vida de regra (CRUD com filhos + cascata + execução real), runtime do watcher (arquivo novo, não-PDF, subpasta recursiva, regra inativa), scan manual (ScanService com pasta única e recursiva), e o teste-âncora E2E que reproduz o cenário do briefing original (pasta digitalizadoras + regra procuração + scan inicial + monitor em runtime). Total: 16 testes de integração novos. |
| 2026-05-09 | Sprint 8 — UI Fyne aba Monitorar completa. Janela principal 1024×720 abre com duas abas. Aba "Monitorar" tem CRUD completo de pastas com seleção explícita (botões habilitam/desabilitam conforme seleção), picker nativo de pasta, validação de existência do diretório, mensagens de erro amigáveis (HumanizeError), confirmação de remoção, empty state. MonitorService inicia automaticamente ao abrir e reinicia após qualquer mudança. Aba "Regras" tem placeholder explícito remetendo à Sprint 9. Decisões D-07 a D-11 registradas. |
| 2026-05-09 | Sprint 9 — UI Fyne aba Regras completa. Dialog modal único (D-12) com metadados (nome, prioridade, ativa, pastas em grade de checkboxes) + acordeon com 3 seções: Extrações, Condições, Ações. Sub-dialogs dedicados para cada item composto (D-13), com helpers de operador/tipo de ação em português. Cache local + commit explícito (D-10): nada vai pro banco até "Salvar" no dialog principal. Validação em camadas: sub-dialogs validam campos próprios, commit() valida UX (>= 1 watch, >= 1 ação), service.Create/Update valida domínio (nome, watches). Aviso quando remover extração em uso por condição/ação. MonitorService reinicia após qualquer mudança em regras. Feature "caminho entre aspas" formalmente adiada para v2 (D-14). |
| 2026-05-09 | Sprint 10 — MVP fechado. Adicionada barra de topo com "Atualizar tudo" (ScanService.ScanAll, novo). Botão "Atualizar regra" por linha na aba Regras (ScanService.ScanRule, novo). Aba Histórico nova: lista com TwoLineItem colorido (vermelho=falha, laranja=sem texto), filtros locais por regra e por status, botão refresh. Indicador "(N falhas)" no título da aba, com persistência via app_state.last_visit (D-15.1). Migration 002_app_state.sql adicionada. README criado com requisitos por SO, comandos de build/run, fluxo de uso. ScanWatch agora retorna (count, error). Decisões D-15 a D-17 registradas. |

---

*Este arquivo deve ser atualizado a cada Sprint concluída e sempre que uma decisão de design for tomada ou alterada.*

---
