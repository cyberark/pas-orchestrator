pipeline {
  agent {
    node {
      label 'ansible'
    }
  }
  environment {
    AWS_REGION = sh(script: 'curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | python3 -c "import json,sys;obj=json.load(sys.stdin);print (obj[\'region\'])"', returnStdout: true).trim()
    // shortCommit = sh(script: "git log -n 1 --pretty=format:'%h'", returnStdout: true).trim()
    CYBERARK_VERSION = "v12.2"
    ENV_TIMESTAMP = sh(script: "date +%s", returnStdout: true).trim()
  }
  stages {
    stage('Install virtual environment') {
      steps {
        sh '''
            python3 -m pip install --user virtualenv
            python3 -m virtualenv .testenv
            source .testenv/bin/activate
            pip install -r requirements.txt
            pip install -r tests/requirements.txt
        '''
      }
    }
    stage('yamllint validation') {
      steps {
        sh '''
            source .testenv/bin/activate
            yamllint .
        '''
      }
    }
    stage('Install ansible roles') {
      steps {
        sh '''
          source .testenv/bin/activate
          ansible-galaxy install -r requirements.yml
        '''
      }
    }
    stage('Download packages') {
      steps {
        withCredentials([
          string(credentialsId: 'default_packages_bucket', variable: 'default_packages_bucket')
        ]) {
          dir ('/tmp/packages') {
            s3Download(file:'/tmp/packages/psm.zip', bucket:"$default_packages_bucket", path:"Packages/${env.CYBERARK_VERSION}/Privileged Session Manager-Rls-${env.CYBERARK_VERSION}.zip", pathStyleAccessEnabled: true, force:true)
            s3Download(file:'/tmp/packages/cpm.zip', bucket:"$default_packages_bucket", path:"Packages/${env.CYBERARK_VERSION}/Central Policy Manager-Rls-${env.CYBERARK_VERSION}.zip", pathStyleAccessEnabled: true, force:true)
            s3Download(file:'/tmp/packages/pvwa.zip', bucket:"$default_packages_bucket", path:"Packages/${env.CYBERARK_VERSION}/Password Vault Web Access-Rls-${env.CYBERARK_VERSION}.zip", pathStyleAccessEnabled: true, force:true)
          }
        }
      }
    }
    stage('Deploy Environments') {
      parallel {
        stage('Deploy Environment for TC1') {
          stages {
            stage('Deploy Vault') {
              steps {
                withCredentials([
                  usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                  string(credentialsId: 'default_keypair', variable: 'default_keypair'),
                  string(credentialsId: 'default_s3_bucket', variable: 'default_s3_bucket')
                ]) {
                  sh '''
                    source .testenv/bin/activate
                    ansible-playbook tests/playbooks/deploy_vault.yml -v -e "keypair=$default_keypair bucket=$default_s3_bucket ansible_user=$ansible_user ansible_password=$ansible_password tc_number=1 env_timestamp=$ENV_TIMESTAMP"
                  '''
                }
              }
            }
            stage('Provision in-domain testing environment') {
              steps {
                withCredentials([
                  usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                  string(credentialsId: 'default_keypair', variable: 'default_keypair')
                ]) {
                  sh '''
                    source .testenv/bin/activate
                    ansible-playbook tests/playbooks/pas-infrastructure/ec2-infrastructure.yml -e "aws_region=$AWS_REGION keypair=$default_keypair ec2_instance_type=m4.large public_ip=no pas_count=1 indomain=yes tc_number=1 ansible_user=$ansible_user ansible_password=$ansible_password env_timestamp=$ENV_TIMESTAMP"
                  '''
                }
              }
            }
            stage('Run pas-orchestrator in-domain #0 failure (TC5)') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user')]) {
                  sh '''
                    source .testenv/bin/activate
                    VAULT_IP=$(cat /tmp/vault_ip_tc_1.txt)
                    cp -r tests/playbooks/pas-infrastructure/outputs/hosts_tc_1.yml inventories/staging/hosts_tc_1.yml
                  '''
                  catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                      source .testenv/bin/activate
                      ansible-playbook pas-orchestrator.yml -i inventories/staging/hosts_tc_1.yml -v -e "accept_eula=yes vault_ip=$VAULT_IP vault_password='blahblah' cpm_zip_file_path=/tmp/packages/cpm.zip psm_zip_file_path=/tmp/packages/psm.zip pvwa_zip_file_path=/tmp/packages/pvwa.zip connect_with_rdp=Yes ansible_user='cyberark.com\\\\$ansible_user' ansible_password=$ansible_password cpm_username=TcOne"
                    '''
                  }
                }
              }
            }
            stage('Run pas-orchestrator in-domain #1') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user')]) {
                  sh '''
                    source .testenv/bin/activate
                    VAULT_IP=$(cat /tmp/vault_ip_tc_1.txt)
                    cp -r tests/playbooks/pas-infrastructure/outputs/hosts_tc_1.yml inventories/staging/hosts_tc_1.yml
                    ansible-playbook pas-orchestrator.yml -i inventories/staging/hosts_tc_1.yml -v -e "accept_eula=yes vault_ip=$VAULT_IP vault_password=$ansible_password psm_hardening=false cpm_zip_file_path=/tmp/packages/cpm.zip psm_zip_file_path=/tmp/packages/psm.zip pvwa_zip_file_path=/tmp/packages/pvwa.zip connect_with_rdp=Yes ansible_user='cyberark.com\\\\$ansible_user' ansible_password=$ansible_password cpm_username=TcOne"
                  '''
                }
              }
            }
            stage('Run pas-orchestrator in-domain #2 (TC4)') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user')]) {
                  sh '''
                    source .testenv/bin/activate
                    VAULT_IP=$(cat /tmp/vault_ip_tc_1.txt)
                    cp -r tests/playbooks/pas-infrastructure/outputs/hosts_tc_1.yml inventories/staging/hosts_tc_1.yml
                    ansible-playbook pas-orchestrator.yml -i inventories/staging/hosts_tc_1.yml -v -e "accept_eula=yes vault_ip=$VAULT_IP vault_password=$ansible_password psm_hardening=false cpm_zip_file_path=/tmp/packages/cpm.zip psm_zip_file_path=/tmp/packages/psm.zip pvwa_zip_file_path=/tmp/packages/pvwa.zip connect_with_rdp=Yes ansible_user='cyberark.com\\\\$ansible_user' ansible_password=$ansible_password cpm_username=TcOne"
                  '''
                }
              }
            }
          }
        }
        stage('Deploy Environment for TC2') {
          stages {
            stage('Deploy Vault') {
              steps {
                sleep 10
                withCredentials([
                  usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                  string(credentialsId: 'default_keypair', variable: 'default_keypair'),
                  string(credentialsId: 'default_s3_bucket', variable: 'default_s3_bucket')
                ]) {
                  sh '''
                    source .testenv/bin/activate
                    ansible-playbook tests/playbooks/deploy_vault.yml -v -e "keypair=$default_keypair bucket=$default_s3_bucket ansible_user=$ansible_user ansible_password=$ansible_password tc_number=2 env_timestamp=$ENV_TIMESTAMP"
                  '''
                }
              }
            }
            stage('Deploy Vault DR') {
              steps {
                withCredentials([
                  usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                  string(credentialsId: 'default_keypair', variable: 'default_keypair'),
                  string(credentialsId: 'default_s3_bucket', variable: 'default_s3_bucket')
                ]) {
                  sh '''
                    VAULT_IP=$(cat /tmp/vault_ip_tc_2.txt)
                    source .testenv/bin/activate
                    ansible-playbook tests/playbooks/deploy_vaultdr.yml -v -e "keypair=$default_keypair bucket=$default_s3_bucket ansible_user=$ansible_user ansible_password=$ansible_password tc_number=2 vault_ip=$VAULT_IP env_timestamp=$ENV_TIMESTAMP"
                  '''
                }
              }
            }
            stage('Provision in-domain testing environment') {
              steps {
                withCredentials([
                  usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                  string(credentialsId: 'default_keypair', variable: 'default_keypair')
                ]) {
                  sh '''
                    source .testenv/bin/activate
                    ansible-playbook tests/playbooks/pas-infrastructure/ec2-infrastructure.yml -e "aws_region=$AWS_REGION keypair=$default_keypair ec2_instance_type=m4.large public_ip=no pas_count=5 indomain=yes tc_number=2 ansible_user=$ansible_user ansible_password=$ansible_password env_timestamp=$ENV_TIMESTAMP"
                  '''
                }
              }
            }
            stage('Run pas-orchestrator in-domain #1') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user')]) {
                  sh '''
                    source .testenv/bin/activate
                    VAULT_IP=$(cat /tmp/vault_ip_tc_2.txt)
                    VAULT_DR_IP=$(cat /tmp/vaultdr_ip_tc_2.txt)
                    cp -r tests/playbooks/pas-infrastructure/outputs/hosts_tc_2.yml inventories/staging/hosts_tc_2.yml
                    ansible-playbook pas-orchestrator.yml -i inventories/staging/hosts_tc_2.yml -v -e "accept_eula=yes vault_ip=$VAULT_IP dr_vault_ip=$VAULT_DR_IP vault_password=$ansible_password cpm_zip_file_path=/tmp/packages/cpm.zip psm_zip_file_path=/tmp/packages/psm.zip pvwa_zip_file_path=/tmp/packages/pvwa.zip pvwa_installation_drive=\'D:\' cpm_installation_drive=\'D:' psm_installation_drive=\'D:\' connect_with_rdp=Yes ansible_user='cyberark.com\\\\$ansible_user' ansible_password=$ansible_password"
                  '''
                }
              }
            }
            stage('Run pas-orchestrator in-domain #2 (TC4)') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user')]) {
                  sh '''
                    source .testenv/bin/activate
                    VAULT_IP=$(cat /tmp/vault_ip_tc_2.txt)
                    VAULT_DR_IP=$(cat /tmp/vaultdr_ip_tc_2.txt)
                    cp -r tests/playbooks/pas-infrastructure/outputs/hosts_tc_2.yml inventories/staging/hosts_tc_2.yml
                    ansible-playbook pas-orchestrator.yml -i inventories/staging/hosts_tc_2.yml -v -e "accept_eula=yes vault_ip=$VAULT_IP dr_vault_ip=$VAULT_DR_IP vault_password=$ansible_password cpm_zip_file_path=/tmp/packages/cpm.zip psm_zip_file_path=/tmp/packages/psm.zip pvwa_zip_file_path=/tmp/packages/pvwa.zip pvwa_installation_drive=\'D:\' cpm_installation_drive=\'D:' psm_installation_drive=\'D:\' connect_with_rdp=Yes ansible_user='cyberark.com\\\\$ansible_user' ansible_password=$ansible_password"
                  '''
                }
              }
            }
          }
        }
        stage('Deploy Environment for TC3') {
          stages {
            stage('Deploy Vault') {
              steps {
                sleep 20
                withCredentials([
                  usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                  string(credentialsId: 'default_keypair', variable: 'default_keypair'),
                  string(credentialsId: 'default_s3_bucket', variable: 'default_s3_bucket')
                ]) {
                  sh '''
                    source .testenv/bin/activate
                    ansible-playbook tests/playbooks/deploy_vault.yml -v -e "keypair=$default_keypair bucket=$default_s3_bucket ansible_user=$ansible_user ansible_password=$ansible_password tc_number=3 env_timestamp=$ENV_TIMESTAMP"
                  '''
                }
              }
            }
            stage('Deploy Vault DR') {
              steps {
                withCredentials([
                  usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                  string(credentialsId: 'default_keypair', variable: 'default_keypair'),
                  string(credentialsId: 'default_s3_bucket', variable: 'default_s3_bucket')
                ]) {
                  sh '''
                    VAULT_IP=$(cat /tmp/vault_ip_tc_3.txt)
                    source .testenv/bin/activate
                    ansible-playbook tests/playbooks/deploy_vaultdr.yml -v -e "keypair=$default_keypair bucket=$default_s3_bucket ansible_user=$ansible_user ansible_password=$ansible_password tc_number=3 vault_ip=$VAULT_IP env_timestamp=$ENV_TIMESTAMP"
                  '''
                }
              }
            }
            stage('Provision out-of-domain testing environment') {
              steps {
                withCredentials([
                  usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                  string(credentialsId: 'default_keypair', variable: 'default_keypair')
                ]) {
                  sh '''
                    source .testenv/bin/activate
                    ansible-playbook tests/playbooks/pas-infrastructure/ec2-infrastructure.yml -e "aws_region=$AWS_REGION keypair=$default_keypair ec2_instance_type=m4.large public_ip=no pas_count=1 indomain=no tc_number=3 ansible_user=$ansible_user ansible_password=$ansible_password env_timestamp=$ENV_TIMESTAMP"
                  '''
                }
              }
            }
            stage('Run pas-orchestrator out-of-domain') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user')]) {
                  sh '''
                    source .testenv/bin/activate
                    VAULT_IP=$(cat /tmp/vault_ip_tc_3.txt)
                    VAULT_DR_IP=$(cat /tmp/vaultdr_ip_tc_3.txt)
                    cp -r tests/playbooks/pas-infrastructure/outputs/hosts_tc_3.yml inventories/staging/hosts_tc_3.yml
                    ansible-playbook pas-orchestrator.yml -i inventories/staging/hosts_tc_3.yml -v -e "accept_eula=yes vault_ip=$VAULT_IP dr_vault_ip=$VAULT_DR_IP vault_password=$ansible_password pvwa_hardening=false cpm_hardening=false psm_hardening=false psm_out_of_domain=true cpm_zip_file_path=/tmp/packages/cpm.zip psm_zip_file_path=/tmp/packages/psm.zip pvwa_zip_file_path=/tmp/packages/pvwa.zip connect_with_rdp=Yes ansible_user='$ansible_user' ansible_password=$ansible_password cpm_username=VeryLongNameRealyIAmVeryLongAskAnybody"
                  '''
                }
              }
            }
          }
        }
      }
    }
  }
  post('Archiving artifacts and Cleanup') {
    always {
      archiveArtifacts artifacts: 'inventories/staging/hosts_tc_*.yml', fingerprint: true
      archiveArtifacts artifacts: 'logs/**/*.log', fingerprint: true
      archiveArtifacts artifacts: 'logs/ansible.log', fingerprint: true
    }
    cleanup {
      sh '''
        source .testenv/bin/activate

        # Terminate EC2 instances
        instance_ids=$(aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[].InstanceId' --filters "Name=tag:Timestamp,Values=$ENV_TIMESTAMP" --output text)
        aws ec2 terminate-instances --region $AWS_REGION --instance-ids $instance_ids
        aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $instance_ids

        instance_ids=$(aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[].InstanceId' --filters "Name=tag:aws:cloudformation:stack-name,Values=$(cat /tmp/cf_vault_tc_1.txt)" --output text)
        aws ec2 terminate-instances --region $AWS_REGION --instance-ids $instance_ids
        aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $instance_ids

        instance_ids=$(aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[].InstanceId' --filters "Name=tag:aws:cloudformation:stack-name,Values=$(cat /tmp/cf_vault_tc_2.txt)" --output text)
        aws ec2 terminate-instances --region $AWS_REGION --instance-ids $instance_ids
        aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $instance_ids

        instance_ids=$(aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[].InstanceId' --filters "Name=tag:aws:cloudformation:stack-name,Values=$(cat /tmp/cf_vaultdr_tc_2.txt)" --output text)
        aws ec2 terminate-instances --region $AWS_REGION --instance-ids $instance_ids
        aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $instance_ids

        instance_ids=$(aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[].InstanceId' --filters "Name=tag:aws:cloudformation:stack-name,Values=$(cat /tmp/cf_vault_tc_3.txt)" --output text)
        aws ec2 terminate-instances --region $AWS_REGION --instance-ids $instance_ids
        aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $instance_ids

        instance_ids=$(aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[].InstanceId' --filters "Name=tag:aws:cloudformation:stack-name,Values=$(cat /tmp/cf_vaultdr_tc_3.txt)" --output text)
        aws ec2 terminate-instances --region $AWS_REGION --instance-ids $instance_ids
        aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $instance_ids

        # Delete security groups
        sleep 60
        aws ec2 describe-security-groups --region $AWS_REGION --query 'SecurityGroups[*].{ID:GroupId}' --filters "Name=tag:Timestamp,Values=$ENV_TIMESTAMP" --output text | awk '{print $1}' | while read line; do aws ec2 delete-security-group --region $AWS_REGION --group-id $line; done

        # Delete Vault Cloudformations
        aws cloudformation delete-stack --region $AWS_REGION --stack-name $(cat /tmp/cf_vault_tc_1.txt)
        aws cloudformation delete-stack --region $AWS_REGION --stack-name $(cat /tmp/cf_vaultdr_tc_2.txt)
        aws cloudformation delete-stack --region $AWS_REGION --stack-name $(cat /tmp/cf_vault_tc_2.txt)
        aws cloudformation delete-stack --region $AWS_REGION --stack-name $(cat /tmp/cf_vaultdr_tc_3.txt)
        aws cloudformation delete-stack --region $AWS_REGION --stack-name $(cat /tmp/cf_vault_tc_3.txt)
      '''
    }
  }
}
