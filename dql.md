```sh
fetch dt.maintenance.windows, from:-30d

| summarize
    registros = count(),
    by:{
        dt.settings.object_id,
        maintenance_window.title,
        maintenance_window.filter,
        start_time,
        end_time
    }

| fieldsAdd
    inicio = formatTimestamp(
        start_time,
        format:"yyyy-MM-dd HH:mm",
        timezone:"America/Sao_Paulo"
    ),
    horario = formatTimestamp(
        start_time,
        format:"HH:mm",
        timezone:"America/Sao_Paulo"
    ),
    data = formatTimestamp(
        start_time,
        format:"yyyy-MM-dd",
        timezone:"America/Sao_Paulo"
    )

| filter horario == "19:00"

| summarize
    ocorrencias = count(),
    datas = collectDistinct(data),
    primeira_ocorrencia = min(start_time),
    ultima_ocorrencia = max(start_time),
    by:{
        dt.settings.object_id,
        maintenance_window.title,
        maintenance_window.filter
    }

| sort ocorrencias desc
```
