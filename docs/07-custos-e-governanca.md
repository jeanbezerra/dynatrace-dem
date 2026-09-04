# Custos e governança do RUM

Este artigo explica a lógica implementada em `calculadora.html`. É uma estimativa educacional. O custo real vem da rate card contratada e dos Billing Usage Events (BUEs).

## Dois modelos de licenciamento

### Dynatrace Platform Subscription (DPS)

Valores públicos de exemplo documentados, em USD:

| Item | Preço de lista usado |
|---|---:|
| RUM sem Session Replay | US$ 0,00225 por sessão |
| RUM com Session Replay | US$ 0,0045 por sessão |
| RUM Property | US$ 0,0001 por propriedade faturável por sessão |

Esses preços podem ser diferentes da sua rate card. A calculadora deixa todos editáveis.

### Licenciamento Classic/DEM

| Item | Consumo documentado |
|---|---:|
| Sessão RUM sem replay | 0,25 DEM |
| Sessão capturada com Session Replay | 1,0 DEM |
| Propriedade adicional por sessão | 0,01 DEM |

O preço monetário por DEM é contratual; por isso, a calculadora só converte DEM em moeda quando o usuário informa esse valor.

## Regras de contagem

- uma visita a cada aplicação conta separadamente;
- visita acima de uma hora gera uma sessão faturável para cada hora iniciada;
- uma aplicação híbrida que combina componente mobile e web é exceção e conta como uma sessão;
- sessão com apenas uma ação é bounce e não consome RUM;
- aplicações com menos de duas ações por hora não geram consumo de sessão;
- sessões synthetic/robot não entram no consumo de sessões reais;
- sessões com replay usam a linha de preço “RUM with Session Replay”, não somam a linha base novamente;
- até 20 propriedades de sessão por aplicação estão incluídas;
- propriedades acima de 20 são faturadas por sessão em que aparecem.

## Fórmula DPS

Defina:

- `V`: visitas brutas no mês;
- `B`: taxa de bounce;
- `A`: aplicações faturadas por visita;
- `H`: blocos de hora iniciados por visita/aplicação;
- `C`: taxa de captura RUM;
- `R`: taxa de replay sobre sessões capturadas;
- `P`: propriedades além das 20 incluídas;
- `O`: presença média das propriedades nas sessões;
- `pRum`, `pReplay`, `pProp`: preços da rate card.

```text
sessoes = V × (1 − B) × A × H × C
comReplay = sessoes × R
semReplay = sessoes − comReplay
ocorrenciasPropriedade = sessoes × P × O

custo = semReplay × pRum
      + comReplay × pReplay
      + ocorrenciasPropriedade × pProp
```

`H` é um fator médio. Como o faturamento usa cada hora iniciada, não calcule simplesmente `média de minutos / 60`. Exemplo: visita de 70 minutos conta 2. Se houver distribuição real, estime `H` por faixas: `1 × %até60 + 2 × %61–120 + 3 × %121–180 + ...`.

## Exemplo

Considere 5.000 visitas/dia, 30 dias, 10% de bounce, uma app por visita, um bloco de hora, RUM 100%, replay 20% e 25 propriedades presentes em todas as sessões.

```text
visitas mensais = 150.000
sessoes faturáveis = 150.000 × 90% = 135.000
sem replay = 108.000 × US$ 0,00225 = US$ 243,00
com replay = 27.000 × US$ 0,0045 = US$ 121,50
propriedades extras = (25 − 20) × 135.000 = 675.000
custo propriedades = 675.000 × US$ 0,0001 = US$ 67,50
total mensal = US$ 432,00
```

## Fonte autoritativa: BUE

Não tente reproduzir a fatura contando `user.events`: as regras já são aplicadas em `billing_usage_event`. Use `dt.system.events`:

```dql
fetch dt.system.events
| filter event.kind == "BILLING_USAGE_EVENT"
| filter event.type == "Real User Monitoring"
    or event.type == "Real User Monitoring with Session Replay"
    or event.type == "Real User Monitoring Property"
| fieldsAdd app = coalesce(dt.entity.application, dt.entity.device_application),
    platform = if(isNotNull(dt.entity.application), "Web", else:"Mobile")
| dedup event.id
| summarize
    rum = sum(billed_sessions),
    replay = sum(billed_replay_sessions),
    properties = sum(billed_property_sessions),
    by:{platform, app, event.type}
```

Os BUEs já refletem sessões por aplicação/hora, exclusão de bounce e ocorrências faturáveis. Concilie com **Account Management > Subscription > Overview > Cost and usage details > Usage summary**.

## Controles de custo

1. **Sampling RUM:** reduza de forma consciente e preserve jornadas críticas.
2. **Sampling de replay:** normalmente deve ser menor que a captura RUM.
3. **Propriedades:** revise as acima das 20 incluídas e a presença real por sessão.
4. **Separação por app:** frontends demais podem multiplicar sessões por visita.
5. **Duração:** usuários operacionais com app aberto por horas geram blocos adicionais.
6. **Ambientes:** não deixe testes de carga alimentarem produção.
7. **BUE por aplicação:** atribua owner e budget.
8. **Alertas:** compare consumo diário com baseline e orçamento mensal.

Reduzir coleta pode prejudicar diagnóstico. Faça mudanças com hipótese, janela de observação e critério de rollback.

## O que a calculadora não faz

- não consulta a rate card do tenant;
- não converte câmbio automaticamente;
- não prevê desconto, compromisso ou overage;
- não inclui Synthetic, logs, spans ou outras capacidades;
- não substitui BUE/fatura;
- usa médias para bounce, presença de propriedades e blocos de hora.

## Referências oficiais

- [Consumo de RUM no DPS](https://docs.dynatrace.com/docs/license/capabilities/real-user-synthetic-monitoring/real-user-monitoring)
- [DEM units — Classic](https://docs.dynatrace.com/docs/license/classic-licensing/digital-experience-monitoring-units)
- [Visão de RUM e Synthetic no DPS](https://docs.dynatrace.com/docs/license/capabilities/real-user-synthetic-monitoring)

Valores e regras verificados em 4 de setembro de 2026.

