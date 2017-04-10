---
layout: post
title: HashiCorp Vault mounting AppRole
---
Quick post on how to mount approle, tune it and add a role.
I like and use python so code is going to be in python... even API REQUESTS

### pre-requsit:
Running Vault (we have a `-dev` instance running locally `http://127.0.0.1:8200`)


Mounting backend:

```python
url = "http://127.0.0.1:8200/v1/sys/auth/approle"
payload = "{
        "type": "approle",
        "description": "approle mounted at /approle ",
        "config": {
            "default_lease_ttl": "2h",
            "max_lease_ttl": "24h"
        }
    }"
headers = {
    'x-vault-token': "<TOKEN>"
    }
requests.request("POST", url, data=payload, headers=headers)
```

Now we have approle mounted, all calles we want to make to approle will start with - `http://127.0.0.1/v1/auth/approle/`

Next, we need to create an approle role - we are going to call it `service-access`, it will be accasable only to instances from 10.10.10.0/24 subnet and the full authentication process will have to be completed in 60 seconds. There are more options and restriction that can be added to a role, more details can be found at [vault's approle backend website] 

```python
url = "http://127.0.0.1:8200/v1/auth/approle/role/all-write"
payload = "{
	"role_name": "api-write",
	"bound_cidr_list": "10.10.10.0/24",
	"policies": "service-access-acl",
	"secret_id_ttl": "60s"
    }"
headers = {
    'x-vault-token': "..."
    }
requests.request("POST", url, data=payload, headers=headers)
```