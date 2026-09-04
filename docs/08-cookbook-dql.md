# Cookbook DQL para RUM mobile

Consultas para o **Latest Dynatrace/Grail**. Substitua `SEU_FRONTEND` e os IDs. Comece com janelas curtas e filtros antecipados para reduzir leitura e custo.

## Campos essenciais

| Campo | Uso |
|---|---|
| `frontend.name` | filtrar aplicação |
| `dt.rum.application.type` | `mobile` ou `web` |
| `dt.rum.user_type` | `real_user`, `synthetic`, `robot` |
| `dt.rum.session.id` | reconstruir sessão |
| `dt.rum.instance.id` | instância pseudônima |
| `view.name` | tela mobile |
| `app.short_version` | release funcional |
| `os.name`, `os.version` | plataforma/versão |
| `device.model.identifier` | modelo do device |
| `trace.id` | correlação backend |

## 1. Sanidade por frontend, plataforma e versão

```dql
fetch user.events, from: now() - 24h
| filter dt.rum.application.type == "mobile"
| filter dt.rum.user_type == "real_user"
| summarize
    eventos = count(),
    sessoes = countDistinct(dt.rum.session.id),
    usuarios_aprox = countDistinct(dt.rum.instance.id, precision:9),
    by:{frontend.name, os.name, app.short_version}
| sort eventos desc
```

## 2. Descobrir tipos de evento

Não confie em `characteristics.classifier` como rótulo final. Derive o tipo pelas flags:

```dql
fetch user.events, from: now() - 2h
| filter frontend.name == "SEU_FRONTEND"
| fieldsAdd event_type = if(characteristics.is_invalid, "Invalid",
    else: if(characteristics.has_error, "Error",
    else: if(characteristics.has_view_summary, "View summary",
    else: if(characteristics.has_app_start, "App start",
    else: if(characteristics.has_user_action, "User action",
    else: if(characteristics.has_navigation, "Navigation",
    else: if(characteristics.has_user_interaction, "Interaction",
    else: if(characteristics.has_request, "Request",
    else:"Other"))))))))
| summarize eventos = count(), by:{event_type}
| sort eventos desc
```

## 3. Timeline de uma sessão

```dql
fetch user.events, from: now() - 510m
| filter dt.rum.session.id == "SESSION_ID"
| fieldsAdd event_type = if(characteristics.has_error, "Error",
    else: if(characteristics.has_view_summary, "View summary",
    else: if(characteristics.has_app_start, "App start",
    else: if(characteristics.has_user_action, "User action",
    else: if(characteristics.has_navigation, "Navigation",
    else: if(characteristics.has_request, "Request", else:"Other"))))))
| fields start_time, event_type, frontend.name, app.short_version,
    view.name, view.sequence_number, user_action.custom_name,
    interaction.type, url.domain, url.path,
    http.response.status_code, error.type, trace.id
| sort start_time asc
```

`510m` cobre 8,5 horas; literais DQL não aceitam `8.5h`.

## 4. Engajamento por view

```dql
fetch user.events, from: now() - 24h
| filter characteristics.has_view_summary
| filter dt.rum.application.type == "mobile"
| filter dt.rum.user_type == "real_user"
| summarize
    views = count(),
    sessoes = countDistinct(dt.rum.session.id),
    foreground_medio = avg(view.foreground_time),
    by:{frontend.name, view.name}
| sort views desc
| limit 50
```

## 5. Profundidade da jornada

```dql
fetch user.events, from: now() - 24h
| filter characteristics.has_view_summary
| filter dt.rum.application.type == "mobile"
| summarize max_sequencia = max(view.sequence_number),
    by:{frontend.name, dt.rum.session.id}
| summarize
    sessoes = count(),
    views_medias = avg(max_sequencia),
    p50_views = percentile(max_sequencia, 50),
    p90_views = percentile(max_sequencia, 90),
    by:{frontend.name}
```

## 6. Startup por tipo e versão

```dql
fetch user.events, from: now() - 7d
| filter characteristics.has_app_start
| filter dt.rum.application.type == "mobile"
| summarize
    starts = count(),
    p50 = percentile(duration, 50),
    p75 = percentile(duration, 75),
    p95 = percentile(duration, 95),
    by:{frontend.name, os.name, app.short_version, app_start.type}
| sort p95 desc
```

Referência prática: cold bom abaixo de 3 s e ruim acima de 5 s; warm bom abaixo de 1,5 s e ruim acima de 2 s; hot bom abaixo de 500 ms e ruim acima de 1 s. Valide thresholds com seu produto e devices reais.

## 7. Ações e motivos de conclusão

```dql
fetch user.events, from: now() - 24h
| filter characteristics.has_user_action
| filter dt.rum.application.type == "mobile"
| summarize
    acoes = count(),
    sessoes = countDistinct(dt.rum.session.id),
    duracao_p95 = percentile(duration, 95),
    by:{frontend.name, user_action.custom_name,
        user_action.complete_reason}
| sort acoes desc
```

## 8. Conversão de checkout

```dql
fetch user.events, from: now() - 7d
| filter characteristics.has_user_action
| filter user_action.custom_name == "Checkout Process"
| summarize
    iniciados = count(),
    sucesso = countIf(event_properties.checkout_successful == true),
    falha = countIf(event_properties.checkout_successful == false),
    by:{frontend.name, app.short_version,
        event_properties.payment_method}
| fieldsAdd conversao_pct = 100.0 * sucesso / iniciados
| sort iniciados desc
```

## 9. Requests falhos por endpoint

```dql
fetch user.events, from: now() - 2h
| filter characteristics.has_failed_request
| filter dt.rum.application.type == "mobile"
| summarize
    falhas = count(),
    sessoes_afetadas = countDistinct(dt.rum.session.id),
    by:{frontend.name, http.response.status_code,
        url.domain, url.path}
| sort falhas desc
| limit 50
```

## 10. Cobertura de trace

```dql
fetch user.events, from: now() - 2h
| filter characteristics.has_request
| filter dt.rum.application.type == "mobile"
| summarize
    requests = count(),
    traced = countIf(isNotNull(trace.id)),
    by:{frontend.name, url.domain}
| fieldsAdd cobertura_pct = 100.0 * traced / requests
| sort cobertura_pct asc
```

## 11. Requests lentos com trace

```dql
fetch user.events, from: now() - 2h
| filter characteristics.has_request
| filter duration > 2s
| filter isNotNull(trace.id)
| fields start_time, frontend.name, view.name,
    url.domain, url.path, duration, trace.id, span.id,
    http.response.status_code, request.trace_context_hint
| sort duration desc
| limit 50
```

Para o request escolhido:

```dql
fetch spans, from:"2026-09-04T12:00:00Z", to:"2026-09-04T12:05:00Z"
| filter trace.id == toUid("TRACE_ID")
| fields start_time, span.name, span.kind,
    dt.smartscape.service, duration
| sort start_time asc
```

Use janela curta. Um join amplo de `user.events` com `spans` tende a ser caro ou inviável.

## 12. Crashes e ANRs

```dql
fetch user.events, from: now() - 7d
| filter characteristics.has_crash or characteristics.has_anr
| summarize
    ocorrencias = count(),
    sessoes_afetadas = countDistinct(dt.rum.session.id),
    usuarios_afetados = countDistinct(dt.rum.instance.id, precision:9),
    by:{frontend.name, os.name, app.short_version,
        error.type, exception.type, exception.crash_signal_name}
| sort ocorrencias desc
| limit 50
```

Detalhe para engenharia:

```dql
fetch user.events, from: now() - 24h
| filter characteristics.has_crash
| fields start_time, frontend.name, app.short_version,
    os.name, os.version, device.model.identifier,
    exception.type, exception.message,
    exception.crash_signal_name, exception.stack_trace,
    dt.rum.session.id
| sort start_time desc
| limit 20
```

## 13. Sessões agregadas com erro

Exclua a última hora para evitar sessões ainda abertas:

```dql
fetch user.sessions, from: now() - 25h, to: now() - 1h
| filter dt.rum.application.type == "mobile"
| filter dt.rum.user_type == "real_user"
| filter error.count > 0
| expand frontend.name
| summarize
    sessoes = count(),
    erros = sum(toLong(error.count)),
    crashes = countIf(error.has_crash),
    by:{frontend.name, os.name, app.short_version}
| sort erros desc
```

## 14. Usuários identificados afetados

Requer fieldset sensível:

```dql
fetch user.sessions, from: now() - 7d, to: now() - 1h
| filter dt.rum.application.type == "mobile"
| filter error.count > 0 and isNotNull(user.identifier)
| expand frontend.name
| summarize
    sessoes = count(),
    erros = sum(toLong(error.count)),
    teve_crash = countIf(error.has_crash),
    by:{frontend.name, user.identifier}
| sort erros desc
| limit 50
```

Sem permissão ou sem identidade explícita, use `dt.rum.instance.id` para análise pseudônima.

## 15. Bounce e duração de sessão

```dql
fetch user.sessions, from: now() - 8d, to: now() - 1h
| filter dt.rum.application.type == "mobile"
| filter dt.rum.user_type == "real_user"
| expand frontend.name
| summarize
    sessoes = count(),
    bounces = countIf(characteristics.is_bounce),
    duracao_media_s = avg(toLong(duration)) / 1000000000,
    p75_duracao_s = percentile(toLong(duration), 75) / 1000000000,
    by:{frontend.name}
| fieldsAdd bounce_pct = 100.0 * bounces / sessoes
| sort sessoes desc
```

## 16. Uso faturável por aplicação

Billing Usage Events são a fonte autoritativa:

```dql
fetch dt.system.events, from: now() - 30d
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
| sort app asc
```

## Notas de diagnóstico

- `frontend.name` é array em `user.sessions`; use `expand`.
- Campos agregados de sessão usam underscore (`request_count`); erros usam ponto (`error.count`).
- `user.sessions` só contém sessões materializadas após agregação.
- Um campo sensível sem permissão pode retornar `null` e filtros podem resultar em zero silenciosamente.
- Filtre `real_user` quando não quiser synthetic/robot.
- Use `view.name`, não `page.name`, para mobile.
- Compare métricas agregadas com eventos, mas não espere contagens idênticas em toda situação.

## Referências oficiais

- [DQL](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language)
- [Modelo de user events](https://docs.dynatrace.com/docs/semantic-dictionary/model/rum/user-events)
- [User actions](https://docs.dynatrace.com/docs/semantic-dictionary/model/rum/user-events/user-actions)
- [Consumo RUM e BUE](https://docs.dynatrace.com/docs/license/capabilities/real-user-synthetic-monitoring/real-user-monitoring)

