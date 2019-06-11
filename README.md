# object-storage
Using IBM Cloud Object Storage as a persistent storage for your apps that run on IKS cluster
For more details on how to get started, use [this link](https://cloud.ibm.com/docs/containers?topic=containers-object_storage#add_cos)

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
      This repo contains a sample yaml for pvc creation that you can use [pvc_create_bucket.yaml]
      
      For more details on yaml file components, refer [here](https://cloud.ibm.com/docs/containers?topic=containers-object_storage#add_cos)
    
    
5. Once you create a PVC, a PV will be created for you. Get the name of the PV as you should refer to the PV in your deployment.yaml file
    ```
    $ kubectl get pv -n <namespace>
    ```
6.  Create your deployment to mount your PV and specify the PVC created from [step 4]
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

7. Verification 
    
    This repo contains a sample [deployment.yaml] file that can be used to verify if you can successfully write data to the       COS bucket you used to mount
    1. Create the deployment
      ```$ kubectl create -f deployment.yaml```
    2. Verify the pods created by the deployment
    3. Shell into the pod using
       ```$ kubectl exec -it <pod-name> -it bash```
    4. Navigate to the mount path specified in your deployment file (`mountPath: /<file_path>`)
    5. Create a file
       ```$ echo "Hello World" > hello.txt"```
    6. Verify that your COS bucket has the file `hello.txt` in it
    
#### Troubleshooting ####
  ##### References #####
  https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#cos_pvc_pending
  https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#cos_nonroot_access
  https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#missing_permissions
  
