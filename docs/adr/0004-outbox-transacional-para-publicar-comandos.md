---
status: accepted
---

# Outbox transacional para publicar comandos

Ao concluir o upload, a API grava `status = QUEUED` e uma linha na tabela `outbox` na mesma transação do PostgreSQL, sem publicar direto no RabbitMQ. Um relay lê a outbox, publica com publisher confirms e só marca `published_at` depois do confirm. Assim, se o RabbitMQ estiver fora do ar ou a API cair entre o commit e o publish, o pedido não se perde: ele é publicado quando o broker voltar.

## Alternativas consideradas

- **Publicar direto depois do commit**: perde a mensagem se o processo cair entre os dois passos ou se o broker estiver indisponível. Isso viola o requisito de não perder requisição.
- **Publicar antes do commit**: o worker pode receber um vídeo que o banco ainda não conhece ou que sofreu rollback.

## Consequências

- A entrega é *at-least-once*: o relay pode publicar a mesma mensagem duas vezes, então os consumidores precisam ser idempotentes (ver [ADR 0006](0006-idempotencia-por-message-id-e-chaves-deterministicas.md)).
- Um job de varredura republica vídeos parados em `QUEUED` sem `processing-started` há mais de X minutos, como rede de segurança.

Ver §5.5 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
