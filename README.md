# gdex-redis

Standalone Redis deployment used as the broker/result backend for
`gdex-web-services` Celery tasks.

## Contents

- **`Dockerfile`** — builds a container image from the upstream `redis:8` base
  image.
- **`chart/`** — Helm chart (`redis`) that deploys the image as a single-replica
  `Deployment` + `Service` on Kubernetes.
- **`.github/workflows/container-build.yaml`** — CI pipeline that builds and
  publishes the image and updates the chart's image tag.

## Image

The image is built from [`Dockerfile`](Dockerfile) and published to UCAR's
Harbor registry:

```
hub.k8s.ucar.edu/calie-gdex/redis
```

### CI pipeline

On every pull request merged into `main` (or via manual `workflow_dispatch`),
[`container-build.yaml`](.github/workflows/container-build.yaml):

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

## Local development

Build and run the image locally:

```bash
docker build -t gdex-redis .
docker run -p 6379:6379 gdex-redis
```
