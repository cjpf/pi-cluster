---
title: Cluster Node configuration
permalink: /docs/node/
redirect_from: /docs/node.md
---

4 Raspberry Pi (4GB), `node1`, `node2`, `node3` and `node4`, will be used as nodes for the Kubernetes cluster.
`node1` will be acting as **master node** and `node2/3/4` as **worker nodes**

## Hardware

`node1-4` are based on a Raspberry Pi 4B 4GB booting from a USB Flash Disk or SSD Disk depending on storage architectural option selected.

- Dedicated disks storage architecture: Kingston A400 480GB SSD Disk and a USB3.0 to SATA adapter will be used connected to `node1`. Kingston A400 240GB SSD Disk and USB3.0 to SATA adapter will be used connected to `node2-node4`.
- Centralized SAN architecture: A Samsung USB 3.1 32 GB Fit Plus Flash Disk will be used connected to one of the USB 3.0 ports of the Raspberry Pi.

## Storage configuration. Dedicated Disks

SSD Disk will be partitioned in boot time reserving 30 GB for root filesystem (OS installation) and the rest will be used for creating a logical volumes (LVM) mounted as `/storage`. This will provide local storage capacity in each node of the cluster, used mainly by Kuberentes distributed storage solution and by backup solution.

{{site.data.alerts.note}}

As a reference of how cloud images partitions grow in boot time check this blog [entry](https://elastisys.com/how-do-virtual-images-grow/).

{{site.data.alerts.end}}

`cloud-init` configuration (`user-data` file) includes commands to be executed once in boot time changing partition table and creating a new partition before the automatic growth of root partitions to fill the entire disk happens.

Command executed in boot time is:

```shell
sgdisk /dev/sda -e .g -n=0:30G:0 -t 0:8e00
```

This command:
  - First convert MBR partition to GPT (-g option)
  - Second moves the GPT backup block to the end of the disk  (-e option)
  - then creates a new partition starting 30GiB into the disk filling the rest of the disk (-n=0:10G:0 option)
  - And labels it as an LVM partition (-t option)

For `node1-node4`, the new partition created in boot time, `/dev/sda2`, uses most of the disk space leaving just 30GB for the root filesystem. This partition is then added to a LVM Volume Group and a unique Logical Volume is created. Lastly the Logical Volume is formatted (`ext4`) and mounted as `/storage`.

LVM partition and formatting tasks have been automated with Ansible developing the ansible role: **ricsanfre.storage** for managing LVM.

Specific `node1-node4` ansible variables to be used by this role are stored in [`vars/dedicated_disks/local_storage.yml`]({{ site.git_edit_address }}/vars/dedicated_disks/local_storage.yml)

## Network Configuration

Only ethernet interface (eth0) will be used connected to the lan switch. Interface will be configured through  DHCP using `gateway` DHCP server.

## Unbuntu boot from USB

Follow the procedure indicated [here](/docs/ubuntu/) using following cloud-init configuration files (`user-data` and `network-config`). `user-data` file used depends on the storage architectural option selected. Since DHCP is used it is not needed to change default `/boot/network-config` file.


| Storage Architeture | node1   | node2 | node3 | node 4 |
|-----------| ------- |-------|-------|--------|
| Dedicated Disks | [user-data]({{ site.git_edit_address }}/cloud-init/dedicated_disks/node1/user-data) | [user-data]({{ site.git_edit_address }}/cloud-init/dedicated_disks/node2/user-data)| [user-data]({{ site.git_edit_address }}/cloud-init/dedicated_disks/node3/user-data) | [user-data]({{ site.git_edit_address }}/cloud-init/dedicated_disks/node4/user-data) |
| Centralized SAN | [user-data]({{ site.git_edit_address }}/cloud-init/centralized_san/node1/user-data) | [user-data]({{ site.git_edit_address }}/cloud-init/centralized_san/node2/user-data)| [user-data]({{ site.git_edit_address }}/cloud-init/centralized_san/node3/user-data) | [user-data]({{ site.git_edit_address }}/cloud-init/centralized_san/node4/user-data) |
{: .table }

## Ubuntu OS Initital Configuration

After booting from the USB3.0 external storage for the first time, the Raspberry Pi will have SSH connectivity and it will be ready to be automatically configured from the ansible control node `pimaster`.

Initial configuration tasks includes removal of snap package, and Raspberry PI specific configurations tasks such as: intallation of fake hardware clock, installation of some utility packages scripts and change default GPU Memory Split configuration. See instructions [here](/docs/os_basic/).

For automating all this initial configuration tasks, ansible role **basic_setup** has been developed.

## NTP Server Configuration

`node1-node4` will be configured as NTP clients using NTP server running in `gateway`
See NTP Configuration instructions [here](/docs/gateway/#ntp-server-configuration)

NTP configuration in `node1-node4` has been automated using ansible role **ricsanfre.ntp**

## iSCSI configuration. Dedicated Disks

Open-iscsi is used by Longhorn as a mechanism to expose Volumes within Kuberentes cluster. All nodes of the cluster need to be configured as iSCSI initiators, When configurin iSCSI initiator, authentication default parameters should not be included in `iscsid.conf` file and per target authentication parameters need to be specified because Longhorn local iSCSI target is not using any authentication.

iSCSI initiator configuration in `node1-node4` have been automated with Ansible developing the ansible role: **ricsanfre.iscsi_initiator**.

## iSCSI configuration. Centralized SAN

`node1-node4` are configured as iSCSI Initiator to use iSCSI volumes exposed by `gateway`

iSCSI configuration in `node1-node4`and iSCSI LUN mount and format tasks have been automated with Ansible developing a couple of ansible roles: **ricsanfre.storage** for managing LVM and **ricsanfre.iscsi_initiator** for configuring a iSCSI initiator.

Further details about iSCSI configurations and step-by-step manual instructions are defined [here](/docs/san/).

Each node add the iSCSI LUN exposed by `gateway` to a LVM Volume Group and create a unique Logical Volume which formatted (ext4) and mounted as `/storage`.

Specific `node1-node4` ansible variables to be used by these roles are stored in [`vars/centralized_san/centralized_san_initiator.yml`]({{ site.git_edit_address }}/vars/centralized_san/centralized_san_initiator.yml)

{{site.data.alerts.important}}

`open-iscsi` is used by Longhorn as a mechanism to expose Volumes within Kuberentes cluster. Authentication default parameters should not be included in `iscsid.conf` file and per target authentication parameters need to be specified because Longhorn local iSCSI target is not using any authentication.

{{site.data.alerts.end}}
