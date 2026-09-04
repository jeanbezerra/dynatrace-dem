# Validação e troubleshooting do RUM mobile

Use este runbook após instrumentar e em cada upgrade do SDK/plugin. O objetivo não é apenas “apareceu dado”, mas provar cobertura, semântica, correlação, privacidade e custo.

## Matriz de teste mínima

| Área | Cenário | Evidência esperada |
|---|---|---|
| Startup | cold, warm e hot | `app_start.type`, duração e fases |
| Views | jornada feliz e alternativa | sequência/nome estáveis |
| Ações | sucesso, falha e cancelamento | uma ação fechada por tentativa |
| HTTP | sucesso, 4xx, 5xx, timeout | request, status e `trace.id` quando aplicável |
| Erro tratado | exceção controlada | evento API reported e contexto permitido |
| Crash/ANR | teste controlado | stack/sinal, sessão e versão |
| Identidade | login/logout | ID pseudônimo só no escopo correto |
| Consentimento | primeira abertura, opt-in, opt-out | coleta coerente com cada escolha |
| Replay | telas sensíveis e customizadas | masking correto e acesso restrito |
| Release | debug e release/obfuscado | mesma cobertura essencial |

## 1. Validar build e configuração

### Android

```powershell
.\gradlew.bat --version
.\gradlew.bat tasks --group=build
.\gradlew.bat assembleDebug
```

Confirme:

- plugin no build raiz;
- `com.android.application` no módulo instrumentado;
- configuração correspondente ao variant executado;
- `applicationId` e `beaconUrl` sem placeholder;
- `mavenCentral()` disponível;
- build release com R8 passa;
- SDK/plugin têm versões compatíveis.

### iOS

```bash
xcodebuild -version
xcodebuild -list -project SeuApp.xcodeproj
xcodebuild -project SeuApp.xcodeproj -scheme SeuApp -sdk iphonesimulator build
```

Confirme:

- pacote SPM no app target;
- produto correto (`Dynatrace` ou Session Replay);
- plist no target/bundle final;
- valores corretos por configuração;
- `import Dynatrace` compila;
- deployment target iOS 15+;
- archive gera e publica `dSYM`.

## 2. Gerar telemetria reproduzível

Registre horário, plataforma, versão, device, build e conta pseudônima. Execute:

1. limpar/instalar o app;
2. escolher consentimento;
3. iniciar e aguardar primeira view;
4. autenticar;
5. percorrer a jornada crítica;
6. provocar erro tratado conhecido;
7. enviar ao background e retornar;
8. encerrar/logout quando aplicável.

Use valores de teste fáceis de filtrar em propriedades não sensíveis, como `session_properties.test_run = "rum-smoke-20260904"`, desde que essa chave esteja permitida e não seja enviada em produção sem necessidade.

## 3. Confirmar chegada de eventos

Pode levar alguns minutos após o primeiro launch. Comece amplo:

```dql
fetch user.events, from: now() - 24h
| filter dt.rum.application.type == "mobile"
| summarize eventos = count(), by:{frontend.name, os.name, app.short_version}
| sort eventos desc
```

Depois restrinja:

```dql
fetch user.events, from: now() - 2h
| filter frontend.name == "SEU_FRONTEND"
| summarize
    total = count(),
    starts = countIf(characteristics.has_app_start),
    views = countIf(characteristics.has_view_summary),
    actions = countIf(characteristics.has_user_action),
    requests = countIf(characteristics.has_request),
    errors = countIf(characteristics.has_error)
```

Se zero:

1. remova o filtro de frontend;
2. amplie para 24h;
3. confirme consentimento e nível de coleta;
4. confirme conectividade ao beacon;
5. revise IDs e variant/scheme;
6. habilite debug logging temporário;
7. verifique relógio/proxy/VPN/firewall/certificate pinning;
8. compare debug e release.

## 4. Entender o atraso de sessões

`user.events` pode chegar em minutos. `user.sessions` é gravado depois do encerramento/agregação, normalmente após mais de 30 minutos de inatividade. Uma consulta só da última hora pode não mostrar sessões em andamento.

Para análise estável:

```dql
fetch user.sessions, from: now() - 25h, to: now() - 1h
| filter dt.rum.application.type == "mobile"
| expand frontend.name
| summarize sessoes = count(), by:{frontend.name, os.name}
```

Sessões “zumbi”, apenas com atividade automática/background e sem navegação/interação real, podem não ser materializadas. Isso não implica perda de eventos.

## 5. Validar nomes e cardinalidade

```dql
fetch user.events, from: now() - 24h
| filter characteristics.has_view_summary
| filter dt.rum.application.type == "mobile"
| summarize views = count(), sessoes = countDistinct(dt.rum.session.id),
    by:{frontend.name, view.name}
| sort views desc
| limit 100
```

Sinais de problema:

- centenas/milhares de nomes com sufixos numéricos;
- e-mail, CPF, IDs ou URLs em nomes;
- nomes diferentes entre iOS e Android para a mesma tela;
- classe técnica onde se esperava conceito funcional;
- view nunca encerrada ou sequência incoerente.

## 6. Validar ações

```dql
fetch user.events, from: now() - 24h
| filter characteristics.has_user_action
| filter dt.rum.application.type == "mobile"
| summarize acoes = count(), duracao_p95 = percentile(duration, 95),
    by:{frontend.name, user_action.custom_name,
        user_action.complete_reason}
| sort acoes desc
```

Investigue `timeout`, interrupções, ações duplicadas e ausência de resultado. Para ações manuais, confirme `complete()` em todos os caminhos e que propriedades são adicionadas antes dele.

## 7. Validar correlação de requests

```dql
fetch user.events, from: now() - 2h
| filter characteristics.has_request
| filter dt.rum.application.type == "mobile"
| summarize
    requests = count(),
    traced = countIf(isNotNull(trace.id)),
    by:{frontend.name, url.domain}
| fieldsAdd cobertura_trace_pct = 100.0 * traced / requests
| sort cobertura_trace_pct asc
```

Baixa cobertura pode indicar framework HTTP não suportado, CORS/proxy, headers removidos, backend não instrumentado ou endpoint externo. Escolha um `trace.id` e consulte spans numa janela curta com `toUid()`.

## 8. Crashes, ANRs e símbolos

### Android

- ANR e native crash requerem Android 11+;
- em alguns casos o app precisa reabrir em até dez minutos;
- teste em device real e build de produção;
- confira compatibilidade com crash handlers concorrentes.

### iOS

- debugger conectado desativa crash e ANR;
- reinicie o app após o crash;
- confirme upload do `dSYM` exato do build;
- evite múltiplos handlers baseados em swizzling.

Consulta:

```dql
fetch user.events, from: now() - 7d
| filter characteristics.has_crash or characteristics.has_anr
| summarize ocorrencias = count(), sessoes = countDistinct(dt.rum.session.id),
    by:{frontend.name, app.short_version, os.name,
        error.type, exception.type}
| sort ocorrencias desc
```

## 9. Propriedades ausentes

Quando uma propriedade não aparece:

1. confirme que foi criada em **Capture properties**;
2. confira prefixo `event_properties.` ou `session_properties.`;
3. valide tipo e regras de nome;
4. confirme que é adicionada antes de `complete()`;
5. verifique se o evento/ação realmente foi enviado;
6. remova filtros e inspecione um evento bruto;
7. considere sampling;
8. para sessão, aguarde agregação.

## 10. Privacidade e permissões

Se `user.identifier` ou `client.ip` aparece nulo, pode ser falta de consentimento, ausência de identificação ou falta do fieldset `builtin-sensitive-user-events-and-sessions`. Consultas podem retornar zero silenciosamente ao filtrar campo sensível sem permissão.

## 11. O que anexar a um ticket de suporte

- frontend/plataforma/ambiente, sem segredos;
- horário UTC e timezone;
- versão do app e do OneAgent/plugin;
- modelo/OS;
- Gradle/AGP/Java/Kotlin ou Xcode/iOS;
- passos mínimos e resultado esperado/observado;
- logs de debug com PII e chaves removidas;
- configuração relevante sanitizada;
- query DQL e resultado;
- diferença entre debug/release.

## Referências oficiais

- [Troubleshooting mobile RUM](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends)
- [Suporte Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-02-support-and-limitations)
- [Suporte iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-02-support-and-limitations)
- [Erros Android](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/android/id-07-error-and-crash-reporting)
- [Erros iOS](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/ios/id-07-error-and-crash-reporting)

