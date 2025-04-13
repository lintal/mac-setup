# mac-setup
Just documenting the steps I have gone through to configure my Mac / Development Environment

## Basics

To start with, let's get some basics installed:

### Homebrew

Install docs: [https://brew.sh/](https://brew.sh/)

```bash
# To install:
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# To configure loading brew in to your bash path (as instructed by the Homebrew installer:
echo >> ~/.zprofile
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```
And now to install a few key packages using Homebrew:

```bash
brew install visual-studio-code 
```
### NodeJS

I'm going to be using NVM to manage my versions of node on my machine. You can install this using a command **LIKE** the following (noting you will want to change the version numbers as they increment in the command:
Instructions modified from [nodejs.org](https://nodejs.org/en/download)

```bash
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash

# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"

# Download and install Node.js:
nvm install --lts
```

### Git

```bash
# To instasll:
brew install git

# Setup some alias commands:
git config --global alias.tree "log --graph --decorate --pretty=oneline --abbrev-commit"
```

## Security Key (Yubikey)
I use my security keys (yes -  you should have more than one!) in my workflows extensively. These include basic OTP MFA, as well as more complex usecases including PGP signing GIT commits, SSH login using GPG keys etc. There are also some tools available from Yubico to install to help out with some tasks related to the keys.

### Yubico Tools

```bash
brew install yubico-yubikey-manager ykman
```

Note, there is also the `yubico-authenticator` package that you can install using brew, however our IT department makes this available through other means.

### Import Existing Keys
As I already have PGP keys on my various Yubikeys, I want to import references to them into my installed GPG Keychain. Simply plug each one in turn and run the following for each:

```bash
gpg --card-edit
# Key info is displayed
gpg/card> quit
```

## PKCS#11 Certificate
I also have my DevCert installed on my Yubikey. To enable access to this from your browser, you need to do the following:

Source Docs:
https://github.com/OpenSC/OpenSC/wiki/macOS-Quick-Start

```bash
# This works best with Firefox, so let's install that:
brew install firefox

# To install PKCS#11 capability:
brew install opensc --cask

# Test it's working (may require reload of the shell environment):
pkcs11-tool --login --test
```

Then following the following docs to get this working with Firefox:
https://github.com/OpenSC/Wiki/blob/master/Installing-OpenSC-PKCS11-Module-in-Firefox%2C-Step-by-Step.md

### Passwordless Mac Login
With PKCS#11, you can also enable smartcard login for your Mac. Note, this isn't actually dependent on OpenSC installed above, but instead uses the mac built-in `sc_auth` command.

Source docs: https://developers.yubico.com/PIV/Guides/Smart_card-only_authentication_on_macOS.html

```bash
sc_auth identities
# From the output of the above command, grab the hash of the certificate stored on your card

sc_auth pair -h {HASH} -u {MAC_USERNAME}

# Check it's now paired with:
sc_auth list
```

If you keep getting prompted for your Mac password as well from the login screen, just open Keychain Access and unlock it from there. Next time you lock your Mac, you should only get prompted for your Yubikey PIV PIN.

## SSH
I use my PGP keys stored on my Yubikeys to authenticate my SSH connections. We need to setup a few things in connection with this.

Firstly, the BBC ssh tunneling config needs to be configured along with the proxy capability to allow access from On-Reith.

```
# ~/.ssh/config:
Host *,??-*-?
  User YOUR_COSMOS_USERNAME
  IdentityFile ~/.ssh/id_rsa
  ProxyCommand >&1; h="%h"; r=${h##*,}; i=${h%%,*}; v=$(($(cut -d. -f2 <<<$i) / 32)); exec ssh -q -p 22000 bastion-tunnel@$v.access.$r.cloud.bbc.co.uk nc $i %p
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
 
Host github.com
 HostName github.com
  User YOUR_GITHUB_USERNAME
  IdentityFile ~/.ssh/id_rsa
  ProxyCommand nc -x socks-gw.reith.bbc.co.uk:1085 %h %p
```

We then need to configure GnuPG to act as an agent to SSH:

```
# ~/.gnupg/gpg-agent.conf
pinentry-program /usr/local/MacGPG2/libexec/pinentry-mac.app/Contents/MacOS/pinentry-mac
enable-ssh-support
default-cache-ttl 600
max-cache-ttl 7200
```

```
# Add the following to ~/.zprofile
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

You should then be able to SSH out to a Cosmos instance. If you need to export your ssh public key, run `ssh-add -L`.

Source docs:
- https://developers.yubico.com/PIV/Guides/Securing_SSH_with_OpenPGP_or_PIV.html
- https://florin.myip.org/blog/easy-multifactor-authentication-ssh-using-yubikey-neo-tokens
- https://gist.github.com/ixdy/6fdd1ecea5d17479a6b4dab4fe1c17eb
