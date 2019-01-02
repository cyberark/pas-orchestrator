# pas-orchestrator

In today’s modern infrastructure, organizations are moving towards hybrid environments, which consist of multiple public clouds, private clouds and on-premise platforms. 

CyberArk has created a tailored installation and deployment method for each platform to enable easy implementation. For example, CloudFormation templates enable easy deployment on AWS, while Azure Resource Manager (ARM) templates enable easy deployment on Azure. However, it is difficult to combine the different methods to orchestrate and automate a hybrid deployment.

PAS Orchestrator is a set of Ansible Roles which provides a holistic solution to deploying CyberArk Core PAS components simultaneously in multiple environments, regardless of the environment’s location. 

The Ansible Roles are responsible for the entire deployment process, and can be integrated with the organization’s CI/CD pipeline.

Each PAS component’s Ansible Role is responsible for the component end-2-end deployment, which inclues the following stages for each component:
- Copy the installation package to the target server
- Installing prerequisites
- Installing the component silently
- Post installation procedure and hardening
- Registration in the Vault

Ansible Roles for PVWA, CPM and PSM can be found in the following links:
- PSM: https://github.com/cyberark/psm
- CPM: https://github.com/cyberark/cpm
- PVWA: https://github.com/cyberark/pvwa

The PAS Orchestrator role is an example of how to use the component roles 
demonstrating paralel installation on multiple remote servers 

Requirements
------------

- IP addresses / hosts to execute the playbook against with Windows 2016 installed on the remote hosts
- WinRM open on port 5986 (**not 5985**) on the remote host 
- Pywinrm is installed on the workstation running the playbook
- The workstation running the playbook must have network connectivity to the remote host
- The remote host must have Network connectivity to the CyberArk vault and the repository server
  - 443 port outbound
  - 443 port outbound (for PVWA only)
  - 1858 port outbound 
- Administrator access to the remote host 
- CyberArk components CD image on the workstation running the playbook 



## Role Variables

These are the variables used in this playbook

**Deployment Variables**

| Variable                         | Required     | Default                                                                        | Comments                                 |
|----------------------------------|--------------|--------------------------------------------------------------------------------|------------------------------------------|
| vault_ip                         | yes          | None                                                                           | Vault ip to perform registration         |
| dr_vault_ip                      | no           | None                                                                           | vault dr ip to perform registration      |
| vault_port                       | no           | 1858                                                                           | vault port                               |
| vault_username                   | no           | "administrator"                                                                | vault username to perform registration   |
| vault_password                   | yes          | None                                                                           | vault password to perform registration   |
| accept_eula                      | yes          | "No"                                                                           | Accepting EULA condition                 |
| connect_with_rdp                  | yes          | "No"                                                                           | This will disable NLA on the server      |
| cpm_zip_file_path                | yes          | None                                                                           | Path to zipped CPM image                 |
| pvwa_zip_file_path               | yes          | None                                                                           | Path to zipped PVWA image                |
| psm_zip_file_path                | yes          | None                                                                           | Path to zipped PSM image                 |


Variables related to the components can be found on the Components README

## Usage

The Role consists of two parts, each part runs independently:

**Part 1 - Components Deployment**

The task will trigger the components main roles, each role will trigger it's sub tasks (prerequisities/installation, etc.)
by default, all tasks are set to true except registration.
This process executes tasks on all hosts in parallel, reducing deployment time

*IMPORTANT: Component Registration should be always set to false in this phase

**Part 2 - Components Registration**

This task will execute the registration process of the components, all the previous tasks are set to false and only registration is enabled
This process executes the registration of each component in serial

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

    ansible-playbook -i ./inventories/hosts.yml pas-orchestrator.yml -e "vault_ip=VAULT_IP vault_password=VAULT_PASSWROD ansible_user=DOMAIN\USER ansible_password=DOMAIN_PASSWORD cpm_zip_file_path=/tmp/pas_packages/cpm.zip pvwa_zip_file_path=/tmp/pas_packages/pvwa.zip psm_zip_file_path=/tmp/pas_packages/psm.zip psm_out_of_domain=false accept_eula=Yes"
    
It is possible to set any of the roles parameter through the PAS Orchestrator  , 
For example : if you need to deploy the PSM without hardening add the following to the execution line: psm_hardening=false

## Troubleshooting
In case of a failure a Log folder with be created on the Ansible workstation with the relevant logs copied from the remote host machine. 


## Idempotence
Every stage in the roles contains validation and can be run multiple time without error .


## License

Apache 2
