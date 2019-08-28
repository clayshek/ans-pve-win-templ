# ans-pve-win-templ

## Summary

Ansible playbook to create Windows templates for Proxmox (PVE) based lab environment (somewhat customized for mine, but easy enough to tweak), prepped for further Ansible automation. Tested with Proxmox v6.0.1 & Ansible v2.8.1.

### Goals:
- Create a base Windows Server 2019 template image for [Proxmox VE](https://www.proxmox.com/en/proxmox-ve)
- Start with only a base ISO from Microsoft (no other components like MDT necessary)
- Apply necessary [VirtIO drivers](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/index.html#virtio-win-direct-downloads) (to run on Proxmox/KVM) during Windows Setup
- Pre-configure Windows image for Ansible to allow for further downstream configuration
- Apply latest Windows Updates as of image build
- Allow for minor customization to image via variables passed to either AutoUnattend.xml file, or to qemu provisioner (ex: name, resource specs, admin password, time zone, Windows image version, etc)
- Sysprep and templatize the image

## Usage

- Modify variables in vars.yml as appropriate
- Populate proxmox hosts in inventory.yml with the PVE hosts on which to create a template VM. If using shared storage, this only needs to be one host. 
- For secure vars (passwords) in both inventory.yml & vars.yml, as alternative to clear text, use Ansible Vault or pass variables on command line to ansible-playbook.

`ansible-playbook -i inventory.yml --ask-vault-pass provision-template.yml`

## Requirements

- Windows Server ISO (eval is fine) available to Proxmox node(s). Update 'os_iso_location' in vars.yml to reflect the location of the PVE storage and ISO file name.
- Additional VirtIO drivers added (/drivers/virtio folder) if necessary. Repo includes VirtIO SCSI, NIC (NetKVM), Balloon drivers and qemu guest agent.
- Genisoimage installed on Proxmox host (present by default)


## Why Not Packer? (And Other Misc Notes)

- Packer would be ideal solution for this, using the [Proxmox builder](https://www.packer.io/docs/builders/proxmox.html), which works great for an OS that can be supplied a kickstart config over http, but for Windows, to use the Unattend.xml automated setup capability, there are two issues:
    - Proxmox does not support floppy drives, so using a 2nd ISO / CDROM image is required (also used for VirtIO drivers)
    - The Packer Proxmox builder does not support (at time of writing) adding more than one ISO. [Related Github issue](https://github.com/hashicorp/packer/issues/7950).

- Unable to use Packer, next looked to use [Proxmox_KVM](https://docs.ansible.com/ansible/latest/modules/proxmox_kvm_module.html) Ansible module. However found multiple issues with this, including:
    - Bug related to Proxmox v6 version numbering scheme, which caused outright module failure. The issue & fix are in [this pull request](https://github.com/ansible/ansible/pull/59356), but at time of writing, still open & unmerged. Editing the proxmox_kvm.py as described in PR is a workaround
    - Multiple issues with the module's clone operation, incl: changes OSType back to Linux, does not allow disk resizing, does not allow modifying RAM or CPU resources, does not allow addition of another disk or CDROM, does not allow changes to network interface. Some of these are described in [this issue](https://github.com/ansible/ansible/issues/28277), and [this pull request](https://github.com/ansible/ansible/pull/49001/files).
    - Issues above, plus a few other minor ones, made using proxmox_kvm module an unattractive option; generally seems like both Packer builder and Ansible module aren't fully developed.

- Ultimately found solution by using [qm command](https://pve.proxmox.com/pve-docs/qm.1.html) line tool from Ansible command module.     

- There are a couple of 'Start-Sleep' calls in the AutoUnattend.xml file. The method is a bit [kludgy](https://www.merriam-webster.com/dictionary/kludge), but has been the only reliable method I've found to allow for 2 Windows Updates passes with reboots, and also ensure that the final phase for Sysprep is not interrupted. 

