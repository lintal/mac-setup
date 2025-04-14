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

There are some global packages I like having installed as well - so installing them now:
```bash
npm i -g tsx
```

### OhMyZsh
I use this for doing some config of my shell environment. See the [docs](https://github.com/ohmyzsh/ohmyzsh). To install:

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

My list of plugins used:
```
# ~/.zshrc
plugins=(git npm node sudo)
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
I also have my DevCert installed on my Yubikey. To enable access to this from your browser, you need to do the following. Note: I have installed FireFox using the BBC Software Catalogue.

Source Docs:
https://github.com/OpenSC/OpenSC/wiki/macOS-Quick-Start

```bash
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

## Git

I use my PGP key on my Yubikey(s) to sign all my git commits, and I want this configured by default. To do this:

Some basic setup to start:

```bash
# To install:
brew install git

# Setup basic user details:
git config --global user.email "{YOUR_EMAIL_ADDRESS}"
git config --global user.name "{YOUR_NAME}"

# Setup some alias commands:
git config --global alias.tree "log --graph --decorate --pretty=oneline --abbrev-commit"

# Configure auto-signing commits:
git config --global commit.gpgsign true
git config --global tag.gpgSign true
```

We may also want to add Github's SSH key for verifying commits made through the web console. You can get the certificate from here: https://github.com/web-flow.gpg, then add this to your GPG-Keychain.

Given I have multiple Yubikeys, I also want the ability to easily switch between them where required. To do this, I have configured a few bash alias commands:

```
# Add to ~/.zprofile:
alias git-sign-bbc="git config --global user.signingkey {SUBKEY1_HASH}!"
alias git-sign-personal="git config --global user.signingkey {SUBKEY2_HASH}!"
```

We can then activate one of the keys as follows (in this case, using my work issued Yubikey):
```bash
git-sign-bbc
```

## AWS CLI

Install the AWS CLI as per the published instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

Set a couple of basic config values:

```
# ~/.aws/config
[default]
region = eu-west-1
output = json
```

### GoBBC
Tool used to authenticate AWS access to BBC accounts - see [here](https://github.com/bbc/gobbc).

Note, the version of XCode Command-Line-Tools needs to be sufficiently modern. To check, you can run the following:

```bash
brew doctor
```

This may give you instruction to upgrade this, either using the Mac `System Settings` app, or manual removal & re-installation commands.

**Certificates**

I store my DevCert outside the normal area where GoBBC (and other Sandbox structured projects) look for certificates for mTLS. Therefore I link them in to the appropriate places as follows:

```bash
sudo mkdir -p /etc/pki/tls/certs
sudo mkdir -p /etc/pki/tls/private

sudo ln -s ~/Certificates/devcerts/2024/lintoa01.crt /etc/pki/tls/certs/client.crt
sudo ln -s ~/Certificates/devcerts/2024/lintoa01-unsafe.key /etc/pki/tls/private/client.key
sudo chmod 600 /etc/pki/tls/private/client.key

gobbc doctor
# Should hopefully give you a clean result.
```

**Account Aliases**

You can setup a number of account aliases for ease of reference using:

```bash
gobbc set-aws-account-alias -accountId <ACCOUNT_ID> -alias <ALIAS>
```

You can get the list of configured aliases using:
```bash
gobbc list-aws-account-aliases
```

**Command Shortcuts**
I like having a few shortcut commands to ease working with GoBBC. These are:


| Command | Equivalent | Description |
|-|-| - |
| `gbc` | `gobbc aws-console -a` | Open AWS console in the browser for a particular account. |
|`gbt`| `gobbc aws-credentials -a` | GoBBC terminal - for use with CLI commands. |


To configure:
```
# Add to ~/.zprofile
alias gbc="gobbc aws-console -a"
alias gbt="gobbc aws-credentials -a"
```

## Time Machine Backups

I use Time Machine to run regular backups of various parts of my system. I have an extensive list of excluded paths to reduce the amount of unnecessary data stored, taking the view that a replacement BBC Laptop will come pre-installed with a bunch of stuff already, and that I should be re-installing software through the Service-Catalogue rather than recovering from Time-Machine backup. Also you typically aren't given the option to recover from TimeMachine backup anyway on a BBC Mac.

My list of excluded paths (not exhaustive): 
```
/Applications/Library
/Users/Shared
/Users/localadmin
~/Applications
~/BBC Dropbox
~/Library/CloudStorage/OneDrive-BBC
~/Movies
~/Music
~/Pictures
```

Given I have various software development projects, I also want to prevent dependencies being unnecessarily backed-up (you can always recover by running `npm i` or similar right).
To do this, I followed the instructions for [this package](https://github.com/stevegrunwell/asimov).

```bash
brew install asimov
sudo brew services start asimov
```