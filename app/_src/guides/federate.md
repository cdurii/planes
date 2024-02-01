---
title: Federate zone control plane
---

With {{site.mesh_product_name}} you can first start with just one zone control plane
and then federate it to a multi-zone deployment.
This way you can:
- see your mesh deployment in one centralized place
- connect multiple zones and introduce cross zone connectivity
- manage policies that are pushed to all zones

## Prerequisites
- Completed [quickstart](/docs/{{ page.version }}/quickstart/kubernetes-demo/) to set up a zone control plane with demo application

## Start a global control plane

### Start a new Kubernetes cluster

Currently, it's not possible to deploy global and zone control plane in the same Kubernetes cluster, therefore we need a new Kubernetes cluster.

{% tip %}
You can skip this step if you already have a Kubernetes cluster running.
It can be a cluster running locally or in a public cloud like AWS EKS, GCP GKE, etc.
{% endtip %}

```sh
kind create cluster --name=mesh-global
```

By default, Kind does not support LoadBalancer therefore we need to configure it by following this [guide](https://kind.sigs.k8s.io/docs/user/loadbalancer/).
Note that all public cloud providers support load balancers out of the box.

### Deploy a global control plane

```sh
helm install --kube-context=kind-mesh-global --create-namespace --namespace {{site.mesh_namespace}} \
--set {{site.set_flag_values_prefix}}controlPlane.mode=global \
--set {{site.set_flag_values_prefix}}controlPlane.defaults.skipMeshCreation=true \
{{ site.mesh_helm_install_name }} {{ site.mesh_helm_repo }}
```

We skip default mesh creation as we will bring mesh from zone control plane in the next steps.

### Sync endpoint

Find the external IP and port of the `{{site.mesh_cp_zone_sync_name_prefix}}global-zone-sync` service in the `{{site.mesh_namespace}}` namespace:

```sh
kubectl get services -n {{site.mesh_namespace}}
NAMESPACE     NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                  AGE
{{site.mesh_namespace}}   {{site.mesh_cp_zone_sync_name_prefix}}global-zone-sync     LoadBalancer   10.105.9.10     35.226.196.103   5685:30685/TCP                                                           89s
{{site.mesh_namespace}}   {{site.mesh_cp_name}}     ClusterIP      10.105.12.133   <none>           5681/TCP,443/TCP,5676/TCP,5677/TCP,5678/TCP,5679/TCP,5682/TCP,5653/UDP   90s
```

In this example the value is `35.226.196.103:5685`. You pass this as the value of `<global-kds-address>` when you federate the zone control plane.
If you see `<Pending>` you either need to wait until load balancer is provisioned or you need to make sure that your Kubernetes cluster support provisioning load balancers. 

## Copy resources from zone to global control plane

To federate zone control plane without any traffic interruption, we need to copy resources like secrets, meshes etc.
First, we need to expose API server of zone control plane:

```sh
kubectl --context=kind-mesh-zone port-forward svc/{{site.mesh_cp_name}} -n {{site.mesh_namespace}} 5681:5681
```

Then we export resources:
```sh
export ZONE_USER_ADMIN_TOKEN=$(kubectl --context=kind-mesh-zone get secrets -n kuma-system admin-user-token -ojson | jq -r .data.value | base64 -d)
kumactl config control-planes add \
  --address http://localhost:5681 \
  --headers "authorization=Bearer $ZONE_USER_ADMIN_TOKEN" \
  --name "zone-cp" \
  --overwrite  
  
kumactl export --profile=federation --format=kubernetes > resources.yaml
```

And finally, we apply resources on global control plane
```sh
kubectl apply --context=kind-mesh-global -f resources.yaml
```

## Connect zone control plane to global control plane

Update Helm deployment of zone control plane to configure connection to the global control plane.

```sh
helm upgrade --kube-context=mesh-zone --namespace {{site.mesh_namespace}} \
--set {{site.set_flag_values_prefix}}controlPlane.mode=zone \
--set {{site.set_flag_values_prefix}}controlPlane.zone=zone-1 \
--set {{site.set_flag_values_prefix}}ingress.enabled=true \
--set {{site.set_flag_values_prefix}}controlPlane.kdsGlobalAddress=grpcs://<global-kds-address>:5685 \
--set {{site.set_flag_values_prefix}}controlPlane.tls.kdsZoneClient.skipVerify=true \
{{ site.mesh_helm_install_name }} {{ site.mesh_helm_repo }}
```

## Verify federation

To verify federation, first port-forward the API service from the global control plane to port 15681 to avoid collision with previous port forward. 

```sh
kubectl --context=kind-mesh-global port-forward svc/{{site.mesh_cp_name}} -n {{site.mesh_namespace}} 15681:5681
```

And then navigate to [127.0.0.1:15681/gui](http://127.0.0.1:15681/gui) to see the GUI.

You should eventually see
* a zone in list of zones
* policies including `redis` MeshTrafficPermission that we applied in the quickstart guide.
* data plane proxies for the demo application that we installed in the quickstart guide.

### Apply policy on global control plane

We can check policy synchronization from global control plane to zone control plane by applying a policy on global control plane:

```
echo "apiVersion: kuma.io/v1alpha1
kind: MeshCircuitBreaker
metadata:
  name: demo-app-to-redis
  namespace: {{site.mesh_namespace}}
  labels:
    kuma.io/mesh: default
spec:
  targetRef:
    kind: MeshService
    name: demo-app_kuma-demo_svc_5000
  to:
  - targetRef:
      kind: MeshService
      name: redis_kuma-demo_svc_6379
    default:
      connectionLimits:
        maxConnections: 2
        maxPendingRequests: 8
        maxRetries: 2
        maxRequests: 2" | kubectl --context=kind-mesh-global apply -f -
```

If we execute the following command:
```sh
kubectl get --context=kind-mesh-zone meshcircuitbreakers -A
```
The policy should be eventually available in zone control plane
```
NAMESPACE     NAME                                                TARGETREF KIND   TARGETREF NAME
kuma-system   demo-app-to-redis-65xb45x2xfd5bf7f                  MeshService      demo-app_kuma-demo_svc_5000
kuma-system   mesh-circuit-breaker-all-default                    Mesh
```

## Next steps

* Read the [multi-zone](/docs/{{ page.version }}/production/cp-deployment/multi-zone) docs to learn more about this deployment model and cross-zone connectivity.
