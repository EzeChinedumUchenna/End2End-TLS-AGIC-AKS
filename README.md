# Setting up End-to-End SSL for AKS Behind AGIC

## Introduction

In this guide, we will delve into the process of configuring end-to-end SSL for Azure Kubernetes Service (AKS) behind Azure Application Gateway Ingress Controller (AGIC). Our objective is to establish end-to-end Transport Layer Security (TLS) encryption using AGIC on Application Gateway while ensuring that traffic remains encrypted from the client to the Application Gateway and continues to be encrypted as it traverses from the Application Gateway to our application, residing within the pod.

## Prerequisites
Before proceeding, ensure the following prerequisites are met:

- You have deployed an AKS cluster with the kubenet plugin.
- An Application Gateway is already set up.
- Azure CLI (Az CLI) and OpenSSL are installed on your computer.

## Configuration Steps

### 1. Generate the Frontend and Backend Certificates

#### Frontend Certificate
Generate a frontend certificate to present to clients connecting to the Application Gateway. This certificate should have the subject name CN=frontend.

```bash
openssl ecparam -out frontend.key -name prime256v1 -genkey
openssl req -new -sha256 -key frontend.key -out frontend.csr -subj "/CN=frontend"
openssl x509 -req -sha256 -days 365 -in frontend.csr -signkey frontend.key -out frontend.crt
```
#### Backend Certificate
Generate a backend certificate to be presented by the backends to the Application Gateway. This certificate should have the subject name CN=backend.

~~~~bash
openssl ecparam -out backend.key -name prime256v1 -genkey
openssl req -new -sha256 -key backend.key -out backend.csr -subj "/CN=backend"
openssl x509 -req -sha256 -days 365 -in backend.csr -signkey backend.key -out backend.crt
~~~~

#### Install the generated certificates on your Kubernetes cluster:

```bash
kubectl create secret tls frontend-tls --key="frontend.key" --cert="frontend.crt"
kubectl create secret tls backend-tls --key="backend.key" --cert="backend.crt"
```

### 2. Deploy a Simple Application with HTTPS

A .NET application is hosted in a pod. Create a deployment file that contains two containers: the application itself and Nginx as a sidecar to handle HTTPS traffic. Nginx will receive encrypted traffic and redirect it to the application over HTTP.

```bash
kubectl apply -f nedum.yaml
```

#### Ensure the pod is running and test the application:

```bash

kubectl get pods
kubectl exec -it website-deployment-<pod-id> -- curl -k https://localhost:8443
```

### 3. Upload the Backend Certificate's Root Certificate to Application Gateway
Upload the self-signed backend certificate's root certificate to the Application Gateway:

```bash

applicationGatewayName="<gateway-name>"
resourceGroup="<resource-group>"
az network application-gateway root-cert create \
    --gateway-name $applicationGatewayName  \
    --resource-group $resourceGroup \
    --name backend-tls \
    --cert-file backend.crt
```

### 4. Enable the AGIC Add-on in Existing AKS Cluster Through Azure CLI
Enable the AGIC add-on in your existing AKS cluster using the Azure CLI:

```bash

appgwId=$(az network application-gateway show -n myApplicationGateway -g myResourceGroup -o tsv --query "id") 
az aks enable-addons -n myCluster -g myResourceGroup -a ingress-appgw --appgw-id $appgwId
```

### 5. Setup Ingress for End-to-End TLS
Configure your ingress to use the frontend certificate for frontend SSL and the backend certificate as the root certificate so that Application Gateway can authenticate the backend. Refer to the ingress.yaml file for configuration details.

```bash
tls:
    - secretName: frontend-tls
      hosts:
        - website.com
```

#### For backend SSL, add the following annotations:

```
appgw.ingress.kubernetes.io/backend-protocol: "https"
appgw.ingress.kubernetes.io/backend-hostname: "backend"
appgw.ingress.kubernetes.io/appgw-trusted-root-certificate: "backend-tls"
```

### After completing the above steps, you should be able to access your website securely.

## Key Notes
Ensure that the virtual network Application Gateway resides in is the same as the AKS nodes' resource group. Delegate the Microsoft.Network/virtualNetworks/subnets/join/action permission to the identity used by AGIC for the subnet that the Application Gateway is deployed into.
Troubleshooting steps include checking AGIC logs, verifying AGIC configuration, and confirming permissions.
This revised version provides a more technical explanation of the steps involved in setting up end-to-end SSL for AKS behind AGIC.


## Troubleshooting Azure Application Gateway Ingress Controller (AGIC)

This guide provides troubleshooting steps for issues related to Azure Application Gateway Ingress Controller (AGIC) when it is used as an ingress controller with Azure Kubernetes Service (AKS).

### Key Note

If your virtual network Application Gateway is not in the same resource group as the AKS nodes, follow these steps to ensure proper functionality:

1. Ensure the identity used by AGIC has the `Microsoft.Network/virtualNetworks/subnets/join/action` permission delegated to the subnet where the Application Gateway is deployed.

   If no custom role is defined with this permission, you can use the built-in "Network Contributor" role, which already includes the necessary permission.

### Permission Configuration

To configure the necessary permissions for AGIC, follow these steps:
```shell
# Get application gateway id from AKS addon profile
appGatewayId=$(az aks show -n myCluster -g myResourceGroup -o tsv --query "addonProfiles.ingressApplicationGateway.config.effectiveApplicationGatewayId")


# Get Application Gateway subnet id
appGatewaySubnetId=$(az network application-gateway show --ids $appGatewayId -o tsv --query "gatewayIPConfigurations[0].subnet.id")


# Get AGIC addon identity
agicAddonIdentity=$(az aks show -n myCluster -g myResourceGroup -o tsv --query "addonProfiles.ingressApplicationGateway.identity.clientId")


# Assign network contributor role to AGIC addon identity to subnet that contains the Application Gateway
az role assignment create --assignee $agicAddonIdentity --scope $appGatewaySubnetId --role "Network Contributor"

````

### Note: When encountering issues with Azure Application Gateway as an ingress controller, where AGIC is not updating the Application gateway, the following steps can be taken to troubleshoot the problem:

Check the logs of the Application Gateway Ingress Controller (AGIC) pod for any error messages.
```shell
kubectl logs -n kube-system -l app=ingress-azure
```

- Verify that the AGIC pod is running and the desired number of replicas is met.
```
kubectl get pods -n kube-system -l app=ingress-azure
```

- Verify that the AGIC is correctly configured by checking that the pod’s environment variables match the expected values.
```
kubectl exec -it -n kube-system <agic-pod-name> env
```

- Verify that the AGIC has the correct permissions to access the Application Gateway.Use the below to assign the right permission
```
# Get application gateway id from AKS addon profile
appGatewayId=$(az aks show -n myCluster -g myResourceGroup -o tsv --query "addonProfiles.ingressApplicationGateway.config.effectiveApplicationGatewayId")


# Get Application Gateway subnet id
appGatewaySubnetId=$(az network application-gateway show --ids $appGatewayId -o tsv --query "gatewayIPConfigurations[0].subnet.id")


# Get AGIC addon identity
agicAddonIdentity=$(az aks show -n myCluster -g myResourceGroup -o tsv --query "addonProfiles.ingressApplicationGateway.identity.clientId")


# Assign network contributor role to AGIC addon identity to subnet that contains the Application Gateway
az role assignment create --assignee $agicAddonIdentity --scope $appGatewaySubnetId --role "Network Contributor"
```

- Check the status of the Application Gateway using the Azure Portal or the Azure CLI.
```
az network application-gateway show -g <resource-group> -n <app-gateway-name>
````

- Check the routing rules defined in the Application Gateway to ensure they match the ingress rules defined in the Kubernetes cluster.
- Check that the Application Gateway’s public IP address is correctly associated with the ingress rules.
- Check that the target service is responding and that it has the correct IP address and ports.
- Check the service endpoint
```
Kubectl get ep
```

### Please note that these steps provide a basic guide, and more complex issues may require further investigation.



