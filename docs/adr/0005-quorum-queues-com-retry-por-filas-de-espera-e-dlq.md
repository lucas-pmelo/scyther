---
status: accepted
---

# Quorum queues, retry por filas de espera e DLQ

Todas as filas do RabbitMQ são quorum queues duráveis, ligadas à exchange topic `scyther.events` (nomes no padrão da ADR 0015 da Fase 4). O consumo usa ack manual, feito só depois de o resultado (`evt.video.completed` ou `evt.video.failed`) ser publicado com confirm. Assim, se um pod do worker morrer, a mensagem volta para a fila. Falhas recuperáveis são reenviadas com backoff por filas de espera com TTL (`*.retry.5s`, `*.retry.30s`, `*.retry.2m`), que fazem dead-letter de volta para a fila principal. Depois de 3 tentativas, a mensagem vai para `*.dlq`. Erros não recuperáveis (`INVALID_MEDIA`, `TOO_LONG`) viram `evt.video.failed` na hora, sem retry.

## Alternativas consideradas

- **Classic queues**: mais leves, mas com garantias de durabilidade e replicação mais fracas. As quorum queues continuam duráveis em disco mesmo com 1 nó no Academy.
- **Retry com nack + requeue**: reentrega imediata, sem backoff, e um vídeo problemático fica em loop.
- **Plugin de delayed message**: exige instalar plugin no broker. As filas com TTL usam só recursos nativos.
- **Kafka**: possível pelo enunciado, mas sem ganho para um volume de comandos discretos, e mais pesado para o cluster do Academy.

## Consequências

- O worker só emite `evt.video.failed` quando a falha é definitiva, para o usuário não receber e-mail de falha que depois se resolve.
- Qualquer mensagem na DLQ dispara alerta no Grafana, porque significa erro inesperado.
- O worker precisa classificar cada erro como recuperável ou não.

Ver §5.4 e §5.5 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
