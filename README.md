# Kube  -- VM Networking

A demo setup that will be required for showing how to make Kubernetes and VM can communicate over ClusterIP and Pod IP

## Tools

- [direnv](https://direnv.net)
- [multipass](https://multipass.run/)
- [k3s](https://k3s.io/)
- [helm](https://helm.sh/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [calico](https://projectcalico.docs.tigera.io/)
- [jq](https://stedolan.github.io/jq/)

## Kubernetes Cluster

### Settings

The k3s cluster will be a single node cluster run via multipass VM. We will configure that to with the following flags,

- `--cluster-cidr=172.16.0.0/24` allows us to create 65 – 110 Pods on this node
- `--service-cidr=172.18.0.0/20` allows us to create 4096 services
- `--disable=traefik` disable `traefik` deployment

For more information on how to calculate the number of pods and service per CIDR rang, check the [GKE doc](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips).

### Create Kubernetes Cluster

The following command will create kubernetes(k3s) cluster and configure it with no network plugin. Check the [k3s with Calico](./configs/k3s-cni-calico)

```shell
multipass launch \
  --name cluster1 \
  --cpus 4 --mem 8g --disk 20g \
  --cloud-init configs/k3s-cni-calico
```

Retrieve and save the `kubeconfig` and save it locally to allow the access to the cluster from the host,

```shell
mkdir -p "$KUBECONFIG_DIR"
chmod -R 644 "$KUBECONFIG_DIR"
export CLUSTER1_IP=$(multipass info cluster1 --format json  | jq -r '.info.cluster1.ipv4[0]')
# Copy kubeconfig
multipass exec cluster1 sudo cat /etc/rancher/k3s/k3s.yaml > "$KUBECONFIG_DIR/config"
sed -i -E "s|127.0.0.1|${CLUSTER1_IP}|" "$KUBECONFIG_DIR/config"
sed -i -E "s|(^.*:\s*)(default)|\1cluster1|g" "$KUBECONFIG_DIR/config"
```

### Deploy Calico Network Plugin

Let us deploy [Calico](https://projectcalico.docs.tigera.io) plugin. The steps are based on the [Quickstart](https://projectcalico.docs.tigera.io/getting-started/kubernetes/k3s/quickstart) except we make few changes to the custom resources.

Deploy the operator,

```shell
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```

Wait for the operator to be running,

```shell
kubectl rollout status -n tigera-operator deploy/tigera-operator --timeout=180s
```

Let us now deploy the Calico custom resource with ipPool as `172.16.0.0/24`, this will allow us to run max of `110` pods. We also make sure we enable ipforwarding in calico container settings.

```shell
kubectl create -f manifests/calico-cr.yaml
```

Watch for all pods in all namespaces to be running ( changed from pending to running),

```shell
watch kubectl get pods -A 
```

Deploy a simple nignx service to test the connectivity from the VM,

```shell
kubectl --context=cluster1 run nginx --image=nginx --expose --port 80 --dry-run=client -o yaml | k --context=cluster1  apply  -f -
```

### Virtual Machine

```shell
multipass launch --name vm1 \
  --cpus 2 --mem 4g --disk 20g \
  --cloud-init configs/workload-vm
```

Lets mount the local `.kube` directory inside the vm

```shell
multipass  mount "$PWD/.kube" vm1:/home/ubuntu/.kube
```

Shell into the vm,

```shell
multipass exec vm1 bash
```

### Add routes on VMs

```shell
sudo ip route add 172.16.0.0/28 via $CLUSTER1_IP
sudo ip route add 172.18.0.0/20 via $CLUSTER1_IP
```

`CLUSTER1_IP` - is the VM where the k3s cluster is running

Note down the `service IP(Cluster-IP)` of the nginx pod,

```shell
export NGINX_SVC_IP=$(kubectl get svc nginx -ojsonpath='{.spec.clusterIP}')
```

Note down the `POD IP` of the nginx pod,

```shell
export NGINX_POD_IP=$(kubectl get pod nginx -ojsonpath='{.status.podIP}')
```

Now you can check the connectivity to service ip using curl,

```shell
curl $NGINX_SVC_IP
```

Now you can check the connectivity to pod ip using curl,

```shell
curl $NGINX_POD_IP
```

For both the commands you should see the NGINX default home page like,

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Cleanup

Delete the vms,

```shell
multipass delete cluster1 --purge
multipass delete vm1 --purge
rm -rf "$PWD/.kube/config"
```
