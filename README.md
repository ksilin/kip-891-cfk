# Three versions of one connector plugin on one Connect cluster

Verified: 2026-09-22 on Confluent for Kubernetes 3.3.0, Confluent Platform 8.3.2, kind.

## What we test

Can one Kafka Connect cluster run three versions of the same connector plugin
at the same time, when Confluent for Kubernetes manages the cluster?

Three things must be true:

1. Each plugin version is in its own directory on `plugin.path`.
2. `GET /connector-plugins` lists the plugin once per version.
3. A connector config names a version with `connector.plugin.version`, and
   the worker loads that version.

We install `kafka-connect-jdbc` 10.9.7, 10.9.8 and 10.9.9 on one worker. We
apply three connectors: two pinned, one without a pin. We read the catalog and
the status of each connector.

## How it works

1. Each plugin version sits in its own directory on `plugin.path`. The worker
   gives each directory its own classloader (KIP-146).
2. At startup the worker calls each plugin's `version()` method. The result is
   the version in the catalog (KIP-891, Apache Kafka 4.1).
3. A connector config selects a version with `connector.plugin.version`. A bare
   value is exact. No value means the newest installed version.

The operator's `spec.build.onDemand` cannot create the directories. Its
installer writes every version into the same directory, and the last one wins.
So the directories are created another way. This repository shows two:

- **Image.** `image/Dockerfile` installs each version into its own directory.
  `k8s/30-connect-image.yaml` points `spec.image.application` at it.
- **Volume.** `k8s/25-plugins-pvc.yaml` fills a PersistentVolumeClaim with the
  same directories. `k8s/31-connect-volume.yaml` mounts it and adds it to
  `plugin.path`.

Pick one. Both were tested.

## Requirements

`docker`, `kind`, `kubectl`, `helm`, `curl`, `jq`. About 6 GiB free in Docker.

## Run

```bash
kind create cluster --name kip891
kubectl create namespace confluent
kubectl config set-context --current --namespace confluent
helm upgrade --install confluent-operator \
  https://packages.confluent.io/helm/confluent-for-kubernetes-0.1718.10.tgz \
  --namespace confluent

kubectl apply -f k8s/10-kafka.yaml -f k8s/20-postgres.yaml
kubectl wait kafka/kafka --for=jsonpath='{.status.phase}'=RUNNING --timeout=600s
kubectl apply -f k8s/21-topics.yaml
```

Then one of the two routes.

Image:

```bash
docker build -t kip891/connect-jdbc-3v:8.3.2 image
kind load docker-image kip891/connect-jdbc-3v:8.3.2 --name kip891
kubectl apply -f k8s/30-connect-image.yaml
```

Volume:

```bash
kubectl apply -f k8s/25-plugins-pvc.yaml
kubectl wait job/fill-connect-plugins --for=condition=complete --timeout=300s
kubectl apply -f k8s/31-connect-volume.yaml
```

Then the connectors:

```bash
kubectl wait connect/connect --for=jsonpath='{.status.phase}'=RUNNING --timeout=600s
kubectl apply -f k8s/40-connectors.yaml
```

## Look

Open a port-forward once:

```bash
kubectl port-forward connect-0 8085:8083 &
sleep 3
```

One plugin, three versions:

```bash
curl -s localhost:8085/connector-plugins \
  | jq -r '.[] | select(.class | test("JdbcSource")) | "\(.class) \(.version)"'
```

```
io.confluent.connect.jdbc.JdbcSourceConnector 10.9.7
io.confluent.connect.jdbc.JdbcSourceConnector 10.9.8
io.confluent.connect.jdbc.JdbcSourceConnector 10.9.9
```

Three connectors, three versions:

```bash
for c in jdbc-10-9-7 jdbc-10-9-8 jdbc-latest; do
  curl -s localhost:8085/connectors/$c/status \
    | jq -r '"\(.name): \(.connector.state) \(.connector.version)"'
done
```

```
jdbc-10-9-7: RUNNING 10.9.7
jdbc-10-9-8: RUNNING 10.9.8
jdbc-latest: RUNNING 10.9.9
```

Rows moved by each connector. The numbers grow by three every poll:

```bash
for t in pg-10-9-7-items pg-10-9-8-items pg-latest-items; do
  kubectl exec kafka-0 -c kafka -- kafka-get-offsets --bootstrap-server localhost:9071 --topic $t
done
```

## Expected

The catalog lists `JdbcSourceConnector` three times, once per version. Each
connector is RUNNING at the version its config names. The unpinned connector
runs the newest, 10.9.9. All three topics receive rows.

## Notes

- A pin to a version that is not installed fails the connector. It does not
  fall back. The `Connector` resource shows a null-pointer message from the
  worker (Apache Kafka issue KAFKA-20751); the useful message, with the list
  of installed versions, is in the operator log once.
- Every worker in a cluster needs the same directories. Task assignment does
  not look at versions.
- A plugin that does not report its own version cannot be pinned. Two such
  versions collapse into one catalog entry. Check `GET /connector-plugins`.
- The volume route was tested with `ReadWriteOnce` and one replica. More
  replicas across nodes need `ReadWriteMany`.

## Clean up

```bash
kind delete cluster --name kip891
```

## Versions

Confluent for Kubernetes chart 0.1718.10 (operator 3.3.0), Confluent Platform
8.3.2, kafka-connect-jdbc 10.9.7, 10.9.8, 10.9.9. Tested 2026-09-22.
