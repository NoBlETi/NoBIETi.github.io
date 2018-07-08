---
layout: post
title: Multiple git hub accounts on one PC
---
## PROBLEM

I recently had a problem with getting multiple GitHub accounts to work on one machine. For some reason, most guides are for linux/mac. Getting all this to work on mac was **easy**, everything works as expected. 

So, let's say I have my private account and now I'm trying to add my work account. Github obviously doesn't allow us to reuse the same key (+1 for security)...

## SOLUTION

Creating a key:

1. Go to your .ssh directory <br/>
    `cd ~/.ssh`
2. Generate a new key (**Change the default name or you will override your other key!**): <br/>
    `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`
3. Add the public part (ends with .pub) to GitHub (account settings -> SSH and GPG Keys)
4. In your .ssh folder create a `config` file where you specify both your accounts:

```bash
# Main account
Host github.com
        HostName github.com
        User git
        IdentityFile ~/.ssh/id_rsa

# Work Account
Host github.com-work
        HostName github.com
        User git
        IdentityFile ~/.ssh/id_rsa_work
```

#### Windows Users
If this config is giving you an error, use absolute path to get to your key, for example: <br />
`c:/Users/Alfred/.ssh/id_rsa_work`

The **Host** part is very important here, as that's how git will know which key to use. You can use any value there, but where you change it, remember to to update the config file in the repository. <br />
`gitproject/.git/config`

Normal:

```bash
[remote "origin"]
	url = git@github.com:user/gitproject.github.io.git
	fetch = +refs/heads/*:refs/remotes/origin/*
```

New:

```bash
[remote "origin"]
	url = git@github.com-work:user/gitproject.github.io.git
	fetch = +refs/heads/*:refs/remotes/origin/*
```

Now git knows which key to use, based on the repository and the `host` name.