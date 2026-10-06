# gdex-redis

Standalone Redis deployment used as the broker/result backend for
`gdex-web-services` Celery tasks.

## Contents

- **`Dockerfile`** — builds a container image from the upstream `redis:8` base
  image.
- **`chart/`** — Helm chart (`redis`) that deploys the image as a single-replica
  `Deployment` + `Service` on Kubernetes.
- **`.github/workflows/build-push.yaml`** — CI pipeline that builds and
  publishes the image and updates the chart's image tag.

## Image

The image is built from [`Dockerfile`](Dockerfile) and published to UCAR's
Harbor registry:

```
hub.k8s.ucar.edu/calie-gdex/redis
```

### CI pipeline

On every pull request merged into `main` (or via manual `workflow_dispatch`),
[`build-push.yaml`](.github/workflows/build-push.yaml):

1. Logs in to Harbor.
2. Builds the image and tags it `v<run_id>`.
3. Pushes both the versioned tag and `latest`.
4. Updates `chart/values.yaml`'s `image.tag` to the new versioned tag and
   commits/pushes that change to `main`.

### Test instances

To build and try out changes before merging to `main`, use a personal branch
prefix (e.g. `<username>/description`, like `calie/bump-redis-version`) and
trigger the pipeline manually against that branch:

1. Push your changes to `<username>/my-change`.
2. In GitHub, go to **Actions → Container Build → Run workflow**, pick your
   branch from the branch dropdown, and run it. `workflow_dispatch` builds
   and pushes an image tagged `v<run_id>` without touching `chart/values.yaml`
   on `main` (that update only happens on the merge-triggered path).
3. Deploy that image to your own namespace/release so it doesn't collide with
   the shared instance:

   ```bash
   helm install redis-<username> chart/ \
     -n <username>-test --create-namespace \
     --set image.tag=v<run_id>
   ```

4. Tear it down when done:

   ```bash
   helm uninstall redis-<username> -n <username>-test
   ```

This keeps test instances isolated by release name/namespace and image tag,
while `main` and its auto-updated `image.tag` remain the source of truth for
the shared deployment.

## Helm chart

Deploy with:

```bash
helm install redis chart/ -n <namespace>
```

Or render manifests locally without installing:

```bash
helm template chart/
```

Key values in [`chart/values.yaml`](chart/values.yaml):

| Key                  | Description                          | Default                                  |
|----------------------|---------------------------------------|-------------------------------------------|
| `replicaCount`       | Number of Redis replicas              | `1`                                       |
| `image.repository`   | Image path                            | `hub.k8s.ucar.edu/calie-gdex/redis`       |
| `image.tag`          | Image tag                             | `latest`                                  |
| `image.pullPolicy`   | Pod image pull policy                 | `Always`                                  |
| `resources`          | CPU/memory requests & limits          | `0.5`/`1Gi` requests, `1`/`2Gi` limits    |
| `service.port`       | Redis port exposed by the Service     | `6379`                                    |
| `persistence.enabled`| Store `/data` on a PVC (else `emptyDir`) | `true`                                |
| `persistence.size`   | PVC size                              | `5Gi`                                     |
| `persistence.storageClass` | StorageClass (empty = cluster default) | `""`                              |
| `persistence.existingClaim` | Use an existing PVC instead of creating one | `""`                        |
| `redis.appendonly`   | Enable the append-only file (AOF)     | `yes`                                     |
| `redis.appendfsync`  | AOF fsync policy                      | `everysec`                                |
| `redis.maxmemory`    | Redis memory cap (keep below the pod limit) | `1536mb`                            |
| `redis.maxmemoryPolicy` | Eviction policy                    | `noeviction`                              |

### Persistence

Redis data (`/data`) lives on a PVC (`redis-data`), with append-only file (AOF) enabled
(`appendfsync everysec`), so queued tasks and results survive pod restarts and
reschedules; at most ~1 second of writes can be lost on a crash. The Deployment
uses `strategy: Recreate` because the volume is `ReadWriteOnce`, so a rollout
briefly takes Redis down. The PVC is annotated `helm.sh/resource-policy: keep`, so it survives
`helm uninstall`; delete `redis-data` manually to wipe the data (test instances
should do this too).

### Celery configuration

The Redis server does not choose a DB number; clients do, via the URL path.
Use separate DBs for the broker and results (set in `gdex-web-services`), e.g.:

```
broker_url     = redis://redis:6379/0
result_backend = redis://redis:6379/1
```

## Local development

Build and run the image locally:

```bash
docker build -t gdex-redis .
docker run -p 6379:6379 gdex-redis
```
