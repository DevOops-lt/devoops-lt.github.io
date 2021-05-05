---
layout: single
title:  "Encrypt secrets in Terraform"
date:   2021-03-11 13:34:56 +0200
categories: Jekyll update
---

In case you wanted to encrypt a value which should not be seen in source control Terraform allows for encryption in `.tf` and `.tvars` using [RSA + Base64 Encryption](https://www.terraform.io/docs/language/functions/rsadecrypt.html).

It so seems that Terraform currently does not have a mechanism to encrypt values itself so this has to involve some external script.

## Generate key pair

Personally I like having Terraform for generating these keys as I have this template ready but you are free to use `openssl` for this.

**NOTE:** if you don't yet have `openssl` installed `choco install openssl` on other OS than Windows use your favorite package manager

### The boring way

```powershell
PS> openssl genrsa -out id_rsa 2048
```

### The Terraform Way

```hcl
resource "tls_private_key" "ppk" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "local_file" "priv_key" {
  filename = "${path.module}/id_rsa"
  content  = tls_private_key.ppk.private_key_pem
}

resource "local_file" "pub_key_pem" {
  filename = "${path.module}/id_rsa.pem"
  content = tls_private_key.ppk.public_key_pem
}
```

## Encrypt secret values

Now that we have private and public keys we can encrypt secrets using `openssl`. As `rsautil` doesn't seem to work with in memory variables we pass in a `secretfile`

```powershell
PS> New-Item -Name cleartext.txt
PS> Set-Content .\cleartext.txt -Value "mysecretvalue"
PS> openssl rsautl -encrypt -inkey .\id_rsa -in .\cleartext.txt -out encrypted.txt
```

Now we should have an encrypted secret file within the same directory, now we can encode it.

```powershell
PS> openssl enc -a -in .\encrypted.txt -out .\encrypted64.txt -none
```

Now we should have a base64 encoded string which we can only decrypt using the private key that we've created previously. I pipe `clip` function here to get the value to my clipboard

```powershell
PS> Get-Content .\encrypted64.txt | clip
```

Don't forget to cleanup

```powershell
PS> rm cleartext.txt
PS> rm encrypted.txt
```

## Use encrypted secrets

Now we can use the secret value in Terraform files using the `rsadecrypt()` function. To test this I wrote a small terraform code snippet.

```hcl
locals {
    encrypted_string = <<EOT
pezs7ThtbvR5S9oMsUo4UmRh3yMMkYzMyrTugpXOLXASccu5MVOJTZviD14PvZ02
HBC8C2gA6hF/LSoXT6A4ba9CAbA9ZtDQHq9Gldf6BB20KArbzNDAFrHdgUhJQC/n
TEuSHrIc5FNsm9JOYGCVEe/n/uhc/W9vkwgE5iAN1ACH3BLAn+KZ/5fV9r7IhtLs
Ar3sjGBJWGp9hQN8FcD/s9EoEtrhy4Gp3cuXmlseVJx5LuWeSSS6HT7glr7FCg38
ow0jSp/Ucue/ak19TYdmBD2c4m+2lz95jkRl3R1Ip+m9cn+1R5TVeihgYUAhbY7/
bXsPe6BvjcaP98h3J5Tbqw==
EOT
    private_key_path = "${path.module}/id_rsa"
}
```

First I've pasted my encoded secret in [heredoc](https://www.terraform.io/docs/language/expressions/strings.html#heredoc-strings) format to preserve any new lines that were created for this encoded document, however after testing I realized that this didn't matter much and we can trim new lines and get the same results, so the above can also be written in single line like this:

```hcl
locals {
    encrypted_string = "pezs7ThtbvR5S9oMsUo4UmRh3yMMkYzMyrTugpXOLXASccu5MVOJTZviD14PvZ02HBC8C2gA6hF/LSoXT6A4ba9CAbA9ZtDQHq9Gldf6BB20KArbzNDAFrHdgUhJQC/nTEuSHrIc5FNsm9JOYGCVEe/n/uhc/W9vkwgE5iAN1ACH3BLAn+KZ/5fV9r7IhtLsAr3sjGBJWGp9hQN8FcD/s9EoEtrhy4Gp3cuXmlseVJx5LuWeSSS6HT7glr7FCg38ow0jSp/Ucue/ak19TYdmBD2c4m+2lz95jkRl3R1Ip+m9cn+1R5TVeihgYUAhbY7/bXsPe6BvjcaP98h3J5Tbqw=="
    private_key_path = "${path.module}/id_rsa"
}
```

**EDIT** My colleague suggested that we can get encoded output in single line format by adding a `-A` flag.

```powershell
PS> openssl enc -a -A -in .\encrypted.txt -out .\encrypted64.txt -none
```

Of course we could define this as a variable and not a local and have it in `.tfvars` file or we could use `file()` function to read the value from encrypted file but for my testing I did this just by copying straight into the file.

Now for the main code body I'll output a new file with the decrypted value, also going to use the decrypted value as a prefix in `random_pet` resource and have it output to console.

```hcl
resource "local_file" "secret" {
    filename = "./secretout.txt"
    content = rsadecrypt(local.encrypted_string,file(local.private_key_path))
}

resource "random_pet" "mypet" {
    prefix = trimspace(rsadecrypt(local.encrypted_string,file(local.private_key_path)))
}

output "rsadecrypt_output" {
    value = trimspace(rsadecrypt(local.encrypted_string,file(local.private_key_path)))
}

output "random_pet_name" {
    value = trimspace(random_pet.mypet.id)
}
```

Let's do a `terraform init` followed by `terraform plan`

```hcl
Terraform will perform the following actions:

  # local_file.secret will be created
  + resource "local_file" "secret" {
      + content              = <<-EOT
            mysecretvalue
        EOT
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "./secretout.txt"
      + id                   = (known after apply)
    }

  # random_pet.mypet will be created
  + resource "random_pet" "mypet" {
      + id        = (known after apply)
      + length    = 2
      + prefix    = "mysecretvalue"
      + separator = "-"
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

Lastly `terraform apply -auto-approve`

```hcl
local_file.secret: Creating...
random_pet.mypet: Creating...
random_pet.mypet: Creation complete after 0s [id=mysecretvalue-master-airedale]
local_file.secret: Creation complete after 0s [id=aaf05d4870a3af4a183c0885592a85035f8bbf0f]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

random_pet_name = "mysecretvalue-master-airedale"
rsadecrypt_output = "mysecretvalue"
```

Happy coding :)
