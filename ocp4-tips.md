# Tips for troubleshooting OpenShift 4 installation

## Before installation

These are things that you can do before installation:

1. Save your install-config.yaml - the `openshift-install` command consumes (deletes) the file to generate the ignition file.

2. Save your private key - the pair of the public key that is deployed as ssh-key in the install-config.yaml file. You can use this key to login to the provisioned RedHat Core OS machines. 

3. Check or resolve all hostnames and IP addresses that will be used in the installation (unless the entries will be provisioned with the cluster)

	- master (control plane) 
	- worker (compute)
	- bootstrap 
	- api endpoint
	- api-int endpoint
	- apps endpoint
	- etcd-* endpoint

4. Ensure load-balancers are listening to the proper ports (6443, 22623, 80 and 443).

## Debugging bootstrap node

Installation commences once the CoreOS nodes are started. The installation program just check the installation status. The first part of the installation is involving the bootstrap node. The bootstrap node perform the following (using the bootkube.service):

- Open port 22623 and generates machine configurations for the master nodes and worker nodes. 
- It then waits for the ETCD cluster is initialized (it checked the addresses of `etcd-<n>.<name>.<domain>` port 2379 as noted by the number of control-plane nodes)
- Once the ETCD cluster is initialized, it deploys the important containers to the contol plane nodes and checked to make sure those are initialized
- API server version is then initialized on port 6443 of the bootstrap node, eventually both port 6443 and 22623 on all control plane nodes are initialized

Debugging bootstrap node:

- Login to the bootstrap node:

		ssh -i <ssh-private-key> core@<bootstrap-ip>

- Check the bootkube-service:

		journalctl -f -u bootkube.service

	or to get all data from boot, but not the streaming one:

		journalctl -b -u bootkube.service > bootkube.log 

- Determine on which stage the boot is happening:

	- Initial bootkube rendering, when this is completed, the machine config operator (MCO) is initialized 
```
-- Logs begin at Wed 2019-10-02 16:51:10 UTC, end at Wed 2019-10-02 17:25:23 UTC. --
Oct 02 16:51:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local systemd[1]: Started Bootstrap a Kubernetes cluster.
Oct 02 16:51:50 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Pulling release image...
Oct 02 16:52:08 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: 9e928444323ac8efafc26ecc9b42be1afcfd46706ee351826098db7f33b6e7b4
Oct 02 16:52:28 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Rendering Cluster Version Operator Manifests...
Oct 02 16:52:29 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Rendering cluster config manifests...
Oct 02 16:52:32 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Rendering Kubernetes API server core manifests...
Oct 02 16:52:37 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Rendering Kubernetes Controller Manager core manifests...
Oct 02 16:52:42 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Rendering Kubernetes Scheduler core manifests...
Oct 02 16:52:46 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Rendering MCO manifests...
Oct 02 16:52:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: I1002 16:52:49.168818       1 bootstrap.go:86] Version: 4.1.18-2
01909201915-dirty (a2175e587b007272f26305fe7d8b603c49e8f1fc)
Oct 02 16:52:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: I1002 16:52:49.175400       1 bootstrap.go:142] manifests/machin
econfigcontroller/controllerconfig.yaml
Oct 02 16:52:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: I1002 16:52:49.179413       1 bootstrap.go:142] manifests/master
.machineconfigpool.yaml
Oct 02 16:52:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: I1002 16:52:49.179835       1 bootstrap.go:142] manifests/worker
.machineconfigpool.yaml
Oct 02 16:52:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: I1002 16:52:49.180259       1 bootstrap.go:142] manifests/bootst
rap-pod-v2.yaml
Oct 02 16:52:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: I1002 16:52:49.180828       1 bootstrap.go:142] manifests/machin
econfigserver/csr-bootstrap-role-binding.yaml
Oct 02 16:52:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: I1002 16:52:49.181374       1 bootstrap.go:142] manifests/machin
econfigserver/kube-apiserver-serving-ca-configmap.yaml
```

	- Connecting to control plane ETCD cluster 
```
Oct 02 16:52:49 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Starting etcd certificate signer...
Oct 02 16:52:52 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: ce8ff0beb9c84d21bdec5001d8a971ec019b04e9254187e405e67e581462610c
Oct 02 16:52:52 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Waiting for etcd cluster...
Oct 02 16:57:39 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: https://etcd-0.vbd-tf-ocp41.csplab.local:2379 is healthy: successfully committed proposal: took = 2.308351ms
Oct 02 16:57:39 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: https://etcd-1.vbd-tf-ocp41.csplab.local:2379 is healthy: successfully committed proposal: took = 2.308351ms
Oct 02 16:57:39 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: https://etcd-2.vbd-tf-ocp41.csplab.local:2379 is healthy: successfully committed proposal: took = 2.308351ms
Oct 02 16:57:39 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: etcd cluster up. Killing etcd certificate signer...
Oct 02 16:57:40 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: ce8ff0beb9c84d21bdec5001d8a971ec019b04e9254187e405e67e581462610c
Oct 02 16:57:40 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Starting cluster-bootstrap...
Oct 02 16:57:43 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]: Starting temporary bootstrap control plane...
```

	- Waiting for critical pods on the control plane (DoesNotExist - Pending - RunningNotReady - Ready):
```
Oct 02 16:58:28 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]:         Pod Status:openshift-kube-apiserver/kube-apiserver
  DoesNotExist
Oct 02 16:58:28 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]:         Pod Status:openshift-kube-scheduler/openshift-kube-sched
uler        DoesNotExist
Oct 02 16:58:28 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]:         Pod Status:openshift-kube-controller-manager/kube-contro
ller-manager        DoesNotExist
Oct 02 16:58:28 vbd-tf-ocp41-bootstrap.vbd-tf-ocp41.csplab.local bootkube.sh[1047]:         Pod Status:openshift-cluster-version/cluster-version-ope
rator        Pending
```

 
## Check control plane host (masters)

Login to the master node:

                ssh -i <ssh-private-key> core@<master-ip>

The container in OpenShift 4 are running using CRI-O container manager, not docker, you can do the following:

- Check `kubelet` processing: `journalctl -f -u kubelet`

- Check running processes: `ps -ef`

- Check running containers: `sudo crictl ps`

- Getting logs from a running container: `sudo crictl logs <containerId>

- Run a command from within a pod `sudo crictl exec <containerId> command`

- Check running pods: `sudo crictl pods`


