### Getting Started

```cmd
minikube start -p arc --cpus=4 --memory=8192
```

## Helm Chart Installations

### Controller

```cmd
helm install arc \
--namespace arc-systems \
--create-namespace \
-f ./controller/values.yaml \
oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

to confirm installation

```sh
helm list -A
```

or

```sh
kubectl get pods -n arc-systems
```

## Auth

First create the secret

```cmd
kubectl create secret generic arc-runner-app \
    -n arc-runners \
    --from-literal=github_app_id=1116585 \
    --from-literal=github_app_installation_id=59776397 \
    --from-file=github_app_private_key=./stackdev-arc-controller.2025-01-19.private-key.pem
```

## Runner Scale Set

```sh
helm install arc-runner-set \
--namespace arc-runners \
--create-namespace \
-f ./runner-scale-set-1/values.yaml \
oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

TO upgrade

```sh
helm upgrade arc-runner-set \
--namespace arc-runners \
--create-namespace \
-f ./runner-scale-set-1/values.yaml \
oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

## Injecting PVC

Creating the storage class

```sh
kubectl apply -f ./custom-storage/storage-class.yaml
```

Create the pvc

```sh
❯ kubectl apply -f ./custom-storage/pvc.yaml
```

To confirm the pvc in the arc-runners

```sh
kubectl get pvc -n arc-runners
```

- - If you want dynamic provisioning, you need to use a different provisioner that supports auto-creation of PVs, like kubernetes.io/aws-ebs for AWS, kubernetes.io/gce-pd for Google Cloud, or nfs-client for NFS.
- - If static provisioning is intended, you need to manually create a PV that matches your PVC's requirements.

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gh-runner-storage-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gh-runner-storage-class
  local:
    path: /mnt/docker-cache # Or any local path on your node
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - arc # Replace with your node's name or remove if not needed
```

Finally lets create the mount path in the node

shh into the node

```cmd
ssh minikube -p arc
```

Create the folder and grant some perms

```cmd
sudo mkdir -p /mnt/docker-cache
sudo chmod 777 /mnt/docker-cache
```

### Binding the pvc in the values.yaml of the scaleset

Add this:

```yml
...
...
volumeMounts:
          - name: docker-build-cache
            mountPath: /mnt/docker-cache
    volumes:
      - name: docker-build-cache
        persistentVolumeClaim:
          claimName: gh-runner-storage-pvc
```

Then Update the scaleset helm

```cmd
helm upgrade arc-runner-set \
--namespace arc-runners \
--create-namespace \
-f ./runner-scale-set-1/values.yaml \
oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set

```

### Stopping the cluster

```cmd
minikube stop -p arc
```

## Check the build cache in the node

```cmd
minikube ssh -p arc
```

Navigate to the cache folder

```cmd
cd /mnt/docker-cache/
ls -lh
```

-lh give the laout, size and headings

Then check the blobs sizee

## Cache Clean Up

This cron job will delete cache files older than 7 days from the /mnt/docker-cache directory. Adjust the schedule and retention period as needed.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cache-cleanup
  namespace: arc-runners
spec:
  schedule: "* * * * *" # Run daily at midnight >> 0 0 * * *
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cache-cleanup
              image: busybox
              command: [
                  "sh",
                  "-c",
                  "echo 'Starting cache cleanup'; \
                  find /mnt/docker-cache -type f -mmin +60 -print -delete; \
                  echo 'Cache cleanup completed'",
                ] #Delete files older than 7 days >>  -mtime +7   older than 60m >> -mmin +60
              volumeMounts:
                - name: docker-build-cache
                  mountPath: /mnt/docker-cache
          restartPolicy: OnFailure
          volumes:
            - name: docker-build-cache
              persistentVolumeClaim:
                claimName: gh-runner-storage-pvc
```

# List the cron jobs

kubectl get cronjob -n arc-runners

# List the jobs created by the cron job

kubectl get jobs -n arc-runners

# List the pods created by the job

kubectl get pods -n arc-runners --selector=job-name=cache-cleanup-<timestamp>

# Check the logs of the pod

kubectl logs -n arc-runners <pod-name>
