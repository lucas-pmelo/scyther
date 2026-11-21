---
status: proposed
---

# Implantação moldada pelo AWS Academy

A nuvem é o AWS Academy (Learner Lab): cota limitada, sessões que expiram e role `LabRole` fixa, sem criar IAM roles próprias. Isso decide onde cada peça roda:

- **PostgreSQL no RDS**, como na Fase 4.
- **RabbitMQ e Redis dentro do cluster EKS via Helm**, com 1 réplica e requests baixos. Amazon MQ e ElastiCache consumiriam cota e nem sempre estão disponíveis.
- **Front servido por nginx no cluster**, atrás do mesmo ingress da API, porque o CloudFront pode não estar disponível.
- **Node group de 2 a 4 nós com Cluster Autoscaler** usando a `LabRole`. Se o Academy não permitir escalar nós, o pool fica fixo e os `requests` do worker são dimensionados para que o máximo de réplicas caiba nele.
- **`docker compose` local completo** como plano B da demo, caso o Academy falhe no dia da gravação.

## Consequências

- O RabbitMQ e o Redis no cluster não têm alta disponibilidade real. Isso é aceito e documentado como não-objetivo.
- Escalar nós é bônus. A garantia de não perder requisição vem da fila, não da capacidade (ver [ADR 0007](0007-escalar-o-worker-com-keda-pela-fila.md)).
- A observabilidade usa retenção curta para caber no cluster.

Ver §3, §5.11 e §8 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
