---
title: "Azure Storage HA/DR, Private Networking, and Client Caveats"
date: 2025-12-26T14:00:00-05:00
language: en
featured_image: ../assets/images/posts/storage-ha-private-networking-and-client-caveats.png
summary: How I set up disaster recovery for Azure Datalake across two un-paired regions using Terraform
description: How I set up disaster recovery for Azure Datalake across two un-paired regions using Terraform
author: Andrew
authorimage: ../assets/images/global/icon.webp
categories: career
tags: Career, Cloud, Azure, Terraform, Infrastructure as Code, Disaster Recovery
---
**As a DevOps/ Platform Engineer,** one of the questions stakeholders are going to ask at some point is "what is our plan for disaster recovery?" As I was approaching a year in with this particular client, it was unsurprising to hear this question brought up again. At just two years into my own Cloud and DevOps journey, this was my trial by fire moment. The other Architect level resource had just rolled off the project, and it was on me to design and implement full fail-over of the client's Azure Big Data Analytics stack.

## The Stack
**A quick background** to the tooling we were using for this client. Data gets ingested to a single Azure Data Lake from multiple different sources. This could be from live streaming data, logs, product shipment data, etc. From there, a combination of automated or manual data translation jobs from Azure Data Factory and Azure Databricks - Azure Data Factory was originally used to orchestrate jobs in Databricks, but as Databricks has rolled out new automation features it was slowly retired. These jobs would record changes in a "delta table," and store these tables in a standard medallion architecture in folders labeled as such - bronze, silver, and gold - in storage containers within the datalake. Each instance of this stack was deployed over three different environments for data engineers - dev, test, and prod - with a fourth environment solely dedicated to infrastructure development. We had a fifth environment as well, but it was for deploying data governance with Azure Purview, and is irelevent to this article.

## The Requirements
**Our team had no strict SLAs to meet,** since this project supported a smaller group of about 20 or so engineers across two teams. As long as the data was still accessible by those that needed to consume it, the ETL pipelines could go down for a bit and no-one internally would be affected by an outage. Additionally, I was accommodating the client's increasingly tighter budget constraints, and knew they would be more than happy with a "good-enough" solution, rather than something that could withstand a nuclear-war and keep ticking.
After a bit of back-and-forth with the client, the client and I decided on a hot/warm with active standby approach. Azure Data Lake had already been set up to use GRS - geo-redundant storage - and  did not include a hot standby instance. The rest of the stack was deployed one-to-one in a secondary Azure region. A complete secondary standby may seem costly at first, but this satisfied the budget requirements as well. Most of the stack uses SaaS or PaaS products on Azure. The cost of Azure Databricks is dependent on DBU and compute consumption, Data Factory doesn't charge until it has run, and empty Key Vaults don't cost anything. For the nominal cost of some pre-wired networking, and bootstrapping secrets in some of those Vaults, we were able to keep costs at about $100 a month. These were further reduced when the client started to use Datafactory and Databricks' built-in ingestion mechanisms instead of running Self Hosted Integration Runtimes (SHIRs).

## The Biggest Challenge
**In consulting, one of the biggest challenges** is adhering to some unique customer requirements. Azure has paired regions for the High Availability of many of their resources. If one region goes down for some reason, the underlying data is accessible at the second region, albeit with some depredations to the data management plain. The client's main Azure deployment region is East US. The paired region for East US is West US, but the client's backup Azure deployment region is North Central US. 
This raised some complications. For starters, everything ran behind Azure Private Link Private Endpoints. That meant that I wasn't able to rely on the mechanisms that Azure uses for automatic fail-over. However, there were some saving graces. A resource in Azure can have multiple private endpoints attached to it, and the endpoints can cross regions to attach to different resources. Step one complete, create a secondary endpoint in North Central US and attach it to the existing Data Lake. 
But what about DNS? How do we route traffic away from the default paired region in Azure during a fail-over? How does Azure Datalake's DNS fail-over for GRS even work to begin with under the hood?

## A CNAME on a CNAME
**Azure Storage has a globally unique public FQDN** for each of the resources created on its public cloud. This is one of the reasons why Azure Storage account needs a globally unique name. This FQDN comes up as `<storage_acount_name>.blob.core.windows.net.` However, this isn't the A record for the storage account, but rather a CNAME record that points to the host's A record `blob.<regionally_specific_storage_stamp>.store.core.windows.net` On top of this, added Private Link inserts another CNAME in-between the `<storage_account_name>.blob.core.windows.net` and `blob.<regionally_specific_storage_stamp>.store.core.windows.net.` The full chart looks something like this:

| **Seq** | **Name** | **Type** | **Record Value** |
| --- | --- | --- | --- |
| 1 | <storage_acount_name>.blob.core.windows.net | CNAME | <storage_acount_name>.**privatelink.blob.core.windows.net** |
| 2 | <storage_acount_name>. **privatelink.blob.core.windows.net** | CNAME | blob.<regionally_specific_storage_stamp>.store.core.windows.net |
| 3 | blob.<regionally_specific_storage_stamp>.store.core.windows.net | HOST (A) | <IP Address> |
chart from: [dmauser - Private Link/Endpoint DNS Integration Resources](https://github.com/dmauser/PrivateLink/blob/master/DNS-Integration-Scenarios/README.md)

**When a fail-over happens,** the region specific storage stamp in the A record gets rewritten in the background to represent the change to the new regional endpoint target for the storage account. The critical part here, is that this rewrite to the new region stamp happens after any CNAME rewrites. The good news is, that means I didn't have to worry about routing DNS queries from North Central US to West US, and I had free-reign to to design the DNS failover. The bad news is that now I'm on the hook for designing the DNS failover. 

## Design
**I had two design options** depending on which part of the infrastructure needs to be pre-configured. I could either use a single DNS zone for all the private endpoints, or have each private endpoint link to a unique private DNS zone in their respective regions. 
| DNS Zone | Pros | Cons | Fail-over Steps |
| --- | --- | --- | --- |
| Global | Easier post fail-over steps | Not all DNS can be pre-configured, more difficult to set up in Terraform | Update DNS Record to Secondary Endpoint |
| Multi Regional | Infrastrcuture can be pre-configured, faster/ automatic fail-over from Azure internal resources | More challenging Fail-over steps for non-Azure traffic | External Traffic: Re-route traffic to Secondary Region by using Traffic Manager or updating routing tables. Internal Traffic: Dependent on Status of Other Secondary Resources |

**Option 1 - Single Global Private DNS Zone**
With a single Private DNS zone, the client could continue to use their existing DNS forwarding methods. A second Private Endpoint is created in North Central US, and linked back to the Storage Account in East US. The drawback of this setup, is that the DNS record cannot be set up in advance for the new private endpoint, as it is connected to the same Private DNS Zone, and has the same FQDN as the existing endpoint. 
![Global/ Common Private DNS Zone Design](../assets/images/posts/Cross\ Region\ Data\ Lake\ Failover - Global\ DNS.png)

Option 2 - Regional Private DNS Zones
With option two, two DNS Zones are set up and can route queries independently of each-other. This has a few benefits: the infrastructure can be a true Active/Active or Active/Standby as all infrastructure is already set up, DNS records don't have to be updated in a fail-over, and standby resources in the secondary region will automatically be routed through the second endpoint. However, this now brings in a new challenge. External traffic now has a point of failure since there is no global DNS resolution to redirect traffic at fail-over. Traffic needs to be redirected from East US to North Central US, either with Azure Traffic Manager pointing to an Ingress point for automatic routing changes, or updating routing tables for on-premises to Azure traffic.
![Regional Private DNS Zone Design](../assets/images/posts/Cross\ Region\ Data\ Lake\ Failover - Regional\ DNS.png)

I presented these two options to the client, with the expectation that they would take the "Single-global DNS zone" option. The existing DNS zone for this storage account already used this design, and it would be easier to slot in the first design option. Additionally, this option posed the lowest cost burden.
You can read more about HA/DR consideration for Azure Private Link in Adam Stewart's whitepaper here, or watch his accompanying YouTube video here.

---

## Terraform
**Let's talk about the code.** On top of the challenges faced from a design standpoint, the terraform code had to be re-factored as to not redeploy - and lose any existing data in - the Datalakes already deployed in the lower environments. The DNS configurations for private link also needed to be changed in a way that creates the new private endpoint, but doesn't re-create the DNS configuration - I found out that this has to be done only after doing the re-create, I'll touch on that later.
Since we already had our environment names passed in through our deployment pipelines, I decided on using a `count = lower(var.environment) != "disasterrecovery" ? 1 : 0` ternary expression to gate the re-deplyment of the datalake. If I were to set up an out-of-the-box module for this in the future, I would add a feature-flag boolean variable akin to `is_disaster_recovery` for more flexibility. I then added a moved block to account for this change. Frustratingly, this then broke every single resource that depended on the Datalake resource block. Due to how Terraform creates its dependency tree, it's easy to run into issues with computed fields that depend on other resource blocks. The workaround for this was not adding another `count` field, but adding a ternary condition based on the length of the `azurerm_storage_account.datalakeresource` set created by the earlier `count` expression I set. 

**example:** customer managed encryption key that depends on the Datalake.
```hcl
resource "azure_storage_accout_customer_managed_key" "datalake" {
  count = length(azurerm_storage_account.datalake) > 0 ? 1 : 0

  storage_account_id = length(azurerm_storage_account.datalake) > 0 ? azurerm_storage_account.datalake[0].id : null

  key_vault_id = azurerm_key_vault.this.id
  key_name     = length(azurerm_storage_account.datalake) > 0 ? azurerm_storage_account.datalake[0].name : null
}
```
To wrap everything up, I added in a dynamic block for the DNS configuration of of the Blob and Datalake DFS private endpoint connections based on the same ternary condition as the datalake. 
```hcl
resource "azurerm_private_endpoint" "datalake_endpoint" {
  name                  = "example"
  resource_group_name   = var.resource_group_name
  location              = var.location
  subnet_id             = var.subnet_id
  
  private_service_connection {
    <removed for brevity>
  }

  dynamic "private_dns_zone_group" {
    for_each = lower(var.environment) != "disasterrecovery" ? [1] : []
    content {
      name                   = "default"
      private_dns_zone_ids   = var.networking.private_dns_zone_ids
    }
  }
}
```
With these changes, I was able to add the disaster recovery environment to our deployment pipeline, and enable the secondary region without making changes to the existing environments.
