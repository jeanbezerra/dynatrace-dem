# Arquitetura e modelo de dados do RUM mobile

## O que é um frontend

No Latest Dynatrace, um frontend é a representação da interface usada pelo cliente. Em mobile, o mapeamento é determinado pelos identificadores embutidos no app durante a instrumentação. Cada evento carrega o contexto do frontend, da plataforma, da versão e da sessão.

Evite misturar produção e teste no mesmo frontend. Separar ambientes reduz ruído, permite taxas de captura distintas e melhora o controle de acesso.

## O pipeline de dados

```text
SDK / OneAgent Mobile
  ├─ instrumentação automática: lifecycle, views, taps, HTTP, crash/ANR
  ├─ APIs: view, user action, erro, identidade, propriedades
  └─ consentimento e sampling
             ↓ beacon
Grail
  ├─ user.events      evento individual e diagnóstico
  ├─ user.sessions    agregado após a sessão fechar
  └─ dt.frontend.*    tendências, dashboards e alertas
             ↓ trace.id
  spans               análise do backend
```

Comece por métricas para enxergar a forma do problema, vá a `user.events` para a causa e use `user.sessions` para perguntas de jornada, retenção e impacto por usuário.

## Eventos que compõem a jornada

| Conceito | O que representa | Campos/filtro úteis |
|---|---|---|
| App start | Inicialização cold, warm ou hot | `characteristics.has_app_start`, `app_start.type`, `duration` |
| View | Tela visível e seu contexto | `view.name`, `view.sequence_number` |
| Navigation | Transição entre views/estados | `characteristics.has_navigation` |
| View summary | Resumo ao encerrar uma view | `characteristics.has_view_summary`, `view.foreground_time` |
| User interaction | Toque, clique, gesto | `characteristics.has_user_interaction`, `interaction.type` |
| User action | Operação significativa e seus efeitos | `characteristics.has_user_action`, `user_action.*` |
| Request | Chamada HTTP vista pelo app | `characteristics.has_request`, `url.*`, `trace.id` |
| Error | Exceção, crash, ANR ou request falho | `characteristics.has_error`, `error.*`, `exception.*` |

Uma ação de usuário agrupa a interação disparadora, requests, navegações e erros resultantes. Uma view é contexto de tela; não a substitua por ações de negócio.

## Identificadores

- `dt.rum.session.id`: identifica uma sessão/visita.
- `dt.rum.instance.id`: identificador pseudônimo persistente do dispositivo/instância.
- `user.identifier`: identidade fornecida pela aplicação; use apenas quando necessário e com permissão.
- `user_action.instance_id`: identifica uma ação.
- `trace.id`: conecta um request do frontend aos spans do backend.
- `frontend.name`: filtro preferencial pelo nome do frontend.

Em `user.sessions`, `frontend.name` é um array porque a sessão pode envolver mais de um frontend; aplique `expand frontend.name` antes de agrupar.

## Diferenças entre `user.events` e `user.sessions`

| Pergunta | Fonte |
|---|---|
| Qual request falhou? | `user.events` |
| Em qual tela ocorreu um crash? | `user.events` |
| Qual a sequência exata da visita? | `user.events`, ordenado por `start_time` |
| Quantas sessões tiveram erro? | `user.sessions` |
| Qual foi a duração e o motivo de encerramento? | `user.sessions` |
| Qual usuário identificado foi afetado? | `user.sessions` |

Campos agregados em `user.sessions` usam underscore, por exemplo `user_action_count`, `request_count`, `view_summary_count`; contadores de erro continuam com ponto, como `error.count` e `error.has_crash`.

Sessões recentes ainda podem não existir em `user.sessions`: o agregador aguarda o encerramento, tipicamente mais de 30 minutos de inatividade. Uma consulta de sessão por período retorna sessões **iniciadas** no intervalo. Amplie o lookback para cobrir sessões longas.

## Views e navegação por plataforma

### Android

O agente observa lifecycle de `Activity`/views e Jetpack Compose suportado. O nome automático pode ser uma classe técnica. Para uma taxonomia funcional, defina view manual quando necessário e avalie exclusões para telas internas.

### iOS

UIKit é observado por callbacks e swizzling de lifecycle de `UIViewController`. Em SwiftUI, o instrumentador build-time reconhece padrões como `NavigationLink`, `navigationDestination`, `sheet` e `popover`. Cenários customizados podem usar `Dynatrace.startView(name:)`.

Somente uma view fica ativa. Iniciar uma nova encerra a anterior e gera navegação.

## Correlação com backend

Para RUM mobile no Latest Dynatrace, o agente propaga W3C Trace Context pelos headers `traceparent` e `tracestate` em requests suportados. O request em `user.events` recebe `trace.id` quando a correlação funciona.

Evite joins amplos entre `user.events` e `spans`. Primeiro localize um request lento/falho e seu `trace.id`; depois consulte `spans` numa janela curta:

```dql
fetch spans, from:"2026-09-04T12:00:00Z", to:"2026-09-04T12:05:00Z"
| filter trace.id == toUid("TRACE_ID")
| fields start_time, span.name, duration, span.kind, dt.smartscape.service
| sort start_time asc
```

`trace.id` em spans é UID; sem `toUid()` o filtro por string pode retornar zero.

## Retenção e acesso

Na documentação consultada, `user.events`, `user.sessions` e user replays têm retenção padrão de 35 dias no Latest Dynatrace, sujeita ao contrato e a programas de retenção estendida. Campos como `user.identifier` e `client.ip` pertencem ao fieldset sensível e ficam ocultos sem política apropriada:

```text
ALLOW storage:fieldsets:read WHERE storage:fieldset-name="builtin-sensitive-user-events-and-sessions"
```

Conceda essa permissão pelo menor privilégio e não a use como justificativa para coletar PII desnecessária.

## Referências oficiais

- [Conceito de frontends](https://docs.dynatrace.com/docs/observe/digital-experience/rum/concepts/frontends)
- [Dicionário de user events](https://docs.dynatrace.com/docs/semantic-dictionary/model/rum/user-events)
- [User actions](https://docs.dynatrace.com/docs/semantic-dictionary/model/rum/user-events/user-actions)
- [App performance Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-05-app-performance)
- [App performance iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-05-app-performance)
- [Retenção](https://docs.dynatrace.com/docs/manage/data-privacy-and-security/data-privacy/data-retention-periods)

