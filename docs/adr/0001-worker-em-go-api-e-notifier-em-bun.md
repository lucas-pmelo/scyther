---
status: accepted
---

# Worker em Go, API e notifier em Bun

O protótipo dos investidores é um `main.go` que chama o ffmpeg. Reescrevemos a API e o notifier em Bun + Elysia, reaproveitando o template `templates/service` da Fase 4 (Clean Architecture, OTel, Sonar, CI). O worker é escrito em Go a partir do núcleo do protótipo. Ele é pequeno (consumir fila, ffmpeg, zip, S3, eventos) e faz o trabalho pesado de CPU e I/O, para o qual Go se encaixa melhor que Bun. Dois toolchains custam pouco para um serviço desse tamanho.

## Alternativas consideradas

- **Tudo em Bun**, com o worker chamando o ffmpeg por subprocesso: reaproveitaria o máximo da Fase 4, mas não teria nada de novo para mostrar, e Bun é menos adequado a trabalho pesado.
- **Tudo em Go**: ficaria perto do protótipo, mas jogaria fora o template, os testes e o ritmo da equipe em Bun.

## Consequências

- O padrão de testes, Sonar, lint (golangci-lint) e OTel precisa ser replicado em Go.
- Os contratos de mensagem viram JSON Schema validado nos dois lados (Bun e Go), porque não há tipos compartilhados.
- Um trace distribuído atravessa as duas linguagens, o que é um diferencial na apresentação.

Ver §4 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
