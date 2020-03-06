# MineMeld in Azure Container Instances (ACI)
## About MineMeld & ACI
MineMeld is an extremely useful tool for aggregating threat intelligence feeds. When combined with External Dynamic Lists (EDL's) on a Palo Alto Networks firewall, malicious traffic can be automatically blacklisted, and sanctioned applications can be whitelisted for only trusted destinations - the most notorious example being to only allow O365 traffic to reach legitimate Microsoft servers and domains.

Although typically deployed as part of AutoFocus or within a Linux virtual machine, MineMeld is fully supported in [container format](https://live.paloaltonetworks.com/t5/MineMeld-Articles/Running-MineMeld-using-Docker/ta-p/289062); this guide specifically addresses how to deploy it with persistence in the Azure Cloud. For additional information on containers and persistence in ACI, consult [this article.](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-volume-azure-files)

### Step 1: Create a storage account and file share
When you stop a container in ACI, all data within that container is lost. In the context of MineMeld, that means all user information and miners will be lost between restarts. We therefore need to deploy persistent storage ***before*** deploying our container. Although MineMeld typically calls for two volumes (one for config, another for logs), we will be skipping over logs due to both limitations in ACI and their limited usefulness when troubleshooting MineMeld errors. Logs created while the container is running will always be accessible.

Modify the **resource group** and **location** to reflect your desired deployment location, then paste the following into Azure Cloud Shell. We are using a random number for the storage account name to ensure uniqueness.

```
# Change these four parameters as needed
ACI_PERS_RESOURCE_GROUP=myResourceGroup
ACI_PERS_STORAGE_ACCOUNT_NAME=mystorageaccount$RANDOM
ACI_PERS_LOCATION=westus2
ACI_PERS_SHARE_NAME=acishare

# Create the storage account with the parameters
az storage account create \
    --resource-group $ACI_PERS_RESOURCE_GROUP \
    --name $ACI_PERS_STORAGE_ACCOUNT_NAME \
    --location $ACI_PERS_LOCATION \
    --sku Standard_LRS

# Create the file share
az storage share create \
  --name $ACI_PERS_SHARE_NAME \
  --account-name $ACI_PERS_STORAGE_ACCOUNT_NAME
```
Now run the following to get your storage account name:

`echo $ACI_PERS_STORAGE_ACCOUNT_NAME`

And then retrieve and store your storage account key:

`STORAGE_KEY=$(az storage account keys list --resource-group $ACI_PERS_RESOURCE_GROUP --account-name $ACI_PERS_STORAGE_ACCOUNT_NAME --query "[0].value" --output tsv)`

Validate that your storage key has been retrieved and stored in your variable with the following. You will need it for the next step.

`echo $STORAGE_KEY`

### Step 2: Create the MineMeld container instance

Paste the following into your Azure Cloud Shell to provision a single MineMeld container. We are using 1 vCPU and 1 GB of RAM; this is plenty for an initial deployment. As DNS names must be unique across instances within a region, we are also prepending the resource group name to the end of the label.

***Important Note***: We have chosen to use a Public IP Address here for the sake of rapid deployment. Once your container is up, you will need to change the default username and password and, ideally, move it behind a firewall or limit access to your private network. This can be done by changing the --ip-address flag to Private and by assigning a --vnet.

```
az container create \
    --resource-group $ACI_PERS_RESOURCE_GROUP \
    --name minemeld \
    --image paloaltonetworks/minemeld \
    --location $ACI_PERS_LOCATION \
    --cpu 1 \
    --memory 1 \
    --os-type linux \
    --ip-address Public \
    --dns-name-label minemeld-$ACI_PERS_RESOURCE_GROUP \
    --ports 80 443 \
    --azure-file-volume-account-name $ACI_PERS_STORAGE_ACCOUNT_NAME \
    --azure-file-volume-account-key $STORAGE_KEY \
    --azure-file-volume-share-name $ACI_PERS_SHARE_NAME \
    --azure-file-volume-mount-path /opt/minemeld/local/
```

### Step 3: Log in, change the default account, validate persistence
It will only take a minute for the container to start, at which point you should be able to log in with the default credentials, admin/minemeld. Add a new user account with a strong password and delete the admin account immediately.

To validate that your persistent storage is working, you can restart your container now. If you are still able to log in with your new account, persistence is working; if not, you will need to delete the container instance and validate the information provided in Step 2.

### Step 4: Add your miners and connect your firewalls
Your instance of MineMeld is ready to go! You can now import an existing configuration or add the feeds that meet your needs, such as the [Azure & Office 365 Whitelist](https://live.paloaltonetworks.com/t5/MineMeld-Articles/Enable-Access-to-Office-365-with-MineMeld-Updated/ta-p/224148).

Lastly, connect [PAN-OS to MineMeld using EDL's](https://live.paloaltonetworks.com/t5/MineMeld-Articles/Connecting-PAN-OS-to-MineMeld-using-External-Dynamic-Lists/ta-p/190414)!
