---
layout: single
title:  "Terraform coding guidelines"
date:   2021-05-05 13:20:00 +0200
categories: Jekyll update
author: Nilsas Firantas
---

## Terraform code writing practices

### General

* Terraform modules should introduce ease of use not the other way around.
Rule of thumb is do not write a module for one resource, it is probably easier to implement without a module. Exception is for a large resource to be able to deploy it with minimal inputs, when modules uses sane defaults for everything else.

* Do not shorten on change names of variables in the resource, it makes for much easier use of the module comparing it to the official documentation of the resource.

* Make modules easy to use for other engineers if I can write less lines when using your module than using a resource block I'm going to use the module.

### File structure

* Module should consist of main.tf, variables.tf outputs.tf versions.tf

  * main.tf - Should host all resource definitions, internal processing and local values if needed

  * variables.tf - Should contain all variable definitions, no processing or local values

  * outputs.tf - Should contain all output definitions and nothing else. Output value accepts some processing before return a value, that is permitted.

  * versions.tf - Should only contain `terraform{}` block with appropriate values.

### Naming

* Always follow the naming convention set by your organization, it will help you in many cases when used consistently

* Use of prefix variable is encouraged to describe the first part of a resource name, it usually is location-client name-tier/environment that are mostly consistent throughout the resources

* When authoring a module use prefix variable to pass fully made prefix rather than trying to assemble it inside the module, for example use this:

```hcl
variable "prefix" {
  type        = string
  default     = "prefix"
  description = "(Required) Specifies the Prefix to prepend on all resources. Changing this forces a new resource to be created."
}
```

* To maintain consistency the naming of a resource should be done through `format()` function which is build-in Terraform feature. This allows us to have a complex naming resolution within the module while keeping consistent through out the modules.

```hcl
resource "azurerm_app_service" "prod" {
  name = format("%s-app", var.prefix)
  ....
}
```

### Versioning

* Terraform has a versioning block in it self to prevent outdated dependencies, this should be kept in separate `versions.tf` file within the project directory

* We should lock down lowest possible version of a dependency for example `for_each` block has been introduced in Terraform version `0.12.6` so if your module uses `for_each` this should be noted.

* Restrictions should apply to major providers of Terraform meaning that you may leave out the restriction block for build in providers like `random` or `tls` but should apply to `AzureRM`, `AWS`, `OCI` etc.

* Terraform and Provider versions are usually locked down in the root module, so child modules may just have `">=x.x.x"` notation meaning no less than provided version.

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 2.8"
    }
  }
  required_version = ">= 0.12.26"
}
```

**NOTE**: The above example is valid from Terraform `0.12.26`

Reference [https://www.terraform.io/docs/configuration/provider-requirements.html#requiring-providers]

### Variable and Output definitions

Both variables and outputs should be defined in separate files from the main code, `variables.tf` and `outputs.tf` respectively.

#### Variables

Should have `name`, `type`, `default`, `description` values.

```hcl
variable "prefix" {
  type        = string
  default     = "prefix"
  description = "(Required) Specifies the Prefix to prepend on all resources. Changing this forces a new resource to be created."
}
```

* `Name` would usually match the name described in Terraform resource, for example if a resource `azurerm_resource_group` requires a value for `location` our module should ask the name `location` exactly.

* `Type` similar to Go types but has some uniqueness for example:
  * `string`
  * `number`
  * `bool`
  * `map(string)`
  * `list(string)`
  * `list(number)`
  * `object({key = value})`

* `Default` should be a sane value, for any optional features we would use false for naming like prefix default value would be "prefix" to clearly see if this has not been provided, for any features and options that should be omitted by default use `null`

* `Description` Use common information that you would want to see here, for example load_balancer_type can be ether `"Basic"` or `"Standard"` but it is a string value thus a free text field, in the description you can mentions that these are the only possible values.

**NOTE:** prefix variable block above is considered correct.

#### Outputs

Similarly to variables it must have a name, value and description. From modules you should pass the full object as output rather than just one value. So the below is considered correct:

```hcl
output "resource_group" {
  description = "Outputs the full resource group object from this module"
  value       = azurerm_resource_group.rg
}
```

`Name` should match the resource you are outputting minus the provider

`Value` should be the full object of what you are outputting (exceptions apply)

`Description` mostly required to provide accurate documentation by automation so should be useful common information. Favored by Terraform module repository

### Formatting

For linting use command `terraform fmt`

Some IDEs provide formatting for HCL out of the box but tends to break a lot of stuff so `terraform fmt` is still the preferred way.

### Commenting

For single line commenting we use '#' sign. It is the default commenting style for terraform and should be used in most cases. Comment style used should usually be on top of the line rather than an inline comment.

Reference: [https://www.terraform.io/docs/configuration/syntax.html#comments](https://www.terraform.io/docs/configuration/syntax.html#comments)

### Features

Module should be able to access most or all the features exposed by the resource it provisions, although all not mandatory features should be disabled by default. Sane defaults should be used.

When introducing a new feature, please keep in mind that it may break other workflows so do not make required variables needlessly.

### Testing

As any code we write, Terraform code can be tested, there is still a heated argument within IaC community on how should the infrastructure code be tested and verified. We use [Terratest](https://terratest.gruntwork.io/) for some sanity tests and to make sure code actually runs.

For now complex tests are not required although we need to make sure the module runs with the basic configuration so the test can look like this:

```go
package tests
 
import (
    "testing"
 
    "github.com/gruntwork-io/terratest/modules/terraform"
)
 
func TestTerraformBuild(t *testing.T) {
    t.Parallel()
 
    terraformOptions := &terraform.Options{
        TerraformDir: "./fixture_simple",
 
        Vars: map[string]interface{}{},
    }
 
    terraform.InitAndApply(t, terraformOptions)
    defer terraform.Destroy(t, terraformOptions)
}
```

### Documentation

While I do not prophesize any specific template please create a `README.md` file and describe what the module does as well some example usage.

You can also use terraform-docs to generate basic information of your module.

### Advanced

#### Get comfy in your chair maybe get your favorite beverage or something stronger before going further

Advanced module development requires making some key decisions when thinking of scaling the deployment.

First topic is to use for_each rather than count because of how count works it will not be aware in some situations that we need to get rid of or modify something in the middle of machine array.

Let's image a possible implementation

```hcl
locals {
  vm_count = 2
  disk_count = 5
}
 
resource "provider_vm" "vm" {
  count   = local.vm_count
  os_disk = true
}
 
resource "provider_additional_data_disk" "disk" {
  count          = vm_count * disk_count
  attach_disk_to = provider_vm.vm[count.index % disk_count]
}
```

This is how Terraform will make the assignments from out imagined code:

* machine[0] will get data_disk[0]
* machine[1] will get data_disk[1]
* machine[0] will get data_disk[2]
* machine[1] will get data_disk[3] and so on....

This implies that if we add additional Machine into the mix like `vm_count = 3` everything in the workflow will change

The assignments will screw up as an extra Machine is inserted in the queue.

* machine[0] will get data_disk[0]
* machine[1] will get data_disk[1]
* machine[2] will get data_disk[2]
* machine[0] will get data_disk[3]
* machine[1] will get data_disk[4]
* machine[2] will get data_disk[5] and so on..

To circumvent this we need to use for_each where applicable this will use names for iterating over resources. To make this happen we need some internal processing to generate a set of Machines and Disks.

For this we can use `setproduct()` another Terraform built in feature which will generate correct lists for us.

To make this a bit more visual

```hcl
setproduct(["machine01", "machine02", "machine03"], ["disk01", "disk02"])
```

will generate:

```hcl
[
  [
    "machine01",
    "disk01",
  ],
  [
    "machine01",
    "disk02",
  ],
  [
    "machine02",
    "disk01",
  ],
  [
    "machine02",
    "disk02",
  ],
  [
    "machine03",
    "disk01",
  ],
  [
    "machine03",
    "disk02",
  ],
]
```

Which will always make sense and will not drift like count would.

Note: this example is incomplete, if you wish to find the values needed for each there is a good example in Terraform documentation: [https://www.terraform.io/docs/configuration/functions/setproduct.html#finding-combinations-for-for_each](https://www.terraform.io/docs/configuration/functions/setproduct.html#finding-combinations-for-for_each)
