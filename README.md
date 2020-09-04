# fedoraCoreOs_on_openstack

Running fedora core os on openstack with ansible.

## How to create the fedora core os ignition file

### Generate ssh keys

1. Generate ssh keys using `ssh-keygen`. This [link](https://www.ssh.com/ssh/keygen/) will help. The public key will be needed at a later point in this tutorial!
2. It might be wise to pass ssh key management to `ssh-agent`. This [link](https://www.ssh.com/ssh/agent) will help. It is important that you remember the passwort for the private key!

### Fedora core os configuration

1. Create a fedora core os configuration file ending with `...fcc.yaml`. Minimal example:

    ```yaml
    variant: fcos
    version: 1.0.0
    passwd:
      users:
        - name: core
          ssh_authorized_keys:
            - ssh-rsa AAAAB3NzaC1...
    ```

    The ssh key is omitted on purpose. Replace `ssh-rsa AAAAB3NzaC1...` with the **complete** contents of the public key file `<public-key-file-name>.pub` generated in the previous chapter **Generate ssh keys**.

### Generate fedora core os ignition file from config

The ignition file generation can be easily done using [fcct](https://docs.fedoraproject.org/en-US/fedora-coreos/using-fcct/). I recommend the following procedure for obtaining fcct and using it (requires docker and/or podman, both cli commands can be used interchangeably):

1. Obtain fcct according to above link. As of today:

    ```bash
    podman pull quay.io/coreos/fcct:release
    ```

2. Run fcct on configuration file according to above link. Powershell command is slightly different than bash. As of today for powershell:

    ```powershell
    Get-Content <example-fcc-path>.yaml | docker run -i --rm quay.io/coreos/fcct --pretty --strict > <transpiled-config-path>.ign
    ```

## Prepare openstack to run fedora core os

### How to create the openstack fedora core os image

1. First get the correct `.qcow2` image from the fedora core os download page.

    ```bash
    curl -O <download-link>
    ```

    The bare metal version should be perfectly fine.
2. Upload the image to openstack:

    ```bash
    openstack image create \
        --disk-format qcow2 \
        --container-format bare \
        --file <qcow-image-name>.qcow \
        --min-disk 10 \
        --min-ram 2048
        <desired-openstack-image-name-here>
    ```

### How to create the openstack network infrastructure

1. Create network:

    ```bash
    openstack network create <desired-network-name>
    ```

2. Create subnet of network:

    ```bash
    openstack subnet create <desired-subnet-name> \
        --subnet-range <desired-subnet-range> \
        --dns-nameserver <dns-ip> \
        --network <name-of-network-to-add-subnet-to>
    ```

    - `<desired-subnet-range>` could for example be `10.5.5.0/24`.
    - `<dns-ip>` could for example be `8.8.8.8` (googles dns nameserver).
3. Create router:

    ```bash
    openstack router create <desired-router-name>
    ```

4. Link router to subnet:

    ```bash
    openstack router add subnet <created-router-name> <created-subnet-name>
    ```

5. Link router to external provider network:

    ```bash
    openstack router set <created-router-name> --external-gateway <external-network-name>
    ```

### How to change the openstack security group rules

We want to enable pings and ssh traffic in the security group (possibly default) that belongs to our openstack project

1. Allow incoming pings (icmp) for all requesting ips:

    ```bash
    openstack security group rule create --src-ip 0.0.0.0/0 --protocol icmp --ingress <security-group-name>
    ```

2. Allow incoming tcp for all requesting ips on dest port 22 (ssh):

    ```bash
    openstack security group rule create --src-ip 0.0.0.0/0 --dst-port 22 --protocol tcp --ingress <security-group-name>
    ```

### How to create a floating ip

1. Create the floating ip and allocate the ip from the public network:

    ```bash
    openstack floating ip create <public-network-name>
    ```

## Run fedora core os on openstack

### How to create the openstack fedora core os instance

1. Create fedora core os openstack server (taken from [link](https://remote-lab.net/fedora-coreos-openstack):

    ```bash
    openstack server create \
        --flavor <flavor-name> \
        --image <image-name> \
        --nic net-id=<network-name> \
        --user-data <path-to-ignition-file>.ign \
        --config-drive True \
        <desired-instance-name>
    ```

    - Choose `<flavor-name>` to respect the minimum resource requirements of our image!
    - Supply our created internal network (not the subnet) as `<network-name>`.

### How to bind a floating ip to the created instance

1. Bind floating ip to the recently created instance:

    ```bash
    openstack server add floating ip <instance-name> <floating-ip>
    ```

## Access fedora core os via ssh

Now you can access fedora core os via ssh!
