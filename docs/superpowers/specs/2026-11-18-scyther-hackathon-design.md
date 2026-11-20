# Scyther — Sistema de Processamento de Vídeos (Hackathon Fase 5) — Design

- Data: 2026-11-18
- Status: aguardando aprovação
- Enunciado: [`_referencia/enunciado.pdf`](../../../_referencia/enunciado.pdf)
- Protótipo dos investidores: [`_referencia/projeto-base/`](../../../_referencia/projeto-base/main.go)

## 1. Intenção e porquê

A FIAP X tem um protótipo (Go + Gin, um `main.go`) que recebe um vídeo, extrai 1 frame/s com
ffmpeg e devolve um `.zip`. Ele é síncrono, guarda tudo em disco local, não tem usuários e
expõe os arquivos de todo mundo. O hackathon pede a reconstrução com boas práticas:
arquitetura, microsserviços, mensageria, qualidade, CI/CD.

O hackathon tem **ranking entre equipes**, e a nota é dada pela qualidade das entregas
(documentação, código, vídeo). A equipe já entregou na Fase 4 um pacote acima da média
(Bunzina: 17 ADRs, Terraform, EKS, Helm umbrella, observabilidade LGTM, SonarCloud, 80% de
cobertura, saga sobre RabbitMQ). Portanto **o patamar a superar é o Bunzina**, não o
enunciado. Os diferenciais vêm do que este problema tem de específico: arquivos grandes,
processamento pesado e picos de carga.

## 2. Objetivos, não-objetivos e anti-objetivos

### Objetivos (requisitos do enunciado → como são atendidos)

| Requisito | Atendimento |
|---|---|
| Processar mais de um vídeo ao mesmo tempo | Workers Go concorrentes (goroutines com limite) × N réplicas escaladas por KEDA |
| Não perder requisição em picos | Upload direto ao S3 + outbox transacional + quorum queues + publisher confirms + ack manual + retry/DLQ |
| Protegido por usuário e senha | Cadastro/login com senha argon2id, JWT |
| Listagem de status dos vídeos do usuário | `GET /videos` paginado + progresso em tempo real via SSE |
| Notificação em caso de erro | `scyther-notifier` envia e-mail em falha (e em conclusão) |
| Persistir dados | PostgreSQL (estado), S3 (arquivos), Redis (progresso efêmero) |
| Arquitetura escalável | Serviços stateless, fila como buffer, KEDA no worker, HPA na API |
| Versionado no GitHub | Organização com um repositório por componente |
| Testes | Unit + integração (testcontainers) + E2E + carga (k6), ≥ 80% de cobertura |
| CI/CD | GitHub Actions: lint, testes, Sonar, scan de imagem, push ECR, deploy Helm |

### Diferenciais (o "bem mais")

1. **Front-end real** (React): login, upload múltiplo com barra de upload, lista de vídeos com
   progresso de processamento ao vivo, miniatura, download, reprocessar falhas.
2. **Upload direto para o S3 via URL pré-assinada** (multipart para arquivos grandes): o vídeo
   nunca passa pela API.
3. **Prova de "zero perda" sob pico**: script k6 dispara centenas de uploads; KEDA escala os
   workers pela profundidade da fila; dashboard Grafana mostra fila subindo e esvaziando,
   com contador "solicitados = concluídos + falhos".
4. **Resiliência demonstrável**: matar um pod do worker no meio do processamento e o vídeo
   ser concluído por outro pod; retry com backoff; DLQ; idempotência.
5. **Extras de produto**: FPS configurável, miniatura de preview, expiração automática dos
   zips, reprocessar vídeo com falha.
6. **Trace distribuído poliglota**: um trace único atravessa web → API (Bun) → RabbitMQ →
   worker (Go) → notifier (Bun), visível no Tempo.

### Não-objetivos

- Edição de vídeo, transcodificação ou qualquer saída além de frames + zip + miniatura.
- Papéis/admin (RBAC). Um único tipo de usuário.
- Recuperação de senha, OAuth, MFA.
- Multi-região, alta disponibilidade real do banco (Academy não comporta).
- App mobile.

### Anti-objetivos (falharia mesmo funcionando)

- Demonstração que depende de sorte: todo cenário do vídeo (pico, kill de pod, falha) precisa
  ser reproduzível por script.
- Diferencial que não aparece no vídeo nem na documentação: se não é mostrável, não entra.
- Escopo que deixe o caminho principal (upload → processa → download) frágil. O caminho
  principal fica pronto e estável antes de qualquer extra.
- Copiar o Bunzina sem adaptar: ADRs e diagramas precisam falar deste problema.

## 3. Restrições

- **Prazo**: janeiro de 2027 (data exata no Portal do Aluno). ~7 semanas a partir de hoje.
- **Equipe**: 2 pessoas codando, com apoio de IA.
- **Nuvem**: AWS Academy (Learner Lab), como na Fase 4: cota limitada, sessões que expiram,
  role `LabRole` fixa (sem criar IAM roles próprias).
- **Vídeo de apresentação**: no máximo 10 minutos, com documentação, arquitetura e o sistema
  funcionando.
- **Entregáveis**: documentação da arquitetura, scripts de banco/recursos, links do GitHub.

## 4. Abordagens consideradas

| | A) Tudo em Bun | **B) Poliglota (escolhida)** | C) Tudo em Go |
|---|---|---|---|
| Ideia | API, worker, notifier em Bun/Elysia; worker chama ffmpeg via subprocesso | API e notifier em Bun; worker em Go refatorado do protótipo | Reescrever tudo em Go a partir do protótipo |
| Ganho | Máximo reaproveitamento do template da Fase 4; um toolchain | História forte ("o núcleo do protótipo virou worker de produção"); microsserviços independentes de verdade; ferramenta adequada para CPU/IO; trace poliglota | Uma linguagem, próxima do protótipo |
| Custo | Nada de novo para mostrar; Bun menos adequado a trabalho pesado | Dois toolchains; replicar em Go o padrão de testes/Sonar/OTel | Joga fora template, testes e ritmo da equipe em Bun |

**Escolha da equipe: B.** O worker é pequeno (consumir fila, ffmpeg, zip, S3, eventos), então
o custo extra do Go é baixo para duas pessoas com IA, e ele vira um diferencial.

## 5. Arquitetura

### 5.1 Componentes e repositórios

Organização GitHub nova, um repositório por componente (padrão da Fase 4):

| Repositório | Stack | Responsabilidade |
|---|---|---|
| `scyther-web` | React + Vite + TypeScript, TanStack Query | SPA: login/cadastro, upload, lista com progresso, download |
| `scyther-api` | Bun + Elysia, Clean Architecture | Autenticação, ciclo de vida do vídeo, URLs pré-assinadas, listagem, SSE de progresso, outbox, consumo de eventos do worker, job de expiração |
| `scyther-worker` | Go | Consome pedidos de processamento, ffprobe/ffmpeg, zip em streaming para o S3, miniatura, publica progresso e resultado |
| `scyther-notifier` | Bun | Consome eventos de conclusão/falha e envia e-mail |
| `scyther-infra` | Terraform | VPC, EKS, ECR, S3, RDS Postgres, addons (KEDA, ingress) |
| `scyther-chart` | Helm | Chart umbrella: serviços + RabbitMQ + Redis + stack de observabilidade |
| `scyther-docs` (opcional, ou dentro de `scyther-api`) | Markdown + diagramas | Documentação central, ADRs, roteiro do vídeo |

Cada serviço tem **dono único dos seus dados**: só a `scyther-api` acessa o Postgres. O worker e o
notifier são stateless e se comunicam apenas por mensagens, S3 e Redis.

### 5.2 Fluxo principal

```
web ──POST /videos──────────────▶ api  (cria vídeo PENDING_UPLOAD, devolve URL(s) pré-assinada(s))
web ──PUT (multipart)───────────▶ S3   uploads/{userId}/{videoId}/source
web ──POST /videos/:id/complete─▶ api  (HEAD no S3, valida tamanho; em 1 transação:
                                         status=QUEUED + linha na outbox)
api outbox relay ──publish(confirm)─▶ RabbitMQ  cmd.video.process  → fila scyther-worker.process
worker ── consome (prefetch = concorrência) ──
   ├─ publica evt.video.processing-started
   ├─ ffprobe (valida que é vídeo, duração ≤ limite)
   ├─ ffmpeg -vf fps={fps} -progress pipe:1  → frames em disco efêmero
   │     └─ progresso (%) → Redis PUBLISH video:{id}:progress + SET last
   ├─ miniatura → S3 outputs/{userId}/{videoId}/thumb.jpg
   ├─ zip em streaming (io.Pipe + multipart upload) → S3 outputs/{userId}/{videoId}/frames.zip
   ├─ publica evt.video.completed (frameCount, zipKey, zipSize)  ── ou evt.video.failed
   └─ ack
api consome evt.video.* → atualiza status (máquina de estados, idempotente)
notifier consome evt.video.failed / evt.video.completed → e-mail
web ◀──SSE /videos/events── api (assina Redis pub/sub dos vídeos do usuário + mudanças de status)
web ──GET /videos/:id/download──▶ api (checa dono) → 302 para URL pré-assinada de GET
```

### 5.3 Estados do vídeo

```
PENDING_UPLOAD ──complete──▶ QUEUED ──started──▶ PROCESSING ──completed──▶ COMPLETED ──ttl──▶ EXPIRED
      │                         ▲                    │
      └─(sem complete em 24h)─▶ ABANDONED            └──failed (esgotou retries)──▶ FAILED ──retry pelo usuário──▶ QUEUED
                                                                                        └──ttl──▶ EXPIRED
COMPLETED ──reprocessar com outro FPS (antes de expirar)──▶ QUEUED
```

- Transições inválidas ou fora de ordem (evento antigo chegando depois) são ignoradas e
  registradas em log. Ex.: `processing-started` depois de `COMPLETED` não regride o status.
- `FAILED` guarda `error_code` (ex.: `INVALID_MEDIA`, `TOO_LONG`, `FFMPEG_ERROR`,
  `STORAGE_ERROR`) e mensagem amigável.

### 5.4 Mensageria (RabbitMQ)

Mesmo padrão de nomes da ADR 0015 do Bunzina:

- Exchange `scyther.events` (topic, durável).
- Comandos: `cmd.video.process`. Eventos: `evt.video.processing-started`,
  `evt.video.completed`, `evt.video.failed`.
- Filas (todas **quorum queues**, duráveis):
  - `scyther-worker.process` ← `cmd.video.process`
  - `scyther-api.inbox` ← `evt.video.*`
  - `scyther-notifier.inbox` ← `evt.video.completed`, `evt.video.failed`
- **Retry com backoff**: filas de espera `*.retry.5s`, `*.retry.30s`, `*.retry.2m` (TTL +
  dead-letter de volta à fila principal). Após 3 tentativas → `*.dlq`. Erros
  não-recuperáveis (`INVALID_MEDIA`, `TOO_LONG`) vão direto para `evt.video.failed` sem retry.
- O worker só emite `evt.video.failed` quando a falha é definitiva (não-recuperável ou última
  tentativa). Mensagem que cai na DLQ por erro inesperado gera alerta no Grafana.
- Envelope de toda mensagem: `messageId` (UUID), `type`, `version`, `occurredAt`,
  `correlationId` (= `videoId`), `attempt`, `payload`; header AMQP `traceparent` (W3C).
- Payload de `cmd.video.process`: `videoId`, `userId`, `userEmail` (snapshot, para o notifier
  não precisar consultar a API), `sourceKey`, `fps`, `frameFormat`, `originalFilename`.
- Contrato documentado em `docs/contracts/` com JSON Schema por mensagem, e testes de
  contrato nos dois lados (Bun e Go validam contra os mesmos schemas).

### 5.5 Garantia de "não perder requisição"

Camada por camada, para documentar e demonstrar:

1. **Upload**: vai direto ao S3 (durável); a API não segura bytes, então pico de upload não
   derruba a API.
2. **Aceite**: `complete` grava status + outbox na mesma transação. Se o RabbitMQ estiver fora,
   o pedido fica na outbox e é publicado quando ele voltar.
3. **Publicação**: relay da outbox com publisher confirms; só marca como publicado após o
   confirm.
4. **Fila**: quorum queue persistente; mensagens sobrevivem a reinício do broker.
5. **Consumo**: ack manual só depois de `evt.video.completed`/`failed` publicado com confirm.
   Pod morto → mensagem volta para a fila (redelivery).
6. **Idempotência**: chaves de saída determinísticas (`outputs/{userId}/{videoId}/...`),
   reprocessar sobrescreve o mesmo resultado; a API registra `messageId` em
   `processed_events` e aplica a máquina de estados.
7. **Varredura**: job periódico na API republica vídeos em `QUEUED` há mais de X minutos sem
   `processing-started` (rede de segurança) e marca `ABANDONED` os uploads não concluídos.

### 5.6 Worker Go

- Estrutura hexagonal leve: `domain` (regras: validação, cálculo de progresso), `ports`
  (Storage, Publisher, ProgressSink, MediaProbe, FrameExtractor), `adapters` (S3, AMQP, Redis,
  ffmpeg exec), `cmd/worker`.
- Concorrência: `WORKER_CONCURRENCY` (padrão = nº de CPUs do pod); prefetch do canal = mesmo
  valor.
- Entrada do ffmpeg: download do vídeo para `emptyDir` (mais robusto que ler via HTTP para
  formatos que precisam de seek).
- Progresso: `ffprobe` dá a duração; `ffmpeg -progress pipe:1` dá `out_time`; publica no
  Redis no máximo a cada 1s.
- **Graceful shutdown**: no SIGTERM para de consumir; espera os jobs em curso até o
  `terminationGracePeriodSeconds`; o que não terminar é devolvido com nack/requeue.
- Health: `/healthz` (vivo) e `/readyz` (conectado ao broker), `/metrics` Prometheus.
- Limites configuráveis: tamanho máximo do vídeo (padrão 2 GB), duração máxima (padrão
  30 min), FPS permitido (0.1 a 5).
- Imagem: build multi-stage, runtime com ffmpeg estático, usuário não-root.

### 5.7 API (Bun + Elysia)

Reaproveita o template `templates/service` do Bunzina (Clean Architecture, migrations
próprias, OTel, logger, Sonar, CI).

Endpoints:

| Método e rota | Descrição |
|---|---|
| `POST /auth/register` | Cria usuário (nome, e-mail, senha) |
| `POST /auth/login` | Devolve JWT; rate limit por IP/e-mail no Redis |
| `GET /me` | Usuário atual |
| `POST /videos` | Cria vídeo, recebe `filename`, `sizeBytes`, `contentType`, `fps`, `frameFormat`; devolve `videoId` + dados de upload multipart pré-assinado |
| `POST /videos/:id/complete` | Conclui o multipart, valida o objeto, enfileira |
| `GET /videos` | Lista paginada do usuário, filtro por status |
| `GET /videos/:id` | Detalhe + último progresso (Redis) |
| `GET /videos/:id/download` | 302 para URL pré-assinada do zip |
| `GET /videos/:id/thumbnail` | 302 para URL pré-assinada da miniatura |
| `POST /videos/:id/retry` | Reenfileira vídeo `FAILED` |
| `DELETE /videos/:id` | Remove vídeo e artefatos |
| `GET /videos/events` | SSE: progresso e mudanças de status dos vídeos do usuário |
| `GET /health`, `/metrics` | Operação |

- Validação com Zod; OpenAPI gerado pelo Elysia (Swagger em `/docs`).
- Senhas com `Bun.password` (argon2id).
- JWT HS256 como na ADR 0004, expiração de 1h. O SSE recebe o token por query string,
  porque `EventSource` não envia header.
- Toda consulta filtra por `user_id` do token (um usuário nunca vê vídeo de outro).

### 5.8 Notifier (Bun)

- Consome `evt.video.failed` (obrigatório pelo enunciado) e `evt.video.completed` (extra).
- Templates de e-mail em HTML; SMTP via Nodemailer (Mailpit local; em nuvem, um provedor
  SMTP gratuito, ver §8).
- Idempotência por `messageId` em Redis (SET NX com TTL), porque o notifier não tem banco.

### 5.9 Front-end (`scyther-web`)

- Telas: cadastro, login, painel de vídeos.
- Painel: área de arrastar e soltar com vários arquivos; cada arquivo mostra o progresso do
  upload (multipart, em paralelo) e depois o processamento (SSE); lista com miniatura, status,
  nº de frames, tamanho do zip, botões de download/reprocessar/excluir; escolha do FPS no
  envio.
- Servido como estático por nginx no cluster, atrás do mesmo ingress da API.

```
┌──────────────────────────────────────────────────────────┐
│ Scyther                                    lucas ▾ Sair  │
├──────────────────────────────────────────────────────────┤
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐  FPS: [ 1 ▾]   │
│    Arraste vídeos aqui ou clique                          │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘                │
│                                                          │
│ [▣] aula1.mp4   Processando ███████░░░ 68%               │
│ [▣] demo.mov    Concluído · 312 frames · 48 MB  [Baixar] │
│ [ ] corrompido  Falhou · arquivo inválido   [Reprocessar]│
│ [ ] grande.mkv  Enviando  ████░░░░░░ 41%                 │
└──────────────────────────────────────────────────────────┘
```

### 5.10 Dados

**PostgreSQL** (só a `scyther-api` acessa), com migrations versionadas, que são também o
"script de criação do banco" do entregável:

- `users` (id, name, email único, password_hash, created_at)
- `videos` (id, user_id, original_filename, content_type, size_bytes, fps, status,
  frame_format, source_key, zip_key, zip_size_bytes, thumbnail_key, frame_count, duration_seconds,
  error_code, error_message, attempts, queued_at, started_at, completed_at, expires_at,
  created_at, updated_at); índice `(user_id, created_at desc)` e `(status, updated_at)`
- `outbox` (id, message_id, routing_key, payload jsonb, headers jsonb, created_at,
  published_at)
- `processed_events` (message_id pk, processed_at)

**Redis**: progresso (`video:{id}:progress`, pub/sub + último valor com TTL), rate limit de
login, idempotência do notifier.

**S3**: bucket privado; `uploads/` e `outputs/` com lifecycle de 7 dias contados da criação
do objeto (regra nativa do S3). O worker **não** apaga o vídeo original: ele fica disponível
para reprocessar (falha ou outro FPS) enquanto não expira. Um job da API marca como `EXPIRED`
os vídeos com `expires_at` vencido (`COMPLETED` ou `FAILED`), e `retry` só é aceito antes
disso. Uploads incompletos de multipart são abortados por lifecycle em 1 dia. CORS liberado só
para a origem do front. Local: MinIO.

### 5.11 Escalabilidade

- Worker: KEDA `ScaledObject` com gatilho `rabbitmq` (tamanho da fila
  `scyther-worker.process`), mín. 1, máx. definido pela cota do Academy.
- Nós: node group gerenciado com mín. 2 e máx. 4, e Cluster Autoscaler usando a `LabRole`.
  Se o Academy não permitir escalar nós, o node group fica fixo e os `requests` do worker são
  dimensionados para que o máximo de réplicas caiba no pool. A demo mostra os pods escalando
  de qualquer forma; os nós escalando são um bônus. O sistema nunca depende de escalar nós
  para não perder nada, porque a fila absorve o excesso.
- Volume de referência para dimensionar: vídeos curtos (10–60 s, até ~30 MB) no teste de carga;
  um vídeo grande (centenas de MB) é usado à parte na demo manual, para exercitar o upload
  multipart. O `emptyDir` do worker tem limite proporcional ao tamanho máximo do vídeo.
- API e notifier: HPA por CPU, mín. 2 réplicas na API.
- Todos stateless (estado em Postgres/Redis/S3), então qualquer réplica atende qualquer
  requisição, inclusive o SSE (via Redis pub/sub).

### 5.12 Observabilidade

Mesma stack da Fase 4 (Alloy + Prometheus + Loki + Tempo + Grafana), provisionada como
código no chart:

- OTel nos quatro componentes; `traceparent` propagado nos headers AMQP nos dois sentidos
  (Bun e Go).
- Métricas de negócio: vídeos solicitados/concluídos/falhos, duração do processamento
  (histograma), frames por vídeo, profundidade das filas e DLQ, réplicas de worker.
- Dashboard "Scyther — Operação" com o painel do pico (fila × réplicas × concluídos) usado no
  vídeo.
- Alertas: DLQ > 0, taxa de falha > X%, fila parada.

### 5.13 Qualidade e testes

- **API/notifier (Bun)**: `bun test`, unitários dos casos de uso e da máquina de estados;
  integração com Postgres/Redis/RabbitMQ/MinIO reais via testcontainers ou compose.
- **Worker (Go)**: `go test` com unitários do domínio e adapters com fakes; integração com
  testcontainers-go (RabbitMQ, MinIO, Redis) e vídeo de amostra pequeno no repositório.
- **Contrato**: JSON Schemas das mensagens validados nos dois lados.
- **E2E**: Playwright no `scyther-web` contra o compose completo (cadastra, envia, espera
  concluir, baixa).
- **Carga**: script k6 versionado, com relatório anexado à documentação.
- Cobertura ≥ 80% por repositório, quality gate no SonarCloud, lint (oxlint/oxfmt;
  golangci-lint).

### 5.14 CI/CD

GitHub Actions por repositório, derivado dos workflows da Fase 4:

- PR: lint + formatação, testes com cobertura, Sonar, build da imagem, scan com Trivy.
- `main`: push no ECR e `helm upgrade` no EKS.
- `scyther-infra`: `terraform fmt/validate/plan` no PR; `apply` manual.
- Proteção de branch na `main` (PR + checagens obrigatórias).

### 5.15 Ambiente local

`docker compose up` sobe tudo: Postgres, Redis, RabbitMQ (com painel de gestão), MinIO,
Mailpit, os quatro componentes e a stack de observabilidade. Serve para desenvolver, para
os testes E2E e como **plano B da demo** caso o Academy falhe no dia da gravação.

### 5.16 Segurança

- Bucket privado, URLs pré-assinadas curtas (upload 15 min, download 5 min), chaves com
  `userId` e checagem de dono na API.
- Validação real de mídia com ffprobe (o protótipo só olhava a extensão).
- Sem path traversal: nenhum caminho de arquivo vem do cliente.
- Imagens não-root, scan Trivy no CI, segredos via Kubernetes Secrets (nada no repositório).
- Rate limit no login.

### 5.17 Eficiência de recursos

O cluster do Academy é pequeno e sem máximo conhecido, então tudo é desenhado para caber
enxuto num pool fixo, e escalar nós é bônus:

- **Zip sem recompressão**: PNG/JPEG já são comprimidos; o zip usa `Store` em vez de
  `Deflate` (o protótipo usava Deflate e gastava CPU sem reduzir o tamanho).
- **Formato do frame**: PNG (fiel ao protótipo) ou JPEG (padrão sugerido no front, de 5 a 10x
  menor), escolhido no envio.
- **ffmpeg sob controle**: `-threads` por job alinhado com `WORKER_CONCURRENCY`, para não
  disputar CPU dentro do pod; requests/limits do worker medidos com o vídeo de referência.
- **Streaming em tudo**: o zip vai para o S3 em multipart sem materializar o arquivo em disco;
  os frames são apagados conforme entram no zip.
- **Imagens mínimas**: Go estático em imagem distroless + ffmpeg estático; Bun em imagem slim;
  nginx alpine para o front.
- **Infra de apoio enxuta**: RabbitMQ e Redis com 1 réplica e requests baixos no Academy
  (quorum queue com 1 nó continua durável em disco); observabilidade com retenção curta.
- **API sem bytes de vídeo**: upload e download vão direto ao S3, então a API roda com pouca
  memória e escala só por requisições leves.
- **Worker escala a zero opcionalmente** fora da demo (KEDA `minReplicaCount: 0`), economizando
  a sessão do Academy.

## 6. Entregáveis e documentação

- README central com visão geral, como rodar, links para tudo.
- Diagramas C4 (contexto, contêineres, componentes do worker), sequência do fluxo principal e
  do retry, máquina de estados, ER, infraestrutura AWS. Em Mermaid (versionável) e exportados
  em PNG.
- ADRs desta fase (mínimo: poliglota Bun+Go, upload pré-assinado, outbox, quorum queues e
  retry, KEDA, SSE+Redis para progresso, S3 lifecycle para expiração).
- Contratos de mensagens (JSON Schema).
- Coleção Bruno da API.
- Relatório do teste de carga (k6 + prints do Grafana).
- Roteiro do vídeo de 10 minutos, com cenários scriptados:
  1. Problema e arquitetura (2 min).
  2. Fluxo de usuário no front (2 min).
  3. Pico com k6 + KEDA + Grafana, com contagem de zero perda (2 min).
  4. Kill de pod do worker e recuperação; vídeo inválido → e-mail de falha (2 min).
  5. Trace distribuído no Tempo, CI/CD e qualidade (Sonar) (2 min).

## 7. Ordem de construção (para o plano detalhar)

O escopo é completo; não há linha de corte prévia. A equipe acompanha o prazo e decide cortes
durante a execução, por isso a ordem abaixo põe primeiro o que sustenta o resto.

1. Caminho principal ponta a ponta local (compose): API + worker + upload pré-assinado +
   status + download. Nenhum extra antes disso funcionar.
2. Resiliência: outbox, retry/DLQ, idempotência, graceful shutdown, notifier.
3. Front-end com progresso em tempo real.
4. Infra AWS + CI/CD + KEDA.
5. Observabilidade, k6, roteiro e documentação.
6. Extras de produto (miniatura, reprocessar, excluir) conforme sobrar tempo.

## 8. Decisões tomadas pelo Claude (podem ser contestadas)

- Autenticação dentro da `scyther-api` (sem Lambda/API Gateway como na Fase 4): o enunciado só
  pede usuário e senha, e um serviço a menos libera tempo para os diferenciais.
- JWT HS256 sem refresh token, como na ADR 0004.
- Redis em pub/sub para o progresso, em vez do RabbitMQ: é alta frequência e efêmero, não
  precisa de durabilidade.
- Worker baixa o vídeo para disco efêmero em vez de ler direto do S3 via HTTP.
- Front servido por nginx dentro do cluster (CloudFront pode não estar disponível no Academy).
- RabbitMQ e Redis dentro do cluster via Helm; Postgres no RDS (como na Fase 4).
- E-mail: Mailpit rodando no cluster é o padrão (sempre funciona e aparece bem no vídeo).
  SMTP real (Gmail com senha de app ou Brevo) fica configurável por variável de ambiente, para
  usar se a saída SMTP do Academy permitir. O SES do Academy fica em sandbox.
- Repositórios públicos na organização GitHub, como na Fase 4 (ADR 0017): SonarCloud
  gratuito, proteção de branch e acesso da banca.
- O vídeo original é mantido por 7 dias para permitir reprocessar (ver §5.10).
- Node group de 2 a 4 nós com Cluster Autoscaler, com plano B de pool fixo (ver §5.11).

## 9. Deixado para quem constrói

- Bibliotecas específicas em Go (cliente AMQP, SDK AWS v2 é o esperado) e no front
  (componentes de UI).
- Nomes finais de métricas e layout do dashboard.
- Divisão de trabalho entre as duas pessoas.
- Valores exatos de limites (tamanho, duração, réplicas máximas), respeitando a cota do
  Academy.
