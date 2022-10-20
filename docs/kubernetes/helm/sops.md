---
title: SOPS
date: 20220220
author: Adrián Martín García
---

# SOPS

[SOPS](https://github.com/mozilla/sops) is a tool that allows the encryption of local files. For our use case we use a Key from Key Vault declared in the **.sops.yaml** file for file encryption.

## SOPS Commands

### Show file
```sh
$ cat file.yaml
---
secrets:
    admin-creds:
        stringData:
            user: user-fake
            password: password-fake
```

### Encrypt file
```sh
$ sops -e -i file.yaml
$ cat file.yaml
secrets:
    admin-creds:
        stringData:
            user: ENC[AES256_GCM,data:qiZNFqw=,iv:n/rXJHTD8D8DD4iI3zF5JpjDeZKp+SPItQSq0ue5VXc=,tag:DeD5PA4OwbsmoSNzzdQtSg==,type:str]
            password: ENC[AES256_GCM,data:7uDh0tJ0c/0VOw==,iv:ft33mHwy4zpmTvvq1mHOasy5/6rQR70Gv189IImeRoQ=,tag:9yP4pM8Y8oSwYCcuPso0jQ==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv:
        - vault_url: <vault-url>
          name: <key-name>
          version: <version-name>
          created_at: "2022-10-20T09:33:41Z"
          enc: Z130UV0JoF4XLZHnJ-Y0hIHVfSLVqAGEr3niWMosL0vRF00TaoreX2bm_JLy8LvvxNKgS3jZXKW5RGujq4bA2_scKrAarkAvOhdg2N4CgykO1Yq4_9KP0ZbmD_FE0nzj3VN8fQsin5kOYrOPjVl5u1x8YLpsFQekt6E_Jj8TEcPaJPmj32sfdxvSWdASYxuJandU5o2aDeuZ_dkX6H9MaNdD68SJCNJQaSMm0IaWENVsyE24KaPKgkkqkXW8Pv92BtKw-Xgg6O2jYb4trokobkraE-Siaq0EwGbGPCi7zVlNH6ImPLG5vHTCLa9HEiG5Nd88A5OOA1yJ45f5NMpFug
    hc_vault: []
    age: []
    lastmodified: "2022-10-20T09:33:42Z"
    mac: ENC[AES256_GCM,data:cq4UrhEJq0x+66pZ7/Ga3DLeMohXnLiCiFOorF+cZYBJXviIj2Ncaw/jYTiEF3D5i8OBoZCz2O9pxSOpHHvrdiETvNfj/0/pGmPWmuflsPJ6Ehen8pKOdh91tJ0K2cBUWzym5X21SxcTjhjg1h8pIsK32e5mgWDu6Oai/G6gnjs=,iv:v6Ds1O2w8T5OE8X9wh8Q9Fez/Blwdxd87Wy7hSnoozM=,tag:a66zoL/C1U97tOE1/jI4cQ==,type:str]
    pgp: []
    unencrypted_suffix: _unencrypted
    version: 3.7.3
```

### Decrypt file
```sh
$ sops -d -i file.yaml
```