```dql
timeseries {
    depth = max(ibmmq.queue.depth),
    depth_max = max(ibmmq.queue.depth, scalar: true),

    depth_pct = max(ibmmq.queue.depth_percent),
    depth_pct_max = max(ibmmq.queue.depth_percent, scalar: true)
},
by: {
    queue_name,
    `dt.entity.ibmmq:local_queue`
}

| fieldsAdd
    `Fila` = coalesce(
        queue_name,
        entityName(`dt.entity.ibmmq:local_queue`)
    ),

    `Depth Atual` = arrayLast(depth),

    `Depth Máximo` = depth_max,

    `% Atual` = arrayLast(depth_pct),

    `% Máximo` = depth_pct_max

| fieldsAdd
    `MAXDEPTH Estimado` =
        if(
            `% Atual` > 0,
            (`Depth Atual` * 100.0) / `% Atual`
        )

| fieldsAdd
    `Status` =
        if(
            `% Atual` >= 95,
            "SEV3 - CRÍTICO",
            else: if(
                `% Atual` >= 85,
                "SEV2 - ALERTA",
                else: if(
                    `% Atual` >= 70,
                    "ATENÇÃO",
                    else: "OK"
                )
            )
        )

| fields
    `Fila`,
    `Depth Atual`,
    `Depth Máximo`,
    `MAXDEPTH Estimado`,
    `% Atual`,
    `% Máximo`,
    `Status`

| sort `% Atual` desc
```
