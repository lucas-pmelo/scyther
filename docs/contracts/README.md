# Contratos de mensagens — Scyther

Contratos das mensagens trocadas entre os serviços, em JSON Schema (draft 2020-12). Bun e Go validam contra **estes mesmos arquivos** nos testes de contrato. Não há tipos compartilhados entre as linguagens (ADR [0001](../adr/0001-worker-em-go-api-e-notifier-em-bun.md)), então este diretório é a fonte da verdade.

## Mensagens

Todas passam pela exchange `scyther.events` (topic, durável). A topologia completa está no [diagrama 11](../diagramas/README.md#11-topologia-do-rabbitmq).

| Routing key | Schema | Produtor | Consumidores (fila) | Exemplo |
|---|---|---|---|---|
| `cmd.video.process` | [v1](cmd.video.process.v1.schema.json) | scyther-api (outbox relay) | scyther-worker (`scyther-worker.process`) | [json](examples/cmd.video.process.json) |
| `evt.video.processing-started` | [v1](evt.video.processing-started.v1.schema.json) | scyther-worker | scyther-api (`scyther-api.inbox`) | [json](examples/evt.video.processing-started.json) |
| `evt.video.completed` | [v1](evt.video.completed.v1.schema.json) | scyther-worker | scyther-api, scyther-notifier (`scyther-notifier.inbox`) | [json](examples/evt.video.completed.json) |
| `evt.video.failed` | [v1](evt.video.failed.v1.schema.json) | scyther-worker | scyther-api, scyther-notifier | [json](examples/evt.video.failed.json) |

Fora do RabbitMQ, o progresso vai pelo Redis (ADR [0008](../adr/0008-progresso-em-tempo-real-via-sse-e-redis-pub-sub.md)):

| Canal / chave | Schema | Produtor | Consumidor | Exemplo |
|---|---|---|---|---|
| `PUBLISH video:{videoId}:progress` e `SET video:{videoId}:progress` (com TTL) | [v1](progress.v1.schema.json) | scyther-worker (no máximo 1×/s) | scyther-api (repassa no SSE) | [json](examples/progress.json) |

## Envelope

Toda mensagem do RabbitMQ segue o [envelope](envelope.schema.json):

| Campo | Significado |
|---|---|
| `messageId` | UUID único. Chave de idempotência (ADR [0006](../adr/0006-idempotencia-por-message-id-e-chaves-deterministicas.md)). |
| `type` | A routing key. |
| `version` | Versão do contrato do `payload`. |
| `occurredAt` | Quando o fato aconteceu (UTC), não quando foi publicado. |
| `correlationId` | Sempre o `videoId`. |
| `attempt` | Tentativa de processamento (1 a 3). |
| `payload` | Conteúdo específico do tipo. |

Os tipos compartilhados (`videoId`, `fps`, chaves do S3, códigos de erro) ficam em [`defs.schema.json`](defs.schema.json).

### Propriedades e headers AMQP

| AMQP | Valor |
|---|---|
| `content_type` | `application/json` |
| `delivery_mode` | `2` (persistente) |
| `message_id` | igual a `messageId` |
| `type` | igual a `type` |
| `correlation_id` | igual a `correlationId` |
| header `traceparent` | contexto W3C do trace (obrigatório) |
| header `tracestate` | opcional |

O RabbitMQ acrescenta `x-death` quando a mensagem passa pelas filas de retry. Os consumidores usam `attempt` do envelope, não `x-death`, para decidir a próxima tentativa.

## Por que os eventos repetem dados do pedido

`evt.video.completed` e `evt.video.failed` levam `userEmail` e `originalFilename` de novo. O notifier não tem banco nem consulta a API (ADR [0002](../adr/0002-um-repositorio-por-componente-e-dono-unico-dos-dados.md)), então tudo de que o e-mail precisa tem de estar na mensagem. O worker copia esses campos do `cmd.video.process` que recebeu.

## Códigos de erro

`errorCode` em `evt.video.failed`:

| Código | Causa | Retry automático |
|---|---|---|
| `INVALID_MEDIA` | ffprobe não reconhece o arquivo como vídeo | não, falha na hora |
| `TOO_LONG` | duração acima do limite (padrão 30 min) | não, falha na hora |
| `TOO_LARGE` | tamanho acima do limite (padrão 2 GB), se escapar da checagem da API | não, falha na hora |
| `FFMPEG_ERROR` | ffmpeg terminou com erro durante a extração | sim, até 3 tentativas |
| `STORAGE_ERROR` | falha ao ler ou gravar no S3 | sim, até 3 tentativas |

Um erro inesperado fora desta lista não gera `evt.video.failed`: a mensagem vai para a DLQ e dispara alerta (ADR [0005](../adr/0005-quorum-queues-com-retry-por-filas-de-espera-e-dlq.md)).

## Regras de evolução

- **Mudança aditiva** (campo novo opcional, valor novo de enum que os consumidores já tratam): atualize o schema mantendo a `version`. Primeiro os consumidores, depois o produtor.
- **Mudança incompatível** (remover ou renomear campo, mudar tipo, tornar obrigatório): crie `*.v2.schema.json`. O produtor passa a publicar v2 só depois que todos os consumidores aceitam v1 e v2.
- Os schemas são estritos (`additionalProperties: false`) para pegar erro de digitação nos testes do **produtor**. Em tempo de execução, o **consumidor** ignora campos desconhecidos em vez de rejeitar a mensagem.

## Testes de contrato

- **Produtor** (api para `cmd`, worker para `evt.*` e progresso): toda mensagem que o código monta é validada contra o schema antes do teste passar.
- **Consumidor**: os arquivos de `examples/` são desserializados e tratados pelo handler real. Os exemplos também precisam passar no próprio schema.
- Bun: Ajv (draft 2020-12, com `ajv-formats`). Go: `github.com/santhosh-tekuri/jsonschema`. Os dois resolvem os `$ref` relativos entre arquivos deste diretório.
