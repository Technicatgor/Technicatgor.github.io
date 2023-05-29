---
layout: post
title: LunarVim - Turn Vim into an IDE
date: 2023-05-16 15:00:00 +800
categories: [Linux, Software]
tags: [neovim,lunarvim,lazygit,linux]
---
![lunarvim](https://www.lunarvim.org/assets/images/lunarvim_logo-ea848294964e3ff5fd5af2ea28e2f23f.png)
## Prerequisites
- [Neovim v0.9.0+](https://neovim.io/)
- git, make, pip, python, npm, [node](https://github.com/nodesource/distributions/blob/master/README.md) and [cargo](#cargo)
- [lazygit](https://github.com/jesseduffield/lazygit#installation) (**optional**)

> **_NOTE:_**  This docs is based on Ubuntu 22.04 for example.

## Neovim v0.9
Adding this [PPA](https://launchpad.net/~neovim-ppa/+archive/ubuntu/stable) to your system
### Ubuntu
```sh
sudo add-apt-repository ppa:neovim-ppa/stable
sudo apt update
sudo apt install neovim -y
```
### Debian
```
wget https://github.com/neovim/neovim/releases/download/nightly/nvim.appimage -O nvim
chmod +x nvim && mv nvim /usr/bin/
mkdir -p ~/.config/nvim
```

check your neovim version
```sh
nvim --version
Neovim v0.9.0+.
```
> remeber export your PATH env `export PATH="/usr/bin:$PATH"`

## pip3
```sh
sudo apt-get install python3-pip
```
## NodeJS v20
remove and purge the exist nodejs 
```sh
sudo apt remove --purge nodejs
sudo apt autoremove
```
install v20
```sh
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && \
sudo apt install -y nodejs
```
check node version
```sh
node -v
v20.0.1
```
## Cargo
```
curl https://sh.rustup.rs -sSf | sh
```

## LunarVim
### Install
```sh
LV_BRANCH='release-1.3/neovim-0.9' bash <(curl -s https://raw.githubusercontent.com/LunarVim/LunarVim/release-1.3/neovim-0.9/utils/installer/install.sh)
```

### Set your PATH
if you using bash:
```sh
cat <<EOF >> ~/.bashrc 
export PATH="$HOME/.local/bin:$PATH"
alias vim='lvim'
EOF
```

Download my lunarvim config file
```sh
wget https://raw.githubusercontent.com/Technicatgor/launcher/main/lunarvim/config.lua -O ~/.config/lvim/config.lua
```

### Update
press leader key(default is spacebar), press `L` then `u`


### Uninstall
```sh
bash ~/.local/share/lunarvim/lvim/utils/installer/uninstall.sh
```
