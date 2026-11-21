---
status: accepted
---

# Upload e download direto no S3 por URL pré-assinada

O vídeo nunca passa pela API. Em `POST /videos`, a API devolve URLs pré-assinadas de upload multipart, e o navegador envia as partes direto ao S3. Em `POST /videos/:id/complete`, a API conclui o multipart e confere o objeto (HEAD, tamanho). O download do zip e da miniatura é um 302 para uma URL pré-assinada de GET, depois de a API checar se o vídeo é do usuário. Com isso um pico de uploads não derruba a API, os arquivos já ficam duráveis no S3 antes de qualquer processamento, e a API roda com pouca memória.

## Alternativas consideradas

- **Upload pela API** (como no protótipo): simples, mas a API passaria a segurar gigabytes em memória ou disco e viraria o gargalo justamente no pico que o enunciado pede para suportar.

## Consequências

- O bucket precisa de CORS liberado só para a origem do front.
- URLs curtas: 15 min para upload e 5 min para download. As chaves incluem o `userId`.
- Uploads que nunca chamam `complete` deixam o vídeo em `PENDING_UPLOAD`. Um job da API marca `ABANDONED` depois de 24 h, e um lifecycle do S3 aborta multipart incompleto em 1 dia.
- Como a API não vê os bytes, a validação real do arquivo é feita pelo worker (ffprobe).

Ver §5.2 e §5.16 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
