---
status: accepted
---

# Zip sem compressão, em streaming para o S3

O zip dos frames usa o método `Store` (sem compressão), e não `Deflate` como no protótipo. PNG e JPEG já são comprimidos, então o Deflate gastava CPU sem reduzir o tamanho de forma relevante. O zip é escrito em streaming (`io.Pipe` + upload multipart) direto para o S3, sem materializar o arquivo em disco.

## Consequências

- Não "otimize" isto voltando para `Deflate`: a economia é desprezível e o custo de CPU disputa com o ffmpeg dentro do pod.
- Para reduzir o tamanho de verdade, a alavanca é o formato do frame: JPEG (sugerido no front) é de 5 a 10x menor que PNG.
- Como o tamanho final não é conhecido de antemão, `zipSizeBytes` é medido ao fim do upload e enviado em `evt.video.completed`.

Ver §5.17 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
