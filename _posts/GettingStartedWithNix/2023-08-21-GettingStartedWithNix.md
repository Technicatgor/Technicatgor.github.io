---
layout: post
title: Getting Started With Nix
date: 2023-08-21 13:00:00 +800
categories: [Linux,Nix]
tags: [linux,nix,dev]
image:
  path: /assets/img/headers/preview-nix.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---
## Nix 
Nix is a declarative package manager that enables users to declare the desired system state in configuration files(declarative configuration), and it takes responsibility for achieving that state.

## Prerequisites
- curl
Supported Platform:
- Linux (i686, x86_64, aarch64).
- macOS (x86_64, aarch64).

## Installation
### Multi Users
This option requires either:
- Linux running systemd, with SELinux disabled
- MacOS
```
bash <(curl -L https://nixos.org/nix/install) --daemon
mkdir /nix
chown $USER /nix
```

### Single User
Single-user is not supported on Mac.
```
bash <(curl -L https://nixos.org/nix/install) --no-daemon
```
Verify installation:
```
nix --version
```

## Using Nix with Docker
```
docker run -ti nixos/nix
```

## Install Global Packages
```
nix-env -i --attr nixpkgs.nodejs
```
```
node -v
v18.17.1

which node
/root/.nix-profile/bin/node
```

## Uninstall Global Packages
```
nix-env --uninstall nodejs
uninstalling 'nodejs-18.17.1'
```

## Update the channels
```
nix-channel --update nixpkgs
nix-env --upgrade '*'
```

## Rollback
```
nix-env --rollback
```

## Delete unused packages
We should run it periodically.
```
nix-collect-garbage --delete-old
```

## Nix Shell(Without install)
You can use Nix packages without installing them globally on your machine. This is a good way to bring in the tools you need for individual projects.
```
nix-shell -p powershell
[nix-shell:~]# pwsh
PowerShell 7.3.4
PS /root>
```

## Set up a development environment
Let’s build a Python web application using the Flask web framework as an exercise.

For our Flask web application, create a new file called myapp.py and add the following code:
```
#! /usr/bin/env python

from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return {
        "message": "Hello, Nix!"
    }

def run():
    app.run(host="0.0.0.0", port=5000)

if __name__ == "__main__":
    run()
```
This is a simple Flask application which serves a JSON document with the message “Hello, Nix!”.

To declare the development environment, create a new file shell.nix:
```
{ pkgs ? import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/eabc38219184cc3e04a974fe31857d8e0eac098d.tar.gz") {} }:

pkgs.mkShell {
  packages = [
    (pkgs.python3.withPackages (ps: [
      ps.flask
    ]))

    pkgs.curl
    pkgs.jq
  ];
}

```
We can now use nix-shell to launch the shell environment we just declared:
```
nix-shell
python ./myapp.py
```
Launch a new terminal to start another session:
```
nix-shell
curl 127.0.0.1:5000 | jq '.message'
```

## Uninstallation
### Single User
```
rm /nix
```
### Multi Users
```
sudo systemctl stop nix-daemon.service
sudo systemctl disable nix-daemon.socket nix-daemon.service
sudo systemctl daemon-reload

sudo rm -rf /etc/nix /etc/profile.d/nix.sh /etc/tmpfiles.d/nix-daemon.conf /nix ~root/.nix-channels ~root/.nix-defexpr ~root/.nix-profile
```
Remove build users and their group:
```
for i in $(seq 1 32); do
  sudo userdel nixbld$i
done
sudo groupdel nixbld
```

### MacOS
[https://nixos.org/manual/nix/stable/installation/uninstall#macos](https://nixos.org/manual/nix/stable/installation/uninstall#macos)

