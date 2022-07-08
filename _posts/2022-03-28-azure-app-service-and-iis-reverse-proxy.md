---
layout: single
title:  "Azure App Service and IIS based reverse proxy (Part 1)"
date:   2022-03-28 13:20:00 +0200
categories: update azure devops web
author: Nilsas Firantas
teaser: "/assets/images/app_service_proxy_teaser.png"
---

## Azure App Service and IIS based reverse proxy

### General

At some point you might find yourself in a spot needing a quick proxy and think to yourself: "Well cloud has me covered, right?".
Unfortunatelly that is not quite true, while cloud has a lot of amazing things to offer quick, easy and cheap proxy services are absent.
This brings me to the current story. I found myself in a situation where I need to whitelist an IP address for Azure Cloud.
Anyone whos ever had this pleasure already knows where I'm going with this. 
Almost all Azure services has some possible IP ranges, some separated by region, some global, some can have 15-ish some close to a thousand possible IP address _ranges_.
At this point I need to have one to three IP addresses or ranges any more and I consider that not manageble.
Oh did I mention I'm in a predicament of using Hybrid infrastructure so just whitelisting a Service Tag and calling it a day is not an option.
To make things harder I'm also not able to whitelist a FQDN, just a plain old CIDR range.

### Enter Azure App Services

Alright, fixing one issue at a time, I decide to tackle IP whitelisting issue first and I'll get back to routing traffic later.
Here cloud has us covered all I need to do is deploy a Web App setup Outbound traffic routing and presto it should just habve a single IP that I manage.
Although it is a bit harder than the previous sentence might have sounded it gets done pretty quickly.

Let's bring out Terraform and do a bit of Infrastructure coding.
First we need App Service Plan to host our App Service.
I'm using Standard tier here, but feel free using Free and F1 for testing and playing around and once you're okay with it switch it to a production ready SKU.
I will probably make this into a ready to grab code on Github at some point, but for now you'll just have to image the missing code snippets.
For the record no magic is used behind the scenes, generic state setup, version bindings, resource group setup, nothing fancy.

```hcl
resource "azurerm_app_service_plan" "main" {
  name                = "asp-my-proxy"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  sku {
    tier = "Standard"
    size = "S1"
  }

  tags = var.tags
}
```

Next I need to build an App Service itself, this has some stuff I need for functionality and some stuff to be secure. Let's see:

```hcl
resource "azure_app_service" "main" {
  name                    = "app-my-proxy"
  location                = azurerm_resource_group.main.location
  resource_group_name     = azurerm_resource_group.main.name
  app_service_plan_id     = azurerm_app_service_plan.main.id
  client_affinity_enabled = true
  https_only              = true
  tags                    = var.tags

  site_config {
    dotnet_framework_version    = "v4.0"
    always_on                   = true
    vnet_route_all_enabled      = true
    ftps_state                  = "FtpsOnly"
    scm_use_main_ip_restriction = true
    ip_restriction = [
      {
        service_tag               = "PowerPlatformInfra"
        name                      = "PowerPlatformInfra"
        priority                  = 100
        action                    = "Allow"
        headers                   = null
        ip_address                = null
        virtual_network_subnet_id = null
      }
    ]
  }
}

lifecycle {
  ignore_changes = [
    site_config["scm_type"]
  ]
}
```

There are a few interesting bits in the above code snippet.

The obvious stuff first: 
`https_only` and `ftps_state` are a must have as Security Center or Microsoft Defender for Cloud as it is now called will flip out on you.

Next up using `ip_restriction` to only allow a select Azure Resource. 
PowerPlatform in this example in one of the most if not the most possible IP range having resource.
Obviously I didn't count all possible IP addresses in all possible service tags, 
but if you are interested enough you can download a JSON list with possible Azure IP addresses from [here](https://www.microsoft.com/en-us/download/details.aspx?id=56519).

Lastly there is a `lifecycle` block which ignores `"scm_type"` and this is just a habbit of adding this through the years. 
If this is not present each time you'd deploy your code to the App Service via Azure DevOps, next time Terraform would be ran it would detect an Infrastructure drift.
If memory serves me right, Terraform would try to fix this drift, but would throw an error on the first try and only would go through on a second try, which can get real annoying real fast.

#### Single Outbound IP address

Now to setup Virtual Network assignment for our new App Service which will in turn route all our outbound traffic through a single IP address.

At first it seems like it would be really hard to do, but it takes just a few blocks of Terraform to solve this,
first we create Virtual Network and Subnet:

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "vnet-my-proxy"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.243.0.0/16"]

  tags = var.tags
}

resource "azurerm_subnet" "main" {
  name                 = "subnet-my-proxy"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.243.0.0/24"]

  delegation {
    name = "app-service-delegation"

    service_delegation {
      name    = "Microsoft.Web/serverFarms"
      actions = ["Microsoft.Network/virtualNetworks/subnets/action"]
    }
  }
}
```

The only interesting bit in the above code is the subnet `delegation` pay attention to it, as this is needed to assign subnet to our App Service.

Next we need a Public IP, easy:

```hcl
resource "azurerm_public_ip" "main" {
  name                = "pip-my-proxy"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "Standard"
  allocation_method   = "Static"
  domain_name_label   = "pip-my-proxy"
  availability_zone   = "No-Zone"

  tags = var.tags
}
```

Azure provider now requires me to explicitly set `availability_zone` parameter to `No-Zone` it seems to me that it used to just default to it previously, but what ever this works.

Next to route App Service Traffic through this IP we need a NAT Gateway. When I think of Gateways it's always a huge piece of code with loads of moving parts, well thank the Gods this one is simple enough.

```hcl
resource "azurerm_nat_gateway" "main" {
  name                    = "nat-gw-my-proxy"
  location                = azurerm_resource_group.main.location
  resource_group_name     = azurerm_resource_group.main.name
  sku_name                = "Standard"
  idle_timeout_in_minutes = 10

  tags = var.tags
}
```

Okay now that we have all the resource, we need to glue them together, this is where we have to define `association` resources.

```hcl
resource "azurerm_nat_gateway_public_ip_association" "main" {
  nat_gateway_id       = azurerm_nat_gateway.main.id
  public_ip_address_id = azurerm_public_ip.main.id
}

resource "azurerm_subnet_nat_gateway_association" "main" {
  subnet_id      = azurerm_subnet.main.id
  nat_gateway_id = azurerm_nat_gateway.main.id
}

resource "azurerm_app_service_virtual_network_swift_connection" "main" {
  app_service_id = azurerm_app_service.main.id
  subnet_id      = azurerm_subnet.main.id
}
```

Here we go now NAT Gateway knows about Public IP and the Subnet it should use
and the app service also knows about the subnet it should use.

That is it for Part 1 we have now setup the Infrastructure we need for our Proxy, 
in the next part I will overview what files we need to deploy to make use of IIS that runs in the backend of Azure App Services.
Also I'll introduce some PowerShell to deploy Proxy configuration and
we'll also look at automating it all with CI/CD.
