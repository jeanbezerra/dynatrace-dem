```sh
fetch dt.maintenance.windows, from:-30d
| filter event.type == "MAINTENANCE_WINDOW_START"

| fieldsAdd
    data = formatTimestamp(
        start_time,
        format:"yyyy-MM-dd",
        timezone:"America/Sao_Paulo"
    ),
    dia_semana = formatTimestamp(
        start_time,
        format:"EEE",
        timezone:"America/Sao_Paulo"
    ),
    horario = formatTimestamp(
        start_time,
        format:"HH:mm",
        timezone:"America/Sao_Paulo"
    )

| summarize
    execucoes = count(),
    horarios = collectDistinct(horario),
    dias_semana = collectDistinct(dia_semana),
    primeira_execucao = min(start_time),
    ultima_execucao = max(start_time),
    by:{
        `Window ID` = dt.settings.object_id,
        `Nome` = maintenance_window.title,
        `Filtro` = maintenance_window.filter
    }

| sort execucoes desc
```

```sh
fetch dt.maintenance.windows, from:-30d
| filter event.type == "MAINTENANCE_WINDOW_START"

| fieldsAdd
    hora = getHour(start_time, timezone:"America/Sao_Paulo"),
    minuto = getMinute(start_time, timezone:"America/Sao_Paulo"),
    data = formatTimestamp(
        start_time,
        format:"yyyy-MM-dd",
        timezone:"America/Sao_Paulo"
    ),
    dia_semana = formatTimestamp(
        start_time,
        format:"EEE",
        timezone:"America/Sao_Paulo"
    )

| filter hora == 19

| summarize
    execucoes = count(),
    datas = collectDistinct(data),
    dias_semana = collectDistinct(dia_semana),
    primeira_execucao = min(start_time),
    ultima_execucao = max(start_time),
    by:{
        `Window ID` = dt.settings.object_id,
        `Nome` = maintenance_window.title,
        `Filtro` = maintenance_window.filter
    }

| sort execucoes desc
```
