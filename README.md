# Naumachia
[![Discord](https://img.shields.io/discord/404881131058626570.svg)](https://discord.gg/gH9ZgeT) 

If you are interested in using or contributing to this project let me (nategraf) know! It will grow based on input from those who care to give it.

### A multi-tenant network sandbox for security challenges

**The ambition of [Naumachia](https://en.wikipedia.org/wiki/Naumachia)** is to enable the deployment of multihost interactive exploit challenges for fun and non-profit. The origonal inspiration was to enable network exploit challenges. The main target is providing fun and challenging exercises for CTFs, with the secondary goal of being used in a classroom setting, and more applications to come.

It **provides challengers with layer two (LAN) access to their own instance of a vitualized network and target machines** as defined by the challenge specification. When a user logs in, a cluster is brought online just for them, and when they log out it is torn down. Optionally, any data may be persisted.

**Naumachia is primarly built on** Docker Compose, which manages the challenege environements and provides the language through which these environments are defined, and OpenVPN, which provide authentication and layer two access accross the Internet. A Python applciation backed by a Redis datastore exists to couple these systems; coordinating the cluster creation/teardown, creation of user credentials, orchistration of platform networking, and various other functions which turn these powerful tools into Naumachia. 

## Deployment

#### Initial Setup
1. Install [Docker](https://docs.docker.com/engine/installation/) and [Docker Compose](https://docs.docker.com/compose/install/)
2. Clone Naumachia onto a Linux server (developed and tested with Ubuntu)
3. Install requirements.txt into Python3 (i.e. `pip3 install -r requirements.txt`)

#### Create a Challenge
1. Write a [`docker-compose.yml` template](https://docs.docker.com/compose/compose-file/) and put it and any associated files in directory within the `challenges` directory. See `challenges/example` for some guidence
2. Modify `config.yml` to include your challenge
3. Run `configure.py` to generate the `docker-compose.yml` file from a Jinja2 template, OpenVPN config files, and PKI

WARNING: When writing the compose file, do not use bind volumes (i.e. mount local directories to the container). It will not mount properly when started from the cluster-manager which handles creating and stopping challenge instances. No workaround is provided as it is the eventual intention to move toward a scalable model when you cannot control (or care about) where your challenges are deployed. See [moby/moby#28124](https://github.com/moby/moby/issues/28124) for techinal discussion of the underlying reason

#### Distriute Access Credentials
In order to log into the VPN tunnel and access Naumachia a client needs the correct configuration, and a registered certificate. These two are bundled in an OpenVPN client config file

To generate a client config for your challenge either:
* Use the registrar CLI
  * Ex: `./registrar/registrar.py mitm add alice` will create certs for alice and `./registrar/registar.py mitm get alice` with output the configuration needed for alice to connect to the 'mitm' challenege
* Use the registrar server
  * Add `registrar: true` to the challenge config
    * NOTE: When using the registrar server ensure it's inaccesible by the public. The registrar server is unathenticated and can be trivialy used to issue a DOS attack or worse to your Naumachia instance
  * Issue REST API calls to registrar server to manage certificates and retrieve configuration files
    * /\<chal\>/list?cn=\<cn\> (cn optional) : List all registered certificates or certificates for a specific cn
    * /\<chal\>/add?cn=\<cn\> : Create a new certificate with the specified common name (cn)
    * /\<chal\>/revoke?cn=\<cn\> : Revoke the certificate with the specified common name (cn)
    * /\<chal\>/remove?cn=\<cn\> : Remove the certificate with the specified common name (cn)
    * /\<chal\>/get?cn=\<cn\> : Get the OpenVPN configuration file for the user with specified common name (cn)

#### Run it!
To run Naumachia simply bring up the enviroment with the [Docker Compose CLI](https://docs.docker.com/compose/reference/overview/) (e.g. `docker-compose up -d`)

#### On Each Server Reboot
For lack of a better method there are two steps that will need to be completed on intial installation and every time Naumachia will be run after reboot. It is my intention to obviate the need for these steps as development continues.
1. Disable bridge-nf enforcement of iptables rules by running `disable-bridge-nf-iptables.sh`. This is needed to allow unrestricted access within the sandbox network Naumachia creates for each user. By default, Docker blocks certain traffic from connections into a Docker configured bridge which were not configured through Docker Networks. *This does not effect layer 3 restrictions imposed by iptables*
2. Run an arbitrary docker container on the host network (e.g. `docker run --rm --net=host alpine /bin/true`) This create a link in the `/var/run/docker/netns` folder called `default` which allows access to the host network namespace. This will be added to the cluster-manager container at runtime to allow it to modify the bridges generated by Docker on the host.

## How to Create a Challenge

Challenges in Naumachia are defined by a docker-compose.yml file and the rousources it launches

Consider the example provided as [challenges/example/docker-compose.yml](https://github.com/nategraf/Naumachia/blob/master/challenges/example/docker-compose.yml)

```yaml
version: '2.1'

# The file defines the configuration for simple Nauachia challenge
# where a sucessful man-in-the-middle (MTIM) attack
# (such as ARP poisoning) provides a solution

# If you are unfamiliar with docker-compose this might be helpful:
# * https://docs.docker.com/compose/
# * https://docs.docker.com/compose/compose-file/
#
# But the gist is that the services block below specifies two containers,
# which act as parties in a vulnerable communication

services:
    bob:
        build: ./bob
        image: naumachia/example/bob
        environment:
            - CTF_FLAG=FOOBAR

    alice:
        build: ./alice
        image: naumachia/example/alice
        depends_on:
            - bob
        environment:
            - CTF_FLAG=FOOBAR

# To avoid users from using this challenge as a personal VPN
# gateway to the internet, it is important to always specify the
# default network as internal (i.e. not connected to the internet)

networks:
    default:
        internal: true
```

This example defines a challenge which will feature two containers networked together through the default network, which has been modified to be inaccessible from the external world (as should be down for all challenges unless you have a good reason not to)

The code defining alice's behavior is in the folder [./alice](https://github.com/nategraf/Naumachia/tree/master/challenges/example/alice) where you will find a Dockerfile defining the containers properties, and a python script which will be run as defined in the Dockerfile (This python script send a message to bob "asking" if she hass the right flag repeatedly)

Simmilarly bob's definition is in [./bob](https://github.com/nategraf/Naumachia/tree/master/challenges/example/bob) which is a simple server listening for the flag alice sends and responding yes or no if it is correct

The user will log in to the VPN tunnel with a config provided by the registrar, and execute an attack to intecept the traffic and obtain the flag

## Connection Instructions

Clients can use any OS supported by OpenVPN, although Linux is recomended for it's large number of hacking tools

To connect each user will need to:
0. Install OpenVPN
  * Can be found on most package managers (e.g. apt, brew, choco) or [downloaded](https://openvpn.net/index.php/open-source/downloads.html)
  * Ensure the TAP driver is installed
1. Obtain a configuration file with certificates for the challenge they want to connect from the host
2. Launch openvpn with the correct configuration
  * CLI: `openvpn --config`
  * Windows GUI: Place the config file in `%HOMEPATH%\OpenVPN\config` and right-click the VPN icon on the status bar, then select the config you want

If using the CLI on Linux, or MacOS you may still need to more steps
3. Bring up the new TAP network interface
  * Ex: `ip link set tap0 up`
4. Obtain an IP address by DHCP (if DHCP is enabled for the challenge)
  * Ex: `udhcpc -i tap0``
  * Ex: `dhclient tap0``
