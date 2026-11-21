---
status: proposed
---

# E-mail via Mailpit, com SMTP real configurável

O notifier envia e-mails por SMTP (Nodemailer) para um Mailpit rodando no cluster, que é o padrão tanto local quanto na nuvem. Ele sempre funciona e aparece bem no vídeo. Um SMTP real (Gmail com senha de app ou Brevo) fica configurável por variável de ambiente, para usar se a saída SMTP do Academy permitir. O SES do Academy fica em sandbox e só envia para endereços verificados.

## Consequências

- Na configuração padrão, o e-mail de falha não chega à caixa real do usuário. Ele aparece na interface do Mailpit, e é isso que a demo mostra.
- Trocar para SMTP real é só configuração, sem mudança de código.

Ver §5.8 e §8 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
