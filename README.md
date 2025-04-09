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
