---
tags: [dynatrace, dql, blueprint]
---
# DQL – Blueprint timeseries + join

Plantilla para comparar dos métricas por las mismas dimensiones (p. ej. recomendación VPA vs request/uso real). Cambia solo lo que está en `<...>`.

## Plantilla

```
timeseries from: now() - <RANGO>, interval: <INTERVALO>, {
    <col_a1> = <AGG>(<METRICA_A1>),
    <col_a2> = <AGG>(<METRICA_A2>)
  },
  by: {<DIM_A_NS>, <DIM_A_WORKLOAD>, <DIM_A_CONTAINER>},
  filter: matchesValue(k8s.cluster.name, $Cluster)
| join [
    timeseries from: now() - <RANGO>, interval: <INTERVALO>, {
        <col_b1> = <AGG>(<METRICA_B1>),
        <col_b2> = <AGG>(<METRICA_B2>)
      },
      by: {<DIM_B_NS>, <DIM_B_WORKLOAD>, <DIM_B_CONTAINER>},
      filter: matchesValue(k8s.cluster.name, $Cluster)
  ],
  on: {left[<DIM_A_NS>]        == right[<DIM_B_NS>],
       left[<DIM_A_WORKLOAD>]  == right[<DIM_B_WORKLOAD>],
       left[<DIM_A_CONTAINER>] == right[<DIM_B_CONTAINER>]},
  fields: {<col_b1>, <col_b2>}
| fields <DIM_A_NS>, <DIM_A_WORKLOAD>, <DIM_A_CONTAINER>,
         <col_a1> = round(arrayAvg(<col_a1>) / <DIVISOR>, decimals: <N>),
         <col_a2> = round(arrayAvg(<col_a2>) / <DIVISOR>, decimals: <N>),
         <col_b1> = round(arrayAvg(<col_b1>) / <DIVISOR>, decimals: <N>),
         <col_b2> = round(arrayAvg(<col_b2>) / <DIVISOR>, decimals: <N>)
| fieldsAdd <pct> = if(<col_b1> > 0, toLong(round(<col_a1> / <col_b1> * 100)))
| sort <pct> asc
```

## Qué cambiar

| Placeholder | Qué es | Ejemplo |
|---|---|---|
| `<RANGO>` / `<INTERVALO>` | Ventana de tiempo y resolución | `5m` / `1m` |
| `<AGG>` | Cómo juntar las series de un grupo | `avg` (por pod) · `sum` (total workload) · `max` |
| `<METRICA_*>` | Nombre de la métrica | `dt.kubernetes.container.requests_cpu` |
| `<DIM_*>` | Labels por las que agrupar y unir | `k8s.namespace.name`, `k8s.workload.name`, `k8s.container.name` |
| `<DIVISOR>` | Conversión de unidades | `1000` (milicores→cores) · `1073741824` (bytes→GiB) · `1` |
| `<N>` | Decimales | `2` |
| `<pct>` | Columna de % | `cpu_vpa_eff_pct` |

## Chuleta rápida

- `timeseries` devuelve **arrays** (un valor por intervalo) → `arrayAvg()` para dejar un número.
- `by` = una fila por combinación de esas labels. Si la label no existe, sale vacía.
- `join` es **inner**: solo salen las filas que casan en los dos lados. `kind: leftOuter` para no perder las de la izquierda.
- `on` = qué columna de la izquierda equivale a cuál de la derecha.
- `fields:` (dentro del join) = qué columnas traer de la derecha.
- `fields` final = elige y **ordena** columnas. `fieldsAdd` = añade al final usando lo ya calculado.
- Colores → thresholds del tile, no en la query.
- Ver labels de una métrica: `fetch metric.series | filter metric.key == "<METRICA>" | limit 5`

## Ejemplo real: VPA vs request/uso

```
timeseries from: now() - 5m, interval: 1m, {
    cpu_target = avg(kube_verticalpodautoscaler_status_recommendation_container_recommendations_target_cpu),
    mem_target = avg(kube_verticalpodautoscaler_status_recommendation_container_recommendations_target_memory)
  },
  by: {k8s.namespace.name, target_name, container},
  filter: matchesValue(k8s.cluster.name, $Cluster)
| join [
    timeseries from: now() - 5m, interval: 1m, {
        cpu_req = avg(dt.kubernetes.container.requests_cpu),
        cpu_use = avg(dt.kubernetes.container.cpu_usage),
        mem_req = avg(dt.kubernetes.container.requests_memory),
        mem_use = avg(dt.kubernetes.container.memory_working_set)
      },
      by: {k8s.namespace.name, k8s.workload.name, k8s.container.name},
      filter: matchesValue(k8s.cluster.name, $Cluster)
  ],
  on: {left[k8s.namespace.name] == right[k8s.namespace.name],
       left[target_name] == right[k8s.workload.name],
       left[container] == right[k8s.container.name]},
  fields: {cpu_req, cpu_use, mem_req, mem_use}
| fields k8s.namespace.name, target_name, container,
         cpu_req    = round(arrayAvg(cpu_req) / 1000, decimals: 3),
         cpu_target = round(arrayAvg(cpu_target), decimals: 3),
         cpu_use    = round(arrayAvg(cpu_use) / 1000, decimals: 3),
         mem_req    = round(arrayAvg(mem_req) / 1073741824, decimals: 2),
         mem_target = round(arrayAvg(mem_target) / 1073741824, decimals: 2),
         mem_use    = round(arrayAvg(mem_use) / 1073741824, decimals: 2)
| fieldsAdd cpu_vpa_eff_pct = if(cpu_req > 0, toLong(round(cpu_target / cpu_req * 100))),
            mem_vpa_eff_pct = if(mem_req > 0, toLong(round(mem_target / mem_req * 100)))
| sort cpu_vpa_eff_pct asc
```

`*_vpa_eff_pct`: 100 = request ideal · <100 sobra · >100 falta. Thresholds: ≥0 rojo · ≥30 naranja · ≥70 verde · ≥110 morado.
