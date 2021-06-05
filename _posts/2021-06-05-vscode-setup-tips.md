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
