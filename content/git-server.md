+++
title = "Self hosted Git server on OpenBSD"
date = 2021-07-14
+++

# Self hosted Git server on OpenBSD

This post shows how to set up a git server on OpenBSD, but it should work on Linux or other *Nixes as well. User management will be done via an authorized keys file. Keys from persons which should have access to the git repos will be added there. To add a new repo somebody has to ssh into the server and initialize a new bare git repo.

<br></br>

## Install necessary packages

```sh
$ doas pkg_add git
```

Read the readme of the package for additional information the mantainer thinks could be helpful after installation. Readmes are located in /usr/local/share/doc/pkg-readmes. Install complementary packages if needed.

## Add user/SSH configuration

Add a new user.

```sh
$ doas adduser git
```

Prepare ssh folders/files.

```sh
$ mkdir /home/git/.ssh
$ chmod 700 .ssh
$ touch .ssh/authorized_keys
$ chmod 600 .ssh/authorized_keys
```

Add required ssh keys to the authorized_keys file. Next lock down the authorized_keys file. To do that use the restrict option. Restrict options includes amongst other things the following options: "no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty".
To also restrict user supplied commands, set the command option; command="".

Config should look something like this.

```cfg
restrict,command="" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIfozwE/mnCuklL9lVpZRyGS65aVHi6Ki0wDPG4hJtI2
```

Change the login shell of the git user to git-shell. It severely limits available shell commands. Basically only git functionality is enabled. Read `man git-shell` for more information.

This can be done with "chsh".

```sh
$ doas chsh git
```

## Set up git 

Initialize a bare repo on the server.

```sh
$ git init --bare
```

Change default branch name if desired.

```sh
$ git branch -m main
```

### (Optional) Add a repo/Change existing remote repository to new remote repository

Show current remote:

```sh
$ git remote -v
```
Change remote origin if it already exists:

```sh
$ git remote set-url origin git@domain:/git-repository/new-repository.git
```

Add remote origin if it does not exist yet:

```sh
$ git remote add origin git@domain:/git-repository/new-repository.git
```

Check if the remote repo can be read.

```sh
$ git ls-remote
```

Finally push the local repo to the remote repo.

```sh
$ git push
```

<br></br>

# Recommended/Further reading

Most info in this blogpost is from the: [Pro Git Book](https://git-scm.com/book/en/v2/Git-on-the-Server-Setting-Up-the-Server)

`man git-shell`

Infos about authorized_keys options(under the "authorized_keys file format" section):

`man sshd`
