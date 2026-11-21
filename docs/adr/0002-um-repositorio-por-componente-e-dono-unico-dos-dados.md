---
status: accepted
---

# Um repositório por componente e dono único dos dados

Cada componente (`scyther-web`, `scyther-api`, `scyther-worker`, `scyther-notifier`, `scyther-infra`, `scyther-chart`) tem um repositório próprio numa organização do GitHub, como na Fase 4. Só a `scyther-api` acessa o PostgreSQL. O worker e o notifier não têm banco: se comunicam apenas por mensagens, S3 e Redis, e recebem nas mensagens os dados de que precisam (por exemplo, `userEmail` vai como snapshot em `cmd.video.process`, para o notifier não consultar a API).

## Consequências

- O worker não atualiza o status do vídeo. Ele publica eventos, e a API aplica a máquina de estados.
- Os repositórios são públicos (como na ADR 0017 da Fase 4), para ter SonarCloud gratuito, proteção de branch e acesso da banca.
- Mudanças de contrato entre serviços exigem coordenação entre repositórios, por isso os contratos têm versão (`version` no envelope).

Ver §5.1 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
