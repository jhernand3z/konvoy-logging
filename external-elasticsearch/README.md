# Konvoy logging with external elasticsearch

Konvoy provides a cluster-level logging solution consisting of three primary addons. These compnents are automatically configured to work together and do not require anything else to get up and running.

- Fluent Bit: Log forwarding
- Elasticsearch: Log storage and searching
- Kibana: Log visualization and analysis

Some teams may already have an existing Elasticsearch backend, which may be preferable to store logs instead of the addon that ships with Konvoy. However, the default behavior of the Fluent Bit addon is to forward logs to the Elasticsearch backed running inside Konvoy. If we want the Fluent Bit addon to forward logs to an external Elasticsearch backend, it will be necessary to override the addon configs in our Konvoy cluster.yaml file.

## Fluent Bit overrides

Here, he want to edit the cluster.yaml file and override the Fluent Bit addon configuration so that it will forward all host and container logs to the preferred Elasticsearch cluster running outside Konvoy.

Right after the fluentbit addon section, values can be added so that a different backend is used for Elasticsearch. Specifically, these are the es.host and es.port backend values. These should be set like this:

```bash
   - name: fluentbit
      enabled: true
      values: |
        backend:
          es:
            # Use the host and port for the external Elasticsearch backend.
            # Default port is 9200, but showing this to illustrate that a different port could be specified if needed.
            host: external-elasticsearch.example.com
            port: 9200
```

If the Konvoy cluster has already been deployed and these Fluent Bit changes are being made later, make sure to rerun the addon deployment.

```bash
  konvoy deploy addons -y
```

After the addons have been redeployed successfully with an OK status, you can confirm the config changes by viewing the ConfigMap used for Fluent Bit:

```bash
  kubectl describe cm --namespace kubeaddons fluentbit-kubeaddons-fluent-bit-config
```

Describing the ConfigMap will display the standard Fluent Bit configuration format explained here: https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/configuration-file

Look for the Fluent Bit OUTPUT entry under "fluent-bit-output.conf" and confirm the Host setting has been overridden with the desired Elasticsearch backend.

```bash
fluent-bit-output.conf:
----

[OUTPUT]
    Name  es
    Match *
    Host external-elasticsearch.example.com
    Port  9200
    Logstash_Format On
    Retry_Limit False
    Type  flb_type
    Time_Key @ts
    Replace_Dots On
    Logstash_Prefix kubernetes_cluster
```

In the external Elasicsearch environment, confirm that logs from Konvoy are now being ingested there. A quick way to confirm this is to filter a search by a hostname or pod ID from the source Konvoy cluster. This should return search results and demonstrate that logs from Konvoy are being forwarded to the external Elasticsearch environment. A good example is to check that etcd logs from a Konvoy master host are being sent to Elasticsearch. Here's an example:

```bash
  kubernetes.pod_name: "etcd" AND kubernetes.host: "konvoy-master-1.example.com"
```

## Using additional overrides

Notice the Host and Port entries from the ConfigMap displayed in the previous section. These will map directly to configuration items for Fluent Bit's **es** output plugin explained here: https://docs.fluentbit.io/manual/pipeline/outputs/elasticsearch. This means additional overrides can be added to the fluentbit addon section specified in the Konvoy cluster.yaml. For example, the default prefix name used for index creation in Elasticsearch is **kubernetes_cluster**. If an external Elasticsearch cluster is being used, it may be your preference, to use a different index naming standard. Perhaps you have multiple konvoy clusters forwarding logs to the external Elasticsearch. It may be desirable to organize these logs into different Elasticsearch indices such as konvoy-cluster-prod and konvoy-cluster-dev. It's easy to ensure that Konvoy logs are stored in Elasticsearch with a preferred index name by overriding the **Logstash_Prefix** configuration for the Fluent Bit es plugin.

In the Konvoy cluster.yaml file, add a new configration under the backend section for es called **logstash_prefix**. In the example below, the names of the indices created in Elasticsearch will have the format konvoy-prod-source-YYYY.MM.DD.

  ```bash
     - name: fluentbit
        enabled: true
        values: |
          backend:
            es:
              # Use the host and port for the external Elasticsearch backend.
              host: external-elasticsearch.example.com
              port: 9200
              logstash_prefix: konvoy-prod-source
  ```

Redeploy the addons once more:

```bash
  konvoy deploy addons -y
```

Check the Fluent Bit ConfigMap and confirm that Logstash_Prefix has been updated under the OUTPUT section.

```bash
  kubectl describe cm --namespace kubeaddons fluentbit-kubeaddons-fluent-bit-config
```

You can also verify that logs are being ingested under a new Elasticsearch index pattern by querying the indices using Elasticsearch API.

```bash
  curl -s external-elasticsearch.example.com:9200/_cat/indices
```

This should list the indices stored in Elasticsearch. You should see a new index created for konvoy-prod-source-YYYY.MM.DD. This confirms that logs are being ingested and the preferred index pattern is being used.

```bash
health status index                             uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_1                         Fxf2YYYXRqKUCH6NjPDwRw   1   1          9            0     42.7kb         21.3kb
green  open   konvoy-prod-source-2020.08.25     gw_jMyX3RQWqFzcPJy8MUQ   5   1     127709            0    150.1mb         75.3mb
green  open   my-index-000001                   5BXP8b-LThaDWAS2GKrxdg   5   1     250518            0    334.4mb        168.6mb
green  open   my-index-000002                   nYFWZEO7TUiOjLQXBaYJpA   5   1       1200            0     88.1kb         44.5kb
```
