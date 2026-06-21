---
date: 2026-06-21T11:43:12
title: "NixOS, sops-nix and age keys"
---

Heavily based on this epic blog post https://michael.stapelberg.ch/posts/2025-08-24-secret-management-with-sops-nix/ by Michael Stapelberg.

## NixOS repo

My nixos config repo structure.

```
.
├── flake.lock
├── flake.nix
├── hosts
│   ├── nixos-applications
│   │   ├── default.nix
│   │   ├── hardware-configuration.nix
│   │   └── shared-host-folder.nix
│   ├── nixos-server
│   │   ├── default.nix
│   │   ├── disko-config.nix
│   │   ├── hardware-configuration.nix
│   │   ├── secrets
│   │   │   └── example.yml
│   │   └── secrets.nix
│  ...
└── modules
    ├── audio.nix
   ...
```

## Age keys

We need the public key of the user or system which decrypts our secrets.

To get an age key based on the ssh key of a server do this on your server.

``` shell
cat /etc/ssh/ssh_host_ed25519_key.pub | nix run nixpkgs#ssh-to-age
```

This will print something like `age1rcjyvm3d2ythis7tj4zagwu9riswlk7ynotav7fmrealzd6zkeyqmz3jky`.
Rerunning it will always print the same key because its based on `/etc/ssh/ssh_host_ed25519_key.pub`.

Create `.sops.yaml` in your nix configuration repo.

``` yaml
keys:
  - &nixos-server age1rcjyvm3d2ythis7tj4zagwu9riswlk7ynotav7fmrealzd6zkeyqmz3jky
creation_rules:
  - path_regex: hosts/nixos-server/secrets/[^/]+\.(yml|yaml|json|env|ini)$
    key_groups:
    - age:
      - *nixos-server
```

This contains only the key of the server -> only the server can decrypt the secrets.
If you want to edit a secret later on you have to do that on the server.

If the secret file is new you can create it on any system with `sops`.

``` shell
sops ./hosts/nixos-server/secrets/example.yml
```

Content of my unencrypted `example.yml`.

``` yaml
api-key: hello i use arch btw
```

## Edit secret on server.

What i came up with was

`sudo SOPS_AGE_SSH_PRIVATE_KEY_FILE=/etc/ssh/ssh_host_ed25519_key sops ./hosts/nixos-server/secrets/example.yml`

but it did not work.

Probably because it was not encrypted for the ssh public key but instead for the age key generated from the ssh public key.

Instead this is what worked.

``` shell
nix-shell -p sops ssh-to-age
SOPS_AGE_KEY_CMD="sudo ssh-to-age -private-key -i /etc/ssh/ssh_host_ed25519_key" sops ./hosts/nixos-server/secrets/example.yml
```

Your `$EDITOR` will be openend and you can edit the secret.

## Multiple secret files

I added a second secret file.

``` shell
sops ./hosts/nixos-server/secrets/example2.yml
```

Content of my unencrypted `example2.yml`.

``` yaml
example_key_level1:
    example_key_level2:
        example_key_level3: value-in-3
```

## Add sops-nix to nixos

Now lets add sops-nix to our nixos configuration.

### flake.nix

Add sops-nix to inputs, outputs and to the modules of the system you wanna use it on.

``` nix
{
  description = "Multi-machine NixOS configuration";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixpkgs-unstable";

    disko = {
      url = "github:nix-community/disko";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    sops-nix = {
      url = "github:Mic92/sops-nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, disko, sops-nix, ... }: {
    nixosConfigurations = {
      nixos-applications = nixpkgs.lib.nixosSystem {
        modules = [ ./hosts/nixos-applications/default.nix ];
      };
      nixos-server = nixpkgs.lib.nixosSystem {
        modules = [
          disko.nixosModules.disko
          sops-nix.nixosModules.sops
          ./hosts/nixos-server/default.nix
        ];
      };
    };
  };
}
```

### secrets.nix

Create a new file `secrets.nix` and import it on your system on which the secret will be used.

``` nix
{ config, lib, pkgs, ... }:

{
  sops = {
    age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
    secrets = {
      "api-key" = {
        sopsFile = ./secrets/example.yml;
      };
      "example_key_level1/example_key_level2/example_key_level3" = {
        sopsFile = ./secrets/example2.yml;
      };
    };
  };
}
```

## Rebuild

``` shell
sudo nixos-rebuild switch
```

## Server /run/secrets

The secrets are now available under `/run/secrets`.

``` shell
sudo cat /run/secrets/api-key
```

```
hello i use arch btw
```

``` shell
sudo cat /run/secrets/example_key_level1/example_key_level2/example_key_level3
```

```
value-in-3
```
