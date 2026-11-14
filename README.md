# helm-chart-komga

Deploys [Komga](https://komga.org/) from its official image, [`gotson/komga`](https://github.com/gotson/komga).
Komga scans folders of comics, manga, BDs and magazines, and serves them to read in the browser, over OPDS and through its API.

The chart runs one replica:

- A Deployment that stops the old pod before starting the new one, since `/config` is `ReadWriteOnce`.
- A PersistentVolumeClaim for `/config`: Komga's SQLite databases, its Lucene search index and its logs.
  It's an ordinary claim the chart owns, so its size can be raised in the values; most storage classes allow claims to grow but not shrink.
- Each library folder from the values, mounted where it says.
- A Service on Komga's port, 25600, and an optional Gateway API `HTTPRoute` to it.
- A start-up, liveness and readiness probe on `/actuator/health`, which Komga answers without a login.
  Start-up gets 5 minutes, since Komga runs its database migrations before it listens.

Komga's build has Spring's actuator but no Prometheus registry, so there is nothing to scrape, and the chart has no `ServiceMonitor` or dashboards.
Libraries are added in Komga's web UI, from the folders mounted here.

## Values

| Value | Default |
| --- | --- |
| `image.repository` | `gotson/komga` |
| `image.tag` | the chart's `appVersion` |
| `user` | `1000` |
| `group` | `1000` |
| `javaToolOptions` | none |
| `resources` | requests `cpu: 100m`, `memory: 1Gi`; no limits |
| `persistence.size` | `2Gi` |
| `persistence.storageClassName` | the cluster's default class |
| `libraries` | none |
| `affinity` | none |
| `httpRoute.enabled` | `false` |
| `httpRoute.parentRefs` | none; required when enabled |
| `httpRoute.hostnames` | none |

### `/config` and its size

Komga refuses to start with its databases on a network filesystem such as NFS or CIFS, so the storage class has to give block storage.

Most of `/config` is thumbnails.
Komga makes one per book and stores it inside `database.sqlite`, not as separate files.
By default it's 300px on the longest edge; 600, 900 and 1200px are the other settings, in Komga's server settings.
At an estimated 30KB each, 300px thumbnails take about 500MB per 16,700 books, and the larger settings take more.
Size the claim for that, plus the rest of the database, the search index and the logs, with headroom.

### The heap

Without `-Xmx`, the JVM's maximum heap is a quarter of the memory it sees: the container's limit if it has one, otherwise the node's.
Komga's documentation gives the default as usually 1GB, and `javaToolOptions: -Xmx4g` sets a cap.

Leaving `-Xmx` out lets the heap grow with the library instead of failing at a fixed cap.
The JVM then holds on to memory it grew into, so `-XX:G1PeriodicGCInterval=300000` is worth adding: it runs a collection every 5 minutes when Komga is idle, and gives the unused heap back to the node.
Set the memory request from what Komga is seen using.

### User and group

Komga runs as user and group `1000` by default, the user in upstream's examples.
It needs no root: it listens above port 1024 and writes only to `/config` and its libraries.
The pod's `fsGroup` is the same group, so `/config`'s volume is writable.

Komga writes to its libraries when books or series are deleted, converted to CBZ, have their extensions repaired or duplicate pages removed, and when books are imported.
Set `user` and `group` to the libraries' owner when it isn't 1000 and Komga has to write there.
Volumes that map ownership themselves, such as some network shares, may need no change.

### Libraries

`libraries` lists the folders Komga's libraries are added from.
Each entry has a `name`, a `mountPath`, a `volume`, which is any Kubernetes volume source, and `readOnly`, which defaults to `false` so Komga's file features work.

## Requirements

- A storage class giving block storage for `/config`, the cluster's default unless named.
- The Gateway API's `HTTPRoute` CRD, when `httpRoute.enabled` is set.

## Usage

```sh
helm repo add komga https://tonylea.github.io/helm-chart-komga
helm install komga komga/komga --values my-values.yaml
```

```yaml
javaToolOptions: -XX:G1PeriodicGCInterval=300000
persistence:
  size: 2Gi
libraries:
  - name: comics
    mountPath: /data/comics
    volume:
      hostPath:
        path: /srv/comics
        type: Directory
```

## Development

Pull requests run:

- `helm lint --strict`, and the rendered chart checked by kubeconform against Kubernetes 1.37, with `ci/kind-values.yaml`.
- helm-unittest's tests in `tests/`.
- An install and upgrade with chart-testing in a kind cluster.

Commits follow Conventional Commits.
Releases are versioned and tagged automatically from `main`, and published to the Helm repository on this repo's GitHub Pages.
