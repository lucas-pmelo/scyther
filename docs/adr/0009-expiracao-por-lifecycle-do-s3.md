---
status: accepted
---

# Expiração por lifecycle do S3, mantendo o original por 7 dias

Os arquivos em `uploads/` e `outputs/` expiram por uma regra nativa de lifecycle do S3, 7 dias depois da criação do objeto. O worker **não** apaga o vídeo original depois de processar: ele fica disponível para reprocessar (após falha ou com outro FPS) enquanto não expira. Um job da API marca como `EXPIRED` os vídeos com `expires_at` vencido, e o `retry` só é aceito antes disso.

## Alternativas consideradas

- **Apagar o original logo após processar**: economiza armazenamento, mas impede o reprocessamento, que é um dos extras do produto.
- **Job da API apagando objetos**: duplica o que o S3 faz sozinho e falha se o job parar.

## Consequências

- O estado no banco (`EXPIRED`) e a existência do objeto no S3 são controlados por mecanismos diferentes. O `expires_at` gravado na API precisa usar a mesma regra de 7 dias do lifecycle.
- Localmente, o MinIO precisa da mesma regra de lifecycle.

Ver §5.10 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
