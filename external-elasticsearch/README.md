# Konvoy logging with external elasticsearch

Konvoy provides a cluster-level logging solution consisting of three primary addons. These compnents are automatically configured to work together and do not require anything else to get up and running.

- Fluent Bit: Log forwarding
- Elasticsearch: Log storage and searching
- Kibana: Log visualization and analysis

Some teams may already have an existing Elasticsearch backend, which may be preferable to store logs instead of the addon that ships with Konvoy. However, the default behavior of the Fluent Bit addon is to forward logs to the Elasticsearch backed running inside Konvoy. If we want the Fluent Bit addon to forward logs to an external Elasticsearch backend, it will be necessary to override the addon configs in our Konvoy cluster.yaml file.

## Fluent Bit addon overrides

Here, he want to edit the cluster.yaml file and override the Fluent Bit configuration so that it will forward all host and container logs to the preferred Elasticsearch cluster running outside Konvoy.

Right after the fluentbit addon section, values can be added so that a different backend is used for Elasticsearch. Specifically, these are the es.host and es.port backend values. These should be set like this:

```bash
   - name: fluentbit
      enabled: true
      values: |
        backend:
          es:
            # Use the host and port for the external Elasticsearch backend.
            host: external-elasticsearch.example.com
            port: 9200
```

If the Konvoy cluster has already been deployed and these Fluent Bit changes are being made later, make sure to rerun the addon deployment.

```bash
  konvoy deploy addons -y
```

In the external Elasicsearch environment, confirm that logs from Konvoy are now being ingested there. A quick way to confirm this is to filter a search by a hostname or pod ID from the source Konvoy cluster. This should return search results and demonstrate that logs from Konvoy are being forwarded to the external Elasticsearch environment. A good example is to check that etcd logs from a Konvoy master host are being written to the Elasticsearch. Here's an example:

```bash
  kubernetes.pod_name: "etcd" AND kubernetes.host: "konvoy-master-1.example.com"
```
