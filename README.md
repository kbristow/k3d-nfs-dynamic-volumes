# NFS Dynamic Volumes with K3D
This shows how to setup dynamic provisioning of nfs volumes with k3d. The nfs PV's are provisioned using the [nfs-ganesha-serverand-external-provisioner](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner) which is backed
by an initial PV created with [local-path-provisioner](https://github.com/rancher/local-path-provisioner). 
The instructions below show how to configure this setup using the default settings from the nfs-ganesha-server. The README on the nfs-ganesha-server has more details if you wish to modify the config of the nfs server or provisioner.

## Instructions
- Create a k3d cluster
```console
docker volume create kube-nfs-volume
k3d cluster create nfs --agents 2 --volume kube-nfs-volume:/opt/local-path-provisioner --wait
k3d kubeconfig merge nfs -d
```
- Install the local-path storage provisioner (will create a pvc to host the nfs share with this)
```console
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```
- Install the nfs-ganesha provisioner
```
# create the rbac rules needed
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner/master/deploy/kubernetes/rbac.yaml
# create the base pvc
kubectl create -f pvc.yaml
# create the deployment
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner/master/deploy/kubernetes/deployment.yaml
# patch the deployment to use the pvc instead of the default hostPath volume
k patch deploy nfs-provisioner --type=json -p='[{"op": "replace", "path": "/spec/template/spec/volumes", "value": [{ "name": "export-volume", "persistentVolumeClaim": {"claimName": "ganesha-pvc"}}]}]'
# create the storage class
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner/master/deploy/kubernetes/class.yaml
```

Now the `nfs-provisioner` deployment failed to start due to not being able to pull the image from `quay.io`. I tried pulling it locally to no avail. 
The image can be build locally but i used a mirror of it that was uploaded to Dockerhub [here](https://hub.docker.com/r/kvaps/nfs-ganesha-server-and-provisioner/tags?page=1&ordering=last_updated) and pushed it into the cluster directly.
```console
docker pull kvaps/nfs-ganesha-server-and-provisioner:latest
docker tag kvaps/nfs-ganesha-server-and-provisioner:latest quay.io/external_storage/nfs-ganesha-server-and-provisioner:latest
k3d image import quay.io/external_storage/nfs-ganesha-server-and-provisioner:latest -c nfs
```

The `nfs-provisioner` should now start. You can test the pvc creation and usage with:
```console
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner/master/deploy/kubernetes/claim.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner/master/deploy/kubernetes/write-pod.yaml
```
