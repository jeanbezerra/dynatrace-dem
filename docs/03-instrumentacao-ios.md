# Instrumentação do RUM em iOS

Este runbook cobre aplicativo iOS nativo em Swift/Objective-C, UIKit ou SwiftUI, com OneAgent adicionado por Swift Package Manager.

## 1. Pré-requisitos

| Componente | Mínimo atual documentado |
|---|---:|
| iOS | 15.0 |
| tvOS | 15.0 |
| Xcode | 16.0 |

Session Replay não está disponível em tvOS. Verifique também conflitos com ferramentas que fazem swizzling ou instrumentação de requests/crashes.

## 2. Criar o frontend e obter as chaves

Em **Experience Vitals > Add Frontend > Mobile > iOS**, crie o frontend. Copie exatamente:

- `DTXApplicationID`;
- `DTXBeaconURL`.

Separe produção e não produção. O wizard do tenant é a fonte preferencial do snippet compatível com o SDK selecionado.

## 3. Adicionar o pacote

No Xcode:

1. **File > Add Package Dependencies…**;
2. informe `https://github.com/Dynatrace/swift-mobile-sdk.git`;
3. selecione a faixa de versão aprovada pela organização (a integração atual parte da família 8.x);
4. adicione `Dynatrace` ao target do app;
5. use `DynatraceSessionReplay` somente quando replay tiver sido aprovado.

Faça um build antes de usar imports/APIs do SDK.

## 4. Configurar o plist

Adicione ao `Info.plist` ou a um `Dynatrace.plist` com target membership correto:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DTXApplicationID</key>
    <string>SEU_DYNATRACE_APPLICATION_ID</string>
    <key>DTXBeaconURL</key>
    <string>https://SEU_AMBIENTE/mbeacon</string>
    <key>DTXUserOptIn</key>
    <true/>
    <key>DTXStartupLoadBalancing</key>
    <true/>
    <key>DTXStartupWithGrailEnabled</key>
    <true/>
</dict>
</plist>
```

`DTXStartupWithGrailEnabled` permite enviar ao Grail desde o primeiro start, antes de a configuração do cluster ser armazenada. Confirme todas as chaves na documentação da versão usada.

## 5. Aplicar consentimento

Em SwiftUI, o SDK pode ser importado no arquivo do `@main`; em UIKit, no `AppDelegate`. A decisão deve vir da tela de privacidade/CMP:

```swift
import Dynatrace

func applyObservabilityConsent(_ consent: Consent) {
    var privacy = Dynatrace.userPrivacyOptions()
    privacy.dataCollectionLevel = consent.rum
        ? (consent.userBehavior ? .userBehavior : .performance)
        : .off
    privacy.crashReportingOptedIn = consent.crashReporting
    // Quando aplicável e consentido:
    // privacy.crashReplayOptedIn = consent.crashReplay

    Dynatrace.applyUserPrivacyOptions(privacy) { success in
        // Trate/registre localmente o resultado sem PII.
    }
}
```

No quick start, isso pode ficar em `App.init()` ou `application(_:didFinishLaunchingWithOptions:)`; em produção, mova a decisão para o fluxo real de consentimento.

## 6. Views e lifecycle

UIKit tem rastreamento automático de `UIViewController`. SwiftUI usa instrumentação build-time para padrões de navegação suportados. Para uma tela não detectada ou um nome funcional:

```swift
Dynatrace.startView(name: "checkout")
```

Iniciar uma view encerra a anterior. Se optar por controle totalmente manual:

```xml
<key>DTXInstrumentLifecycleMonitoring</key>
<false/>
```

Essa decisão remove cobertura automática e exige instrumentar todas as views relevantes. Evite nomes contendo valores de UI/usuário.

## 7. Ações de usuário manuais

```swift
import Dynatrace

private var checkoutAction: DTXUserAction?

func beginCheckout(cartTotal: Double, itemCount: Int) {
    let config = DTXUserActionConfiguration(customName: "Checkout Process")
    checkoutAction = Dynatrace.createUserAction(configuration: config)
    checkoutAction?.addEventProperty(
        "event_properties.cart_value",
        value: cartTotal
    )
    checkoutAction?.addEventProperty(
        "event_properties.item_count",
        value: itemCount
    )
}

func finishCheckout(success: Bool, reason: String? = nil) {
    checkoutAction?.addEventProperty(
        "event_properties.checkout_successful",
        value: success
    )
    if let reason {
        checkoutAction?.addEventProperty(
            "event_properties.failure_reason",
            value: reason
        )
    }
    checkoutAction?.complete()
    checkoutAction = nil
}
```

Para fechar automaticamente depois dos requests pendentes, use `withCompleteAutomatically(true)` conforme a API atual. Para desligar apenas a criação automática:

```swift
Dynatrace.setAutomaticUserActionDetectionEnabled(false)
```

Ações manuais continuam funcionando.

## 8. Identidade e propriedades de sessão

```swift
Dynatrace.identifyUser("usr_8f6e2d")

Dynatrace.sendSessionPropertyEvent(
    DTXSessionPropertyEventData()
        .addSessionProperty(
            "session_properties.product_tier",
            value: "premium"
        )
        .addSessionProperty(
            "session_properties.onboarding_complete",
            value: true
        )
)
```

Autorize as chaves no frontend antes de enviar. Propriedade desconhecida é descartada no ingest. No logout:

```swift
Dynatrace.identifyUser(nil)
Dynatrace.endVisit()
```

Reaplique a identidade após uma nova sessão/ciclo do app. Prefira IDs pseudônimos.

## 9. HTTP e correlação

A instrumentação automática de requests vem habilitada (`DTXInstrumentWebRequestTiming`). Frameworks de terceiros podem não ser observados; use a API manual nesses casos. Exclua endpoints sem valor ou sensíveis por `DTXURLFilters`, validando o formato na documentação da versão.

O Latest Dynatrace usa `traceparent`/`tracestate` para ligar o request mobile ao backend. Confirme `trace.id` nos eventos e a continuidade nos spans.

## 10. Crash, ANR e símbolos

Crash e ANR estão ativos por padrão. Em iOS, o debugger conectado desativa ambos, então valide em execução sem debugger. Um crash é enviado após reiniciar o app.

Configuração disponível:

```xml
<key>DTXCrashReportingEnabled</key>
<true/>
<key>DTXANRReportingEnabled</key>
<true/>
<key>DTXANRTimeout</key>
<integer>2</integer>
```

Envie arquivos `dSYM` de cada build de produção. Sem symbolication, stack traces exibem endereços em vez de funções e linhas. Não combine handlers de crash/swizzling sem teste de compatibilidade.

## 11. Session Replay

Inclua o produto `DynatraceSessionReplay`, ative a capacidade e a taxa de captura no frontend e aplique o opt-in de crash replay quando requerido. Session Replay mobile exige masking no aplicativo; teste telas e componentes customizados. Comece pela configuração mais restritiva.

## 12. Build e verificação

```bash
xcodebuild -version
xcodebuild -list -project SeuApp.xcodeproj
xcodebuild -project SeuApp.xcodeproj -scheme SeuApp -sdk iphonesimulator build
```

Roteiro de teste:

1. build limpo;
2. execução em simulador e dispositivo físico;
3. consentimento concedido;
4. navegação e requests;
5. background/foreground para flush;
6. eventos em Experience Vitals/DQL;
7. build release com símbolos enviados;
8. crash controlado sem debugger;
9. opt-out e alteração de preferências.

Para diagnóstico temporário, `DTXWriteLogsToFile` e `Dynatrace.shareLogsFile(on:)` podem ajudar, mas a documentação não recomenda logs em arquivo em produção.

## Referências oficiais

- [Setup inicial iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-01-initial-setup)
- [Suporte e limitações iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-02-support-and-limitations)
- [Configuração iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-03-configuration)
- [App performance iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-05-app-performance)
- [Erros e crashes iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-07-error-and-crash-reporting)
- [Usuário e sessão iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-09-user-and-session)
- [Ações iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-10-user-actions)

