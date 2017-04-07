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




[approleDiagram]: {{ site.baseurl }}public/img/vault-approle-diagram.png