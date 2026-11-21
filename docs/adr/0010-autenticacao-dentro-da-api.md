---
status: proposed
---

# Autenticação dentro da scyther-api

Cadastro e login ficam na própria `scyther-api`: senha com argon2id (`Bun.password`), JWT HS256 com expiração de 1 h e sem refresh token (como na ADR 0004 da Fase 4), e rate limit de login por IP/e-mail no Redis. Na Fase 4 a autenticação era uma Lambda atrás do API Gateway. O enunciado só pede usuário e senha, e um serviço a menos libera tempo para os diferenciais.

## Alternativas consideradas

- **Lambda + API Gateway (Fase 4)**: mais uma peça para implantar e testar, sem requisito que justifique.
- **Cognito ou provedor externo**: lock-in e configuração no Academy, para um requisito simples.

## Consequências

- Sem recuperação de senha, OAuth ou MFA (não-objetivos).
- Sem refresh token, o usuário faz login de novo a cada hora.

Ver §5.7 e §8 do [design](../superpowers/specs/2026-11-18-scyther-hackathon-design.md).
