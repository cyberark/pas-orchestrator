# EC2-PAS-Infrastructure

This Playbook provisions an infrastructure on AWS to prepare a DC environment and Domain machines for PAS-Orchestrator

Requirements
------------

- Security Group on the Target VPC with all the neccesary ports open for Domain Controller and WinRM.
- Pip libraries: `jq`, `yq`, `json2yaml`:
    `pip install jq yq json2yaml --user`

## Role Variables

A list of vaiables the playbook is using

| Variable             | Comments           |
|----------------------|--------------------|
| aws_region           | AWS Region         |
| keypair              | KeyPair            |
| ec2_instance_type    | Instance Type      |
| public_ip            | Public Ip Yes/No   |
| subnet_id            | Subnet ID          |
| security_group       | Security Group ID  |
| win2012_ami_id       | AMI for DC         |
| win2016_ami_id       | AMI for PAS EC2    |
| pas_count            | Number of Machines |
| ansible_user         | Ansible User       |
| ansible_password     | Ansible Password   |
| indomain             | Yes/No             |
| comp_sg              | sg-xxxxxx          |
| dc_sg                | sg-xxxxxx          |

## Running the playbook:

To run the above playbook:

    ansible-playbook ec2-infrastructure.yml -e "aws_region=my-region keypair=My-KP ec2_instance_type=t2.size public_ip=yes/no subnet_id=subnet-xxxxxx security_group=sg-xxxxxx win2012_ami_id=ami-xxxxxx win2016_ami_id=ami-xxxxxx dc_sg=sg-xxxxxx comp_sg=sg-xxxxxx pas_count=10 ansible_user=Administrator ansible_password=nopass when: indomain=yes"

## Outputs:

You will get a hosts file in `outputs/hosts.yml` you can use for the PAS-Orchestrator.
When using PAS-Orchestrator, you will see the relevant host groups on: `tag_Type_pvwa`, `tag_Type_cpm`, `tag_Type_psm`.

## License

 **TBD**
