# Ansible Playbook for Configuration Management

This Ansible playbook includes three primary roles for managing network devices: configuring baseline settings, performing firmware upgrades, and backing up configuration files.

## Setup

To switch to the CM service account, enter the command `sudo -u svc.ansible -i`.
To activate the enviroment, enter the command `ansible`. This will change to the correct directory and activate the virtual environment. 
Now the environment is ready to run playbooks.

## Prerequisites

ansible.netcommon.cli_backup requires netcommon 5 or greater, which requires ansible-core 2.14 or higher.

Please see requirements.txt for requirements.

```bash
ansible-galaxy collection list | grep netcommon
ansible-galaxy collection install ansible.netcommon
```

## Support

For any issues or questions regarding this playbook, please reach out to the Configuration Management Team.

## Inventories

Inventory files list the groups of devices that ansible will interact with. The inventory files are broken down by AOR. 
Inventory files are located in the `/inventory` directory. You can specify which inventory file to use with the `-i` option. For example:

### Inventory file usage 

To combine all inventory files:

```bash
ansible-playbook -i inventory networkBackup.yml
```

To use an AOR specific inventory file:

```bash
ansible-playbook -i inventory/inventory1.yml networkBackup.yml
```

### Limit switch usage for groups of devices

Individual devices, groups of devices, or multiple device and groups can be run with the `-l` option. 

Single device:
```bash
ansible-playbook -i inventory/inventory1.yml -l Gateway1 networkBackup.yml
```

Single group:
```bash
ansible-playbook -i inventory/inventory1.yml -l connection_settings_ios networkBackup.yml
```

Multiple devices:
```bash
ansible-playbook -i inventory/inventory1.yml -l Gateway1,Gateway2 networkBackup.yml
```

## Credentials

Anisble uses the CM service account tacacs credentials to connect to network devices. These credentials are stored in group_vars/<group>/vars.yml as `ansible_user` and `ansible_password`.
The actual values of these variables are encrypted and stored in the group_vars/<group>/vault.yml file. 

To view the contents of the file, use the command: 

```bash
ansible-vault view group_vars/inventory1/vault.yml
```

To edit the credentials, use the command:

```bash
ansible-vault edit group_vars/inventory1/vault.yml
```

Ansible cannot natively decrypt the vault files, so cannot pass the credentials to the switch or router. 

Typically, to decrypt the file and pass the credentials to the playbook, a user would run the command:

```bash 
ansible-playbook -i inventory networkBackup.yml --ask-vault-pass 
```

The user would them be prompted for the vault password, and ansible will decrypt the file, passing the tacacs credentials. 

This limits automation, so instead we are using a password file. Ansible will automatically use the password file to decrypt the file with no additional parameters needed.

The password file is located in the home directory of the CM service account with 0600 permissions. 

`-rw-------. 1 svc.ansible svc.ansible    17 Mar 14 08:30 .vault_pass`

The location of the .vault_pass is set in the ansible.cfg file.

```ini  
vault_password_file = /home/svc.ansible/.vault_pass
```

# Roles

## Baseline

### Overview

Confgures baseline settings on the devices. 

The networkBaseline.yml playbook kicks off the baseline process by setting up variables and importing template variables. Next it starts the /roles/networkBaseline role, which starts the templating process. Once the template is created, it is sent to the device via SCP and loading to the running configuration. 

The template is in jinja2 format. Variables are imported from the roles/networkBaseline/vars folder, group_vars folder, and host_vars folder. 

To test the build process, use the tag --build.

### Execution

Baselining Up All Devices

To initiate a baseline for all network devices listed in your inventory, execute the following command:

```bash
ansible-playbook -i inventory networkBaseline.yml
```

This command triggers the baseline process for every device defined in the specified inventory directory.

Targeted Baseline for a Specific Area of Responsibility (AOR)

In scenarios where you need to baseline devices within a specific AOR, you can target the baseline process by specifying an AOR-specific inventory file. Use the `-i` flag followed by the path to the AOR's inventory file:

```bash
ansible-playbook -i inventory/<inventory_file>.yml networkBaseline.yml
```

To baseline a single device or group, use the `'l` switch:

```bash
ansible-playbook -i inventory/<aor_inventory_file>.yml -l ANS-GW1,ANS-GW2 networkBaseline.yml
```

![template schema](img/template.PNG)

### File Structure

```bash
.
├── group_vars
│   ├── connection_settings_ios.yml
│   └── connection_settings_nxos.yml
├── host_vars
│   ├── deviceA.yml
│   ├── deviceB.yml
│   ├── deviceC.yml
│   └── devoceD.yml
├── inventory
│   ├── inventory1.yml
│   └── inventory2.yml
├── networkBaseline.yml
├── README.md
├── requirements.txt
├── roles
│   ├── networkBaseline
│   │   ├── backups
│   │   │   └── staged
│   │   │       └── 2024-03-07
│   │   │           └── device.cfg # this will contain a backup of the current device config, and a copy of the template that will be pushed. This is only for historical pushes and serves as a way to troubleshoot what was pushed to the device.
│   │   ├── handlers
│   │   │   └── main.yml   # the handlers will handle device specifc commands, such as write memory
│   │   ├── staging
│   │   │   └── device.cfg # this is where the template is stored after creation. this directory is cleaned after completion.
│   │   ├── tasks
│   │   │   ├── baseline_ios.yml
│   │   │   ├── baseline_nxos.yml
│   │   │   ├── deployBaseline.yml
│   │   │   └── main.yml
│   │   ├── templates
│   │   │   ├── ios_config.j2
│   │   │   ├── junos_clean_config.j2
│   │   │   ├── junos_config.j2
│   │   │   ├── nxos_config.j2
│   │   │   └── snmp_contact.j2
│   │   └── vars
│   │       ├── acl_ext_100.yml
│   │       ├── acl_ext_200.yml
│   │       ├── acl_ext_300.yml
│   │       ├── acl_std_10.yml
│   │       └── main.yml
```

## Upgrade

### Overview

Performs firmware upgrades.

This playbook is executed by running the neworkUpgrade.yml playbook, which starts the networkUpgrade role. 

The networkUpgrade playbook performs the following actions:

- checks filesystem to ensure firmware does not exist on device
- transfers the firmware file to flash via SCP
- validates md5 checksum

variables used:

datetime
image_path
firmware_filename
firmware_hash

If the firmware packages (ie `cat4500e-universalk9.SPA.03.11.10.E.152-7.E10.bin`) are not in the configmgmt directory, you can add them to the `roles/networkUpgrade/files` directory. The variable for the image path is set in the networkUpgrade.yml playbook as image_path variable. 

Run with:

```bash
ansible-playbook -i inventory networkUpgrade.yml
```


## Backup

### Overview

This section outlines the procedure for utilizing the Ansible playbook designed for backing up network configuration files. The playbook employs a backup role that ensures configuration files are securely copied to two distinct destinations: a timestamped directory and a "latest" directory for easy access to the most recent backup.

The backup process involves creating a new directory each day, named in a date format (YYYY-MM-DD), to store that day's configuration file backups. This method ensures that each day's backups are uniquely preserved. Simultaneously, the backup files are also saved to a directory named "latest," which serves as a centralized location for accessing the most recent network configuration files. It is important to note that the contents of the "latest" directory are updated daily, with older files being replaced by newer backups.

![backup schema](img/backups.PNG)

### Execution

Backing Up All Devices

To initiate a backup for all network devices listed in your inventory, execute the following command:

```bash
ansible-playbook -i inventory networkBackup.yml
```
This command triggers the backup process for every device defined in the specified inventory directory.


Targeted Backup for a Specific Area of Responsibility (AOR)

In scenarios where you need to back up configurations for devices within a specific AOR, you can target the backup process by specifying an AOR-specific inventory file. Use the -i flag followed by the path to the AOR's inventory file:

```bash
ansible-playbook -i inventory/inventory1.yml networkBackup.yml
```

### File Structure

```bash
.
├── inventory
├── networkBackup.yml   #this conducts the pretasks of creating and varifying backup locations, then calls the networkBackup role
│
└── roles
    └── networkBackup
        └── tasks
             └── main.yml   #this performs the actual backup of each device
```

## Directory 

This is the proposed structure - following the default logic ansible uses to parse playbooks:
```bash
├── group_vars                  # this is where group specific variables are placed - they can include variables for AORs, device platform, model, and function, as well as any connection methods 
│   ├── AOR-A.yml
│   ├── AOR-B.yml
│   └── connection_settings_ios.yml
├── host_vars
│   ├── deviceA.yml
│   ├── deviceB.yml
│   ├── deviceC.yml
│   └── devoceD.yml
├── inventory
│   ├── inventory1.yml
│   └── inventory2.yml                  # this is the inventory file. it should be kept as clean as possible to allow for maximum readable.
├── networkBackup.yml                    # this is the master playbook that calls the roles. 
├── networkBaseline.yml 
├── networkUpgrade.yml 
├── README.md
└── roles                       # this is the default structure ansible parses when running playbooks. the roles contain the template modules and the templates themselves. 
    ├── networkBaseline                    # this is the baseline role. it uses a device agnostic method to connect to devices and uses conditional logic to apply the correct template based on the group_vars the device is assigned to
    │   ├── defaults            # default variables can be set here if necessary, but currently unused
    │   │   └── main.yml
    │   ├── handlers            # unuesd
    │   │   └── main.yml
    │   ├── meta                # unused
    │   │   └── main.yml
    │   ├── README.md
    │   ├── tasks               # this is the default section ansible uses to run the playbooks
    │   │   └── main.yml    
    │   ├── templates           #this contains the actual baselining jinja templates. templates are separated by syntax. minor differences in syntax can be overcome by using jinja if statements and group variables. group specific items (for example aor specific items, are pulled from group_vars)
    │   │   ├── cisco_config.j2
    │   │   └── junos_config.j2
    │   ├── tests
    │   │   └── test.yml        # can be later developed for unit testing
    │   └── vars
    │       └── main.yml        # ROLE SPECIFIC variables go here - for example configuration backup directories, date-time variables, anything related to the ROLE itself
    ├
    ├── networkBackup         # a possible role that can be used to perform firmware upgrades. again each role is developed by function
    │   ├── defaults
    │   │   └── main.yml
    │   ├── files
    │   ├── handlers
    │   │   └── main.yml
    │   ├── meta
    │   │   └── main.yml
    │   ├── README.md
    │   ├── tasks
    │   │   └── main.yml
    │   ├── templates
    │   ├── tests
    │   │   ├── inventory
    │   │   └── test.yml
    │   └── vars
    │       └── main.yml
    └── networkUpgrade                # a possible role to query devices for device information (akin to running show commands, but returns structured output instead of ios tables)
        ├── defaults
        │   └── main.yml
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── README.md
        ├── tasks
        │   └── main.yml
        ├── tests
        │   ├── inventory
        │   └── test.yml
        └── vars
            └── main.yml
```

## Robot Framework 

To generate the docs for robot framework librarys run this command:

```bash
python -m robot.libdoc genie.libs.robot.GenieRobot geniedoc.html
python -m robot.libdoc ats.robot.pyATSRobot robotdoc.html
```
These can also be found at:

https://pubhub.devnetcloud.com/media/pyats/docs/robot.html
https://pubhub.devnetcloud.com/media/genie-docs/docs/userguide/robot.html

if using unicon connect, see unicon keywords:

https://pubhub.devnetcloud.com/media.unicon/docs/robot.html

## References

### Robot 

https://pubhub.devnetcloud.com/media/unicon/docs/robot/index.html#
https://github.com/robotframework/QuickStartGuide/blob/master/QuickStart.rst

### Pyats 

To run an arbitrary command using pyats and genie parser run the command:

```bash
genie learn config --testbed-file robot/test/tb.yml --device <device name> --output device_output.txt
```
Parsers are at

lib64/python3.11/site-packages/genie/libs/parser/iosxe/

Note - Device name must match hostname when using pyats

### Ansible 

https://docs.ansible.com/ansible/latest/collections/junipernetworks/junos/junos_config_module.html#ansible-collections-junipernetworks-junos-junos-config-module


## Tools

To check the ansible configuration file run this command:

```bash
ansible-config dump --only-changed -t connection
```
