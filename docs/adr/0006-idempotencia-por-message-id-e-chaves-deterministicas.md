---
status: accepted
---

# Idempotência por messageId e chaves determinísticas

Com outbox, redelivery e retry, toda mensagem pode chegar mais de uma vez ou fora de ordem. Cada consumidor é idempotente:

- **Worker**: as chaves de saída no S3 são determinísticas (`outputs/{userId}/{videoId}/frames.zip`, `thumb.jpg`). Processar de novo sobrescreve o mesmo resultado.
- **API**: grava o `messageId` em `processed_events` na mesma transação da mudança de status e aplica a máquina de estados, que ignora (e registra em log) transições inválidas ou antigas. Por exemplo, `processing-started` depois de `COMPLETED` não faz o status regredir.
- **Notifier**: não tem banco, então usa `SET NX` com TTL no Redis por `messageId`, para não enviar o mesmo e-mail duas vezes.

## Consequências

- Todo envelope de mensagem leva `messageId` (UUID) e `correlationId` (= `videoId`).
- Reprocessar um vídeo (retry do usuário ou outro FPS) gera mensagens novas com `messageId` novo, mas reaproveita as mesmas chaves de saída.

Ver §5.3 e §5.5 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
