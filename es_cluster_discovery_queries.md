# Discovery queries for the Elasticsearch cluster dashboard

Run in Kibana > Management > Dev Tools > Console. Paste the **full output** of each back to Claude
(redact anything sensitive). Queries are read-only.

## 1. Which Elasticsearch monitoring datasets exist? (most important)
```
GET metrics-*,logs-*/_search
{
  "size": 0,
  "query": { "range": { "@timestamp": { "gte": "now-1h" } } },
  "aggs": {
    "ds": { "terms": { "field": "data_stream.dataset", "size": 500,
                       "include": ".*(elasticsearch|stack_monitoring|kibana|logstash|beat).*" } }
  }
}
```

## 2. Legacy Metricbeat / internal monitoring indices?
```
GET _cat/indices/.monitoring-*,metricbeat-*,.ds-metrics-elasticsearch*?v&h=index,docs.count,store.size&s=index
```

## 3. Sample document + field names of the node-stats and cluster-stats metrics
```
GET metrics-elasticsearch.stack_monitoring.node_stats-*/_search?size=1&sort=@timestamp:desc
GET metrics-elasticsearch.stack_monitoring.cluster_stats-*/_search?size=1&sort=@timestamp:desc
GET metrics-elasticsearch.stack_monitoring.index_summary-*/_search?size=1&sort=@timestamp:desc
```
If these return "no such index", run instead (Metricbeat / integration `elasticsearch.*`):
```
GET metrics-elasticsearch.node_stats-*/_search?size=1&sort=@timestamp:desc
GET .monitoring-es-*/_search?size=1&sort=timestamp:desc
```

## 4. Cluster basics
```
GET /
GET _cluster/health
GET _cat/nodes?v&h=name,node.role,master,heap.percent,cpu,load_1m,disk.used_percent,version
GET _cluster/settings?include_defaults=false&flat_settings=true
```

## 5. Index and shard layout (top 25 by size)
```
GET _cat/indices?v&h=index,health,pri,rep,docs.count,store.size&s=store.size:desc&bytes=gb&size=25
GET _cat/allocation?v
GET _cluster/stats?filter_path=indices.shards,indices.count,nodes.count
```

## 6. Lifecycle, snapshots, ingest
```
GET _ilm/policy?filter_path=*.version
GET */_ilm/explain?only_errors=true
GET _slm/policy
GET _snapshot
GET _nodes/stats/ingest?filter_path=nodes.*.name,nodes.*.ingest.total
```

## 7. Thread pools / breakers (for rejection and breaker panels)
```
GET _cat/thread_pool/write,search,bulk?v&h=node_name,name,active,queue,rejected
GET _nodes/stats/breaker?filter_path=nodes.*.name,nodes.*.breakers.*.tripped
```

## 8. Alerting and slow logs
```
GET .kibana_alerting_cases*/_count
GET logs-elasticsearch.slowlog-*/_search?size=1
GET logs-elasticsearch.server-*/_search?size=1
```

## 9. Which role/account is the dashboard user, and can it read monitoring data?
```
GET _security/_authenticate
GET _cat/indices/.monitoring-*,metrics-elasticsearch*,logs-elasticsearch*?v&h=index
```
