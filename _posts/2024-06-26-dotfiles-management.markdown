---
layout: post
title: "Dotfiles Management with Dotbot and Chezmoi!"
date: 2025-06-26 22:32:59
author: Zois Roupas
categories: automation
tags: dotbot chezmoi dotfiles automation ansible git bash
cover: "/assets/2025-06_dotfiles_management/header.png"
---
# Dotfiles Management with Dotbot and Chezmoi!
---
## Summary
Hey everyone! Long time no see!

As a lazy guy myself, always trying to find ways to automate tasks and minimize, as much as possible, manual actions. 

One of my struggles, is keeping my different macOS machines synced. And by synced I mean from aliases, macOS system settings and dotfiles to gpg/ssh keys, homebrew packages and development tools.

So you can probably understand the pain of trying to keep the same environment, your aliases, your git or zsh config, while moving from your personal machine to your work related one.

## Dotfiles

One of the most important aspects of working across multiple machines is ensuring that my dotfiles are always accessible. In my case, there are 2 different dotfile categories, the **normal** ones like aliases, vimrc, zshrc etc, which are the same in each machine and the **templated** ones, as I like to call them.

By *templated* I mean dotfiles that should be different for each machine. For example, I don't want my *.netrc* to have work related tokens. Or in my personal machine I don't need a .*terraformrc*. 

Haven't found any dotfile manager that handles both cases so I decided to combine two different solutions to be able to keep my systems synced.

### Dotbot

[Dotbot](https://github.com/anishathalye/dotbot) is an awesome tool that I'm using for the *non-templated* dotfiles. It's very easy and straight forward to setup and use.

In my setup, I have a repo where dotbot is initialized and my dotfiles are present. The folder structure looks like this:

```
├── centos/
│   ├── dotfiles/
├── common/
│   ├── dotfiles/
│   └── vim/
├── dotbot/
│   ├── bin/
│   ├── dotbot/
│   ├── lib/
│   ├── tests/
│   ├── tools/
│   ├── CHANGELOG.md
│   ├── CONTRIBUTING.md
│   ├── LICENSE.md
│   ├── pyproject.toml
│   ├── README.md
│   ├── setup.py
│   └── tox.ini
├── mac/
│   ├── docker/
│   ├── dotfiles/
│   ├── gnupg/
│   ├── launchagents/
│   ├── m2/
│   └── vim_runtime/
├── install*
├── install.config.yaml
└── README.md
```

As you can see, I have a folder separated files in different folders,
* centos folder, linux specific dotfiles  in case I want to bootstrap a linux machine
* common folder, dotfiles that are relevant for both linux and mac
* dobot folder, is created from dotbot installation (check https://github.com/anishathalye/dotbot?tab=readme-ov-file#getting-started)
* mac folder, mac specific dotfiles and other config files
* install , script that will install the dotfiles
* install.config.yaml , config used by the installer to create the relevant symlinks

The **install.config.yaml** looks like this:

```
# cleans deadlinks , useful if we want to delete where we can rename the main folders and run dotbot to clean deadlinks
- clean: ['~']

- defaults:
    link:
      relink: true
      force: true

- link:
    ~/:
      glob: true
      path: common/dotfiles/.*
    ~/.vim/templates:
      glob: true
      path: common/vim/*
      create: true

- defaults:
    link:
      if: '[ `uname` = Darwin ]'
      relink: true
      force: true

- link:
    ~/:
      glob: true
      path: mac/dotfiles/.*
    ~/.docker:
      glob: true
      path: mac/docker/*
      create: true
    ~/.gnupg:
      glob: true
      path: mac/gnupg/*
    ~/.vim_runtime:
      glob: true
      path: mac/vim_runtime/*
    ~/Library/LaunchAgents:
      glob: true
      path: mac/launchagents/*
      create: true

- shell:
  - [launchctl unload ~/Library/LaunchAgents/]
  - [launchctl load ~/Library/LaunchAgents/]

- defaults:
    link:
      if: '[ `uname` = Linux ]'
      relink: true
      force: true

- link:
    ~/:
      glob: true
      path: centos/dotfiles/.*
      create: true

- shell:
  - [git submodule update --init --recursive, Installing submodules]

```

As you can easily see from the config, if executed it will create symbolic links at specified locations that point to files in your dotfiles repository

There is a separation between operating systems, Linux and Darwin in my case, and some shell commands that are specific to my needs but you can adapt it accodringly.

So after executing `./install -c install.config.yaml`,  you should something like that in your home folder :

```
lrwxr-xr-x     - testuser 26 Jun  2025 .zshr -> /Users/path/to/your/repo/mac/dotfiles/.zshr
lrwxr-xr-x     - testuser 26 Jun  2025 .zshrc -> /Users/path/to/your/repo/mac/dotfiles/.zshrc
lrwxr-xr-x     - testuser 26 Jun  2025 .gitignore_global -> /Users/path/to/your/repo/common/dotfiles/.gitignore_global
lrwxr-xr-x     - testuser 26 Jun  2025 .fzf.zsh -> /Users/path/to/your/repo/mac/dotfiles/.fzf.zsh
```

This is a rough explanation of how I'm handling *non-templated* files via Dotbot

> [!TIP]
> As my repo keeps sensitive data, like docker auth, gitlab tokens etc, I use [transcrypt](https://github.com/elasticdog/transcrypt) to encrypt such data.

### Chezmoi

Chezmoi is another awesome highly customizable tool that manages our dotfiles across multiple different machines, in a secure way.

Chezmoi documentation is very comprehensive, so I won't spend time in the details but you can start [here](https://www.chezmoi.io/user-guide/setup/) . It supports [encryption](https://www.chezmoi.io/user-guide/encryption/) and also [templating](https://www.chezmoi.io/user-guide/templating/) which is my favorite part! 
Have a look on the official official [User Guide](https://www.chezmoi.io/user-guide/command-overview/)

For my setup, I have yet another repo 😆 which is initialized via chezmoi and folder structure looks like this:

```
├── private_dot_ssh/
│   ├── encrypted_private_executable_config.tmpl.age
│   ├── encrypted_private_executable_id_rsa.tmpl.age
│   └── private_executable_id_rsa.pub.tmpl
├── dot_gitconfig.tmpl
├── dot_vimrc.tmpl
├── encrypted_dot_netrc.tmpl.age
├── encrypted_dot_terraformrc.tmpl.age
├── Makefile
└── README.md
```

Now lets see how my *.netrc* looks like:
`❯ chezmoi edit ~/.terraformrc`

```
{{- if eq .chezmoi.username "customuser" }}
credentials "gitlab.com" {
   token = "<REDUCTED>"
 }
{{- end }}
```

With chezmoi's templating power, a `~/.terraformrc` file will only be created if the user of the machine that I'm running `chezmoi apply` is **customuser** . Otherwise, no such file will be created.

The above is just an example and you can use the automatically-populated available variables based on your setup, found [here](https://www.chezmoi.io/reference/templates/variables/).

## Automation

Dotbot and Chezmoi managers can be of course configured and used independently of each other as well as standalone.

Since I wanted to automate the process, I created [MacOS-Ansible-Bootstrap](https://github.com/roupasz/MacOS-Ansible-Bootstrap)repository which uses both dotfiles managers to configure the files needed in every machine

In addition, the repository makes use of Bash and Ansible to configure macOS system settings, import gpg keys, install homebrew packages and plugins with the help of [Bitwarden](https://bitwarden.com/), my favorite Password Manager.

You can check the repo, adapt it accordingly and star it if you like it 😉

## Final Words 💡

Let's be honest, dotfile managers are awesome but all the heavy work is done from git.

Having my most useful files encrypted and tracked in a repository, significantly saves time both productivity and troubleshooting wise.

These awesome tools then act as wrappers adding advanced features that handle file symlinking automatically, saving you the time and effort of writing custom scripts. 

Thanks for reading, until the next post and as Dr Wallace Breen says in Half-Life 2..
Be wise. Be safe. Be aware!