# Signoz Traces in Grafana
- Your Dashboard > Add > Visualization
  - Visualizaton Type: Traces
  - Editor Type: SQL Editor
  - Query Type: Traces  
- Pass traceID value via url: `http://your.grafana.com/dashboard?var-traceID=aaaa-bbbb-cccc-dddd`

```SQL
with spans as (
  select timestamp, model from signoz_traces.signoz_spans
  where traceID = '${traceID}'
  order by timestamp asc
)
select
    timestamp as startTime,
    simpleJSONExtractString(model, 'traceId') as traceID,
    simpleJSONExtractString(model, 'spanId') as spanID,
    JSONExtractString(arrayElement(JSONExtractArrayRaw(model, 'references'), 1), 'spanId') as parentSpanID,
    simpleJSONExtractString(model, 'serviceName') as serviceName,
    simpleJSONExtractString(model, 'name') as operationName,
    simpleJSONExtractUInt(model, 'durationNano') *  0.000001 as duration,
    arrayPushBack(
        arrayMap(
            x -> map('key', x.1, 'value', x.2), -- transform to Map(String,String)
            JSONExtractKeysAndValues(model, 'tagMap', 'String') -- extract tags as Tuple(String,String)
        ),
        map(
            'key', 'error',  -- span error status [true/false]
            'value', if(simpleJSONExtractRaw(model, 'hasError') == 'true', 'true', 'false')
        )
    ) as tags,
    arrayMap(
        x -> map(
            'fields',
            arrayMap(
                x -> map('key',x.1 , 'value', x.2),
                JSONExtractKeysAndValues(JSONExtractString(x), 'attributeMap', 'String')
            )
        ),
        JSONExtractArrayRaw(model, 'event')
    ) as logs
from spans
```
