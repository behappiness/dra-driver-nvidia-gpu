# dra-driver-nvidia-gpu for Talos (patch queue)

This branch builds [kubernetes-sigs/dra-driver-nvidia-gpu](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu)
with the fixes that GPU passthrough to VMs (`vfio.gpu.nvidia.com` claims) needs
on Talos Linux. It holds no driver source. It holds:

- `BASE`: the upstream release tag the queue applies to.
- `patches/`: the queue, in `git am` order.
- `.github/workflows/talos-image.yaml`: the CI that builds the image.

The image is `ghcr.io/behappiness/dra-driver-nvidia-gpu:<base>-talos.<tree>`.
`<tree>` is the source tree hash after patching, so a tag always names exactly
one source. `<base>-talos` is a moving alias to the latest build.

## The patches

| Patch | Fixes | Upstream |
|---|---|---|
| 0001 | `fuser` is missing on Talos, so the busy check always timed out. The patch scans `/proc` by device number instead, which also fixes the wrong `/dev` path. | [#1504](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/pull/1504) |
| 0002 | The nvidia-persistenced socket is looked up under the host root, so persistence mode really is disabled before the unbind. | [#1568](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/pull/1568) |
| 0003 | The card's HDMI audio function stays on vfio across switches. | [#1567](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/pull/1567) |
| 0004 | Adapts 0002 to v0.5.1, which predates `internal/lookup/root`. | drop when the base has it |
| 0005 | The holder scan fails closed: permission errors, or a `/proc` that is not the initial PID namespace (e.g. Talos workload isolation), abort the switch instead of reporting "free". Unbinding a busy GPU hangs in the kernel. | not yet |
| 0006 | `--vfio-unbind-releasing-holders`: named clients that release the GPU when it is unbound do not block the switch. This is meant for dcgm-exporter with its bind/unbind watch (driver 590+, DCGM 4.5+). Empty by default. | not yet |
| 0007 | After a plugin restart, the plugin again withdraws the sibling of a still-prepared device. Before this, the vfio twin of a GPU in use was advertised again. | not yet |

## How it stays current

The workflow runs on every push here and daily at 05:17 UTC.

1. **Push run:** it builds on `BASE`.
2. **Daily run:** it builds on the newest upstream release.
3. **Applying:** it applies the queue with `git am --3way --empty=drop`. A patch that upstream has merged applies empty and is dropped.
4. **Testing and pushing:** it runs `go test -race ./...`, then builds and pushes the image. It skips the build if that tag already exists.
5. **When a new release takes the queue cleanly:** the workflow commits the rebased queue and the new `BASE` here. It also opens an issue naming the new image. That issue is the signal to bump the cluster.
6. **When the queue does not apply, or tests or the build fail:** the workflow opens an issue (or comments on the open one) with the run link. To fix it, rebase the queue by hand. Check out the new tag, `git am -3` the patches, resolve the conflicts, then `git format-patch --zero-commit --no-signature <tag>..HEAD -o patches`. Update `BASE` and push.

Deploying is deliberately manual. The cluster pins an exact tag, and a new image goes in together with the matching chart version:

```yaml
# k8s/dra-driver-nvidia-gpu/values.yaml (chart oci://registry.k8s.io/dra-driver-nvidia/charts/dra-driver-nvidia-gpu)
image:
  repository: ghcr.io/behappiness/dra-driver-nvidia-gpu
  tag: v0.5.1-talos.<tree>
kubeletPlugin:
  containers:
    gpus:
      env:
        - name: VFIO_UNBIND_RELEASING_HOLDERS
          value: dcgm-exporter
```
