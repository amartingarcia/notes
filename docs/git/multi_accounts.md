---
title: Git - Multi-accounts
date: 20221212
author: Adrián Martín García
---

# Multi-Accounts
Sometimes it may be necessary to set up different git accounts for different directories.
In my computer I use the following configuration:

`~/.ssh/config.d/git`
```sh 
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/keys/github

Host ssh.dev.azure.com
    HostName ssh.dev.azure.com
    User git
    IdentityFile ~/.ssh/keys/azdevops    
```

`~/.gitconfig`
```sh
[user]
        name = fullname-fake
        email = username-fake@company-fake.com

[includeIf "gitdir:~/repos/personal/"]
        path = ~/repos/personal/.gitconfig
```

`~/repo/personal/.gitconfig`
```sh
[user]
  name = personal-username-fake
  email = personal-email-fake
```