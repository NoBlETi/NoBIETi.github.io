---
layout: post
title: HashiCorp Vault AppRole - authentication.
---
I will try to explain how does the AppRole authentication backend works, and why you should use it?

## Chicken and egg problem with passwords
So, Vault will store all our secrets for us, all we need is just authenticate to vault to get all the secrets we need! Easy, put Vault's authentication token as an environment variable on the machine, or save it in the config file... 
### **RIGHT?!**
### **NOPE!**

This way we are just adding more complexity, more code lines, more API calls... for no extra security. This is where Vault's AppRole authentication backend comes to the rescue! It allows us to authenticate to vault in a semi-secured way... It's not totally secured, but it is much better than saving secrets in config files or environment variables.

Below diagram briefly shows how the full process works... one pre-requisit for this is some kind of orchestration tool...

![alt text][approleDiagram]


With [approle backed set up] done, we can now move to the service deployment process. The diagram shows ansable, but here we are just going to use a simple python script to replicate the same functionality. This approuch should not be used in production environments, as it is insecure!

First we need to request Role-id (RID) and Wrapped Secret-id (WSID). We will look into wsid later...

To get RID and WSID we need to request it from vault using `service-access` we create earlier. We need to make two calls:

Get RID:
```python
url = "http://127.0.0.1:8200/v1/auth/approle/role/all-write/role-id"
headers = {
    'x-vault-token': "<Ansible's Token>"
    }
requests.request("GET", url, headers=headers)
```
Get WSID:
```python
url = "http://127.0.0.1:8200/v1/auth/approle/role/all-write/secret-id"
headers = {
    'x-vault-token': "<Ansible's Token>",
    'x-vault-wrap-ttl': "600"
    }
requests.request("POST", url, data=payload, headers=headers)
```

Now we have RID and WSID, we need to pass thouse two variables to the container, so out service can pick it up to start the authenticatio process. 

How is this better than saving passwords as environment variables? Well... WSID can be used only **once** to fetch the proper secret-id (SID), this way our envirnment variables are useless after they have been used once  

The below python script, will fetch RID and WSID and save them as environment variables.
```python
import requests
import os
import argparse

CORE_VAULT_URL = "http://127.0.0.1:8200/v1"
PATH = os.path.dirname(os.path.realpath(__file__))


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("-t", "--token", help="vault token")
    parser.add_argument("-r", "--role", help="[OPTIONAL] vault role", default="read-only")
    args = parser.parse_args()
    token = args.token or input("Token? \n")
    role = args.role
    rid = get_approle_role_id(role, token)
    wsid = get_wrapper_token(role, token)
    with open(PATH + "/vault.sh", "w") as env_var:
        env_var.write("export APPROLE_RID=" + rid + "\n")
        env_var.write("export APPROLE_WSID=" + wsid)
    os.system("sudo mv " + PATH + "/vault.sh /etc/profile.d/vault.sh")
    print("File created")


def get_approle_role_id(role_name, token):
    headers = {"X-Vault-Token": token}
    path = CORE_VAULT_URL + "/auth/approle/role/" + role_name + "/role-id"
    r = requests.get(path, headers=headers, verify=False)
    role_id = r.json().get("data").get("role_id")
    return role_id


def get_wrapper_token(role_name, token):
    headers = {"X-Vault-Token": token,
               "X-Vault-Wrap-TTL": "600s"}
    path = CORE_VAULT_URL + "/auth/approle/role/" + str(role_name) + "/secret-id"
    r = requests.post(path, headers=headers, verify=False)
    wrapper_token = r.json().get("wrap_info").get("token")
    return wrapper_token


main()
```

With the above, we should have everything ready to start our service and use APPROLE_WSID and APPROLE_RID to log in to vault.

At the moment WSID is a wrapped secret-id, which uses vault's [cubbyhole] to encapsulate the secret-id. This ensures that Ansible (or whatever we use for orchestration) doesn't have direct access to the user-id and the secret-id.

Now, we need to unwrap the WSID (wrapped secret-id) to get the SID (secret-id). To do this we make the below call setting `x-vault-token` header to the `WSID`, this will returt out SID that we will use next to get the authentication token for the service.

```python
url = "http://127.0.0.1:8200/v1/sys/wrapping/unwrap"
payload = ""
headers = {
    'x-vault-token': "<WSID>",
    }
requests.request("POST", url, data=payload, headers=headers)
```
Secret-id should be retuned by the above request. Now we can authenticate to Vault and get the token required to make requests to the `secrets/` backend

Using UID and SID we can now authenticate, and we should get a token in response...

```python

url = "https://vault.demo.mtkip.com:8200/v1/auth/approle/login"
payload = "{"role_id": "<RID>",
            "secret_id": "<SID>"}"
headers = {
    'content-type': "application/json",
    'cache-control': "no-cache",
    'postman-token': "1264d045-cc1b-1f46-d405-f00a72c9f47d"
    }
requests.request("POST", url, data=payload, headers=headers)
```
Example Response:
```json
{
  "request_id": "8a6d489b-badd-45dd-c804-cd0ad85fcbd0",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": null,
  "wrap_info": null,
  "warnings": null,
  "auth": {
    "client_token": "e40cd856-dc96-c763-4a8f-90e193e57933",
    "accessor": "432af15f-d643-d256-30ef-7dc007c5abb3",
    "policies": [
      "service-access-acl"
    ],
    "metadata": {},
    "lease_duration": 86400,
    "renewable": true
  }
}
```

Now we can use the `cliet_token` to query any (as long as we have permissions) of the secrets backend (generic, aws, PKI) to request secrets.
Our test server runs over http, but obviously anything that is not local should use https!



[vault's approle backend website]: https://www.vaultproject.io/docs/auth/approle.html
[cubbyhole]: https://www.vaultproject.io/docs/secrets/cubbyhole/
[approle backed set up]: {{ site.baseurl }}2017/04/06/HashiCorp-Vault-AppRole-install/
[approleDiagram]: {{ site.baseurl }}public/img/vault-approle-diagram.png