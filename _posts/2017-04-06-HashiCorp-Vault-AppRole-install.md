---
layout: post
title: HashiCorp Vault mounting AppRole .
---
Quick post on how to mount approle, tune it and add a role.
I like and use python so code is going to be in python... even API REQUESTS

### pre-requsit:
Running Vault (we have a `-dev` instance running locally `http://127.0.0.1:8200`)


Mounting backends is easy:

```Python
url = "http://127.0.0.1:8200/v1/sys/auth/approle"
payload = {
        "type": "approle",
        "description": "approle mounted at /approle ",
        "config": {
            "default_lease_ttl": "2h",
            "max_lease_ttl": "24h"
        }
    }
headers = {
    'x-vault-token': "<TOKEN>"
    }
requests.request("POST", url, data=payload, headers=headers)
```

Now we have approle mounted, all calles we want to make to approle will start with - `http://127.0.0.1/v1/auth/approle/`

Now, we need to create an approle role - we are going to call it `service-access`, so if a microservice needs access to vault it should use this role to get credencials and log in. code:

```python
url = "https://vault.demo.mtkip.com:8200/v1/auth/approle/role/all-write"
payload = "{
	"role_name": "api-write",
	"bound_cidr_list": "10.10.20.0/24",
	"policies": "service-access-acl",
	"secret_id_ttl": "60s"
    }"
headers = {
    'x-vault-token': "..."
    }
requests.request("POST", url, data=payload, headers=headers)
```

The above request will create a role that, when used, will allow access as defined in the policy `service-access-acl`