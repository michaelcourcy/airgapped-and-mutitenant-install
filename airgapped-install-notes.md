# Let's create a NFS service in the NFS machine

Use the nfs terminal to install the nfs-kernel-server
```
apt-get update
apt install -y nfs-kernel-server
```

Create and expose the /nfs directory
```
sudo mkdir /nfs
sudo chown nobody:nogroup /nfs
echo "/nfs p-${INSTRUQT_PARTICIPANT_ID}-k8svm.c.instruqt-prod.internal(rw,sync,no_subtree_check)" >> /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

# Use the k8s terminal now

Test the NFS client

Install the nfs client.
```
sudo apt-get update
apt-get install -y nfs-common
```

Mount the share
```
mkdir /mnt/nfs
mount p-${INSTRUQT_PARTICIPANT_ID}-nfs.c.instruqt-prod.internal:/nfs /mnt/nfs
```

# Check nfs work properly

on k8svm
Check nfs work properly
```
touch /mnt/nfs/test
```

On nfs
```
ls /nfs
```

You should see
```
test
```

# Install the internal registry on nfs

On nfs let's create an internal registry with a basic credential testuser/testpassword

```
mkdir auth
docker run \
   --entrypoint htpasswd \
   httpd:2 -Bbn testuser testpassword > auth/htpasswd

docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v "$(pwd)"/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

# Configure docker on k8s to accept this insecure registry

on k8s add this registry in the allowed insecure registry and reload docker
```

cat <<EOF > /etc/docker/daemon.json
{
  "insecure-registries": ["nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000"]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

# Test that you can pull and push an image to this registry from k8s

on k8s
```
docker pull alpine
docker tag alpine nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000/alpine
```

check your images
```
docker images
```

Now connect to the regystry
```
docker login nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000 -u testuser -p testpassword
```

If login is successful you can push
```
docker push nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000/alpine
```

# Recreate the k8s cluster with the registry

On the k8s machine

```
cat <<EOF | kind create cluster --name k10-demo --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.21.1
  extraPortMappings:
  - containerPort: 32000
    hostPort: 32000
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32010
    hostPort: 32010
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32020
    hostPort: 32020
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32030
    hostPort: 32030
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32040
    hostPort: 32040
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32050
    hostPort: 32050
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32060
    hostPort: 32060
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32070
    hostPort: 32070
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32080
    hostPort: 32080
    listenAddress: "0.0.0.0"
    protocol: TCP
  - containerPort: 32090
    hostPort: 32090
    listenAddress: "0.0.0.0"
    protocol: TCP
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000"]
    endpoint = ["http://nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000"]
EOF
```

# Test NFS and external registry all together

Let's create a pod that use a NFS PVC and an image that come from the internal registry

```
kubectl create ns kasten-io
```

Create the nfs PV and the corresponding PVC.
```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-kasten
spec:
  claimRef:
    name: pvc-nfs-kasten
    namespace: kasten-io
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /nfs
    server: p-${INSTRUQT_PARTICIPANT_ID}-nfs.c.instruqt-prod.internal
    readOnly: false
  mountOptions:
      - hard
      - nfsvers=4.1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-kasten
  namespace: kasten-io
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
EOF
```


Create a pull secret
```
kubectl create -n kasten-io secret docker-registry nfs-${INSTRUQT_PARTICIPANT_ID}-registry-secret \
   --docker-username=testuser \
   --docker-password=testpassword \
   --docker-email=unused \
   --docker-server=http://nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000/v2/
```

Now deploy a pod based on an internal registry and mounting this nfs PVC
```
cat <<EOF | kubectl create -n kasten-io -f -
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: airgapped-alpine
  name: airgapped-alpine
spec:
  imagePullSecrets:
  - name: nfs-${INSTRUQT_PARTICIPANT_ID}-registry-secret
  containers:
  - args:
    - tail
    - -f
    - /dev/null
    volumeMounts:
    - name: data
      mountPath: /data
    image: nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000/alpine
    name: airgapped-alpine
    resources: {}
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-nfs-kasten
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF
```

Test the creation of a file from a pod
```
kubectl exec  airgapped-alpine -n kasten-io -- touch /data/test-from-a-pod
```

On nfs machine check the file has been created

```
ls /nfs
```

we don't need anymore this pod but want to keep the nfs PVC storage so let's delete it.

On k8s machine
```
kubectl delete po -n kasten-io airgapped-alpine
```

# Make sure all kasten images are pushed to the internal registries

This example is based on the 4.5.8 version of kasten, change the version number for the last recent version number.

In order to push all the kasten images in the internal images, Kasten feature the k10offline image that can do that for you.

This image is mounting you docker sock for retreiving the credentials to the internal registry (`-v ${HOME}/.docker:/root/.docker`).

```
docker run --rm -ti -v /var/run/docker.sock:/var/run/docker.sock \
    -v ${HOME}/.docker:/root/.docker \
    gcr.io/kasten-images/k10offline:4.5.8 pull images --newrepo nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000/kasten-images
```

When working airgapped we won't have the chart available through internet, we need to download it first.

```
helm repo add kasten https://charts.kasten.io/
helm repo update && \
    helm fetch kasten/k10 --version=4.5.8
```

You should see k10-4.5.8.tgz archive in you directory now.

# Install Kasten K10

At this point you don't need anymore a network connection to the internet.

Everything can work airgapped.

install a lab licence
```
kubectl create -f /root/license-secret.yaml
```

# Install Kasten K10

```
helm install k10 k10-4.5.8.tgz --namespace=kasten-io \
--set global.airgapped.repository=nfs.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:5000/kasten-images \
--set secrets.dockerConfig=$(base64 -w 0 < ${HOME}/.docker/config.json) \
--set global.imagePullSecret="k10-ecr"
```

To ensure that Kasten K10 is running, check the pod status to make sure they are all in the `Running` state:

```
watch -n 2 "kubectl -n kasten-io get pods"
```

Once all pods have a Running status, hit `CTRL + C` to exit `watch`.

# Check the images are all belonging to the internal registry

```
kubectl get po -n kasten-io -ojsonpath='{range .items[*].spec.containers[*]}{.image}{"\n"}{end}'|sort|uniq
```

You may notice has the layout is flat we add the k10 prefix on tag for non kasten images, this is to avoid images collisions.

# Expose the K10 dashboard

While not recommended for production environments, let's set up access to the K10 dashboard by creating a NodePort. Let's first create the configuration file for this:

```console
cat > k10-nodeport-svc.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: gateway-nodeport
  namespace: kasten-io
spec:
  selector:
    service: gateway
  ports:
  - name: http
    port: 8000
    nodePort: 32000
  type: NodePort
EOF
```

Now, let's create the actual NodePort Service

```console
kubectl apply -f k10-nodeport-svc.yaml
```
# View the K10 Dashboard

Once completed, you should be able to view the K10 dashboard in the other tab on the left.

# Use the NFS PVC as a location profile

Once K10 is running and accessible, go on kasten dashboard and create a nfs location profile using the pvc-nfs-kasten pvc.

Test by creating a policy with the export location profile you just created on the default namespace.

After successful policy execution, check the content of the `/nfs` directory in the nfs server. You should find a usual k10 layout.
