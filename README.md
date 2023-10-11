# Torizon for Maivin

This repository provides the repo manifests to build Torizon for Maivin which is Toradex's Torizon Linux distribution customized for Maivin.

# Requirements

- Maivin 2 or Maivin 1 with 4K camera option
    - Maivin 1 with e-ConSystems camera should use the dunfell branch.
- Linux system capable of building Yocto
    - Tested on Ubuntu 20.04 LTS with 32GB of RAM

# Setup

These instructions summarize those from Toradex and adjusted for Maivin builds.  Refer to the original [Build Torizon BSP6][1] documentation for details.

Instructions assume you have already configured your system according to the Yocto requirements and installed the repo command, refer to [Build Torizon BSP6][1] for details.

```shell
mkdir torizon-maivin
cd torizon-maivin
repo init -u ssh://git@github.com/MaivinAI/torizon-maivin.git -b main -m maivin/default.xml
repo sync
MACHINE=verdin-imx8mp source setup-environment build
cp --remove-destination ../layers/meta-maivin/conf/bblayers.conf conf/bblayers.conf
```

# Build

```shell
MACHINE=verdin-imx8mp source setup-environment build
bitbake torizon-core-maivin
```

# Deploy

*Note: these commands should be run from the torizon-maivin/build directory.*

A Toradex Easy Installer "TEZI" feed can be setup from the Yocto deployment directory.  Create the following file as `deploy/images/verdin-imx8mp/image_list.json` with the content below.

```json
{
        "config_format": 1,
        "images": [
                "image-torizon-core-maivin.json"
        ]
}
```

Now you can launch the built-in Python HTTP server.

```
python3 -m http.server --directory deploy/images/verdin-imx8mp 9090
```

# TEZI

Perform a [USB System Recovery][2] of your Maivin platform and point the feeds to your computer, replacing MYCOMPUTER with your computer's hostname or IP address.

```
http://MYCOMPUTER:9090/image_list.json
```

Select "Torizon for Maivin" from the feeds and install as normal.

# Yocto with Git+SSH

*Note: this only applies when working with Git+SSH which is not required by default.*

Due to a quirk with how Yocto works with Git+SSH you will need to add the following to your SSH configuration to ensure Yocto fetches from Git using the `git` user instead defaulting to the current username.

Add the following to your `$HOME/.ssh/config` file, which might need to be created.

```
Host bitbucket.org
        HostName bitbucket.org
        User git
        PreferredAuthentications publickey

Host github.com
        HostName github.com
        User git
        PreferredAuthentications publickey
```

[1]: https://developer.toradex.com/torizon/in-depth/build-torizoncore-from-source-with-yocto-projectopenembedded/
[2]: https://support.deepviewml.com/hc/en-us/articles/4417259322893-Maivin-System-Recovery