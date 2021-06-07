---
layout: single
title:  "Visual Studio Code Setup tips for Terraform development"
date:   2021-06-05 13:00:00 +0200
categories: Jekyll update terraform vscode ide dev
author: Nilsas Firantas
---

## IDE

As the post title suggests we are going to talk about setting up your work environment for developing on Terraform.
But don't be discouraged most of these will benefit you even if you are not targetting Terraform specifically.

When writting this I'm emphasizing `Windows` OS, however I'll try providing intructions for `Linux`/`Mac` as well.

### Installation

For `Windows` users it's easiest to do

```Batchfile
choco install vscode -y
```

It also seems that `winget` supports Visual Studio Code, well both comming from Microsoft should be no surprise I guess...

```Batchfile
winget install -e --id Microsoft.VisualStudioCode
```

For `Mac OS` users `brew` is the obvious choise

```bash
brew install --cask visual-studio-code
```

And for someone running a `Linux` based OS I'll cover `Debian` flavoured systems and you'll need to look it up for your prefered flavour if it doesn't match.

First we need to add an `apt` repository and aquire MS signing key.

```bash
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/trusted.gpg.d/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'
rm -f packages.microsoft.gpg
```

After installing the repo and key, you can run the following to install code (stable or insider build)

```bash
sudo apt install apt-transport-https
sudo apt update
sudo apt install code # or code-insiders
```

More information on Linux installation is provided in the [official documentation](https://code.visualstudio.com/docs/setup/linux)

### Configuration

Initialy VSCode does not require any configuration, it's pretty much plug and play.
However there a few tricks that I always make sure that I have to make my life easier.

#### PowerShell 7+

Yup, I know this is not exactly VSCode setup, 
but as someone that run Powershell everywhere
I need to make sure that I have relevant Cross Platform Powershell setup.

`Windows`

```Batchfile
choco install powershell-core -y
```

`Mac OS`

```bash
brew install --cask powershell
```

`Debian`

```bash
# Update the list of packages
sudo apt-get update
# Install pre-requisite packages.
sudo apt-get install -y wget apt-transport-https software-properties-common
# Download the Microsoft repository GPG keys
wget -q https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb
# Register the Microsoft repository GPG keys
sudo dpkg -i packages-microsoft-prod.deb
# Update the list of products
sudo apt-get update
# Enable the "universe" repositories
sudo add-apt-repository universe
# Install PowerShell
sudo apt-get install -y powershell
```

**Refrence:** [https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-linux?view=powershell-7.1#installation-via-package-repository---ubuntu-2004](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-linux?view=powershell-7.1#installation-via-package-repository---ubuntu-2004)

This should take care of our integrated terminal.
After restarting VSCode Powershell 7 should also be your default shell, if that doesn't happen out fo the box
you can try doing `ctrl+shift+p` start typing `Terminal: Select Default Profile` 

![Select Terminal Profile](./assets/images/terminal-select-01.png)

this should return you a selection of terminal available for you in which you can chose which one do you prefer.
Here's mine:

![Select Terminal Profile](./assets/images/terminal-select-02.png)

Now that we have Powershell setup I always do like to see some source control information in my shell.
Thus lets install `posh-git` with

```powershell
Install-Module -Name "posh-git"
```

**Note:** you may need to approve NuGet as a package manager if you never installed stuff before via this method.

After the instalation I always want it to start so I'll include it into my PS profile.
Easy way to find your profile in Powershell

```powershell
Write-Host $PROFILE
```

Output should look like this:

```powershell
"C:\Users\<YOUR USERNAME>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1"
```

Now if that file does not exist, feel free to create it.
If it does, open it with your favourite text editor and append the follwing:

```powershell
Import-Module -Name 'posh-git'
```

It's unrelated but for conveniance and while we're here, you can also add:

```powershell
Set-Alias -Name tf -Value terraform
```

This let's us to use `tf` instead of typing full word `terraform`

```powershell
tf init && tf plan && tf apply
```

instead of 

```powershell
terraform init && terraform plan && terraform apply
```

#### PSScriptAnalyzer

This one is easy to setup and super useful with real time Static analysis

```powershell
Install-Module -Name PSScriptAnalyzer
```

Real-time Static analysis should be enabled by default, but in case it isn't do this:
`ctrl+shift+p` type `preferences` and find `Preferences: Open Settings (UI)`
in the settings use the search bar and find the setting by typing `script analysis`
tick the checkbox if it is not already ticked

![Enable Real Time Analysis](./assets/images/script-analyzer.png)

### Plugins

Some of my favourite plugins

* Hashicorp terraform
  * Make sure it is published by Hashicorp, it is definately the best plugin for Terraform there is.
  * Excelent language support, auto-complete, up-to-date
  * [https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform)
* ident-rainbow
  * by `oderwat` [https://marketplace.visualstudio.com/items?itemName=oderwat.indent-rainbow](https://marketplace.visualstudio.com/items?itemName=oderwat.indent-rainbow)
  * makes all your ident clearly visible which helps keeping the code clean and consistant



