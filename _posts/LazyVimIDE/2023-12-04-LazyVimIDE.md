---
layout: post
title: LazyVim IDE
date: 2023-12-04 09:00:00 +800
categories: [Linux, Software]
tags: [neovim, lazyvim]
image:
  path: /assets/img/headers/lazyvim_cover.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## Introduction

I change my IDE from LunarVim to LazyVim now. Because LunarVim always get stuck when I install it on a new devices. \
LazyVim is a Neovim setup powered by 💤 lazy.nvim to make it easy to customize and extend your config. Rather than having to choose between starting from scratch or using a pre-made distro, LazyVim offers the best of both worlds - the flexibility to tweak your config as needed, along with the convenience of a pre-configured setup.

## Prerequisite

- Neovim >= 0.9.0 (needs to be built with LuaJIT)
- Git >= 2.19.0 (for partial clones support)
- a Nerd Font (optional)
- a C compiler for nvim-treesitter.

## Install Neovim

#### Debian / Ubuntu

1. Install build prerequisites on your system:
   `sudo apt-get install ninja-build gettext cmake unzip curl`
2. `git clone https://github.com/neovim/neovim`
3. `cd neovim && make CMAKE_BUILD_TYPE=RelWithDebInfo`
4. `cd build && cpack -G DEB && sudo dpkg -i nvim-linux64.deb`
5. `nvim --version`

#### Install Nerd Font

1. Download the [Nerd Fonts](https://www.nerdfonts.com/)
2. Unzip
3. cp to ~/.local/share/fonts/
4. `sudo apt install fontconfig`
5. `sudo fc-cache -fv`

## Install LazyVim

#### lazygit

```
LAZYGIT_VERSION=$(curl -s "https://api.github.com/repos/jesseduffield/lazygit/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
curl -Lo lazygit.tar.gz "https://github.com/jesseduffield/lazygit/releases/latest/download/lazygit_${LAZYGIT_VERSION}_Linux_x86_64.tar.gz"
tar xf lazygit.tar.gz lazygit
sudo install lazygit /usr/local/bin
```

#### ripgrep & fd

`sudo apt-get install ripgrep fd-find`

- Backup

```
# required
mv ~/.config/nvim{,.bak}

# optional but recommended
mv ~/.local/share/nvim{,.bak}
mv ~/.local/state/nvim{,.bak}
mv ~/.cache/nvim{,.bak}
```

- clone
  `git clone https://github.com/LazyVim/starter ~/.config/nvim`

- remove .git
  `rm -rf ~/.config/nvim/.git`

`nvim`

## Clone my nvim config (optional)

`git clone https://github.com/Technicatgor/lazyvim.git`

change the name `nvim`, put it into ~/.config/nvim

```
mv ~/.config/nvim{,.bak2}
cp -r ./lazyvim ~/.config/nvim
```

## Update

`nvim`
