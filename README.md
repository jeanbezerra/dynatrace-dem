# Dynatrace RUM Mobile — guia de estudo

Material em português para instrumentar, configurar, validar e mapear jornadas de Real User Monitoring (RUM) em aplicativos Android e iOS no **Latest Dynatrace / Grail**.

## Comece aqui

1. Leia o [guia completo](docs/00-guia-completo-rum-mobile.md).
2. Siga o roteiro da sua plataforma: [Android](docs/02-instrumentacao-android.md) ou [iOS](docs/03-instrumentacao-ios.md).
3. Modele a jornada com o [guia de mapeamento](docs/04-mapeamento-da-jornada.md).
4. Use o [cookbook DQL](docs/08-cookbook-dql.md) para validar os dados no Grail.
5. Abra [calculadora.html](calculadora.html) no navegador para estimar o consumo e o custo.

## Artigos

| Arquivo | Assunto |
|---|---|
| [00-guia-completo-rum-mobile.md](docs/00-guia-completo-rum-mobile.md) | Visão ponta a ponta e plano de implementação |
| [01-arquitetura-e-modelo-de-dados.md](docs/01-arquitetura-e-modelo-de-dados.md) | Arquitetura, sessões, eventos, views, ações e traces |
| [02-instrumentacao-android.md](docs/02-instrumentacao-android.md) | Instalação e configuração Android |
| [03-instrumentacao-ios.md](docs/03-instrumentacao-ios.md) | Instalação e configuração iOS |
| [04-mapeamento-da-jornada.md](docs/04-mapeamento-da-jornada.md) | Taxonomia, funil, propriedades e instrumentação manual |
| [05-privacidade-session-replay.md](docs/05-privacidade-session-replay.md) | Consentimento, LGPD, masking e Session Replay |
| [06-validacao-e-troubleshooting.md](docs/06-validacao-e-troubleshooting.md) | Testes, aceite e diagnóstico de falhas |
| [07-custos-e-governanca.md](docs/07-custos-e-governanca.md) | DPS, DEM legado, rate card e governança de consumo |
| [08-cookbook-dql.md](docs/08-cookbook-dql.md) | Consultas prontas para jornada e qualidade mobile |

## Escopo e data de referência

O material prioriza o **RUM no Latest Dynatrace**, com `user.events`, `user.sessions` e métricas `dt.frontend.*`. A calculadora também contém um modo **Classic/DEM** apenas para contratos legados. Documentação oficial verificada em **4 de setembro de 2026**.

> Os snippets usam valores fictícios. Nunca publique `applicationId`, URLs internas, identificadores pessoais ou dados de clientes em repositórios públicos.

