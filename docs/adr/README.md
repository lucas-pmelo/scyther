# ADRs — Scyther

Registro das decisões de arquitetura difíceis de reverter, surpreendentes sem contexto e fruto de uma escolha real entre alternativas. O contexto completo está no [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).

Status: `accepted` = decidida pela equipe; `proposed` = decisão tomada no design (§8) que ainda pode ser contestada.

| # | Decisão | Status |
|---|---|---|
| [0001](0001-worker-em-go-api-e-notifier-em-bun.md) | Worker em Go, API e notifier em Bun | accepted |
| [0002](0002-um-repositorio-por-componente-e-dono-unico-dos-dados.md) | Um repositório por componente e dono único dos dados | accepted |
| [0003](0003-upload-e-download-direto-no-s3-por-url-pre-assinada.md) | Upload e download direto no S3 por URL pré-assinada | accepted |
| [0004](0004-outbox-transacional-para-publicar-comandos.md) | Outbox transacional para publicar comandos | accepted |
| [0005](0005-quorum-queues-com-retry-por-filas-de-espera-e-dlq.md) | Quorum queues, retry por filas de espera e DLQ | accepted |
| [0006](0006-idempotencia-por-message-id-e-chaves-deterministicas.md) | Idempotência por messageId e chaves determinísticas | accepted |
| [0007](0007-escalar-o-worker-com-keda-pela-fila.md) | Escalar o worker com KEDA pela profundidade da fila | accepted |
| [0008](0008-progresso-em-tempo-real-via-sse-e-redis-pub-sub.md) | Progresso em tempo real via SSE e Redis pub/sub | proposed |
| [0009](0009-expiracao-por-lifecycle-do-s3.md) | Expiração por lifecycle do S3, mantendo o original por 7 dias | accepted |
| [0010](0010-autenticacao-dentro-da-api.md) | Autenticação dentro da scyther-api | proposed |
| [0011](0011-worker-baixa-o-video-para-disco-efemero.md) | Worker baixa o vídeo para disco efêmero | proposed |
| [0012](0012-zip-sem-compressao-em-streaming-para-o-s3.md) | Zip sem compressão, em streaming para o S3 | accepted |
| [0013](0013-implantacao-moldada-pelo-aws-academy.md) | Implantação moldada pelo AWS Academy | proposed |
| [0014](0014-e-mail-via-mailpit-com-smtp-configuravel.md) | E-mail via Mailpit, com SMTP real configurável | proposed |

Novas ADRs seguem a numeração sequencial (`NNNN-slug.md`) e entram nesta tabela.
