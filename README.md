<p align="center">
  <a href="https://mitrafsnap.github.io"><img src="https://img.shields.io/badge/Project%20page-mitrafsnap.github.io-cc3333" alt="Project page"></a>
  <a href="https://github.com/mitrafsnap/demo"><img src="https://img.shields.io/badge/Live%20demo-mitrafsnap%2Fdemo-cc3333" alt="Live demo"></a>
  <img src="https://img.shields.io/badge/License-Apache--2.0-1c39bb" alt="License Apache-2.0">
</p>

# MitraFSnap snapshotter

> Part of **MitraFSnap**, Build Less, Pull Less, Restart Never: Fast Container Image
> Distribution and In-Place Updates. The paper is under review.
> [Project page, videos and results](https://mitrafsnap.github.io)

MitraFSnap updates the data mounted in a running container without replacing the container.
It consists of an OCI compatible builder, a registry and this containerd snapshotter. The
snapshotter pulls images lazily, so a container starts before its image is downloaded, and it
adds **refresh**, which moves the layers of a running container to a new image version.

A refresh pairs each mounted layer with its counterpart in the new version and compares their
chunk indexes (TOCs). It then replaces the readers of the changed layers and invalidates only
the cached ranges whose chunks changed. Unchanged layers and chunks stay in the cache, the
container keeps its mount, and the application decides when to reopen the updated files.

| Repository | Role in MitraFSnap |
|---|---|
| [builder](https://github.com/mitrafsnap/builder) | builds independent, lazy loadable allotments and their chunk indexes |
| [registry](https://github.com/mitrafsnap/registry) | turns a partition selection into prefetch hints over the complete image |
| **snapshotter** (this repository) | lazy pulling and refresh of running containers |
| [demo](https://github.com/mitrafsnap/demo) | live demo that compares Docker, eStargz, 2DFS and MitraFSnap |
| [evaluation](https://github.com/mitrafsnap/evaluation) | build, pull and update benchmarks |

## Build and install

The snapshotter needs Linux with FUSE, containerd 2 and Go 1.25 or later. The demo runs it
with containerd 2.2.

```sh
git clone https://github.com/mitrafsnap/snapshotter.git
cd snapshotter
make containerd-stargz-grpc ctr-remote
sudo install out/containerd-stargz-grpc out/ctr-remote /usr/local/bin/
```

containerd reaches the snapshotter as a proxy plugin. Add the plugin to
`/etc/containerd/config.toml` and restart containerd.

```toml
[proxy_plugins.stargz]
  type = "snapshot"
  address = "/run/containerd-stargz-grpc/containerd-stargz-grpc.sock"
  [proxy_plugins.stargz.exports]
    root = "/var/lib/containerd-stargz-grpc/"
    enable_remote_snapshot_annotations = "true"
```

Then start the daemon with a configuration file such as the one below
(`/etc/containerd-stargz-grpc/config.toml`).

```toml
[[resolver.host."registry.example:5000".mirrors]]
  host = "registry.example:5000"
  insecure = true

[fuse]
  passthrough = true
```

```sh
sudo containerd-stargz-grpc --config /etc/containerd-stargz-grpc/config.toml
```

## Usage

Pull an image lazily and run it. With the MitraFSnap registry, a partition suffix on the tag
selects the allotments to prefetch, and every other file is fetched when the container first
reads it.

```sh
ctr-remote images rpull --plain-http registry.example:5000/app:v1--0.0.0.1
ctr-remote run -d --snapshotter=stargz registry.example:5000/app:v1--0.0.0.1 app
```

After a new version is pushed under the same tag, refresh the running container in place, or
let the snapshotter watch the tag.

```sh
ctr-remote refresh --plain-http registry.example:5000/app:v1--0.0.0.1
ctr-remote watch --interval 30s registry.example:5000/app:v1--0.0.0.1
```

| Command | What it does |
|---|---|
| `refresh <ref>` | refreshes the containers of `<ref>` to the version the tag points to now |
| `refresh <old-ref> <new-ref>` | refreshes from one image to another |
| `watch <ref> [--interval 30s]` | polls the tag and refreshes whenever it changes |
| `unwatch <ref>` | stops watching a tag |
| `watch-list` | lists the active watches |
| `fetch-toc <repo>@<digest>` | prints the TOC of a layer, for debugging |

`refresh` and `watch` accept `--with-background-fetch`, which fetches the changed chunks of
every refreshed layer in the background after the swap. A refresh also records the new version
in containerd's image store, so a later refresh starts from it.

The configuration option `[fuse] disable_xattrs = true` makes the FUSE mount answer `getxattr`
with ENOSYS. Some kernels, including the one in Docker Desktop, otherwise fail to exec binaries
from an overlay on top of the FUSE mount.

## Relation to the stargz snapshotter

This repository is a fork of the containerd
[stargz snapshotter](https://github.com/containerd/stargz-snapshotter), which provides lazy
pulling of eStargz and zstd:chunked images. MitraFSnap adds refresh and watch, the delta
comparison of chunk indexes, the handling of the registry's prefetch annotations and the
`disable_xattrs` option. Everything else follows upstream, and the upstream documentation is
in [UPSTREAM.md](UPSTREAM.md) and [docs/](docs/).

## License

Apache License 2.0, as upstream. See [LICENSE](LICENSE) and [NOTICE.md](NOTICE.md).
