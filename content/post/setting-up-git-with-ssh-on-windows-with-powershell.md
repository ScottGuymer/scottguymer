+++
author = "Scott Guymer"
date = "2021-10-08T10:12:21Z"
tags = ["Git", "SSH", "Windows"]
title = "Setting up Git with SSH on Windows with PowerShell"
+++

When you activate 2FA on your GitHub account using HTTPS becomes a little more difficult because you can't simply use your password anymore.

The simple solution is to use a [PAT token](https://github.com/settings/tokens) as your password when prompted from the command line.

But managing these tokens can be a bit of pain.

You can also clone using SSH. For our Mac and Linux friends, we know it's easy, it should be on windows, but it isn't yet...

Firstly, you do not need any "extra" software or plugins like PuTTY anymore, in Windows 10 there is a native SSH agent that we can use.

Let's use PowerShell...

Sometimes the SSH client isn't installed, [we can install it as a Windows capability](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)

    $ Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0

Sometimes the SSH client isn't turned on. Let's set it to automatic start and start it now

    $ Get-Service -Name ssh-agent | Set-Service -StartupType Automatic
    $ Start-Service ssh-agent

At this point you will have a working SSH client with no keys. Let's test that out. This should connect but you will get an authentication error.

    $ ssh git@github.com

Now we need a new key so we can tell GitHub who we are and prove it

    $ ssh-keygen #change name/password as prompted

Now we tell the SSH client about the key we just created so it can use it to connect to things.

    $ ssh-add C:\Users\<your_user>\.ssh\id_rsa
    Identity added: C:\Users\<your_user>\.ssh\id_rsa (C:\Users\<your_user>\.ssh\id_rsa)

Next, we need to tell GitHub about the key so let's copy the public key to our clipboard

    $ cat ~/.ssh/id_rsa.pub | clip

[Then you need to add the public key to your account using the instructions here](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)

**!Important!** If you are part of a GitHub org that uses SSO you need make sure you click Enable SSO and authorize for your org.

The last step is to tell your local git about your new SSH client and make sure it knows where to find it. You need to set an env var for this. We can do this temporarily like this

    $env:GIT_SSH = "C:\Windows\System32\OpenSSH\ssh.exe"

I would suggest adding this to your PowerShell profile or to the system environment variables using the windows dialogue.

Now you should be all set to clone your repos over SSH

    git clone git@github.com:philips-internal/my-repo.git

When copying the link to your repo make sure you select the SSH tab in the clone dialogue.

![](/uploads/2021/10/08/git_clone.png)

I have open sourced my own PowerShell Profile which sets up some of these things and makes sure that they will always work for me. [You can get some inspiration here.](https://github.com/ScottGuymer/powershell-profile/)
