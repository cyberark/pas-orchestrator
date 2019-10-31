pipeline {
  agent {
    node {
      label 'ansible'
    }
  }
  environment {
    AWS_REGION = sh(script: 'curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | python -c "import json,sys;obj=json.load(sys.stdin);print obj[\'region\']"', returnStdout: true).trim()
    // shortCommit = sh(script: "git log -n 1 --pretty=format:'%h'", returnStdout: true).trim()
    CYBERARK_VERSION = "v11.1"
    ENV_TIMESTAMP = sh(script: "date +%s", returnStdout: true).trim()
  }
  stages {
    stage('Install virtual environment') {
      steps {
        sh '''
            python -m pip install --user virtualenv
            python -m virtualenv --no-site-packages .testenv
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
    stage('Deploy Vaults') {
      parallel {
        stage('Deploy Vault for in-domain environment') {
            steps {
              withCredentials([
                usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                string(credentialsId: 'default_keypair', variable: 'default_keypair'),
                string(credentialsId: 'default_s3_bucket', variable: 'default_s3_bucket')
              ]) {
                sh '''
                  source .testenv/bin/activate
                  ansible-playbook tests/playbooks/deploy_vault.yml -v -e "keypair=$default_keypair bucket=$default_s3_bucket ansible_user=$ansible_user ansible_password=$ansible_password domain=yes env_timestamp=$ENV_TIMESTAMP"
                '''
              }
            }
        }
        stage('Deploy Vault for out-of-domain environment') {
            steps {
              sleep 10
              withCredentials([
                usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
                string(credentialsId: 'default_keypair', variable: 'default_keypair'),
                string(credentialsId: 'default_s3_bucket', variable: 'default_s3_bucket')
              ]) {
                sh '''
                  source .testenv/bin/activate
                  ansible-playbook tests/playbooks/deploy_vault.yml -v -e "keypair=$default_keypair bucket=$default_s3_bucket ansible_user=$ansible_user ansible_password=$ansible_password domain=no env_timestamp=$ENV_TIMESTAMP"
                '''
              }
            }
        }
      }
    }
    stage('Provision testing environments') {
      parallel {
        stage('Provision in-domain testing environment') {
          steps {
            withCredentials([
              usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
              string(credentialsId: 'default_keypair', variable: 'default_keypair')
            ]) {
              sh '''
                source .testenv/bin/activate
                ansible-playbook tests/playbooks/pas-infrastructure/ec2-infrastructure.yml -e "aws_region=$AWS_REGION keypair=$default_keypair ec2_instance_type=m4.large public_ip=no pas_count=1 indomain=yes ansible_user=$ansible_user ansible_password=$ansible_password env_timestamp=$ENV_TIMESTAMP"
              '''
            }
          }
        }
        stage('Provision out-of-domain testing environment') {
          steps {
            sleep 10
            withCredentials([
              usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user'),
              string(credentialsId: 'default_keypair', variable: 'default_keypair')
            ]) {
              sh '''
                source .testenv/bin/activate
                ansible-playbook tests/playbooks/pas-infrastructure/ec2-infrastructure.yml -e "aws_region=$AWS_REGION keypair=$default_keypair ec2_instance_type=m4.large public_ip=no pas_count=1 indomain=no ansible_user=$ansible_user ansible_password=$ansible_password env_timestamp=$ENV_TIMESTAMP"
              '''
            }
          }
        }
      }
    }
    stage('Run pas-orchestrator') {
      parallel {
        stage('Run pas-orchestrator in-domain') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user')]) {
              sh '''
                source .testenv/bin/activate
                VAULT_IP=$(cat /tmp/vault_ip_domain_yes.txt)
                cp -r tests/playbooks/pas-infrastructure/outputs/hosts_domain_yes.yml inventories/staging/hosts_domain_yes.yml
                ansible-playbook pas-orchestrator.yml -i inventories/staging/hosts_domain_yes.yml -v -e "accept_eula=yes vault_ip=$VAULT_IP vault_password=$ansible_password cpm_zip_file_path=/tmp/packages/cpm.zip psm_zip_file_path=/tmp/packages/psm.zip pvwa_zip_file_path=/tmp/packages/pvwa.zip connect_with_rdp=yes ansible_user='cyberark.com\\\\$ansible_user' ansible_password=$ansible_password"
              '''
            }
          }
        }
        stage('Run pas-orchestrator out-of-domain') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'default_vault_credentials', passwordVariable: 'ansible_password', usernameVariable: 'ansible_user')]) {
              sh '''
                source .testenv/bin/activate
                VAULT_IP=$(cat /tmp/vault_ip_domain_no.txt)
                cp -r tests/playbooks/pas-infrastructure/outputs/hosts_domain_no.yml inventories/staging/hosts_domain_no.yml
                ansible-playbook pas-orchestrator.yml -i inventories/staging/hosts_domain_no.yml -v -e "accept_eula=yes vault_ip=$VAULT_IP vault_password=$ansible_password {psm_out_of_domain:true} cpm_zip_file_path=/tmp/packages/cpm.zip psm_zip_file_path=/tmp/packages/psm.zip pvwa_zip_file_path=/tmp/packages/pvwa.zip connect_with_rdp=yes ansible_user='$ansible_user' ansible_password=$ansible_password"
              '''
            }
          }
        }
      }
    }
  }
  post('Archiving artifacts and Cleanup') {
    always {
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

        instance_ids=$(aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[].InstanceId' --filters "Name=tag:aws:cloudformation:stack-name,Values=$(cat /tmp/cf_vault_domain_yes.txt)" --output text)
        aws ec2 terminate-instances --region $AWS_REGION --instance-ids $instance_ids
        aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $instance_ids

        instance_ids=$(aws ec2 describe-instances --region $AWS_REGION --query 'Reservations[].Instances[].InstanceId' --filters "Name=tag:aws:cloudformation:stack-name,Values=$(cat /tmp/cf_vault_domain_no.txt)" --output text)
        aws ec2 terminate-instances --region $AWS_REGION --instance-ids $instance_ids
        aws ec2 wait instance-terminated --region $AWS_REGION --instance-ids $instance_ids

        # Delete security groups
        sleep 60
        aws ec2 describe-security-groups --region $AWS_REGION --query 'SecurityGroups[*].{ID:GroupId}' --filters "Name=tag:Timestamp,Values=$ENV_TIMESTAMP" --output text | awk '{print $1}' | while read line; do aws ec2 delete-security-group --region $AWS_REGION --group-id $line; done

        # Delete Vault Cloudformations
        aws cloudformation delete-stack --region $AWS_REGION --stack-name $(cat /tmp/cf_vault_domain_yes.txt)
        aws cloudformation delete-stack --region $AWS_REGION --stack-name $(cat /tmp/cf_vault_domain_no.txt)
      '''
    }
  }
}
