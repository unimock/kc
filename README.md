# kc  ... kvm controller 

## Description

kc environment contains a couple of bash scripts to install and manage kvm  
from a central host and directory (master).

It only works as a wrapper for libvirt for convenient usage.

## Requirements

 * ubuntu-server 16.04
 * bare metal with VT support

## Installation

```
git clone https://github.com/unimock/kc.git /opt/kc
/opt/dc/bin/kc-install init     # initialize kc environment
/opt/dc/bin/kc-install packages # install docker-de (edge)
. /etc/profile                  # or re-login
```
