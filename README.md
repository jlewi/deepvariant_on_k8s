# deepvariant_on_k8s
Repository for running [Deepvariant](https://github.com/google/deepvariant) on K8s
  * This is a work in progress.
  * Initial focus on training.


## Cluster setup.

* We setup NFS in our K8s cluster because we need a shared posix compliant filesystem.


## Setup [NFS Provisioner](https://github.com/kubernetes-incubator/nfs-provisioner)

	* We setup the NFS provisioner because we want to use NFS to allow volumes 
	  to be shared across multiple pods.
    * We can't use GCS because parts of Deepvariant stack (e.g. make_examples don't work with GCS).

Create a PD to back the storage

```
PROJECT=<Your project>
PD_NAME=<Disk name>
ZONE=<zone>
gcloud --project=${PROJECT} disks create --zone=${ZONE} ${PD_NAME} --description="PD to back NFS storage on GKE." --size=1TB
```
	
	* TODO(jlewi): Should we use an SSD for better performance?
	* TODO(jlewi): Look at using Rook to run in cluster storage
	
Modify nfs/deployment.yaml to use the newly created PD as the backing store.

Deploy it

```
kubectl apply -f nfs/
```

## Copying Data to NFS

Checkout [Kubeflow](https://github.com/google/kubeflow)
 
Modify the config for JuptyerHub to specify the NFS volume you created e.g. something like

```
    c.KubeSpawner.volumes = [
        {
            'name': 'volume-{username}{servername}',
            'persistentVolumeClaim': {
                'claimName': 'claim-{username}{servername}'
            }
        },
        {
            'name': 'deepvariant-nfs',
            'persistentVolumeClaim': {
                'claimName': 'nfs'
            }
        }
    ]
    c.KubeSpawner.volume_mounts = [
        {
            # This should point to homedir of the user in the image
            'mountPath': '/home/jovyan/pd',
            'name': 'volume-{username}{servername}'
        },
        {
            'mountPath': '/home/jovyan/deepvariant-pd',
            'name': 'deepvariant-nfs'
        }
    ]
```

To copy data to NFS start Juptyer on the cluster using [Kubeflow](https://github.com/google/kubeflow)

  * Connect to Jupyter

  ```
  kubectl port-forward tf-hub-0 8100:8000
  ```

  * You can now access it localhost:8100

* At this point we cann use kubectl exec to start a shell in the datalab container.

* To start a shell

```
kubectl exec -ti ${DATALAB_POD} /bin/bash
```

* Checkout the git repo onto the NFS filesystem

```
mkdir -p /datalab/biotensorflow/home/jlewi
```


```
kubectl port-forward ${DATALAB_POD} 8888:8080
```

# Friction Log

## NFS, Kubeflow, and Jupyterhub

* We need a shared posix file system because some of the Deepvariant code only works with POSIX filesystems.
* We want to be able to mount the data across multiple pods simulataneously with RW access.
* We'd like to be able to mount the NFS volume in our Jupyter notebook 
   * Easiest way to copy data from GCS to NFS is simply via a terminal or notebook (shelling out).
   * To get Juptyer to mount my NFS volume I had to manually modify my Juptyerhub config to specify the volume.
   * Opened https://github.com/jupyterhub/jupyterhub/issues/1580 to allow specifying volumes in the spawner UI
