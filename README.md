# Debian Packaging

[![made-with-Markdown](https://img.shields.io/badge/Made%20with-Markdown-1f425f.svg)](http://commonmark.org) [![Open Source Love](https://badges.frapsoft.com/os/v1/open-source.png?v=103)]() 

## Table of Content

----------

* [Introduction](#introduction)
* [Installing LXC](#installing-lxc)
    * [Configuring LXC in Arch Only ](#configuring-lxc-in-arch)
    * [Installing Debain-sid in LXC](#installing-debain-sid-in-lxc)
* [Installing Docker](#installing-docker)
    * [Docker Prerequisites](#docker-prerequisites)
    * [Installing Debian-Sid in Docker](#installing-debian-sid-in-docker)
* [Installing Packaging Tools](#installing-packaging-tools)
* [Practical Session 1](#practical-session-1)
* [Practical Session 2](#practical-session-2)
* [Practical Session 3](#practical-session-3)
* [Practical Session 4](#practical-session-4)
* [Common Issues](#common-issues)
    - [Permission Denied](#permission-denied)
    - [Cannot connect to Docker](#cannot-connect-to-docker)
* [Use the Source, Luke!](#use-the-source.-luke!)
* [License](#license)



## Introduction

So, what is a **package**?

> "A Debian package is a collection of files that allow 
> for applications or libraries to be distributed via the Debian package 
> management system. The aim of packaging is to allow the automation of 
> installing, upgrading, configuring, and removing computer programs for 
> Debian in a consistent manner. "
>
> --- Debian Wiki



For packaging software, it is recommended to use the Debian Sid as it contain the latest software dependencies. If you have a Debian-Sid Machine you are good to go. If not we need to setup a packaging environment. There are many virtualization softwares in Linux like,

- Docker
- LXC 
- VM (Using vagrant or manually installed)

``Docker`` is pretty much good for normal packaging, but face bandwidth issue when using advanced things like ``sbuild``. This is because Docker doesn't have ``systemd`` inside it. 

But, ``LXC`` have ``systemd`` and it is better to use this due to two reasons,

- Less image size (Minimal Debian-sid image)
- We can save a lot of bandwidth (a lot..)



## Installing LXC

For **Debian** _Stretch_ and later and for **Ubuntu** _Xenial_ 16.04 (LTS) & later,

```bash
sudo apt install lxc
```

For **Arch** Linux,

```bash
sudo pacman -S lxc
```



## Configuring LXC( Debian Only )

....

## Configuring LXC ( Arch Only )

Start the ``lxc`` service.

```bash
sudo systemctl start lxc
```

We need to setup a Host network configuration. LXCs supports different virtual network types and it comes with its own NAT bridge (lxcbr0).

To use LXC's NAT bridge, you need to create its config file in ``/etc/default/lxc-net``

```
# Leave USE_LXC_BRIDGE as "true" if you want to use lxcbr0 for your
# containers.  Set to "false" if you'll use virbr0 or another existing
# bridge, or mavlan to your host's NIC.
USE_LXC_BRIDGE="true"

# If you change the LXC_BRIDGE to something other than lxcbr0, then
# you will also need to update your /etc/lxc/default.conf as well as the
# configuration (/var/lib/lxc/<container>/config) for any containers
# already created using the default config to reflect the new bridge
# name.
# If you have the dnsmasq daemon installed, you'll also have to update
# /etc/dnsmasq.d/lxc and restart the system wide dnsmasq daemon.
LXC_BRIDGE="lxcbr0"
LXC_ADDR="10.0.3.1"
LXC_NETMASK="255.255.255.0"
LXC_NETWORK="10.0.3.0/24"
LXC_DHCP_RANGE="10.0.3.2,10.0.3.254"
LXC_DHCP_MAX="253"
# Uncomment the next line if you'd like to use a conf-file for the lxcbr0
# dnsmasq.  For instance, you can use 'dhcp-host=mail1,10.0.3.100' to have
# container 'mail1' always get ip address 10.0.3.100.
#LXC_DHCP_CONFILE=/etc/lxc/dnsmasq.conf

# Uncomment the next line if you want lxcbr0's dnsmasq to resolve the .lxc
# domain.  You can then add "server=/lxc/10.0.3.1' (or your actual $LXC_ADDR)
# to your system dnsmasq configuration file (normally /etc/dnsmasq.conf,
# or /etc/NetworkManager/dnsmasq.d/lxc.conf on systems that use NetworkManager).
# Once these changes are made, restart the lxc-net and network-manager services.
# 'container1.lxc' will then resolve on your host.
#LXC_DOMAIN="lxc"
```

Then we need to modify ``/etc/lxc/default.conf``

```
lxc.net.0.type = veth
lxc.net.0.link = lxcbr0
lxc.net.0.flags = up
lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
```

_P.S: lxc.net.o.hwaddr is the same as above. No need to be confused.._

You may need to install ``dnsmasq``, which is a dependency for ``lxcbr0`` .

Now, start ``lxc-net`` service.

```bash
sudo systemctl start lxc-net
```



## Installing Debain-sid in LXC

Download the Debian-sid image by,

```bash
sudo lxc-create -n debian-sid -t download -- --dist debian --release sid
```

and choose the required architecture (amd64).

Start the lxc image by,

```bash
sudo lxc-start -n debian-sid
```

To setup root password do,

```bash
sudo lxc-attach -n debian-sid
```

and do ``passwd`` and reset the root password. And exit from it.

Then we need to create a normal user. Log in via root by,

```bash
sudo lxc-console -n debian-sid
```

and ``useradd -m -g sudo <username>`` and set the password. Also use this user to login and packaging.

**Most of the packaging is done in the normal user, not in root**



## Installing Docker

For **Debian** _Stretch_ and later and for **Ubuntu** _Xenial_ 16.04 (LTS) & later,

Setup the Docker Repository by ,

```bash
$ sudo apt-get update
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common
```

Add Docker’s official GPG key,

```bash
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```

Verify whether you have done it right,

```bash
$ sudo apt-key fingerprint 0EBFCD88
```

Add the stable repository by,

```bash
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```

Update ``apt`` package index.

```bash
$ sudo apt-get update
```

Install the latest **Docker Community Engine** (Docker CE),

```bash
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Verify that Docker Engine - Community is installed correctly by running the `hello-world`
image. _You may have to start/enable ``docker.service`` before able to run docker._ [1](#cannot-connect-to-docker)

```bash
$ sudo docker run hello-world
```

And you are ready to move to next section..



For **Arch** Linux,

```bash
sudo pacman -S docker
```

And you know what to do :)



## Docker Prerequisites

We will learn about the basic commands of docker..

|       Docker Command        |                    man                    |
| :-------------------------: | :---------------------------------------: |
|        docker images        | Shows the images pulled/builded in docker |
|        docker ps -a         |       Shows the all the containers        |
|    docker rmi [image-id]    |          Delete the resp. image           |
|  docker pull [repository]   |     Pulls the image from a repository     |
| docker start [containerid]  |            Starts a container             |
| docker attach [containerid] |     Attach the container to terminal      |

*P.S : Default Docker may not work without sudo. For this you may need to add your user to the docker group.* [2](#permission-denied)



## Installing Debian-Sid in Docker

The docker image using here is from [FSCI](https://fsci.org.in/) repositories. We had a [new one](https://gitlab.com/fsci/resources/pipelines) build  especially for this event. Thanks to [@balasankarc](https://balasankarc.in/) for the effort.😘

For pulling the image,

```bash
$ docker pull registry.gitlab.com/fsci/resources/debian-dev:latest
```

We need to start a container using this image.

```bash
$ docker run --name debian-dev -it registry.gitlab.com/fsci/resources/debian-dev:latest bash
```

**For packaging some packages, we may need to run the docker in a privileged mode. Also we might need to move files from the host to container and vice-versa.** To do so,

```bash
sudo docker run --privileged=true -v "$(pwd)"/workarea:/root -w /root -it registry.gitlab.com/fsci/resources/debian-dev bash
```

While you are inside the container do update to get the latest packages.

*P.S: Default sudo password for the user developer is developer itself.*

```bash
$ sudo apt-get update && sudo apt-get dist-upgrade
```

Exit the docker container by using ``Ctrl + D`` or ``$ exit``.

To start and attach the previously made container use,

```bash
$ sudo docker start debian-dev
$ sudo docker attach debian-dev
```

For packaging workshops, pulling from upstream registry use quite some bandwidth and time. To solve this, the image can be exported as a tarball and distributed among others.

Run the following to export the image,

```bash
$ sudo docker save registry.gitlab.com/fsci/resources/debian-dev:latest > debian-dev.tar
$ gzip debian-dev.tar
```

Now we get ``debian-dev.tar.gz`` file which is ready to be distributed.

To load the image from the tarball use,

```bash
$ sudo docker load < debian-dev.tar.gz
```



## Installing Packaging Tools

Now, after setting up our Debian-Sid, we need to install tools for helping us in packaging.

| Packaging Tool |                     man                      |
| :------------: | :------------------------------------------: |
|    gem2deb     |           For packaging ruby gems            |
|    npm2deb     |        For packaging Node.js modules         |
| dh-make-golang |          For packaging go packages           |
|    dh-make     | Generic tool; if no specific tool for a lang |

If the packaging tools are not installed, then install them by,

```bash
$ sudo apt-get install gem2deb npm2deb dh-make
$ npm install npm@latest -g # update npm
```



## Practical Session 1

In this session, we are going to package a small module named [d3-time](https://github.com/d3/d3-time) from [D3.js](https://d3js.org/) .

First of all, what is **d3-time**? It's a replacement for the default JavaScript date-time module. Now, we are diving right into packaging.

Make a directory of your choice, and ``cd`` to it

```bash
$ mkdir vidya
$ cd vidya
```

We need to install the ``d3-time`` module by using npm,

```bash
$ npm i d3-time
```

_For checking the version of npm2deb or other packages use,_

```bash
$ apt policy npm2deb
```

From here onwards is the real core part of packaging..

We need to add our email and name to ``.bashrc`` as the packaging tool needs them. Replace the email and name in the below code and write in to ``~/.bashrc``

```bash
export DEBEMAIL=youremail@domain 
export DEBFULLNAME='Your Name' 
alias lintian='lintian -iIEcv --pedantic --color auto' 
alias git-import-dsc='git-import-dsc --author-is-committer --pristine-tar' 
alias clean='fakeroot debian/rules clean' 
```

Activate the changes in ``~/.bashrc`` by doing ``source ~/.bashrc``.

Now we are going to pull the latest ``d3-time`` using ``npm2deb``(packaging tool for npm). 

```bash
$ npm2deb create d3-time
```

```bash
Downloading source tarball file using debian/watch file... 
uscan: Newest version of node-d3-time on remote site is 1.0.11, specified download version is 1.0.11 
Successfully symlinked ../node-d3-time-1.0.11.tar.gz to ../node-d3-time_1.0.11.orig.tar.gz. 
Creating debian source package... 
uupdate: debian/source/format is "3.0 (quilt)". 
uupdate: Auto-generating node-d3-time_1.0.11-1.debian.tar.xz 
uupdate: -> Use existing node-d3-time_1.0.11-1.debian.tar.xz 
dpkg-source: info: extracting node-d3-time in node-d3-time-1.0.11 
dpkg-source: info: unpacking node-d3-time_1.0.11.orig.tar.gz 
dpkg-source: info: unpacking node-d3-time_1.0.11-1.debian.tar.xz 
uupdate: Remember: Your current directory is changed back to the old source tree! 
uupdate: Do a "cd ../node-d3-time-1.0.11" to see the new source tree and 
    edit it to be nice Debianized source. 
Building the binary package 
dpkg-source: info: using source format '3.0 (quilt)' 
dpkg-source: info: building node-d3-time using existing ./node-d3-time_1.0.11.orig.tar.gz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-1.debian.tar.xz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-1.dsc 
dpkg-buildpackage: info: source package node-d3-time 
dpkg-buildpackage: info: source version 1.0.11-1 
dpkg-buildpackage: info: source distribution UNRELEASED 
dpkg-buildpackage: info: source changed by Your Name <your@email.domain> 
dpkg-buildpackage: info: host architecture amd64 
 dpkg-source --before-build . 
 debian/rules clean 
dh clean --with nodejs 
   dh_auto_clean --buildsystem=nodejs 
        rm -rf ./node_modules/.cache 
   dh_clean 
 dpkg-source -b . 
dpkg-source: info: using source format '3.0 (quilt)' 
dpkg-source: info: building node-d3-time using existing ./node-d3-time_1.0.11.orig.tar.gz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-1.debian.tar.xz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-1.dsc 
 debian/rules build 
dh build --with nodejs 
   dh_update_autotools_config 
   dh_autoreconf 
   dh_auto_configure --buildsystem=nodejs 
   dh_auto_build --buildsystem=nodejs 
No build command found, searching known files 
   dh_auto_test --buildsystem=nodejs 
        /usr/bin/node -e require\(\".\"\) 
internal/modules/cjs/loader.js:583 
    throw err; 
    ^ 
Error: Cannot find module '.' 
    at Function.Module._resolveFilename (internal/modules/cjs/loader.js:581:15) 
    at Function.Module._load (internal/modules/cjs/loader.js:507:25) 
    at Module.require (internal/modules/cjs/loader.js:637:17) 
    at require (internal/modules/cjs/helpers.js:22:18) 
    at [eval]:1:1 
    at Script.runInThisContext (vm.js:96:20) 
    at Object.runInThisContext (vm.js:303:38) 
    at Object.<anonymous> ([eval]-wrapper:6:22) 
    at Module._compile (internal/modules/cjs/loader.js:689:30) 
    at evalScript (internal/bootstrap/node.js:587:27) 
dh_auto_test: /usr/bin/node -e require\(\".\"\) returned exit code 1 
make: *** [debian/rules:8: build] Error 255 
dpkg-buildpackage: error: debian/rules build subprocess returned exit status 2 
dh clean --with nodejs 
   dh_auto_clean --buildsystem=nodejs 
        rm -rf ./node_modules/.cache 
   dh_clean 
Remember, your new source directory is d3-time/node-d3-time-1.0.11 
This is not a crystal ball, so please take a look at auto-generated files. 
You may want fix first these issues: 
d3-time/node-d3-time-1.0.11/debian/control:Description: FIX_ME write the Debian package description 
d3-time/node-d3-time_itp.mail:Subject: ITP: node-d3-time -- FIX_ME write the Debian package description 
d3-time/node-d3-time_itp.mail:  Description     : FIX_ME write the Debian package description 
d3-time/node-d3-time_itp.mail: FIX_ME: This ITP report is not ready for submission, until you are 
d3-time/node-d3-time_itp.mail:FIX_ME: Explain why this package is suitable for adding to Debian. Is 
d3-time/node-d3-time_itp.mail:FIX_ME: Explain how you intend to consistently maintain this package 
```

We got a lot of errors from first one itself. So we need to manually fix these errors.

After the above command, we have a new folder named ``d3-time``. This the folder that is pulled from Debian watch file ie latest files(unstable) . The folder structure of ``d3-time`` is,

```bash
drwxr-xr-x 4 root root  4096 Aug 25 04:17 . 
drwxr-xr-x 4 root root  4096 Aug 25 04:16 .. 
drwxr-xr-x 3 root root  4096 Aug 25 04:16 node-d3-time 
drwxr-xr-x 5 root root  4096 Aug 25 04:16 node-d3-time-1.0.11 
-rw-r--r-- 1 root root 36662 Aug 25 04:16 node-d3-time-1.0.11.tar.gz 
-rw-r--r-- 1 root root  2336 Aug 25 04:17 node-d3-time_1.0.11-1.debian.tar.xz 
-rw-r--r-- 1 root root  1144 Aug 25 04:17 node-d3-time_1.0.11-1.dsc 
lrwxrwxrwx 1 root root    26 Aug 25 04:16 node-d3-time_1.0.11.orig.tar.gz -> node-d3-time-1.0.11.tar.gz 
-rw-r--r-- 1 root root  1574 Aug 25 04:16 node-d3-time_itp.mail 
```

We are going to package the ``node-d3-time-1.0.11``, which is the latest package. We first try to build the package by using ``dpkg-buildpackage`` inside the ``node-d3-time-1.0.11``.

_P.S: While building make sure you are outside the debian/ directory_

```bash
dpkg-buildpackage: info: source package node-d3-time 
dpkg-buildpackage: info: source version 1.0.11-2 
dpkg-buildpackage: info: source distribution UNRELEASED 
dpkg-buildpackage: info: source changed by Abhijith Sheheer <abhijithsheheer@gmail.com> 
dpkg-buildpackage: info: host architecture amd64 
 dpkg-source --before-build . 
 debian/rules clean 
dh clean --with nodejs 
   dh_auto_clean --buildsystem=nodejs 
        rm -rf ./node_modules/.cache 
   dh_clean 
 dpkg-source -b . 
dpkg-source: info: using source format '3.0 (quilt)' 
dpkg-source: info: building node-d3-time using existing ./node-d3-time_1.0.11.orig.tar.gz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-2.debian.tar.xz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-2.dsc 
 debian/rules build 
dh build --with nodejs 
   dh_update_autotools_config 
   dh_autoreconf 
   dh_auto_configure --buildsystem=nodejs 
   dh_auto_build --buildsystem=nodejs 
No build command found, searching known files 
   dh_auto_test --buildsystem=nodejs 
        /usr/bin/node -e require\(\".\"\) 
internal/modules/cjs/loader.js:583 
    throw err; 
    ^ 
Error: Cannot find module '.' 
    at Function.Module._resolveFilename (internal/modules/cjs/loader.js:581:15) 
    at Function.Module._load (internal/modules/cjs/loader.js:507:25) 
    at Module.require (internal/modules/cjs/loader.js:637:17) 
    at require (internal/modules/cjs/helpers.js:22:18) 
    at [eval]:1:1 
    at Script.runInThisContext (vm.js:96:20) 
    at Object.runInThisContext (vm.js:303:38) 
    at Object.<anonymous> ([eval]-wrapper:6:22) 
    at Module._compile (internal/modules/cjs/loader.js:689:30) 
    at evalScript (internal/bootstrap/node.js:587:27) 
dh_auto_test: /usr/bin/node -e require\(\".\"\) returned exit code 1 
make: *** [debian/rules:8: build] Error 255 
dpkg-buildpackage: error: debian/rules build subprocess returned exit status 2 
```

Oops. We got new errors. In every situation error is our friend. All error indicates our mistakes. Here, the tool is searching for module "." which is weird. Let us check the ``package.json`` file.

Look for a field `main` which tells nodejs to load a particular file when that directory is inside a `require` statement. 
For example when you have `require('d3-time');` it looks for a directory called `d3-time` inside `node_modules` directory or in directories mentioned in `NODE_PATH` variable or default load path set by nodejs.
In debian, `/usr/share/nodejs` is the current preferred path. Once it finds the matching directory, it looks for `package.json` file inside it and loads the file mentioned in `main` field.
You can see 
`  "main": "dist/d3-time.js",`
But if you look at your local directory, this file is not present. Usually `dist` contains files generated from source by tools like `rollup`, `webpack`, `babel` etc

```json
{   
.......
	"pretest": "rollup -c",
.......
  },
  "devDependencies": {
    "eslint": "5",
    "rollup": "0.64",
    "rollup-plugin-terser": "1",
    "tape": "4"
  }
}
```

Here, the corresponding command in `pretest` is ``rollup -c`` (in most packages it is `build` instead of `pretest`) and ``rollup`` version required is 0.64. So, we need to make a file named ``build`` in ``debian/nodejs/`` and add ``rollup -c``. And try building again.

```bash
root@012f48e4060b:~/vidya/d3-time/node-d3-time-1.0.11# dpkg-buildpackage 
dpkg-buildpackage: info: source package node-d3-time 
dpkg-buildpackage: info: source version 1.0.11-2 
dpkg-buildpackage: info: source distribution UNRELEASED 
dpkg-buildpackage: info: source changed by Abhijith Sheheer <abhijithsheheer@gmail.com> 
dpkg-buildpackage: info: host architecture amd64 
 dpkg-source --before-build . 
 debian/rules clean 
dh clean --with nodejs 
   dh_auto_clean --buildsystem=nodejs 
        rm -rf ./node_modules/.cache 
   dh_clean 
 dpkg-source -b . 
dpkg-source: info: using source format '3.0 (quilt)' 
dpkg-source: info: building node-d3-time using existing ./node-d3-time_1.0.11.orig.tar.gz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-2.debian.tar.xz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-2.dsc 
 debian/rules build 
dh build --with nodejs 
   dh_update_autotools_config 
   dh_autoreconf 
   dh_auto_configure --buildsystem=nodejs 
   dh_auto_build --buildsystem=nodejs 
Found debian/nodejs/./build 
        sh -e debian/nodejs/build 
[!] Error: Unexpected token 
rollup.config.js (22:4) 
20:   config, 
21:   { 
22:     ...config, 
        ^ 
23:     output: { 
24:       ...config.output, 
dh_auto_build: sh -e debian/nodejs/build returned exit code 1 
make: *** [debian/rules:8: build] Error 255 
dpkg-buildpackage: error: debian/rules build subprocess returned exit status 2 
```

This is a syntax error in the ``rollup.config.js`` file. After some discussion with JS Developer, we understand that it is due to ``rollup`` not understand the syntax.

We can check the version of ``rollup`` installed by,

```bash
$ apt policy rollup
```

```bash
rollup: 
  Installed: 0.50.0-6 
  Candidate: 0.50.0-6 
  Version table: 
 *** 0.50.0-6 500 
        500 http://deb.debian.org/debian sid/main amd64 Packages 
        100 /var/lib/dpkg/status 
```

We know that the ``d3-time`` require ``rollup`` of version 0.64 in the ``package.json`` file. The solution is strip down the ``rollup.config.js`` to,

```js
import * as meta from "./package.json";

const config = {
  input: "src/index.js",
  external: Object.keys(meta.dependencies || {}).filter(key => /^d3-/.test(key)),
  output: {
    file: `dist/${meta.name}.js`,
    name: "d3",
    format: "umd",
    indent: false,
    extend: true,
    banner: `// ${meta.homepage} v${meta.version} Copyright ${(new Date).getFullYear()} ${meta.author.name}`,
    globals: Object.assign({}, ...Object.keys(meta.dependencies || {}).filter(key => /^d3-/.test(key)).map(key => ({[key]: "d3"})))
  },
  plugins: []
};

export default [
  config
];
```

And since these are upstream files, we are not allowed to directly edit the files (except debian/*). Thus, we need to apply a patch.

Patch is applied by,

```bash
$ dpkg-source --commit
```

Add a relevant file named similar to ``something-config.patch`` and add short description on it.

_If you messed up the patch and would like to revert it do_

```bash
$ quilt pop -a
```

And delete ``.pc`` and ``debian/patches`` and restart the patching.

After building again,

```bash
dpkg-buildpackage: info: source package node-d3-time 
dpkg-buildpackage: info: source version 1.0.11-2 
dpkg-buildpackage: info: source distribution UNRELEASED 
dpkg-buildpackage: info: source changed by Abhijith Sheheer <abhijithsheheer@gmail.com> 
dpkg-buildpackage: info: host architecture amd64 
 dpkg-source --before-build . 
 debian/rules clean 
dh clean --with nodejs 
   dh_auto_clean --buildsystem=nodejs 
        rm -rf ./node_modules/.cache 
   dh_clean 
 dpkg-source -b . 
dpkg-source: info: using source format '3.0 (quilt)' 
dpkg-source: info: building node-d3-time using existing ./node-d3-time_1.0.11.orig.tar.gz 
dpkg-source: info: using patch list from debian/patches/series 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-2.debian.tar.xz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-2.dsc 
 debian/rules build 
dh build --with nodejs 
   dh_update_autotools_config 
   dh_autoreconf 
   dh_auto_configure --buildsystem=nodejs 
   dh_auto_build --buildsystem=nodejs 
Found debian/nodejs/./build 
        sh -e debian/nodejs/build 
src/index.js → dist/d3-time.js... 
created dist/d3-time.js in 139ms 
   dh_auto_test --buildsystem=nodejs 
        /usr/bin/node -e require\(\".\"\) 
   create-stamp debian/debhelper-build-stamp 
 debian/rules binary 
dh binary --with nodejs 
   dh_testroot 
   dh_prep 
   dh_auto_install --buildsystem=nodejs 
No "files" field in ./package.json, install all files 
        mkdir -p /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/ 
        cp --reflink=auto -a ./d3-time.sublime-project /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time// 
        cp --reflink=auto -a ./package.json /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time// 
        mkdir -p /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/dist 
        cp --reflink=auto -a ./dist/d3-time.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/dist/ 
        mkdir -p /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src 
        cp --reflink=auto -a ./src/duration.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/minute.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/second.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/hour.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/year.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/month.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/interval.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcYear.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/index.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/week.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcMinute.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/millisecond.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcDay.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/day.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcMonth.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcWeek.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcHour.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
   dh_installdocs 
   dh_installchangelogs 
   dh_perl 
   dh_link 
   dh_strip_nondeterminism 
   dh_compress 
   dh_fixperms 
   dh_missing 
   dh_installdeb 
   dh_gencontrol 
   dh_md5sums 
   dh_builddeb 
dpkg-deb: building package 'node-d3-time' in '../node-d3-time_1.0.11-2_all.deb'. 
 dpkg-genbuildinfo 
 dpkg-genchanges  >../node-d3-time_1.0.11-2_amd64.changes 
dpkg-genchanges: info: including full source code in upload 
 dpkg-source --after-build . 
dpkg-buildpackage: info: full upload (original source is included) 
```

We got the ``.deb`` in the parent directory. 📦

But this is not over quite. Remember we remove some code. Actually we want the same feature. Here, we are going to use ```uglifyjs.terser``` as replacement of the previously deleted code.

```bash
uglifyjs.terser dist/d3-time.js -o dist/d3-time.min.js
```

Append this code to the build file , i.e ```debian/nodejs/build```, and build again.

_P.S: Uglifyjs.terser is a fork of uglyifyjs and it is used to make the *.min.js files which are supposed to create by terser in rollup(but the rollup in debian at this time is outdated)_

```bash
dpkg-buildpackage: info: source package node-d3-time
dpkg-buildpackage: info: source version 1.0.11-2
dpkg-buildpackage: info: source distribution UNRELEASED
dpkg-buildpackage: info: source changed by Abhijith Sheheer <abhijithsheheer@gmail.com>
dpkg-buildpackage: info: host architecture amd64
 dpkg-source --before-build .
 debian/rules clean
dh clean --with nodejs
   dh_auto_clean --buildsystem=nodejs
        rm -rf ./node_modules/.cache
   dh_clean
 dpkg-source -b .
dpkg-source: info: using source format '3.0 (quilt)'
dpkg-source: info: building node-d3-time using existing ./node-d3-time_1.0.11.orig.tar.gz
dpkg-source: info: using patch list from debian/patches/series
dpkg-source: info: local changes detected, the modified files are:
 node-d3-time-1.0.11/dist/d3-time.js
dpkg-source: info: you can integrate the local changes with dpkg-source --commit
dpkg-source: error: aborting due to unexpected upstream changes, see /tmp/node-d3-time_1.0.11-2.diff.saoZeU
dpkg-buildpackage: error: dpkg-source -b . subprocess returned exit status 2
```

We need to add the two files generated while building to ``debian/clean``.

```makefile
dist/d3-time.js
dist/d3-time.min.js
```

Also we need to update the build depends in ``debian/control`` .

```
....

Build-Depends:
 debhelper-compat (= 11)
 , nodejs (>= 6)
 , pkg-js-tools (>= 0.8.10)
 , rollup
 , uglifyjs.terser
Standards-Version: 4.4.0
Homepage: https://d3js.org/d3-time/
Vcs-Git: https://salsa.debian.org/js-team/node-d3-time.git
Vcs-Browser: https://salsa.debian.org/js-team/node-d3-time

....
```

This should be it. Try building again.

```bash
dpkg-buildpackage: info: source package node-d3-time 
dpkg-buildpackage: info: source version 1.0.11-2 
dpkg-buildpackage: info: source distribution UNRELEASED 
dpkg-buildpackage: info: source changed by Abhijith Sheheer <abhijithsheheer@gmail.com> 
dpkg-buildpackage: info: host architecture amd64 
 dpkg-source --before-build . 
 debian/rules clean 
dh clean --with nodejs 
   dh_auto_clean --buildsystem=nodejs 
        rm -rf ./node_modules/.cache 
   dh_clean 
 dpkg-source -b . 
dpkg-source: info: using source format '3.0 (quilt)' 
dpkg-source: info: building node-d3-time using existing ./node-d3-time_1.0.11.orig.tar.gz 
dpkg-source: info: using patch list from debian/patches/series 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-2.debian.tar.xz 
dpkg-source: info: building node-d3-time in node-d3-time_1.0.11-2.dsc 
 debian/rules build 
dh build --with nodejs 
   dh_update_autotools_config 
   dh_autoreconf 
   dh_auto_configure --buildsystem=nodejs 
   dh_auto_build --buildsystem=nodejs 
Found debian/nodejs/./build 
        sh -e debian/nodejs/build 
src/index.js → dist/d3-time.js... 
created dist/d3-time.js in 137ms 
   dh_auto_test --buildsystem=nodejs 
        /usr/bin/node -e require\(\".\"\) 
   create-stamp debian/debhelper-build-stamp 
 debian/rules binary 
dh binary --with nodejs 
   dh_testroot 
   dh_prep 
   dh_auto_install --buildsystem=nodejs 
No "files" field in ./package.json, install all files 
        mkdir -p /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/ 
        cp --reflink=auto -a ./d3-time.sublime-project /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time// 
        cp --reflink=auto -a ./package.json /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time// 
        mkdir -p /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/dist 
        cp --reflink=auto -a ./dist/d3-time.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/dist/ 
        cp --reflink=auto -a ./dist/d3-time.min.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/dist/ 
        mkdir -p /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src 
        cp --reflink=auto -a ./src/duration.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/minute.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/second.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/hour.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/year.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/month.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/interval.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcYear.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/index.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/week.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcMinute.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/millisecond.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcDay.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/day.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcMonth.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcWeek.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
        cp --reflink=auto -a ./src/utcHour.js /root/vidya/d3-time/node-d3-time-1.0.11/debian/node-d3-time//usr/share/nodejs/d3-time/src/ 
   dh_installdocs 
   dh_installchangelogs 
   dh_perl 
   dh_link 
   dh_strip_nondeterminism 
   dh_compress 
   dh_fixperms 
   dh_missing 
   dh_installdeb 
   dh_gencontrol 
   dh_md5sums 
   dh_builddeb 
dpkg-deb: building package 'node-d3-time' in '../node-d3-time_1.0.11-2_all.deb'. 
 dpkg-genbuildinfo 
 dpkg-genchanges  >../node-d3-time_1.0.11-2_amd64.changes 
dpkg-genchanges: info: including full source code in upload 
 dpkg-source --after-build . 
dpkg-buildpackage: info: full upload (original source is included) 
```

Yeah. It worked and successful made a ``.deb`` file in the parent directory.

Now, we need to check the ``.deb`` file have no error. Here, we use tool named ``lintian`` to check for errors in ``.deb`` file.

```bash
$ lintian ../node-d3-time_1.0.11-3_all.deb 
```

```bash
N: Using profile debian/main. 
N: Starting on group node-d3-time/1.0.11-4 
N: Unpacking packages in group node-d3-time/1.0.11-4 
N: ---- 
N: Processing binary package node-d3-time (version 1.0.11-4, arch all) ... 
N: Finished processing group node-d3-time/1.0.11-4 
```

For me, ``lintian`` didn't show any error. The deb file can be only send to upstream if it have no errors.

_Always remember to make ``linitian`` happy_ 😀 

## Practical Session 2

Going to package another simple module ``d3-format `` but using ``git`` and ``sbuild``. For mere introduction, ``git`` is a control-version system and ``sbuild`` is a packaging tool which uses ``dpkg-buildpackage`` and ``lintian`` in its back-end.

First make a account in [Salsa](salsa.debian.org) which is a collaborative development server for debian. Create a new repository named ``node-d3-format``. (Since it is a node package and typically all node packages in salsa are named as ``node-`` + name of package.)

Download the package ``d3-format`` using ``npm2deb``. (Refer above Session)

Get inside ``d3-format``.

```bash
cd d3-format/ && ls -la
total 360
drwxr-xr-x  4 dev dev   4096 Sep  2 16:56 .
drwxr-xr-x 12 dev dev   4096 Sep  5 14:14 ..
drwxr-xr-x  7 dev dev   4096 Sep  2 15:50 node-d3-format
drwxr-xr-x  6 dev dev   4096 Sep  2 14:28 node-d3-format-1.3.2
-rw-rw-r--  1 dev dev  17112 Sep  2 16:56 node-d3-format_1.3.2-1_all.deb
-rw-rw-r--  1 dev dev 129994 Sep  2 16:01 node-d3-format_1.3.2-1_amd64-2019-09-02T15:56:38Z.build
-rw-rw-r--  1 dev dev 134607 Sep  2 16:59 node-d3-format_1.3.2-1_amd64-2019-09-02T16:54:21Z.build
lrwxrwxrwx  1 dev dev     55 Sep  2 16:54 node-d3-format_1.3.2-1_amd64.build -> node-d3-format_1.3.2-1_amd64-2019-09-02T16:54:21Z.build
-rw-rw-r--  1 dev dev   8500 Sep  2 16:56 node-d3-format_1.3.2-1_amd64.buildinfo
-rw-rw-r--  1 dev dev   1110 Sep  2 16:56 node-d3-format_1.3.2-1_amd64.changes
-rw-r--r--  1 dev dev   3504 Sep  2 16:54 node-d3-format_1.3.2-1.debian.tar.xz
-rw-r--r--  1 dev dev   1285 Sep  2 16:54 node-d3-format_1.3.2-1.dsc
lrwxrwxrwx  1 dev dev     27 Sep  2 14:28 node-d3-format_1.3.2.orig.tar.gz -> node-d3-format-1.3.2.tar.gz
-rw-r--r--  1 dev dev  31091 Sep  2 14:28 node-d3-format-1.3.2.tar.gz
-rw-r--r--  1 dev dev   1572 Sep  2 14:28 node-d3-format_itp.mail
```

Here, to initialize our ``git`` here. So we need to use ``gbp import-dsc``, but there is already a directory named ``node-d3-format``. So delete that directory and run ``gbp importdsc node-d3-format_1.3.2-1.dsc`` and the new updated directory is created.

```bash
cd node-d3-format/ && ls -la
total 104
drwxr-xr-x 7 dev dev  4096 Sep  2 15:50 .
drwxr-xr-x 4 dev dev  4096 Sep  2 16:56 ..
-rw-r--r-- 1 dev dev   340 Sep  2 15:27 d3-format.sublime-project
drwxr-xr-x 6 dev dev  4096 Sep  2 16:52 debian
-rw-r--r-- 1 dev dev   226 Sep  2 15:27 .eslintrc.json
drwxr-xr-x 8 dev dev  4096 Sep  2 16:53 .git
-rw-r--r-- 1 dev dev    63 Sep  2 15:27 .gitignore
-rw-r--r-- 1 dev dev  1475 Sep  2 15:27 LICENSE
drwxr-xr-x 2 dev dev  4096 Sep  2 15:27 locale
-rw-r--r-- 1 dev dev    34 Sep  2 15:27 .npmignore
-rw-r--r-- 1 dev dev  1497 Sep  2 15:27 package.json
-rw-r--r-- 1 dev dev 15209 Sep  2 15:27 README.md
-rw-r--r-- 1 dev dev   577 Sep  2 15:49 rollup.config.js
drwxr-xr-x 2 dev dev  4096 Sep  2 15:27 src
drwxr-xr-x 2 dev dev  4096 Sep  2 15:27 test
-rw-r--r-- 1 dev dev 32220 Sep  2 15:27 yarn.lock
```

Here, we are going to upload these files to salsa, but we not need to upload the temporary files.

Some of the temporary files are,

- .eslintrc.json
- .git and .gitignore
- .npmignore

> " You should have separate commits for each logical change with proper commit messages. When pushing, you should ``git push -u --all --follow-tags`` as debian packaging needs master, upstream and pristine-tar branches . Also you should only commit files you create or modify. "
>
> Pirate-Praveen

**Commit Message**

- When you create build file in ``debian/nodejs/`` 
  - " Build with rollup -c "
- When you remove lines from ``rollup.config.js``
  - " Remove unsupported syntax "
- When you applied patch
  - " Remove terser usage with patch "
- When you added uglifyjs.terser in build
  - " Use uglifyjs.terser command to create minified file "
- When you update build-depends in ``debian/control``
  - " Add rollup and uglifyjs.terser to build dependencies "
- When you change bug number in ``debian/changelog``
  - " Add ITP "

And similarly, commit message should say about the changes made.

## Practical Session 3

Clone the following git repo.

```bash
$ git clone https://git.fosscommunity.in/bhe/hello
```

Rename the ``hello`` to ``hello-0.1``.(Version numbering)

```bash
$ mv hello hello-0.1
```

Move into the directory and do ``dh_make``.

```bash
$ dh_make --createorgin
```

Now the ``debian/`` and other files are generated. 

Edit ``control``, ``changelog`` and ``Makefile`` to remove the ``lintain`` errors.

```bash
$ dpkg-buildpackage -us -uc
```

_Note_: ``-us -uc`` stands for *unsigned*, for not signing the binary temporarly.

If successful, this provides us with a ``.deb`` file in parent directory. Install it using,

```bash
$ dpkg -i ./hello_0.1-1_amd64.deb
```

Running ``hello`` in the terminal now, prints ``Hello, world!!``. Thus, a simple C program have been packaged successfully.

## Practical Session 4



## Session 5 (Bonus)

Going to package ``gnujump``.

```bash
$ wget http://ftp.gnu.org/gnu/gnujump/gnujump-1.0.8.tar.gz 
```

```bash
--2019-08-25 04:31:48--  http://ftp.gnu.org/gnu/gnujump/gnujump-1.0.8.tar.gz 
Resolving ftp.gnu.org (ftp.gnu.org)... 209.51.188.20, 2001:470:142:3::b 
Connecting to ftp.gnu.org (ftp.gnu.org)|209.51.188.20|:80... connected. 
HTTP request sent, awaiting response... 200 OK 
Length: 2508641 (2.4M) [application/x-gzip] 
Saving to: 'gnujump-1.0.8.tar.gz' 
gnujump-1.0.8.tar.gz                     100%[================================================================================>]   2.39M   153KB/s    in 17s 
2019-08-25 04:32:06 (143 KB/s) - 'gnujump-1.0.8.tar.gz' saved [2508641/2508641] 
```

```bash
$ cp gnujump-1.0.8.tar.gz gnujump-1.0.8.orig.tar.gz 
```

```bash
 $ tar xf gnujump-1.0.8.orig.tar.gz
```

```bash
$ cd gnujump-1.0.8/
$ dh_make -f ../gnujump-1.0.8.tar.gz
```

Select single binary for now. Now the ``debian`` directory is created.

```bash
total 96 
drwxr-xr-x 3 root      root       4096 Aug 25 04:41 . 
drwxr-xr-x 9 developer developer  4096 Aug 25 04:41 .. 
-rw-r--r-- 1 root      root        174 Aug 25 04:41 README.Debian 
-rw-r--r-- 1 root      root        259 Aug 25 04:41 README.source 
-rw-r--r-- 1 root      root        186 Aug 25 04:41 changelog 
-rw-r--r-- 1 root      root        508 Aug 25 04:41 control 
-rw-r--r-- 1 root      root       1828 Aug 25 04:41 copyright 
-rw-r--r-- 1 root      root         28 Aug 25 04:41 gnujump-docs.docs 
-rw-r--r-- 1 root      root        131 Aug 25 04:41 gnujump.cron.d.ex 
-rw-r--r-- 1 root      root        515 Aug 25 04:41 gnujump.doc-base.EX 
-rw-r--r-- 1 root      root       1629 Aug 25 04:41 manpage.1.ex 
-rw-r--r-- 1 root      root       4649 Aug 25 04:41 manpage.sgml.ex 
-rw-r--r-- 1 root      root      11006 Aug 25 04:41 manpage.xml.ex 
-rw-r--r-- 1 root      root        126 Aug 25 04:41 menu.ex 
-rw-r--r-- 1 root      root        958 Aug 25 04:41 postinst.ex 
-rw-r--r-- 1 root      root        931 Aug 25 04:41 postrm.ex 
-rw-r--r-- 1 root      root        691 Aug 25 04:41 preinst.ex 
-rw-r--r-- 1 root      root        878 Aug 25 04:41 prerm.ex 
-rwxr-xr-x 1 root      root        677 Aug 25 04:41 rules 
drwxr-xr-x 2 root      root       4096 Aug 25 04:41 source 
-rw-r--r-- 1 root      root       1149 Aug 25 04:41 watch.ex 
```

**Under Construction... Help I can't able to package this.. Requies *dh_make* tool**



## Common Issues

### Docker Related

#### Permission Denied

If you are getting permission denied error message, you are probably _not using **sudo**_. 

```bash
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.40/images/json: dial unix /var/run/docker.sock: connect: permission denied
```

You can either use ``sudo`` in every docker command (Recommended) or you have to add your user to the docker group.

#### Cannot connect to Docker

The docker service required may not be running or started in the background.

```bash
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

You need to start the service manually.

```bash
$ sudo systemctl start docker --now
```

You can check the status of the docker service by,

```bash
$ sudo systemctl status docker
```

Also you can enable the ``docker.service`` which enables it to load while your system boots.

```bash
$ sudo systemctl enable docker
```



## Use the Source, Luke!

This documentation will not be possible without the below sources.

- [Debian Prerequisites by Pirate Praveen](https://www.loomio.org/d/LTpSdMuX/debian-packaging-pre-requisites) 
- [Debian Policy Manual](https://www.debian.org/doc/debian-policy/index.html) 
- [DebianWiki](https://wiki.debian.org/Packaging)
- [Official Docker Documentation](https://docs.docker.com/)
- [Sample Packaging Tutorial (Debian Wiki)](https://wiki.debian.org/SimplePackagingTutorial)
- [Debian Manual on debmake](https://www.debian.org/doc/manuals/debmake-doc/ch04.en.html)
- [Stackoverflow (Our last Hope!!)](https://stackoverflow.com)

## To-do List

- [x] Complete  Session 1 (dpkg-buildpackage)
- [x] Complete Session 2 (salsa + sbuild)
- [x] Complete Session 3 (Intoduction dh_make)
- [ ] Complete Session 4 (gem2deb)
- [ ] Bonus packaging
- [ ] Try packaging a module with dependencies
- [ ] Make an official contribution to Debian
- [ ] Become a DM / DD 🤭

## Special Thanks

[@piratepraveen]()

[@sruthi]()

[@bhe]()

[@balasankarc]()

[Typora](https://typora.io/) for helping to make Markdown editing more easy and fast.

[Markdown-Toc](http://ecotrust-canada.github.io/markdown-toc/) for making the TOC.

## License

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a>

This work by <a xmlns:cc="http://creativecommons.org/ns#" href="https://github.com/abspython/" property="cc:attributionName" rel="cc:attributionURL">Abhijith Sheher</a> is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.

