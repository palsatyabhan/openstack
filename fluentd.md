<match **>
  @type elasticsearch
  host elasticsearch.default.svc.cluster.local
  port 9200
  scheme https
  user my-username
  password my-password
  logstash_format true
  logstash_prefix fluentd-logs
  index_name fluentd-logs
  include_timestamp true
  reconnect_on_error true
  reload_connections false
  request_timeout 15s
  ssl_verify false
</match>
