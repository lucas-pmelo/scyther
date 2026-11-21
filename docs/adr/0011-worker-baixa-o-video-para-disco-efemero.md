---
status: proposed
---

# Worker baixa o vídeo para disco efêmero

O worker baixa o vídeo original do S3 para um `emptyDir` do pod antes de chamar ffprobe e ffmpeg, em vez de passar uma URL pré-assinada para o ffmpeg ler via HTTP. Alguns formatos (por exemplo, MP4 com o `moov` no fim do arquivo) exigem seek, que via HTTP vira várias requisições de range e falhas difíceis de diagnosticar. Do disco local a leitura é previsível.

## Consequências

- O `emptyDir` tem limite proporcional ao tamanho máximo do vídeo (padrão 2 GB) vezes `WORKER_CONCURRENCY`, e isso entra no dimensionamento dos nós.
- Os frames extraídos também vão para o disco efêmero e são apagados conforme entram no zip.
- Se o pod morrer, o disco some com ele, e a mensagem volta para a fila (ver [ADR 0005](0005-quorum-queues-com-retry-por-filas-de-espera-e-dlq.md)).

Ver §5.6 e §8 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
