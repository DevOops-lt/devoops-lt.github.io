---
layout: single
title:  "Azure App Service and IIS based reverse proxy"
date:   2022-02-14 13:20:00 +0200
categories: update azure devops web
author: Nilsas Firantas
teaser: "/assets/images/app_service_proxy_teaser.png.png"
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
