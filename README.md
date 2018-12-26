# pas-orchestrator

This Playbook will orchestrate the CyberArk cpm/pvwa/psm products on a Windows 2016 server / VM / instance

Requirements
------------

- Windows 2016 must be installed on the servers
- Administrator credentials (either Local or Domain)
- Network connection to the vault and the repository server
- Location of psm CD image
- PAS packages version 10.6 and above
- IP addresses / hosts to orchestrate the products to


## Role Variables

A list of variables used in this playbook

**Deployment Variables**

| Variable                         | Required     | Default                                                                        | Comments                                 |
|----------------------------------|--------------|--------------------------------------------------------------------------------|------------------------------------------|
| vault_ip                         | yes          | None                                                                           | Vault ip to perform registration         |
| dr_vault_ip                      | no           | None                                                                           | vault dr ip to perform registration      |
| vault_port                       | no           | 1858                                                                           | vault port                               |
| vault_username                   | no           | "administrator"                                                                | vault username to perform registration   |
| vault_password                   | yes          | None                                                                           | vault password to perform registration   |
| accept_eula                      | yes          | "No"                                                                           | Accepting EULA condition                 |
| psm_disable_nla                  | yes          | "No"                                                                           | This will disable NLA on the server      |
| cpm_zip_file_path                | yes          | None                                                                           | Path to zipped CPM image                 |
| pvwa_zip_file_path               | yes          | None                                                                           | Path to zipped PVWA image                |
| psm_zip_file_path                | yes          | None                                                                           | Path to zipped PSM image                 |


Variables related to the components can be found on the Components README

## Usage

The Role consists of two parts, each part runs independently:

**Part 1 - Components Deployment**

The task will trigger the components main roles, each role will trigger it's sub tasks (prerequisities/installation, etc.)
by default, all tasks are set to true except registration.
This process runs all tasks on all hosts parallely, causing reduction in deployment time

*IMPORTANT: Component Registration should be always set to false in this phase

**Part 2 - Components Registration**

This task will executes registration process of the components, all the previous tasks are set to false and only registration is enabled
This process execute the each registration in serial, one registration at a time

## Inventory

Inventory consists of a group of variables:

    ---
    windows:
      children:
        pvwa:
          hosts:
            1.2.3.4;
            1.2.3.14:
            1.2.3.24:
        cpm:
          hosts:
            2.2.2.2;
            2.2.2.22;
            2.2.2.222;
        psm:
          hosts:
            9.8.7.6;
            5.4.3.2;
            9.1.7.3;


## Running the  playbook:

To run the above playbook execute the following command:

    ansible-playbook -i ./inventories/hosts.yml pas-orchestrator.yml -e "vault_ip=VAULT_IP vault_password=VAULT_PASSWROD ansible_user=DOMAIN\USER ansible_password=DOAMIN_PASSWORD cpm_zip_file_path=/tmp/pas_packages/cpm.zip pvwa_zip_file_path=/tmp/pas_packages/pvwa.zip psm_zip_file_path=/tmp/pas_packages/psm.zip psm_out_of_domain=false accept_eula=yES"

## License

Apache 2
