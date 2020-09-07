# fedoraCoreOs_on_openstack

Running fedora core os on openstack with ansible.

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

### How to create a volume

1. Create a volume in openstack with the desired capacity:

    ```bash
    openstack volume create --size <desired-size> <desired-volume-name>
    ```

    This volume will be accessible in fedora core os as `/dev/disk/by-id/virtio-<truncated-volume-id>`, where `<truncated-volume-id>` is the openstack volume id truncated to a length of 20 chars ([taken from chris cowley](https://medium.com/@chriscowleyunix/identify-and-mounting-cinder-volumes-in-openstack-heat-403e08adaa52)).

## How to create the fedora core os ignition file

### Generate ssh keys

1. Generate ssh keys using `ssh-keygen`. This [link](https://www.ssh.com/ssh/keygen/) will help. The key pair will be needed at a later point in this tutorial! The following command creates a 4096 bit rsa key pair. It will prompt for a password to protect the private key with encryption

    ```bash
    ssh-keygen -f <desired-path-to-ssh-key> -t rsa -b 4096
    ```

    - `<desired-path-to-ssh-key>` could be `~/.ssh/my_key`. In that case `ssh-keygen` will output two files: `~/.ssh/my_key` and `~/.ssh/my_key.pub`.

2. It might be wise to pass ssh key management to `ssh-agent`. This [link](https://www.ssh.com/ssh/agent) will help. It is important that you remember the passwort for the private key!

### Fedora core os configuration

1. Create a fedora core os configuration file ending with `...fcc.yaml`. 
    - Minimal example:

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

    - Example specifying a ssh user, device partition and partition formatting:

        ```yaml
        variant: fcos
        version: 1.0.0
        passwd:
        users:
            - name: core
            ssh_authorized_keys:
                - ssh-rsa AAAAB3Nza...
        storage:
        disks:
            - device: /dev/disk/by-id/virtio-<openstack-volume-id-20-chars>
            wipe_table: true
            partitions:
                # Since type_guid is not specified, it will be a Linux native
                # partition.
                # We assign a descriptive label to the partition. This is important
                # for referring to it in a device-agnostic way in other parts of the
                # configuration.
                - label: <desired-partition-label>
                start_mib: 0
                size_mib: 0
                number: 1
                wipe_partition_entry: true
        filesystems:
            - path: /var/<desired-mount-point>
            device: /dev/disk/by-partlabel/<desired-partition-label>
            format: ext4
        ```

        In this example we are creating a partition the full size of the specified disk. We do not mount it though.

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

## Run fedora core os on openstack

### How to create the openstack fedora core os instance

1. Create fedora core os openstack server (taken from [link](https://remote-lab.net/fedora-coreos-openstack):

    ```bash
    openstack server create \
        --block-device-mapping <desired-dev-name>=<openstack-volume-id>:<type>:<size(GB)>:<delete-on-terminate>
        --flavor <flavor-name> \
        --image <image-name> \
        --nic net-id=<network-name> \
        --user-data <path-to-ignition-file>.ign \
        --config-drive True \
        <desired-instance-name>
    ```

    - `<desired-dev-name>` is the symlink in `/dev/` at which the volume should be added, although it seems that core os can also randomly decide to choose a `<desired-dev-name>` of its liking.
    - `<openstack-volume-id>` should be the uuid of the volume we created.
    - `<type>` can be `volume` or `snapshot`, but in this case choose `volume`.
    - `<size (GB)>` is optional but in this case please just leave it blank.
    - `<delete-on-terminate>` is optional as well but here we supply `false`.
    - Choose `<flavor-name>` to respect the minimum resource requirements of our image!
    - Supply our created internal network (not the subnet) as `<network-name>`.

    A real life example of the above command could look like this:

    ```bash
    openstack server create \
        --block-device-mapping sdb=0f53abcf-95d7-4397-af5d-1368d21eae4a:volume::false \
        --flavor standard.1.1905 \
        --image 4259881c-3011-409f-93cc-bde1330d72c5 \
        --nic net-id=35b07761-34bb-4e74-a853-f1aa489f4000 \
        --user-data ./transpiled_config.ign \
        --config-drive True \
        fedora_coreOs_test1
    ```

    The `--block-device-mapping` option is used to attach our created volume to the server during creation.

### How to bind a floating ip to the created instance

1. Bind floating ip to the recently created instance:

    ```bash
    openstack server add floating ip <instance-name> <floating-ip>
    ```

## Access fedora core os via ssh

1. Now you can access fedora core os via ssh:

    ```bash
    ssh -i <path-to-private-key> core@<floating-ip-of-instance>
    ```
