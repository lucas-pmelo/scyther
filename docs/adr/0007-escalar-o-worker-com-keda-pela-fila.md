---
status: accepted
---

# Escalar o worker com KEDA pela profundidade da fila

O worker escala com um `ScaledObject` do KEDA, usando o gatilho `rabbitmq` sobre o tamanho da fila `scyther-worker.process`. Dentro do pod, a concorrência é `WORKER_CONCURRENCY` (padrão: nº de CPUs), com prefetch igual. A API e o notifier usam HPA por CPU. Para o worker, CPU é um sinal atrasado e enganoso: com o pod no limite, a fila pode estar com 3 ou com 300 vídeos. A fila diz diretamente quanto trabalho falta.

## Alternativas consideradas

- **HPA por CPU no worker**: reage tarde e não distingue um pico de 3 vídeos de um de 300.
- **HPA com métrica customizada** (Prometheus Adapter): chega ao mesmo resultado, com mais peças para configurar.

## Consequências

- O sistema nunca depende de escalar para não perder nada: a fila absorve o excesso, e o escalonamento só reduz a espera.
- `minReplicaCount: 0` é opcional fora da demo, para economizar a sessão do Academy.
- O máximo de réplicas é limitado pela cota do Academy (ver [ADR 0013](0013-implantacao-moldada-pelo-aws-academy.md)).
- O `ScaledObject` é o que aparece no vídeo: fila subindo, réplicas aumentando, fila esvaziando.

Ver §5.11 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
