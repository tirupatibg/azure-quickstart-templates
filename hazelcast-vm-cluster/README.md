# Deploy a Hazelcast Cluster

[![Deploy HZ to Azure](http://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fsrinusanchula%2Fazure-quickstart-templates%2Fmaster%2Fhazelcast-vm-cluster%2Fazuredeploy.json)

[Hazelcast](https://hazelcast.org) is an in-memory data platform which can support a variety of data applications such as data grids, nosql data stores, caching and web session clustering.

This template will deploy any number of Ubuntu Hazelcast instances in a vnet using the [official Hazelcast Azure Discovery Provider](https://github.com/hazelcast/hazelcast-azure). Every instance is installed with Hazelcast as an [systemd service](https://www.digitalocean.com/community/tutorials/systemd-essentials-working-with-services-units-and-the-journal) and will continue to run even after subsequent restarts. Each instance will discover every other instance on the network automatically so you can add and remove instances as you see fit.

Use the **Deploy to Azure** button above to get started.

Checkout Hazelcast's [official documentation](https://hazelcast.org/documentation/) to learn more on how to use Hazelcast.

## Azure Service Prinicpal

This template deploys resources that need read access the mangaement api's for the resource group this template is deployed to.

You'll need to setup [Azure Active Directory Service Principal credentials](https://azure.microsoft.com/en-us/documentation/articles/resource-group-create-service-principal-portal/) for your Azure Subscription for this plugin to work. With the credentials, fill in the `aadClientId`, `aadClientSecret`, and `aadTenantId` parameters.

## Architecture

As per the new architecture, each CEM region deployment will have multiple PODs served by two hazelcast clusters, blue & green. Blue & Green PODs will connects to blue & green hz clusters respectively. These HZ clusters are completely independent, won't share anything. They will align with the canary rollout of CEM.

For hazelcast version upgrade, green hz cluster will be provisioned with upgraded hazelcast vesion, CEM green PODs will connects to new cluster and CEM traffic will be routed green PODS after successful deployment.

There may be a data loss of non-persistent data between blue and green HZ clusters during switch. However, CEM has very limited use cases associated to non-persistent data. With short TTL and switching the entire tenant traffic to green cluster instantainously, the affect data loss is mitigated.

HZ cluster will be provisioned in the same resource group where CEM PODs resides.

Please refer [MT Architecture]() for additional info.

## How to Provision

This template can be used to create a new hazelcast cluster or add instances to existing cluster.

###### Prerequisites
> Resource group and vnet should be pre provisioned.

###### Create New HZ Cluster
Following command can be used to create new HZ cluster
```
az group deployment create \
  --subscription [AZURE-SUBSCRIPTION] \
  --name HZClusterDeployment \
  --resource-group [RESOURCE-GROUP-NAME] \
  --template-uri https://raw.githubusercontent.com/srinusanchula/azure-quickstart-templates/master/hazelcast-vm-cluster/azuredeploy.json \
  --parameters clusterName=[CLUSTER-NAME] \
  --parameters adminPassword=[CLUSTER-PASSWORD] \
  --parameters aadClientId=[AAD-CLIENT-ID] \
  --parameters aadClientSecret=[AAD-CLIENT-SECRET] \
  --parameters aadTenantId=[AAD-TENANT-ID] \
  --parameters vNetName=[VNET-NAME] \
  --parameters subnetAddressPrefix=[SUBNET-ADDRESS-RANGE] \
  --parameters instanceStartIndex=1
```

###### Parameter Description
- `clusterName` Cluster name. Ex: 'hzbluecluster' or 'hzgreencluster'.
- `adminUsername` Default user name 'dev'.
- `clusterUsername` By default `adminUsername` is used as `clusterUsername`.
- `adminPassword` Root user password for the VM.
- `clusterPassword` By default `adminPassword` is used as `clusterPassword`.
- `subnetAddressPrefix` Subnet address range for hazelcast instances. It's recommended that blue & green cluster instances stays in a separate subnets. Ex: 10.0.253.0/24 for blue & 10.0.254.0/24 for green clusters. BTW, it should align with the address space of VNET.
- `instanceCount` Default 2.
- `instanceStartIndex`: This index is used to enforce the uniqueness on instance name. Value should be no. of existing instances + 1.

###### Add Instances to an Existing Cluster
Same command as above including parameter values, except `instanceStartIndex` parameter. Value for instanceStartIndex should be 'instance count of existing hz cluster + 1'. Ex: if there are 2 instance in existing cluster, then the instanceStartIndex should be 3.

###### Created Components
These are the list of components that are created after successful provision.
1. Subnet for the cluster will be added to the VNET.
2. NSG (network security group) for the subnet.
3. One storage account
4. NIC and VM pairs as per the instanceCount.

Note: While adding the instances, subnet, NSG and Storage account are reused.

###### Blue and Green HZ cluster
As described above, both of these blue and green HZ clusters resides in the same resource group. Following are the 2 parameters that differentiate these clusters.
```
clusterName - It should be either 'hzbluecluster' or 'hzgreencluster'
subnetAddressPrefix - Different subnets as described above
```
The CEM HZ client auto-discovery service will make use of the `clusterName` (clusterName will be used as a VM tag at HZ instance) to properly resolve the ipAddresses of blue or green HZ cluster.