+++
title = "CalDav/CardDav server with Baïkal"
date = 2021-05-28
+++

certbot on openbsd

certbot with wildcard and ovh necessary to isntall yourself

pipx is good as it installs packages and their dependencies in an isolated environment

build with pipx(installs and isolates applications and their dependencies for pypi packages), needs rust(install from base), needs setting of openssl env variable



make this into a blogpost
show error msg rust, openssl missing

export OPENSSL_INCLUDE_DIR=/usr
OpenBSD$ export OPENSSL_LIB_DIR=/usr/lib                                            
OpenBSD$ export OPENSSL_DIR=/usr/bin     
OpenBSD$ pipx install certbot

should be fixed upstream in the version after version 42.0.8

https://github.com/pyca/cryptography/issues/9144


OpenBSD$ pipx install certbot                                                       
  installed package certbot 2.11.0, installed using Python 3.10.14
  These apps are now globally available
    - certbot
⚠️  Note: '/home/joe/.local/bin' is not on your PATH environment variable. These apps will not be globally accessible until your
    PATH is updated. Run `pipx ensurepath` to automatically add it, or manually modify your PATH in your shell's config file (i.e.
    ~/.bashrc).
done! ✨ 🌟 ✨
you have mail in /var

OpenBSD$ pipx ensurepath
Success! Added /home/joe/.local/bin to the PATH environment variable.

Consider adding shell completions for pipx. Run 'pipx completions' for instructions.

You will need to open a new terminal or re-login for the PATH changes to take effect.

Otherwise pipx is ready to go! ✨ 🌟 ✨

ovh plugin installieren
pipx install certbot-dns-ovh

confirm with certbot plugins

check if pipx is better or simply just use pip

python3 -m venv path
activate venv:
. /path/bin/activate

pip install foo

certbot in venv has the ovh plugin but not externally

fertig fixen, cronjob fixen, error mit mail bei error bei renewal einrichten

fix postgresql for miniflux, also upgrade to new major version

pipx list
