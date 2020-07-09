# spo-si-ocp4-installer
Installer tools for preparing a Private/Private or Offline installation of OCP4

# Guide

Make the script(s) executable
```
sudo chmod +x ./*.sh
```

export two variables:
```
export OCP_VERSION=4.4.3
export PULL_SECRET=$HOME/path/to/my/pull-secret.json
```

run generate.sh
```
./generate.sh
```

You should have two .iso files, split into 4GB or smaller chunks, which can then be burned or transported to private networks and concatenated together.

To concatenate together into one .iso and mount for use:
```
cat openshift* > ocp-installer.iso
sudo mkdir /mnt/ocp-installer
sudo mount ocp-installer.iso /mnt/ocp-installer
```

You can now proceed to use the openshift installer and its offline components as needed.