# Instrumentação do RUM em Android

Este runbook cobre aplicativo Android nativo em Kotlin ou Java usando o Dynatrace Android Gradle plugin. Use o snippet gerado pelo wizard do seu tenant como fonte dos IDs e da configuração específica da versão.

## 1. Pré-requisitos

Segundo a página de suporte consultada em setembro de 2026:

| Componente | Mínimo |
|---|---:|
| Android API level | 23 |
| Gradle | 8.0 |
| Android Gradle Plugin | 8.1.1 |
| Java usado pelo Gradle | 17 |
| Kotlin, quando aplicável | 2.1.0 |
| Jetpack Compose | 1.4–1.10 |

Verifique o JDK realmente usado pelo Gradle:

```powershell
.\gradlew.bat --version
```

Localize o módulo que aplica `com.android.application`; não assuma que se chama `app`. O plugin auto-instrumenta o aplicativo e suas bibliotecas internas quando elas entram como dependência, mas não um projeto library isolado.

## 2. Criar o frontend e obter as chaves

Em **Experience Vitals > Add Frontend > Mobile > Android**, crie o frontend e copie:

- `applicationId`: identifica o frontend Dynatrace, não confundir com o package/application ID Android;
- `beaconUrl`: endpoint de ingestão terminado normalmente em `/mbeacon`.

Use chaves distintas por ambiente. Trate o beacon como configuração; mesmo não sendo uma senha, não exponha topologia interna desnecessariamente.

## 3. Configurar o plugin

### Kotlin DSL — build raiz

```kotlin
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath("com.dynatrace.tools.android:gradle-plugin:8.+")
    }
}

apply(plugin = "com.dynatrace.instrumentation")

configure<com.dynatrace.tools.android.dsl.DynatraceExtension> {
    configurations {
        create("prod") {
            autoStart {
                applicationId("SEU_DYNATRACE_APPLICATION_ID")
                beaconUrl("https://SEU_AMBIENTE/mbeacon")
            }
            userOptIn(true)
            agentBehavior.startupLoadBalancing(true)
            agentBehavior.startupWithGrailEnabled(true)
        }
    }
}
```

### Groovy DSL — build raiz

```groovy
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.dynatrace.tools.android:gradle-plugin:8.+'
    }
}

apply plugin: 'com.dynatrace.instrumentation'

dynatrace {
    configurations {
        prod {
            autoStart {
                applicationId 'SEU_DYNATRACE_APPLICATION_ID'
                beaconUrl 'https://SEU_AMBIENTE/mbeacon'
            }
            userOptIn true
            agentBehavior.startupLoadBalancing true
            agentBehavior.startupWithGrailEnabled true
        }
    }
}
```

Mantenha o plugin no build raiz. Use o wizard para mapear configurações a flavors/build types e excluir classes. Fixar uma versão validada no catálogo de dependências dá builds mais reproduzíveis; `8.+` acompanha a família indicada nos exemplos de setup, mas deve ser governado pelo processo de atualização da organização.

## 4. Aplicar consentimento

Com `userOptIn(true)`, o app precisa aplicar explicitamente a decisão armazenada pela sua CMP/tela de privacidade. No `Application.onCreate()` — ou no ponto de entrada controlado pelo app — recupere a escolha persistida e aplique:

```kotlin
import com.dynatrace.android.agent.Dynatrace
import com.dynatrace.android.agent.conf.DataCollectionLevel
import com.dynatrace.android.agent.conf.UserPrivacyOptions

fun applyObservabilityConsent(consent: Consent) {
    val level = when {
        !consent.rum -> DataCollectionLevel.OFF
        consent.userBehavior -> DataCollectionLevel.USER_BEHAVIOR
        else -> DataCollectionLevel.PERFORMANCE
    }

    val options = UserPrivacyOptions.builder()
        .withDataCollectionLevel(level)
        .withCrashReportingOptedIn(consent.crashReporting)
        // Inclua somente se Session Replay/crash replay estiver contratado e consentido:
        // .withCrashReplayOptedIn(consent.crashReplay)
        .build()

    Dynatrace.applyUserPrivacyOptions(options)
}
```

Não codifique `USER_BEHAVIOR` como decisão permanente em produção. Aplique novamente quando a pessoa alterar a preferência.

## 5. Instrumentação automática

Por padrão, o agente captura lifecycle, app starts, views suportadas, ações acionadas por handlers, requests via `HttpURLConnection`/`OkHttp`, exceções não tratadas e recursos compatíveis.

Limitações importantes:

- frameworks HTTP fora de `HttpURLConnection` e `OkHttp` podem exigir API manual;
- WebSocket e protocolos não HTTP exigem instrumentação manual;
- código NDK não é auto-instrumentado, embora native crash reporting tenha suporte específico em Android 11+;
- a auto-instrumentação modifica `AndroidManifest.xml` e bytecode `.class`, não HTML/JS/resources;
- plugins concorrentes de performance podem conflitar.

Teste o artefato com R8/ProGuard e flavors reais de produção.

## 6. Views e jornada

O agente acompanha app start → view → navegação → view até background/encerramento. Se nomes automáticos forem técnicos ou uma UI customizada não for reconhecida, use a API de view indicada pela versão do SDK/wizard. O princípio é manter nomes funcionais e estáveis:

```text
home
catalog
product_detail
cart
checkout
order_confirmation
```

Não inclua IDs de produto, conta, pedido ou usuário no nome da view. Leve essas dimensões para propriedades autorizadas e com cardinalidade controlada.

## 7. Ações de usuário manuais

Use ação manual para fluxos significativos que atravessam callbacks/telas:

```kotlin
private var checkoutAction: UserAction? = null

fun beginCheckout(cartTotal: Double, itemCount: Int) {
    checkoutAction = Dynatrace.createUserAction(
        UserActionConfiguration("Checkout Process")
    ).apply {
        addEventProperty("event_properties.cart_value", cartTotal)
        addEventProperty("event_properties.item_count", itemCount)
    }
}

fun finishCheckout(success: Boolean, reason: String? = null) {
    checkoutAction?.apply {
        addEventProperty("event_properties.checkout_successful", success)
        reason?.let {
            addEventProperty("event_properties.failure_reason", it)
        }
        complete()
    }
    checkoutAction = null
}
```

Os packages/imports exatos podem variar por versão; use o autocomplete/API do SDK instalado e o snippet do wizard. O contrato funcional é: `createUserAction` → propriedades → `complete()`.

A detecção automática fica ativa por padrão. Pode ser controlada em runtime:

```kotlin
Dynatrace.setAutomaticUserActionDetection(false)
```

Desativá-la não afeta ações manuais. Evite fazê-lo globalmente sem comparar a perda de cobertura.

## 8. Identidade e propriedades de sessão

Depois de login e consentimento:

```kotlin
Dynatrace.identifyUser("usr_8f6e2d")
```

Envie contexto de sessão somente após autorizar as chaves no frontend:

```kotlin
Dynatrace.sendSessionPropertyEvent(
    SessionPropertyEventData()
        .addSessionProperty("session_properties.product_tier", "premium")
        .addSessionProperty("session_properties.customer_segment", "retail")
        .addSessionProperty("session_properties.onboarding_complete", true)
)
```

No logout:

```kotlin
Dynatrace.identifyUser(null)
Dynatrace.endVisit()
```

O `identifyUser` deve ser reaplicado em nova sessão/ciclo do app; prefira um ID pseudônimo e não reversível.

## 9. Erros, crashes e ANRs

Crashes não tratados são automáticos por padrão. ANR e native crashes exigem Android 11+ e podem depender de o app reabrir dentro de dez minutos para o envio/correlação.

Relate exceções tratadas que impactam a jornada:

```kotlin
try {
    paymentRepository.confirm()
} catch (exception: Exception) {
    Dynatrace.sendExceptionEvent(
        ExceptionEventData(exception)
            .addEventProperty("event_properties.operation", "confirm_payment")
    )
    throw exception
}
```

Não use mensagem livre com payload, token ou PII como propriedade.

## 10. Session Replay

Requisitos específicos incluem Android API 23+ e AGP 8.1.1+ nas versões atuais. Ative no frontend, selecione a captura de replay e execute novamente o instrumentation wizard. Para opt-in, inclua consentimento de crash replay na API de privacidade.

Comece com masking **Safest**. Em Compose, elementos específicos podem ser mascarados com `Modifier.dtMask()`. Para Views, use as APIs/tags documentadas. Valide especialmente login, teclado, documentos, saldo, dados de pagamento e WebViews.

## 11. Build e verificação

```powershell
.\gradlew.bat tasks --group=build
.\gradlew.bat assembleDebug
```

Depois:

1. instale em emulador e em pelo menos um dispositivo físico;
2. conceda o consentimento;
3. abra, navegue, faça requests, vá ao background e retorne;
4. verifique os logs do agente sem registrar chaves/PII;
5. confirme eventos em **Experience Vitals > Mobile** e via DQL;
6. repita em build release/obfuscado;
7. teste opt-out e mudança de consentimento.

## Referências oficiais

- [Setup inicial Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-01-initial-setup)
- [Suporte e limitações Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-02-support-and-limitations)
- [App performance Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-05-app-performance)
- [Erros e crashes Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-07-error-and-crash-reporting)
- [Usuário e sessão Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-09-user-and-session)
- [Ações Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-10-user-actions)
- [Session Replay Android](https://docs.dynatrace.com/docs/observe/digital-experience/session-replay-latest/configure-session-replay-mobile/latest-session-replay-android)
