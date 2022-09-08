# Installation of IBM CP4I 2022.2.1 on AWS ROSA with EBS storage

## Introduction notes

We assume here that the following steps are already completed:
1. Catalog sources for IBM operators are added to the OpenShift cluster - please see [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-adding-catalog-sources-your-openshift-cluster](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-adding-catalog-sources-your-openshift-cluster)  
2. Operators for the IBM Cloud Pak for Integration and for the required capabilities installed- please see [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-operators](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-operators)

We also assume that there is an OpenShift project (namespace) with the name **cp4i** (you can create it with `oc new-project cp4i`) and that the IBM Entitlement Key secret is already created in that namespace. For the instructions on how to create that secret see the following chapter from the documentation: [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-applying-your-entitlement-key-online-installation](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-applying-your-entitlement-key-online-installation)

The following storage classes exist in ROSA out-of-the-box:
```
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
gp2             kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true                
gp2-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                
gp3 (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                
gp3-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                
```

All of those classes are of the block, RWO type. For the instance of CP4I Platform UI (Platform Navigator), we need an RWX class. So to proceed with the existing classes as they are, we followed the alternative installation method described here: [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=ui-deploying-platform-rwo-storage](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=ui-deploying-platform-rwo-storage)

This approach means that we create a kind of RWX "facade" or "wrapper" around the existing RWO class. In our case, we decided to use the existing **gp3** class.

## Deploying Rook NFS

1. Clone branch v1.7.3 of the rook nfs git repository:
    ```
    git clone --single-branch --branch v1.7.3 https://github.com/rook/nfs.git
    ```

2. Navigate to this directory:
    ```
    cd nfs/cluster/examples/kubernetes/nfs
    ```

3. Edit file **operator.yaml** Replace the original image **rook/nfs:v1.7.3** with IBM provided one: **icr.io/cpopen/cpd/rook-nfs:kz-220512**

    An example of updated [operator.yaml](artefacts/operator.yaml) file is available here in this repository in the [artefacts](artefacts) directory.

4. Still in the directory **nfs/cluster/examples/kubernetes/nfs** , apply **crds.yaml** to create custom resource definitions:  
    ```
    oc apply -f crds.yaml
    ```

5. Create operator deployment:
    ```
    oc apply -f operator.yaml
    ```

6. Verify that operator is running:
    ```
    oc get pod -n rook-nfs-system
    ```

    The result should be similar to the following:
    ```
    NAME                                 READY   STATUS    RESTARTS   AGE
    rook-nfs-operator-84fff9f699-45v2c   1/1     Running   0          43s
    ```

7. Grant the Rook NFS service account access to the privileged SecurityContextConstraints (SCC) resources:
    ```
    oc adm policy add-scc-to-user privileged system:serviceaccount:rook-nfs:rook-nfs-server
    ```

## Deploying the Rook NFS server

Apply the following YAMLs by storing them to files and running the CLI command: `oc apply -f <file_name>` or by copy/pasting them to the *Import YAML* window in OpenShift web console:

<img width="850" src="images/Snip20220908_76.png">  

1. Create RBAC objects:
    ```yaml
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name:  rook-nfs
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: rook-nfs-server
      namespace: rook-nfs
    ---
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: rook-nfs-provisioner-runner
    rules:
      - apiGroups: [""]
        resources: ["persistentvolumes"]
        verbs: ["get", "list", "watch", "create", "delete"]
      - apiGroups: [""]
        resources: ["persistentvolumeclaims"]
        verbs: ["get", "list", "watch", "update"]
      - apiGroups: ["storage.k8s.io"]
        resources: ["storageclasses"]
        verbs: ["get", "list", "watch"]
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["create", "update", "patch"]
      - apiGroups: [""]
        resources: ["services", "endpoints"]
        verbs: ["get"]
      - apiGroups: ["policy"]
        resources: ["podsecuritypolicies"]
        resourceNames: ["rook-nfs-policy"]
        verbs: ["use"]
      - apiGroups: [""]
        resources: ["endpoints"]
        verbs: ["get", "list", "watch", "create", "update", "patch"]
      - apiGroups:
        - nfs.rook.io
        resources:
        - "*"
        verbs:
        - "*"
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: rook-nfs-provisioner-runner
    subjects:
      - kind: ServiceAccount
        name: rook-nfs-server
        namespace: rook-nfs
    roleRef:
      kind: ClusterRole
      name: rook-nfs-provisioner-runner
      apiGroup: rbac.authorization.k8s.io    
    ```

2. Create PersistenVolumeClaim for NFS. Please note that we selected here **gp3** as an existing RWO class that we are going to use. We also left the recomended size of 200 GB. This size actually depends on the planned number of instances of capabilities that require RWX storage. If we need it just for Platform UI then we can decide for the smaller size. By the documentation the required storage for Platform UI is 40 GB. <br>
**Note:** Please do not be surprised if this PVC remains in the **pending** state. The volume binding mode for those classes is *WaitForFirstConsumer*. So, the volume will be created when it is really needed (in the next step when we deploy the server).
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nfs-pwx-claim
      namespace: rook-nfs
    spec:
      storageClassName: gp3  # we decided to use gp3, change if necessary 
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 200Gi    
    ```

3. Deploy the NFS server (note that the PVC created previously is now bound):
    ```yaml
    apiVersion: nfs.rook.io/v1alpha1
    kind: NFSServer
    metadata:
      name: rook-nfs
      namespace: rook-nfs
    spec:
      replicas: 1
      exports:
      - name: share1
        server:
          accessMode: ReadWrite
          squash: "none"
        # A Persistent Volume Claim must be created before creating NFS CRD instance.
        persistentVolumeClaim:
          claimName: nfs-pwx-claim
      # A key/value list of annotations
      annotations:
        rook: nfs    
    ```

4. Verify that server pod is running:
    ```
    oc get pods -n rook-nfs
    ```
    The result should be similar to the following:
    ```
    NAME         READY   STATUS    RESTARTS   AGE
    rook-nfs-0   2/2     Running   0          20m    
    ```


## Creating the storage class

Apply this following YAML:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  labels:
    app: rook-nfs
  name: integration-storage
parameters:
  exportName: share1
  nfsServerName: rook-nfs
  nfsServerNamespace: rook-nfs
provisioner: nfs.rook.io/rook-nfs-provisioner
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

If you check the classes now, you will see on the list our new class named **integration-storage**:
```
oc get sc

NAME                  PROVISIONER                        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2                   kubernetes.io/aws-ebs              Delete          WaitForFirstConsumer   true                   7d14h
gp2-csi               ebs.csi.aws.com                    Delete          WaitForFirstConsumer   true                   7d14h
gp3 (default)         ebs.csi.aws.com                    Delete          WaitForFirstConsumer   true                   7d14h
gp3-csi               ebs.csi.aws.com                    Delete          WaitForFirstConsumer   true                   7d14h
integration-storage   nfs.rook.io/rook-nfs-provisioner   Delete          Immediate              false                  15s
```

## Creating a ConfigMap

The following ConfigMap has to be created in the namespace where we are going to create an instance of the Platform UI (PlatformNavigator). In our case it is **cp4i**. Besides the namespace, you must specify the newly created storage class -  in our case **integration-storage** and the original RWO class - in our case **gp3**. The locations of those properties are marked with comments in the bellow YAML:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: ibm-zen-config
  namespace: cp4i                             # <-- Platform Navigator namespace
data:
  zen: |-
    storage_data:
      storageClass: integration-storage       # <-- Our new storage class
      zenCoreMetaDbStorageClass: gp3          # <-- The original RWO class
    scale_data:
      Usermgmt:
        name: usermgmt
        kind: Deployment
        container: usermgmt-container
        replicas: 1
        resources:
          limits:
            cpu: 400m
            memory: 512Mi
          requests:
            cpu: 200m
            memory: 256Mi
      ZenCoreMetaDb:
        name: zen-metastoredb
        kind: StatefulSet
        container: zen-metastoredb
        replicas: 3
        resources:
          limits:
            cpu: 500m
            memory: 2048Mi
          requests:
            cpu: 100m
            memory: 512Mi
      Nginx:
        name: ibm-nginx
        kind: Deployment
        container: ibm-nginx-container
        replicas: 1
        resources:
          limits:
            cpu: 400m
            memory: 512Mi
          requests:
            cpu: 200m
            memory: 256Mi
      ZenCore:
        name: zen-core
        kind: Deployment
        container: zen-core-container
        replicas: 1
        resources:
          limits:
            cpu: 400m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 256Mi
      ZenCoreApi:
        name: zen-core-api
        kind: Deployment
        container: zen-core-api-container
        replicas: 1
        resources:
          limits:
            cpu: 400m
            memory: 1024Mi
          requests:
            cpu: 100m
            memory: 256Mi
      ZenWatcher:
        name: zen-watcher
        kind: Deployment
        container: zen-watcher-container
        replicas: 1
        resources:
          limits:
            cpu: 1000m
            memory: 1024Mi
          requests:
            cpu: 100m
            memory: 256Mi
      ZenAudit:
        name: zen-audit
        kind: Deployment
        container: zen-audit-container
        replicas: 1
        resources:
          limits:
            cpu: 200m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 256Mi
      ZenWatchdog:
        name: zen-watchdog
        kind: Deployment
        container: zen-watchdog-container
        replicas: 1
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
      ZenDataSorcerer:
        name: zen-data-sorcerer
        kind: Deployment
        container: zen-data-sorcerer-container
        replicas: 1
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
          requests:
            cpu: 30m
            memory: 128Mi
      DsxInfluxdb:
        name: dsx-influxdb
        kind: StatefulSet
        container: dsx-influxdb
        replicas: 1
        resources:
          limits:
            cpu: 400m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 256Mi

```

## Install the Platform UI

1. In the OpenShift web console navigate to **Operators > Installed Operators**, select the project - in our case **cp4i** and select the **IBM Cloud Pak for Integration** operator:
    <img width="850" src="images/Snip20220908_77.png">  

2. Select **Platform UI** tab and click on the button **Create PlatformNavigator**:
    <img width="850" src="images/Snip20220908_78.png">

3. Define some name and accept the **license**. Optionaly change the number of replicas:
    <img width="850" src="images/Snip20220908_79.png">

4. Scroll down. For the **storage class** select the new class that we just created - in our case it is **integration-storage**. **Do not click Create yet!**
    <img width="850" src="images/Snip20220908_80.png">







