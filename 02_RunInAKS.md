# Lab 02: Running in AKS

---
## Objective
---

In this lab, you will push your container images from the last lab to Azure
Container Registry (ACR) and deploy them to your own Azure Kubernetes Service
(AKS) cluster.

---
## Push Images to Azure Container Registry (ACR)
---

### Provision resources

1. Log onto Azure:

    ```console
    az login
    ```

2. Ensure the current subscription context is correct (replace `MyAzureSub` with
the name of the Azure subscription you want to use):

    ```console
    az account set --subscription MyAzureSub
    ```

3. Re-use your existing `myResourceGroup` to create an ACR with a unique
`myContainerRegistry` name:

    ```console
    az acr create --resource-group myResourceGroup --name myContainerRegistry --sku Standard
    ```

### Log in to ACR

```console
az acr login --name myContainerRegistry
```

### Push images to ACR

1. Pushing images requires that they must first be tagged with the fully qualified name
of your ACR login server. Run the following to get the full login server name:

    ```console
    az acr list --resource-group myResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
    ```

2. Tag the images we'll be pushing, replacing `ACR_LOGIN_SERVER` with the value from
the last command:

    ```console
    # First tag and move the Message Consumer
    docker tag songrequest.messageconsumer:dev ACR_LOGIN_SERVER/songrequest.messageconsumer:dev
    docker push ACR_LOGIN_SERVER/songrequest.messageconsumer:dev

    # Second tag and move the Web Front End
    docker tag songrequest.webfrontend:dev ACR_LOGIN_SERVER/songrequest.webfrontend:dev
    docker push ACR_LOGIN_SERVER/songrequest.webfrontend:dev
    ```

3. Azure Container Registry considers container images repositories, so to list the
container images in ACR, execute the following:

    ```console
    # List all of the container images
    az acr repository list --name myContainerRegistry --output table

    # Show all of the tags available of a container image
    az acr repository show-tags --name myContainerRegistry --repository songrequest.messageconsumer --output table
    ```

---
## Setup an Azure Kubernetes Service (AKS) Cluster
---

1. Create the AKS cluster, replacing `myAKSCluster` with a unique name:

    ```console
    az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --node-vm-size Standard_A2_v2 --generate-ssh-keys
    ```

    > Note: This may take several minutes to complete. You'll receive a JSON-formatted
    > response with information about your cluster when it is finished provisioning.

2. Kubernetes clusters are managed with the Kubernetes command-line client, **kubectl**.
To install it locally, simply execute:

    ```console
    az aks install-cli
    ```

3. Just as docker can use our Azure credentials, we can configure kubectl to use our
credentials as well by executing:

    ```console
    az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

    # Verify connection
    kubectl get nodes
    ```

4. Grant the AKS cluster service principal access to your ACR registry so that it
can pull down images for deployment. Execute the following:

    ```console
    # Get the CLIENT_ID of the service principal configured for AKS
    az aks show --resource-group myResourceGroup --name myAKSCluster --query "servicePrincipalProfile.clientId" --output tsv

    # Get the ACR_ID registry resource ID
    az acr show --name myContainerRegistry --resource-group myResourceGroup --query "id" --output tsv

    # Create the role assignment
    az role assignment create --assignee CLIENT_ID --role Reader --scope ACR_ID
    ```

---
## Deploy Containers to AKS
---

### Create the YAML manifest

Kubernetes uses manifest files to define a desired state for the cluster. This
desired state includes which container images should be running, how many replicas,
on which ports, and how it all ties together. Create a directory at the root (even
with your current project directories) called `deployments`. Create a file called 
`songrequest.yaml` in the directory. Copy the contents below into the file, replace
`ACR_LOGIN_SERVER` with the correct value, and save.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-songrequest-webfrontend
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: songrequest
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: songrequest-messageconsumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: songrequest
      tier: backend
  template:
    metadata:
      name: songrequest-messageconsumer
      labels:
        app: songrequest
        tier: backend
    spec:
      containers:
        - name: messageconsumer
          image: ACR_LOGIN_SERVER/songrequest.messageconsumer:dev
          imagePullPolicy: Always
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: songrequest-webfrontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: songrequest
      tier: frontend
  template:
    metadata:
      name: songrequest-webfrontend
      labels:
        app: songrequest
        tier: frontend
    spec:
      containers:
        - name: webfrontend
          image: ACR_LOGIN_SERVER/songrequest.webfrontend:dev
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```

This manifest describes the deployment of our applications -- the Message Consumer
and Web Front End. It also contains a service that ensures we have an externally
accessible IP address that is load balanced in front of the Kubernetes cluster for
our Web Front End.

### Update AKS cluster desired state

1. Apply the deployments by executing:

    ```console
    kubectl apply -f .\deployments\songrequest.yaml
    ```

2. Watch the service status to see when it has been fully applied:

    ```console
    kubectl get service service-songrequest-webfrontend --watch
    ```
    It will update from displaying a `<pending>` to the port once the external IP
    assignment is complete. Use `CTRL+C` to exit the watch command after the IP
    has updated.

3. Navigate to the Song Request Web Front End in a browser -- using the external
IP address defined by the service-songrequest-webfrontend service.

4. Verify everything is healthy in your cluster by hitting the Kubernetes dashboard:

    ```console
    az aks browse --resource-group myResourceGroup --name myAKSCluster
    ```
    If your browser doesn't automatically open, open your browser and navigate
    to the address specified in the output on which the proxy is running.

5. Push some messages to your event hub using the [Event Generator](https://eventgen.azurewebsites.net/). You can check the container logs by
navigating to the Pods page, and selecting the Logs icon to the right on the
`songrequest-messageconsumer` Pod instance row. You can refresh the logs page
manually, or toggle auto-refresh from the toolbar to the top right. You should
see that song requests are being received by the container.

---
## Summary
---

By completing this lab, you now have two containerized applications hosted in
an Azure Kubernetes Service (AKS) cluster. You pushed your locally-built images
to an Azure Container Registry, created an AKS manifest to describe your
desired state, and deployed to your cluster. You've also seen how to grant access
to the ACR with the use of Role Assignments to the AKS service principal, how to
watch progress towards desired state configuration, and how to open the Kubernetes
dashboard.