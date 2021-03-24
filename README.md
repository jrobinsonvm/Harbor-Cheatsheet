# Setting up Harbor for Tanzu Kubernetes Clusters 
This guide will walk through how to setup Harbor with an auto generated self signed cert that can be used with Tanzu Kubernetes Clusters 

### 1. Setup an ingress controller 
  This guide will use Projectcontour's ingress controller 

1. Ensure your Role Binding has the appropriate Pod Security Policy to create resources on the tanzu cluster. 
   If not, please issue the following command.   
   ```
   kubectl create clusterrolebinding psp:authenticated --clusterrole=psp:vmware-system-privileged --group=system:authenticated
   ```

2. Download Projectcontour's YAML from https://projectcontour.io/getting-started/
   ```
   wget https://projectcontour.io/quickstart/contour.yaml
   ```

3. Edit the YAML and comment out the lines for the aws-load-balancer-backend-protocol and externalTrafficPolicy. 
   
    ```
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    externalTrafficPolicy: Local
    ```

4. Now run kubectl apply against the YAML 
    ```
    kubectl apply -f contour.yaml
    ```

5. Verify all  resources were provisioned
    ```
    kubectl get all -n projectcontour 
    ```

### 2. Setup DNS to point to Ingress Controller IP Address 

1. Create an A record which points to the IP Address of your Ingress.  
   When using Projectcontour this will be the IP address of a service called envoy.   
   To get the IP Address of the envoy service run the following command

   ```
   kubectl get svc -n projectcontour
   ```
   
   ```
   Type	    Name	Value	      Example Domain: mytanzudemo.com
    A	    @	    35.185.32.27 
   ```

   
2. Once you have created an A record; now create a CNAME that you can use as a wildcard DNS record.   
   ```
   Type	    Name	    Value	
    CNAME     *.sys                 @ 
   ```

### 3. Setup Harbor 

1. Download Harbor Helm Chart via https://goharbor.io/docs/2.2.0/install-config/harbor-ha-helm/
   
   Run the following commands to add the GoHarbor Repo and to Download the Harbor helm chart 
   ```
    helm repo add harbor https://helm.goharbor.io
    helm fetch harbor/harbor --untar
   ```

2. Change directories to Harbor and Edit the values file to include the following changes 

   * Line 25 asks for a common name; use the FQDN you wish to use for your registry.   
     (i.e registry.sys.mytanzudemo.com )

   * Replace all occurances of "core.harbor.domain" with the FQDN you wish to use for your registry.  
   (ie. core.harbor.domain -> replace with -> registry.sys.mytanzudemo.com )

   * Replace all occurances of "notary.harbor.domain" with the FQDN you wish to use for your notary. 
   (ie. notary.harbor.domain -> replace with -> notary.sys.mytanzudemo.com )

   * Increase the PVC Size for the Registry and Chartmuseum pods 
     Changes should be made at line 200 and 206.  
     (ie. You can increase both limits to something like 100GB or your preferred limit based on your historical capacity trends.)

   * Replace the annotations section with the following if using projectcontour 

   ```
       annotations:
      # note different ingress controllers may require a different ssl-redirect annotation
      # for Envoy, use ingress.kubernetes.io/force-ssl-redirect: "true" and remove the nginx lines below
      ingress.kubernetes.io/force-ssl-redirect: "true"     # force https, even if http is requested
      kubernetes.io/ingress.class: contour                 # using Contour for ingress
      kubernetes.io/tls-acme: "true"                       # using ACME certificates for TLS
   ```

3. Create a namespace for Harbor 
   ```
   kubectl create ns harbor 
   ```
   
4. Install Harbor 
   ```
   helm install harbor . -n harbor 
   ```

5. Ensure that you are able to login to the Harbor UI 
   Default Harbor Credentials can be found in the Values YAML file. 


### 4. Import Harbor CA Locally to avoid any x509 Errors when accessing Harbor via docker login command.   
1. Download CA from the Harbor UI 
   
   Direct link to Harbor CA provided below.   
   ```
   curl -k https://registry.sys.mytanzudemo.com/api/v2.0/systeminfo/getcert
   ```

   Import Harbor CA to be picked up by your local docker daemon (Needed for local development)
   If on linux or MacOS use the following directions: 

2. Navigate to /etc/docker/certs.d
    If path does not exist create it. 
    ```
    mkdir -p /etc/docker/certs.d
    ```

3. CD to /etc/docker/certs.d and create a directory for your registry.  
    Name the new directory the same name as your registry.  
    Example: 
    ```
    mkdir registry.sys.mytanzudemo.com
    ```

3. Now CD to the new directory and create a file called ca.crt.
    Paste the contents of your Harbor CA here. 

    This could also be done using a single command: 
    Example:  
    ```
    curl -k https://registry.sys.mytanzudemo.com/api/v2.0/systeminfo/getcert >> ca.crt
    ```

4. Restart Docker Daemon 

5. Now login to your Harbor Registry 
   ```
   docker login registry.sys.mytanzudemo.com
   ```

### 5. Import Harbor CA remotely to be used by your Tanzu Kubernetes Clusters
   Please follow steps documented in the VMware vSphere 7 with Tanzu Docs for using external registries 
   https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-376FCCD1-7743-4202-ACCA-56F214B6892F.html

   For reference purposes -  The following steps were taking from the VMware vSphere 7 with Tanzu Docs. 
   1. Login to your Supervisor cluster 

   2. Create a file called tkgserviceconfig.yaml and paste the following contents 

   ```
    apiVersion: run.tanzu.vmware.com/v1alpha1
    kind: TkgServiceConfiguration
    metadata:
    name: tkg-service-configuration
    spec:
    defaultCNI: antrea
    trust:
        additionalTrustedCAs:
        - name: first-cert-name
            data: base64-encoded string of a PEM encoded public cert 1
        - name: second-cert-name
            data: base64-encoded string of a PEM encoded public cert 2
   ```

   3. Copy the contents of your Harbor CA Cert and Base64 Encode it.   
      If you have openssl installed you can do this via command line 
      
   ```
   openssl base64 -in ca.crt -out encodedca.txt
   ``` 
   If using the above command, your encoded string will be in encodedca.txt 

   4. Edit the tkgserviceconfig.yaml file to include the base64 encodded string of your Harbor CA Cert.   
      Under Trust -> AdditionalTrustedCAs -> Name -> you will find a section for Data.   
      Replace the contents of the Data Section with your base64 encoded string and save the file.  

   5. Now run kubectl apply against the tkgserviceconfig.yaml file

   ```
   kubectl apply -f tkgserviceconfig.yaml 
   ```

   6. Finally, create a new Tanzu Kubernetes cluster and start deploying containers using your new Harbor Registry.  
      Don't forget to create an image pull secret on your tanzu kubernetes cluster so that your worker nodes can authenticate to your Harbor instance.  
   
