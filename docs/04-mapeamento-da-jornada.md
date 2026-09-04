# Como mapear a jornada RUM mobile

Instrumentar tudo não equivale a explicar uma jornada. Um bom mapa combina cobertura automática para diagnóstico técnico com poucos marcos de negócio, nomes estáveis e propriedades governadas.

## 1. Comece pelo resultado de negócio

Escolha uma jornada crítica e descreva:

- entrada e saída;
- etapas obrigatórias e alternativas;
- sinal de sucesso;
- sinais de abandono/falha;
- dependências HTTP/backend;
- dimensões necessárias para segmentar o resultado.

Exemplo:

```text
app_start
  → home
  → catalog ──(sem resultado)──→ empty_search
  → product_detail
  → cart
  → checkout ──(erro)──────────→ payment_error
  → order_confirmation
```

## 2. Traduza o mapa para o modelo Dynatrace

| Camada | Pergunta | Exemplo |
|---|---|---|
| Frontend | Qual produto/plataforma/ambiente? | `Store · iOS · Prod` |
| Session | Quem/qual contexto visita o app? | segmento, tier, feature flag |
| View | Em qual tela/estado visível? | `checkout` |
| User action | Qual operação significativa ocorreu? | `Checkout Process` |
| Event property | Qual contexto pertence à ocorrência? | método de pagamento, sucesso |
| Request/trace | Qual dependência respondeu? | `/payments/confirm`, `trace.id` |
| Error | Por que a etapa falhou? | código normalizado, exception |

Não modele cada toque como uma ação manual. Interações automáticas respondem “o que a pessoa tocou”; ações manuais respondem “qual operação de negócio esse gesto iniciou e quando terminou”.

## 3. Convenção de nomes

### Views

- use inglês ou português de forma consistente;
- use substantivos de tela: `home`, `product_detail`, `checkout`;
- não use valores variáveis: `product_8731`, `checkout_user_42`;
- não use texto visível que possa conter PII;
- documente aliases antigos durante migrações.

### Ações

- verbo + objeto/resultado: `Search products`, `Add to cart`, `Checkout Process`;
- o nome deve sobreviver a uma troca de componente visual;
- mantenha conjunto pequeno e intencional;
- não inclua status no nome; use `event_properties.*`.

### Propriedades

Use namespace semântico, tipo estável e valores enumerados quando possível:

| Chave | Tipo | Cardinalidade | Observação |
|---|---|---:|---|
| `session_properties.customer_segment` | string | baixa | `retail`, `business` |
| `session_properties.experiment.checkout` | string | baixa | variante do experimento |
| `event_properties.payment_method` | string | baixa | sem dados do cartão |
| `event_properties.cart_value` | number | contínua | moeda documentada separadamente |
| `event_properties.checkout_successful` | boolean | 2 | resultado da ação |
| `event_properties.failure_reason` | string | baixa | catálogo normalizado |

Evite `order_id`, e-mail, CPF, token e mensagem de erro crua, salvo necessidade formalmente aprovada. Identificadores de alta cardinalidade raramente ajudam dashboards agregados e elevam risco.

## 4. Dicionário de instrumentação

Mantenha um artefato versionado como este:

| ID | Etapa | Plataforma | View | Ação | Início | Término | Propriedades | Owner |
|---|---|---|---|---|---|---|---|---|
| J01 | Abrir app | ambas | `home` | automática | launch | primeiro frame | — | Mobile Platform |
| J02 | Buscar | ambas | `catalog` | `Search products` | submit | resultados/erro | `result_count` | Catalog |
| J03 | Pagar | ambas | `checkout` | `Checkout Process` | toque em pagar | confirmação/falha/cancelamento | `payment_method`, `checkout_successful`, `failure_reason` | Payments |

Inclua ainda: base legal, dado pessoal, retenção, SLO, dashboard e teste de aceite.

## 5. Instrumentação automática versus manual

Prefira automática para:

- lifecycle/app start;
- views e navegações suportadas;
- toques com action handler;
- requests de frameworks suportados;
- crashes/exceções não tratadas.

Use manual para:

- fluxos multi-etapas;
- UI customizada não detectada;
- tarefas assíncronas/background importantes;
- requests de biblioteca não suportada;
- erros tratados relevantes;
- resultado de negócio e contexto controlado.

## 6. Padrão de ação manual

### Android/Kotlin

```kotlin
private var action: UserAction? = null

fun onPaySelected(total: Double, method: String) {
    action = Dynatrace.createUserAction(
        UserActionConfiguration("Checkout Process")
    ).apply {
        addEventProperty("event_properties.cart_value", total)
        addEventProperty("event_properties.payment_method", method)
    }
}

fun onPaymentResult(success: Boolean, reason: String?) {
    action?.apply {
        addEventProperty("event_properties.checkout_successful", success)
        reason?.let {
            addEventProperty("event_properties.failure_reason", it)
        }
        complete()
    }
    action = null
}
```

### iOS/Swift

```swift
private var action: DTXUserAction?

func onPaySelected(total: Double, method: String) {
    let config = DTXUserActionConfiguration(customName: "Checkout Process")
    action = Dynatrace.createUserAction(configuration: config)
    action?.addEventProperty("event_properties.cart_value", value: total)
    action?.addEventProperty("event_properties.payment_method", value: method)
}

func onPaymentResult(success: Bool, reason: String?) {
    action?.addEventProperty(
        "event_properties.checkout_successful",
        value: success
    )
    if let reason {
        action?.addEventProperty(
            "event_properties.failure_reason",
            value: reason
        )
    }
    action?.complete()
    action = nil
}
```

Em ambos, finalize em `success`, `failure`, cancelamento e descarte da tela. Propriedades após `complete()` são ignoradas.

## 7. Preparar propriedades no Dynatrace

1. Abra o frontend em **Experience Vitals**.
2. Vá a **Settings > Capture properties**.
3. Em **Allowed API-reported properties**, crie cada chave.
4. Defina tipo e validação de caixa.
5. Só então faça o deploy do código.

Limites atuais documentados para o Latest RUM incluem nome de até 100 caracteres, até 200 propriedades API permitidas por frontend, até 50 propriedades por evento, até 500 propriedades de sessão por sessão e string de até 1.000 caracteres. Limite técnico não é meta: mantenha um catálogo muito menor.

Propriedades de evento podem virar propriedades de sessão via OpenPipeline. Exemplo para manter o maior carrinho da sessão:

```dql
fieldsAdd session_properties.cart.total_value = event_properties.cart.total_value,
  session_properties_aggregation.cart.total_value = "Max"
```

## 8. Identificar usuário com segurança

Chame `identifyUser()` após autenticação e consentimento. Recomenda-se um identificador interno pseudônimo, sem valor direto fora do sistema. Limpe no logout e reaplique em nova sessão.

`user.identifier` é campo sensível. Para análises anônimas, `dt.rum.instance.id` oferece continuidade pseudônima da instância sem representar uma identidade real.

## 9. Confirmar a jornada no Grail

Encontre sessões do frontend:

```dql
fetch user.events, from: now() - 2h
| filter frontend.name == "Store · Android · Prod"
| filter dt.rum.application.type == "mobile"
| summarize
    eventos = count(),
    acoes = countIf(characteristics.has_user_action),
    views = countIf(characteristics.has_view_summary),
    erros = countIf(characteristics.has_error),
    by:{dt.rum.session.id}
| sort eventos desc
| limit 20
```

Escolha um ID e monte a timeline:

```dql
fetch user.events, from: now() - 510m
| filter dt.rum.session.id == "SESSAO"
| fieldsAdd event_type = if(characteristics.has_error, "Error",
    else: if(characteristics.has_view_summary, "View summary",
    else: if(characteristics.has_app_start, "App start",
    else: if(characteristics.has_user_action, "User action",
    else: if(characteristics.has_navigation, "Navigation",
    else: if(characteristics.has_request, "Request", else: "Other"))))))
| fields start_time, event_type, view.name,
    user_action.custom_name, interaction.type, url.path,
    http.response.status_code, error.type, trace.id
| sort start_time asc
```

## 10. Medir conversão e abandono

As propriedades de resultado permitem uma visão simples do checkout:

```dql
fetch user.events, from: now() - 7d
| filter frontend.name == "Store · Android · Prod"
| filter characteristics.has_user_action
| filter user_action.custom_name == "Checkout Process"
| summarize
    iniciados = count(),
    sucesso = countIf(event_properties.checkout_successful == true),
    falha = countIf(event_properties.checkout_successful == false),
    by:{app.short_version, event_properties.payment_method}
| fieldsAdd conversao_pct = 100.0 * sucesso / iniciados
| sort iniciados desc
```

Ausência de resultado não significa automaticamente abandono: pode indicar ação não finalizada, evento descartado, sampling ou app terminado. Mantenha a semântica de conclusão explícita.

## 11. Checklist de qualidade

- [ ] nomes iguais em iOS e Android para o mesmo conceito;
- [ ] nenhuma PII em views, ações ou propriedades;
- [ ] todas as propriedades estão permitidas antes do deploy;
- [ ] ações fecham em todos os caminhos;
- [ ] requests principais têm `trace.id`;
- [ ] versão do app está disponível;
- [ ] jornada feliz e falhas foram reproduzidas;
- [ ] dashboard usa percentis, não apenas média;
- [ ] filtros separam plataforma/versão quando necessário;
- [ ] taxa de captura e custo foram documentados.

## Referências oficiais

- [Ações Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-10-user-actions)
- [Ações iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-10-user-actions)
- [Propriedades de evento e sessão](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/additional-configuration/event-and-session-properties)
- [Dicionário: user actions](https://docs.dynatrace.com/docs/semantic-dictionary/model/rum/user-events/user-actions)

