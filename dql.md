```sh
fetch dt.maintenance.windows, from:-30d
| filter event.name == "Maintenance Window Start"

| fieldsAdd
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

| summarize {
    execucoes = count(),
    datas = collectDistinct(data),
    primeira_execucao = min(start_time),
    ultima_execucao = max(start_time)
  },
  by:{
    dt.settings.object_id,
    maintenance_window.title,
    maintenance_window.filter
  }

| sort execucoes desc
```
