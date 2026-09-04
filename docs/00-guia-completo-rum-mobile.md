# Guia completo: RUM Dynatrace em Android e iOS

Este guia conduz uma implementação do zero até uma jornada mobile consultável no Grail. O alvo é o **Latest Dynatrace**. Use os artigos específicos deste repositório como runbooks durante a execução.

## 1. O resultado esperado

Ao final, cada frontend mobile deve entregar:

- app starts `cold`, `warm` e `hot`;
- views com nomes estáveis e transições de navegação;
- ações automáticas e ações de negócio manuais;
- requests HTTP correlacionáveis com traces de backend;
- erros tratados, crashes e ANRs;
- identidade pseudonimizada do usuário, quando houver base legal;
- propriedades de evento e sessão previamente autorizadas;
- consentimento aplicado antes da coleta;
- consultas DQL, critérios de aceite e limites de custo documentados.

Fluxo lógico:

```text
Pessoa → app instrumentado → OneAgent Mobile → beacon Dynatrace
       → user.events → agregador de sessão → user.sessions
       → métricas dt.frontend.* → Experience Vitals / Dashboards / DQL
                         ↘ trace.id → spans do backend
```

## 2. Decisões antes de alterar código

Registre estas decisões em um ADR ou ticket técnico:

| Decisão | Recomendação inicial |
|---|---|
| Frontends | Um frontend por app/plataforma e ambiente que precise de controle próprio |
| Nomes de views | Nomes funcionais e estáveis, sem IDs, e-mail ou conteúdo digitado |
| Ações manuais | Apenas marcos de negócio que a auto-instrumentação não representa bem |
| Identidade | ID interno irreversível/pseudônimo; não usar e-mail em texto puro |
| Propriedades | Pequeno dicionário governado; nunca payloads livres |
| Consentimento | Opt-in integrado à CMP/tela de privacidade |
| Session Replay | Começar restrito, com masking mais seguro e amostragem baixa |
| Ambientes | IDs e beacon URLs separados para dev, homologação e produção |
| Custo | Alertas e revisão mensal da rate card/consumo real |

Não use nomes dinâmicos como `Produto 938271`, `Conta joao@email.com` ou URLs inteiras como nomes de view/ação. Isso aumenta cardinalidade e pode expor dados pessoais.

## 3. Criar o frontend no Dynatrace

1. Abra **Experience Vitals > Overview**.
2. Selecione **Add Frontend** e escolha **Mobile**.
3. Escolha **Android** ou **iOS**.
4. Defina um nome que identifique produto, plataforma e ambiente, por exemplo `Banco Mobile · Android · Produção`.
5. Copie o `applicationId`/`DTXApplicationID` e o `beaconUrl`/`DTXBeaconURL` fornecidos pelo wizard.
6. Em **Settings > Enablement and cost control**, ative a nova experiência de RUM e defina a taxa de captura.
7. Repita para a outra plataforma. Não reutilize por conveniência as chaves de um frontend em outro.

Também é possível habilitar o padrão no nível de ambiente em **Settings > Collect and capture > Real User Monitoring > Enablement and cost control > Mobile**; a configuração específica do frontend pode sobrescrevê-lo.

## 4. Instrumentar Android

Pré-requisitos atuais: Android API 23+, Gradle 8.0+, Android Gradle Plugin 8.1.1+, Java 17 e, quando usado, Kotlin 2.1.0. O plugin atua em projetos `com.android.application`.

No build raiz, inclua o plugin Dynatrace e aplique `com.dynatrace.instrumentation`. Em Kotlin DSL, a estrutura básica é:

```kotlin
buildscript {
    repositories { mavenCentral() }
    dependencies {
        classpath("com.dynatrace.tools.android:gradle-plugin:8.+")
    }
}

apply(plugin = "com.dynatrace.instrumentation")

configure<com.dynatrace.tools.android.dsl.DynatraceExtension> {
    configurations {
        create("prod") {
            autoStart {
                applicationId("SEU_APPLICATION_ID")
                beaconUrl("https://SEU_AMBIENTE/mbeacon")
            }
            userOptIn(true)
            agentBehavior.startupLoadBalancing(true)
            agentBehavior.startupWithGrailEnabled(true)
        }
    }
}
```

Aplique as opções de privacidade após o consentimento, não incondicionalmente no startup de produção:

```kotlin
val privacy = UserPrivacyOptions.builder()
    .withDataCollectionLevel(DataCollectionLevel.USER_BEHAVIOR)
    .withCrashReportingOptedIn(true)
    .build()

Dynatrace.applyUserPrivacyOptions(privacy)
```

Detalhes, variações Groovy e checklist: [Instrumentação Android](02-instrumentacao-android.md).

## 5. Instrumentar iOS

Pré-requisitos atuais: iOS 15+, tvOS 15+ e Xcode 16+. Pelo Swift Package Manager, adicione:

```text
https://github.com/Dynatrace/swift-mobile-sdk.git
```

Selecione o produto `Dynatrace` ou, quando aprovado, `DynatraceSessionReplay`. Adicione as chaves ao `Info.plist` (ou a um `Dynatrace.plist` incluído no target):

```xml
<key>DTXApplicationID</key>
<string>SEU_APPLICATION_ID</string>
<key>DTXBeaconURL</key>
<string>https://SEU_AMBIENTE/mbeacon</string>
<key>DTXUserOptIn</key>
<true/>
<key>DTXStartupLoadBalancing</key>
<true/>
<key>DTXStartupWithGrailEnabled</key>
<true/>
```

Depois do consentimento:

```swift
import Dynatrace

var privacy = Dynatrace.userPrivacyOptions()
privacy.dataCollectionLevel = .userBehavior
privacy.crashReportingOptedIn = true
Dynatrace.applyUserPrivacyOptions(privacy) { success in
    // Atualize a UI/telemetria interna conforme o resultado.
}
```

Detalhes de SwiftUI, UIKit, símbolos e aceite: [Instrumentação iOS](03-instrumentacao-ios.md).

## 6. Mapear a jornada

Modele primeiro a jornada de negócio, depois o código. Exemplo de checkout:

| Etapa | View estável | Ação importante | Propriedades permitidas |
|---|---|---|---|
| Entrada | `home` | `Open catalog` | `session_properties.customer_segment` |
| Descoberta | `catalog` | `Search products` | `event_properties.result_count` |
| Consideração | `product_detail` | `Add to cart` | `event_properties.product_category` |
| Conversão | `checkout` | `Checkout Process` | `event_properties.payment_method`, `cart_value` |
| Confirmação | `order_confirmation` | `Order confirmed` | `event_properties.checkout_successful` |

O OneAgent detecta várias interações e views automaticamente. Use API manual quando a ação:

- atravessa telas ou callbacks assíncronos;
- representa um resultado de negócio;
- precisa agrupar requests, navegações e erros em uma unidade;
- não possui um handler que a auto-instrumentação consiga observar.

Regras essenciais:

1. Inicie a ação no gesto do usuário.
2. Termine em todos os caminhos de sucesso, falha e cancelamento.
3. Adicione propriedades antes de `complete()`.
4. Não aninhe ou duplique uma ação manual sobre uma automática sem necessidade.
5. Garanta que as propriedades estejam permitidas no frontend antes do deploy; propriedades desconhecidas são descartadas na ingestão.

Veja exemplos completos em [Mapeamento da jornada](04-mapeamento-da-jornada.md).

## 7. Identidade e propriedades

Após login e consentimento, associe a sessão a um identificador pseudônimo:

```kotlin
Dynatrace.identifyUser("usr_8f6e2d")
```

```swift
Dynatrace.identifyUser("usr_8f6e2d")
```

No logout, limpe a identidade e encerre a visita quando fizer sentido para separar contextos:

```kotlin
Dynatrace.identifyUser(null)
Dynatrace.endVisit()
```

```swift
Dynatrace.identifyUser(nil)
Dynatrace.endVisit()
```

O identificador não é persistido entre novos ciclos do app; reaplique-o em cada nova sessão autenticada. O campo `user.identifier` é sensível no Grail e exige permissão de fieldset.

No Dynatrace, permita primeiro as chaves em **Frontend > Settings > Capture properties > Allowed API-reported properties**. Use prefixos:

- evento: `event_properties.*`;
- sessão: `session_properties.*`.

## 8. Privacidade e Session Replay

Trate privacidade como requisito de arquitetura:

- `OFF`: nada é enviado;
- `PERFORMANCE`: telemetria de desempenho, com identificadores randomizados a cada início;
- `USER_BEHAVIOR`: permite identidade e dados de comportamento conforme consentimento.

Session Replay mobile pode capturar reconstruções visuais, inclusive contexto anterior a crash. Comece com o nível de masking mais restritivo, teste telas de autenticação/pagamento e só então amplie. Consentimento para RUM, crash reporting e replay deve refletir a escolha real da pessoa.

Veja [Privacidade e Session Replay](05-privacidade-session-replay.md).

## 9. Validar o primeiro evento e a jornada

1. Gere um build limpo e confirme que o agente aparece no artefato.
2. Instale em dispositivo/emulador sem proxy bloqueando o beacon.
3. Conceda o consentimento escolhido.
4. Execute uma jornada: abrir → login → catálogo → carrinho → checkout → background → foreground.
5. Aguarde alguns minutos e verifique **Experience Vitals > Mobile > frontend**.
6. Execute:

```dql
fetch user.events, from: now() - 2h
| filter frontend.name == "Banco Mobile · Android · Produção"
| filter dt.rum.application.type == "mobile"
| summarize eventos = count(), by:{os.name, app.short_version}
```

7. Selecione um `dt.rum.session.id` e monte a linha do tempo conforme o [cookbook DQL](08-cookbook-dql.md).
8. Verifique `trace.id` nos requests para confirmar correlação com o backend.
9. Crashes e ANRs exigem testes controlados; em iOS, o debugger desativa essa captura. Símbolos iOS (`dSYM`) devem ser enviados para stack traces legíveis.

`user.events` aparece rapidamente; `user.sessions` só é materializado depois do fechamento/agregação da sessão, normalmente após mais de 30 minutos de inatividade. Não diagnostique ausência de sessão agregada olhando apenas os últimos minutos.

## 10. Aceite de produção

Considere concluído apenas quando:

- há eventos de ambas as plataformas e versões esperadas;
- views têm nomes funcionais, estáveis e baixa cardinalidade;
- ações críticas fecham em sucesso e falha;
- requests principais têm `trace.id` e chegam ao backend instrumentado;
- erros tratados, crash/ANR e símbolos foram validados;
- identificador e propriedades não contêm PII desnecessária;
- opt-in/opt-out e mudança de consentimento foram testados;
- masking do replay foi aprovado por Segurança/Privacidade;
- DQL e dashboards filtram `dt.rum.user_type == "real_user"` quando apropriado;
- owner, SLOs, alertas, taxa de captura e orçamento foram definidos;
- o consumo real é conferido pelos Billing Usage Events, não por recontagem de eventos brutos.

## 11. Operação contínua

- Compare qualidade por `app.short_version` antes e depois de releases.
- Acompanhe p75/p90 de app start e ações, crash-free sessions e requests falhos.
- Revise mensalmente propriedades, cardinalidade, sampling e Session Replay.
- Mantenha um teste de fumaça por plataforma em cada release.
- Use `dt.system.events` com `event.kind == "BILLING_USAGE_EVENT"` como fonte autoritativa de consumo faturável.
- Atualize os pré-requisitos do pipeline quando o SDK/plugin for atualizado.

## Referências oficiais

- [Mobile frontends](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends)
- [Setup Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-01-initial-setup)
- [Setup iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-01-initial-setup)
- [Propriedades de evento e sessão](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/additional-configuration/event-and-session-properties)
- [Privacidade mobile](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/data-privacy)
- [Consumo de RUM (DPS)](https://docs.dynatrace.com/docs/license/capabilities/real-user-synthetic-monitoring/real-user-monitoring)

Documentação verificada em 4 de setembro de 2026.

