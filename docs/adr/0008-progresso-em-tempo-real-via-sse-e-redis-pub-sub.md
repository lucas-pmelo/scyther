---
status: proposed
---

# Progresso em tempo real via SSE e Redis pub/sub

O worker lê o `out_time` de `ffmpeg -progress pipe:1` e publica o percentual no Redis (`PUBLISH video:{id}:progress` + `SET` do último valor com TTL), no máximo uma vez por segundo. A API expõe `GET /videos/events` como Server-Sent Events: assina o pub/sub dos vídeos do usuário e repassa também as mudanças de status. O progresso é de alta frequência, efêmero e não precisa de durabilidade. Colocá-lo no RabbitMQ poluiria filas duráveis com mensagens que perdem o valor em um segundo.

## Alternativas consideradas

- **Progresso pelo RabbitMQ**: funciona, mas gasta quorum queues (escrita em disco) com dados descartáveis.
- **WebSocket**: bidirecional sem necessidade. O SSE é HTTP simples, reconecta sozinho no `EventSource` e passa pelo ingress sem configuração especial.
- **Polling de `GET /videos`**: simples, mas sem a barra de progresso fluida que é um dos diferenciais.

## Consequências

- Qualquer réplica da API atende qualquer conexão SSE, porque o estado está no Redis.
- O `EventSource` não envia header, então o JWT vai na query string do SSE. Isso pede cuidado com logs de acesso (não registrar a query string).
- Se o Redis cair, só o progresso ao vivo se perde. O status do vídeo continua correto no PostgreSQL.

Ver §5.2 e §8 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
