# object-storage
Using IBM Cloud Object Storage as a persistent storage for your apps that run on IKS cluster

NOTE: This document summarizes the info from official IBM Cloud documentation. If you need more info, checkout [Storing data on IBM Cloud Object Storage](https://cloud.ibm.com/docs/containers?topic=containers-object_storage#add_cos)

1. You should have an object storage service instance and service credentials. Details [here](https://cloud.ibm.com/docs/containers?topic=containers-object_storage#create_cos_service)

2. Create a kubernetes secret using the service credentials foryour COS instance to be accessible from IKS cluster
    1. If using API key,
        ```
        $ kubectl create secret generic cos-write-access --type=ibm/ibmc-s3fs --from-literal=api-key=<api_key> --from-literal=service-instance-id=<service_instance_guid> -n <namespace>
        ```
    2. If using HMAC credentials,
        ```
        $ kubectl create secret generic cos-write-access --type=ibm/ibmc-s3fs --from-literal=access-key=<access_key_ID> --from- literal=secret-key=<secret_access_key> -n <namespace>
        ```
3.  Install the IBM Cloud Object Storage plug-in. Follow steps detailed [here](https://cloud.ibm.com/docs/containers?topic=containers-object_storage#install_cos)

4.  Create a persistent volume claim (PVC) to provision IBM Cloud Object Storage for your cluster
    The pvc yaml will look like the below :
      ```
      kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: <pvc_name>
        namespace: <namespace>
        annotations:
          ibm.io/auto-create-bucket: "<true_or_false>"
          ibm.io/auto-delete-bucket: "<true_or_false>"
          ibm.io/bucket: "<bucket_name>"
          ibm.io/object-path: "<bucket_subdirectory>"
          ibm.io/secret-name: "<secret_name>"
          ibm.io/endpoint: "https://<s3fs_service_endpoint>"
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 8Gi # Enter a fictitious value
        storageClassName: <storage_class>
      ```
      This repo contains a sample yaml for pvc creation that you can use [pvc.yaml](pvc.yaml)
      - Create PVC
        ```$ kubectl create -f pvc.yaml```
      - Verify PVC is create and bound to the PV(should be created automatically)
        ```$ kubectl get pvc -n <namespace>```
        Sample output looks like below:
        ```
        NAME           STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS  AGE
        s3fs-test-pvc  Bound     pvc-b38b30f9-1234-11e8-ad2b-t910456jbe12   8Gi        RWO    ibmc-s3fs-standard-cross-region  1h
        ```
      - Get the volume name(`VOLUME`) that PVC is bound to, from the output of above command. This name will be used to replace `<volume_name>` in your `deployment.yaml` file.

      - Troubles with PVC. Refer [this](https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#cos_pvc_pending)

      - Note that the sample [pvc.yaml](pvc.yaml) has `ibm.io/auto-create-bucket: "true"`. So, the service credentials created to access your COS should atleast have a `Writer` role in order to create a bucket. If you want to point to a bucket that already exists, use  `ibm.io/auto-create-bucket: "false"` in your `pvc.yaml` file
      
      For more details on yaml file components, refer [here](https://cloud.ibm.com/docs/containers?topic=containers-object_storage#add_cos)


    
5.  Create a deployment to mount your PV and specify the name of PVC created from above
    
    A deployment yaml will look like this:
      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: <deployment_name>
        labels:
          app: <deployment_label>
      spec:
        selector:
          matchLabels:
            app: <app_name>
        template:
          metadata:
            labels:
              app: <app_name>
          spec:
            containers:
            - image: <image_name>
              name: <container_name>
              securityContext:
                runAsUser: <non_root_user>
                fsGroup: <non_root_user> #only applicable for clusters that run Kubernetes version 1.13 or later
              volumeMounts:
              - name: <volume_name>
                mountPath: /<file_path>
            volumes:
            - name: <volume_name>
              persistentVolumeClaim:
                claimName: <pvc_name>
      ```

#### Verification ####
    
  This repo contains a sample [deployment.yaml](deployment.yaml) file that can be used to verify if you can successfully write data to the COS bucket used for the mount
  - Create the deployment
    ```$ kubectl create -f deployment.yaml```
  - Verify the pods created by the deployment
    ```$ kubectl get pods -n <namespace>```
  - Shell into the pod created by the deployment using
    ```$ kubectl exec -it <pod-name> -it bash```
  - Navigate to the mount path specified in your deployment file (`mountPath: /<file_path>`)
  - Create a file
    ```$ echo "Hello World" > hello.txt```
  - Verify that your COS bucket has the file `hello.txt` in it
    
  ##### Important to note #####
  To run the deployment as non-root user, note that the deployment uses 
    ```
    securityContext:
      runAsUser: <non_root_user_id>
      fsGroup: <non_root_user_id>
    ```
    Specifying the above will change the owner of the s3fs mount point to userID value provided in `runAsUser` and assign the ownership of COS Volume to that user group. Currently, you can modify the id, but `runAsUser` and `fsGroup` should have same id. If these two values do not match, the mount point is automatically owned by the root user.

#### Troubleshooting ####
  If you face any issues during the process, use the following references to troubleshoot them
  ##### References #####
  - [PVC remains in pending state](https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#cos_pvc_pending)
  - [COS non-root user access](https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#cos_nonroot_access)
  - [PVC creation fails because of missing permissions](https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#missing_permissions)
  
