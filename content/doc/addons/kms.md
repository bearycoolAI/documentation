---
type: docs
title: Clever KMS
description: A serverless Hashicorp Vault API compatible FoundationDB layer
tags:
- addons
keywords:
- vault
- iam
- biscuit
- foundationdb

type: docs
---

**Clever KMS** or **KMS Layer** is a serverless [Hashicorp Vault](https://www.vaultproject.io/) API compatible
[FoundationDB layer](https://apple.github.io/foundationdb/layer-concept.html).

It is fully compatible with the official [Vault CLI](https://developer.hashicorp.com/vault/docs/commands)
and can be handled by it or by curl calls.

## Feature

### KV API

- [get](https://developer.hashicorp.com/vault/docs/commands/kv/get)
- [put](https://developer.hashicorp.com/vault/docs/commands/kv/put)

### Transit API

Full support except [BYOK](https://developer.hashicorp.com/vault/docs/secrets/transit#bring-your-own-key-byok),
implemented later.

## Requirements

Running:

- FoundationDB server >= 7.3

Building:

- Rust >= 1.80

## How to use

To running the Clever KMS you'll need a valid Foundation DB installation (standalone or cluster).

You also need a [configuration file](./docs/config/configuration.md).

I'll use the following:

```toml
port = 8202

[cluster]
api_version = 730
cluster_file_path = "/etc/foundationdb/fdb.cluster"

[root_tenant_keyring_provider]
name = "shamir"

[master_keyring_provider]
name = "shamir"
namespace = "kms-c1"
node_name = "kms-c1-n1"

[authenticator]
name = "biscuit"
[authenticator.provider]
name = "iam"
keys = [
    [
        "7cba42eeaed244a4c4d4bec5f2955dd4e40fbbdcd04cacd0b6d02813bcf67b18",
        "017078e99d10f24e706bd215c66ba3fd7f57a2087a5dd94b6881856d1ebfbc19"
    ],
]
service = "toto"
```

To create required Biscuits, you can either handcraft them or use
the [KMS-cli](https://gitlab.corp.clever-cloud.com/clever-cloud/clever-cloud-kms/kms-cli/-/releases).

You'll need 3 kinds of Biscuits:

- cluster operator token to init the cluster and unseal nodes
- tenant operator token to create a new vault and unseal it
- basic token to classic operations

First create the Cluster operator token. Scoped to "kms-c1".

```bash
kms-cli biscuit --addon-id kms-c1 --privilege cluster-operator
``` 

```
Token: 

EmwKAhgDEiQIABIgmLfKBc0UguC115huG6hdftyXOODkO4Y9khFEFcUU4GcaQIGbBg2_JGdkfpybFu6LMC_UmjELGiMkYvyd59UJOhJQwSuiutLBlupP5wqKS2uSiSt-GY69hRli_1ZcwTcE2QcawQIKbQopb3JnYV9mMzAyMmY2MC1iMzA0LTQ3MjItODlhYy1lMjM2MzA1NTgxMTIKBmttcy1jMQoGcmlnaHRzCghvcGVyYXRvchgEIgkKBwgHEgMYgAgiCQoHCAISAxiBCCIOCgwIgggSBzoFCgMYgwgSJAgAEiCiafpzk3Zk5qDM4rE1PpQDGXWpXQvoFnM3vKPxZXLgJRpAq15H_5u2SPwVm-lBLysBrIFFGwrRsU0Q2pCUxjySqpE-Sz9QnaH7goQN8qX3IZxM2hKhxPp9WK8sU9NTqdsZASJoCkAKsNOYobIQHztERxaD1AE2B8YFQVX7F7iehqjaSrHDTO0TIlRgB-eFJZ7G6qq3XFfCD2GhyAJNu2V2o7RSEfEIEiQIABIgAXB46Z0Q8k5wa9IVxmuj_X9Xogh6XdlLaIGFbR6_vBkiIgogurrcCeb6nfCUETIMwPIeuAa_k_cg-8PLkZz6TgN-h9c=
Root Public Key ed25519/7cba42eeaed244a4c4d4bec5f2955dd4e40fbbdcd04cacd0b6d02813bcf67b18
Service Public Key ed25519/017078e99d10f24e706bd215c66ba3fd7f57a2087a5dd94b6881856d1ebfbc19
```

Then the Tenant operator token. Scoped to "vault_8fa9b31c-4c55-4f11-b9bf-6a656c678906"

```bash
kms-cli biscuit --addon-id vault_8fa9b31c-4c55-4f11-b9bf-6a656c678906 --privilege tenant-operator
```

```
Token: 

EmwKAhgDEiQIABIgIDSz4OaBs6qJW4q74aM-kZRB7el0ApcOCGT-iapAdYgaQHzQ0jlHGnQyBE925UP6SWZhizoWJnKQLX1ZP7ry6xVKKbAAJ3BSxfbW86qZ6cGEuDxK3VSr6FAAAMeAyP3Yrwoa4gIKjQEKKW9yZ2FfZjMwMjJmNjAtYjMwNC00NzIyLTg5YWMtZTIzNjMwNTU4MTEyCip2YXVsdF84ZmE5YjMxYy00YzU1LTRmMTEtYjliZi02YTY1NmM2Nzg5MDYKBnJpZ2h0cwoEcm9vdBgEIgkKBwgHEgMYgAgiCQoHCAISAxiBCCIOCgwIgggSBzoFCgMYgwgSJAgAEiAKfbVccGhS5KDzl7eutGPC9VePa4X6QZu4hVP6ylIthxpAqbqe80Hije6Qs8HbFnDViQ0kHMtNvJiHPd3yYUxBpto3XWgeLeQ6Kwy3v2ho6qAGeavTqIVIvrNImjPU7jqWCCJoCkD9q9ob1hhYvAu5WB1PMERMVtcFZY1Oan9iEXxy84SJ29BdaoK6qQhI2pSXKk-S9PHdg036dmxUsFAIKq_ruKsPEiQIABIgAXB46Z0Q8k5wa9IVxmuj_X9Xogh6XdlLaIGFbR6_vBkiIgogwSdVkXt_phTMXtOPXvwSKyQ4s3QGqab6M61zos_3GaA=

Root Public Key ed25519/7cba42eeaed244a4c4d4bec5f2955dd4e40fbbdcd04cacd0b6d02813bcf67b18
Service Public Key ed25519/017078e99d10f24e706bd215c66ba3fd7f57a2087a5dd94b6881856d1ebfbc19
```

And finally a tenant token.

```bash
kms-cli biscuit --addon-id vault_8fa9b31c-4c55-4f11-b9bf-6a656c678906 --privilege tenant
```

```
Token: 

EmwKAhgDEiQIABIgsOkzd2YSNEFCjOXvoTCFzF3eODh5zS7tiaMmKMytruYaQJ9WshlBIvOktMDQPBsl1jQgplgngvm7h0VCgLoGcxWVYpUPBL_n86ZHrCCIGRiDwBssTRrGIUtSRwVnlP5rWgoa1wIKggEKKW9yZ2FfZjMwMjJmNjAtYjMwNC00NzIyLTg5YWMtZTIzNjMwNTU4MTEyCip2YXVsdF84ZmE5YjMxYy00YzU1LTRmMTEtYjliZi02YTY1NmM2Nzg5MDYKBnJpZ2h0cxgEIgkKBwgHEgMYgAgiCQoHCAISAxiBCCIJCgcIgggSAjoAEiQIABIgdPl3oHqGXUkzDYWGQ6kKJeUJI2ZfDgpP1mU_669kga8aQHqISFrDjO5M5aPI65Sl1m0LcRRgLTKQevnhfevQ2pnCJZYZHAUWlNm7FDmx-LqMga1OJhMb6iE1G9VqwZ3vKAsiaApAPa9iUhPf1NBQUEUjK0s6c7mjGKp60euNnbBc-THbJrwC12jr-CGyb_zWE2PDfQPa4VZJHRExcGsHLubcIBUXDxIkCAASIAFweOmdEPJOcGvSFcZro_1_V6IIel3ZS2iBhW0ev7wZIiIKICn-6JUc3fkMug2_QnfOLlTBx198VHmgrNdjGRbkiv6B

Root Public Key ed25519/7cba42eeaed244a4c4d4bec5f2955dd4e40fbbdcd04cacd0b6d02813bcf67b18
Service Public Key ed25519/017078e99d10f24e706bd215c66ba3fd7f57a2087a5dd94b6881856d1ebfbc19
```

Once done, you'll need the vault CLI [installed](https://developer.hashicorp.com/vault/docs/install/install-binary).

```
❯ vault --version
```

You can then start the Clever KMS.

```bash
kms-layer --configuration config/shamir.toml 
```

```
2024-11-05T07:56:29.448944Z  INFO exporting metrics to http://127.0.0.1:9090/metrics    
2024-11-05T07:56:29.451578Z  INFO Initializing schemas
2024-11-05T07:56:29.461375Z  INFO Schemas initialized
2024-11-05T07:56:29.461426Z  INFO get_authenticator:get_authenticator: Getting biscuit authenticator
2024-11-05T07:56:29.461440Z  INFO get_authenticator:get_authenticator: Using root public keys: ["7cba42eeaed244a4c4d4bec5f2955dd4e40fbbdcd04cacd0b6d02813bcf67b18"]
2024-11-05T07:56:29.461559Z  INFO Initializing MasterKeyringShamirProvider node_name="kms-c1-n1"
2024-11-05T07:56:29.461592Z  INFO Starting server host="127.0.0.1" port=8202
2024-11-05T07:56:29.461604Z  INFO Server listening without TLS
2024-11-05T07:56:29.461977Z  INFO starting 16 workers
2024-11-05T07:56:29.462015Z  INFO Actix runtime found; starting in Actix runtime
2024-11-05T07:56:29.462026Z  INFO starting service: "actix-web-service-127.0.0.1:8202", workers: 16, listening on: 127.0.0.1:8202
```

The layer is up and running waiting for connections.

To work with the vault CLI, you need to export the address.

```bash
export VAULT_ADDR="http://127.0.0.1:8202"
```

Then login to the vault with the "cluster-operator" token.

```bash
vault login
```

```
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                EmwKAhgDEiQIABIgmLfKBc0UguC115huG6hdftyXOODkO4Y9khFEFcUU4GcaQIGbBg2_JGdkfpybFu6LMC_UmjELGiMkYvyd59UJOhJQwSuiutLBlupP5wqKS2uSiSt-GY69hRli_1ZcwTcE2QcawQIKbQopb3JnYV9mMzAyMmY2MC1iMzA0LTQ3MjItODlhYy1lMjM2MzA1NTgxMTIKBmttcy1jMQoGcmlnaHRzCghvcGVyYXRvchgEIgkKBwgHEgMYgAgiCQoHCAISAxiBCCIOCgwIgggSBzoFCgMYgwgSJAgAEiCiafpzk3Zk5qDM4rE1PpQDGXWpXQvoFnM3vKPxZXLgJRpAq15H_5u2SPwVm-lBLysBrIFFGwrRsU0Q2pCUxjySqpE-Sz9QnaH7goQN8qX3IZxM2hKhxPp9WK8sU9NTqdsZASJoCkAKsNOYobIQHztERxaD1AE2B8YFQVX7F7iehqjaSrHDTO0TIlRgB-eFJZ7G6qq3XFfCD2GhyAJNu2V2o7RSEfEIEiQIABIgAXB46Z0Q8k5wa9IVxmuj_X9Xogh6XdlLaIGFbR6_vBkiIgogurrcCeb6nfCUETIMwPIeuAa_k_cg-8PLkZz6TgN-h9c=
token_accessor       n/a
token_duration       ∞
token_renewable      false
token_policies       ["default"]
identity_policies    []
policies             ["default"]
```

You can now initialize the cluster.

```bash
vault operator init
```

The KMS-CLI allows to also create Master Key shares, but for simplicity sake, here
a split key

```
AX0/zTf+FLouvPIldi96WnUAPfunNJEeXD+KmWvys4K0
AuZGhgA0qiloj5h4Brrckj4HVcJvaFfyEpFCvWg2TT8O
A7VjcxfTNBAw9SF5DBqRbXyUPFtB7tBgfKEGVdh7hb7v
BAb6YFiczo3Imjr5LVglvCsgPTmKEMFxpfHb8L/8wffJ
BVXflU97ULSQ4IP4J/hoQ2mzVKCklkbjy8GfGA+xCXYo
```

With the same cluster operator token, unseal the node by calling 3 times the command
with 3 different shares.

```bash
vault operator unseal
```

You'll see the Progress field increasing until reaching 3/3.

```
Unseal Key (will be hidden): 
Key                Value
---                -----
Seal Type          n/a
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       n/a
Version            0.1.0
Build Date         n/a
Storage Type       n/a
HA Enabled         false
```

Once done, the "Sealed" field will flip to "false".

```
❯ vault operator unseal
Unseal Key (will be hidden): 
Key             Value
---             -----
Seal Type       n/a
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         0.1.0
Build Date      n/a
Storage Type    n/a
HA Enabled      false
```

Now, login with the "tenant operator" token

```bash
vault login
```

And initialize the Tenant Vault.

```bash
vault operator init
```

```
Unseal Key 1: ASRzXqCk2BmDTQNrZb84LTrmGxefjboqTo7WL3tFVqwN
Unseal Key 2: Aqg+5IT7KYHhzIV+n1t8ObGWYsxgoWYEgqBMayt8ojcq
Unseal Key 3: AykO/19ITYxQDSc/1sTHKW8ZTP1ULpcMUUvbwP2oU3tR
Unseal Key 4: BPfNfNW04iUDVwD906sKpbldt/WW4362Ccqj7WArYnJc
Unseal Key 5: BXb9Zw4HhiiylqK8mjSxtWfSmcSibI++2iE0Rrb/kz4n

Initial Root Token: pUNFexe8FDKMoSosIIM95Gk1JqsCSyKdZUGErZGn4HY

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated root key. Without at least 3 keys to
reconstruct the root key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

Proceed to unsealing.

```bash
vault operator unseal
```

Once done, login with the "tenant" token.

```bash
vault login
```

You can now use the vault like a classic Hashicorp Vault.

Write a secret:

```bash
vault kv put -mount=secret creds passcode=my-long-passcode
```

```
== Secret Path ==
secret/data/creds

==== Metadata ====
Key         Value
---         -----
passcode    my-long-passcode
```

```bash
vault kv get -mount=secret creds
```

```
== Secret Path ==
secret/data/creds

========== Metadata ==========
Key                      Value
---                      -----
cas_required             false
creation_time            2024-11-05T08:34:12+00:00
custom_metadata          map[]
deleted_version_after    n/a
max_version              1
min_version              1
update_time              2024-11-05T08:34:12+00:00
version                  1

====== Data ======
Key         Value
---         -----
passcode    my-long-passcode
```

The documentation of the KV API is [here](https://developer.hashicorp.com/vault/docs/secrets/transit#usage)

The documentation of the Transit API is [there](https://developer.hashicorp.com/vault/docs/secrets/transit#usage)

To seal at tenant level.

Login as "tenant-operator".

```bash
vault operator seal
```

```
Success! Vault is sealed.
```

To seal the node.

Login as "cluster-operator"

```bash
vault operator seal
```

```
Success! Vault is sealed.
```