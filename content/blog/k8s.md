+++
title = "k8s 笔记"
description = ""
date = 2023-03-04
+++

## Pod镜像拉取超时

```
  Warning  Failed     17m                 kubelet            Failed to pull image "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.7.0": rpc error: code = Unknown desc = failed to pull and unpack image "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.7.0": failed to resolve reference "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.7.0": failed to do request: Head "https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/sig-storage/csi-node-driver-registrar/manifests/v2.7.0": dial tcp 74.125.203.82:443: i/o timeout
  Normal   BackOff    12m (x31 over 25m)  kubelet            Back-off pulling image "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.7.0"
```

通过国内源拉取，再打上相应的标签

```
> ctr -n k8s.io i pull registry.cn-hangzhou.aliyuncs.com/google_containers/csi-node-driver-registrar:v2.7.0
> ctr -n k8s.io image tag registry.cn-hangzhou.aliyuncs.com/google_containers/csi-node-driver-registrar:v2.7.0 registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.7.0
```

