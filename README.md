scaleway-kapsule-scale-internals
================================

3 internal components of Kapsule are deployed on the worker nodes side instead of the control plane and thus, can't be managed directly by Scaleway. They are updated with cluster update but their scaling is left to the customer discretion based on their needs and deployed nodes (numbers, type, usage, autoscaling, etc....)

```
$ kubectl get deploy -n kube-system
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
cilium-operator   1/1     1            1           4y100d
coredns           1/1     1            1           4y100d
metrics-server    1/1     1            1           4y100d
```

Nonetheless, to help the scaling of such components for increased nodes and HA, here is a guideline procedure that can be leveraged.

cilium-operator
===============

According to [Cilium documentation](https://docs.cilium.io/en/stable/internals/cilium_operator/#highly-available-cilium-operator), scaling the operator can be done for HA purpose. If the Kapsule cluster has at least 3 nodes (and will not go lower than that), scaling the replica to 3 is enough.

```
$ kubectl scale deploy cilium-operator -n kube-system --replicas=3
```

A leader election is automatically managed by the operator and the deployment includes a pod anti-affinity policy based on node hostname.

coredns
=======

Following [this documentation](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/) the scaling of core-dns is better handled by a propotionnal cluster autoscaler: it will scale the needed number of pods based on the number of nodes. 

But the base deployment of coredns does not include a topology spread constraint, this could lead to having several pods in the same zone, bypassing HA consideration.

We will complement the base core-dns deployment with an HA version by copying the original deployment.

```
$ kubectl get cm coredns -n kube-system -oyaml | sed 's/name: coredns/name: coredns-ha/' | kubectl apply -f -
configmap/coredns-ha created

$ kubectl patch cm coredns-ha -n kube-system --type=json -p='[{"op": "remove", "path": "/metadata/labels/k8s.scw.cloud~1object"}, {"op": "remove", "path": "/metadata/labels/k8s.scw.cloud~1system"}]'
configmap/coredns-ha patched

$ kubectl get deploy coredns -n kube-system -oyaml | sed 's/name: coredns/name: coredns-ha/' | kubectl apply -f -
deployment.apps/coredns-ha created

$ kubectl patch deploy coredns-ha -n kube-system --type=json -p='[{"op": "remove", "path": "/metadata/labels/k8s.scw.cloud~1object"}, {"op": "remove", "path": "/metadata/labels/k8s.scw.cloud~1system"}]'
deployment.apps/coredns-ha patched
```

At this point, you have two copies of the core-dns pod. You can scale down to the zero the original one (it can be scaled up back to 1 for debugging need).

```
$ kubectl scale deploy coredns -n kube-system --replicas=0
deployment.apps/coredns scaled
```

**Important note**

This will freeze the version of the image used for core-dns. In case of upgrade of the Kapsule cluster, you will have to check the image version in the original deployment to update you own.

We will now patch this deployement with topology spread constraints. To leverage the topology spread, a multi-AZ cluster (with pools on multiple AZ) is needed.

```
$ kubectl patch deploy coredns-ha --patch-file=coredns-patch.yaml --type='merge' -n kube-system
deployment.apps/coredns-ha patched
```

Now, we can add the proportional cluster autoscaler.

```
$ kubectl apply -f dns-horizontal-autoscaler.yaml
serviceaccount/kube-dns-autoscaler created
clusterrole.rbac.authorization.k8s.io/system:kube-dns-autoscaler created
clusterrolebinding.rbac.authorization.k8s.io/system:kube-dns-autoscaler created
deployment.apps/kube-dns-autoscaler created
```

You can now fine-tune the autoscaler following [this documentation](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/#tuning-autoscaling-parameters).

metrics-server
==============

For metrics-server HA, we will leverage [this documentation](https://github.com/kubernetes-sigs/metrics-server?tab=readme-ov-file#high-availability). But as for coredns, the Out-the-box deployment is not ready for HA and can't be used as-is with 2 replicas.

First, we will add the missing pod disruption budget:

```
$ kubectl apply -f metrics-server-pdb.yaml
poddisruptionbudget.policy/metrics-server created
```

We now copy the original deployment:

```
$ kubectl get deploy metrics-server -n kube-system -oyaml | sed 's/name: metrics-server/name: metrics-server-ha/' | kubectl apply -f -
deployment.apps/metrics-server-ha created

$ kubectl patch deploy metrics-server-ha -n kube-system --type=json -p='[{"op": "remove", "path": "/metadata/labels/k8s.scw.cloud~1object"}, {"op": "remove", "path": "/metadata/labels/k8s.scw.cloud~1system"}]'
```

**Important note**

This will freeze the version of the image used for metrics-server. In case of upgrade of the Kapsule cluster, you will have to check the image version in the original deployment to update you own.

At this point, you have two copies of the metrics-server pod. You can scale down to the zero the original one (it can be scaled up back to 1 for debugging need).

```
$ kubectl scale deploy metrics-server -n kube-system --replicas=0
deployment.apps/metrics-server scaled
```

And we apply the patch to add HA:

```
$ kubectl patch deploy metrics-server-ha --patch-file=metrics-server-patch.yaml --type='merge' -n kube-system
deployment.apps/metrics-server-ha patched
```

Please note that to avoid unschedulable pods, `preferredDuringSchedulingIgnoredDuringExecution` was used for the pod anti-affinity instead of `requiredDuringSchedulingIgnoredDuringExecution`.

cluster autoscaling consideration
=================================

If the Kapsule cluster has a node pool with autoscaling activated, it would be better to get the internals components to stick to a node pool with a fixed size to avoid repeated deletion and re-creation of the internal components pods, especially for coredns.

To avoid this, an additional patch can be applied.

First, edit the `node-affinity-patch.yaml` file to add the list of node pools without autoscaling activated:

```
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: k8s.scaleway.com/pool-name
                operator: In
                values:
                - <name of a pool>
                - <name of an other pool>
```

Then apply the patch.

```
$ kubectl patch deploy coredns-ha --patch-file=node-affinity-patch.yaml --type='merge' -n kube-system
deployment.apps/coredns-ha patched
```

And eventually, rollout the deployment to ensure the pod are in the right place.

```
$ kubectl rollout restart deploy coredns-ha -n kube-system
```


support & uninstall
===================

The presented methods are based on official documentations BUT modify Kapsule inner components. Questions regarding those modifications can't be brought to Scaleway support and must be handled by the customer Kubernetes experts.

If needed, the different added components can be deleted and the deployments scaled back to their normal size to bring back these components to their supported state.

```
$ kubectl scale deploy cilium-operator -n kube-system --replicas=1
```

```
$ kubectl scale deploy coredns -n kube-system --replicas=1
$ kubectl delete -f dns-horizontal-autoscaler.yaml
$ kubectl delete deploy coredns-ha -n kube-system
$ kubectl delete cm coredns-ha -n kube-system
```

```
$ kubectl scale deploy metrics-server -n kube-system --replicas=1
$ kubectl delete deploy metrics-server-ha -n kube-system
$ kubectl delete -f metrics-server-pdb.yaml
```
