pipeline {
  agent { label 'tool' }

  options {
    timestamps()
    timeout(time: 25, unit: 'MINUTES')
    disableConcurrentBuilds()
  }

  parameters {
    choice(name: 'ACTION', choices: ['apply', 'destroy'], description: 'Terraform action')
    string(name: 'BUILD_JOB', defaultValue: 'AndreyIL/AndreyLAb2', description: 'L2 job name that produces artifacts')
    string(name: 'SSH_CRED_ID', defaultValue: 'Andrey-heatvm', description: 'SSH key to access created VM (private key)')
    string(name: 'OPENRC_PATH', defaultValue: '/home/ubuntu/openrc-jenkins.sh', description: 'Path on agent to openrc.sh')
    string(name: 'TF_VM_NAME', defaultValue: 'Terraform_andrey', description: 'VM name')
    string(name: 'TF_IMAGE_NAME', defaultValue: 'Ununtu 22.04', description: 'Image name (as in main.tf var)')
    string(name: 'TF_FLAVOR_NAME', defaultValue: 'm1.small', description: 'Flavor name')
    string(name: 'TF_NETWORK_NAME', defaultValue: 'sutdents-net', description: 'Network name')
    string(name: 'TF_KEYPAIR_NAME', defaultValue: 'AndreyIL', description: 'OpenStack keypair name')
    string(name: 'TF_SECGROUP_NAME', defaultValue: 'students-general', description: 'Security group name')
  }

  environment {
    TF_IN_AUTOMATION = "1"
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Fetch artifacts from L2') {
      when { expression { params.ACTION == 'apply' } }
      steps {
        script {
          sh 'rm -rf deploy_art && mkdir -p deploy_art'
          step([
            $class: 'CopyArtifact',
            projectName: params.BUILD_JOB,
            selector: [$class: 'StatusBuildSelector', stable: false],
            filter: 'dist/*.whl',
            target: 'deploy_art',
            fingerprintArtifacts: true
          ])
          step([
            $class: 'CopyArtifact',
            projectName: params.BUILD_JOB,
            selector: [$class: 'StatusBuildSelector', stable: false],
            filter: 'app-restoringvalues.tgz',
            target: 'deploy_art',
            fingerprintArtifacts: true
          ])
          sh 'find deploy_art -maxdepth 3 -type f -print'
        }
      }
    }

    stage('Terraform init') {
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          test -f "${OPENRC_PATH}" || (echo "openrc not found at ${OPENRC_PATH}" && exit 1)
          . "${OPENRC_PATH}"
          terraform version
          terraform init -input=false
        '''
      }
    }

    stage('Terraform apply') {
      when { expression { params.ACTION == 'apply' } }
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          . "${OPENRC_PATH}"

          terraform apply -input=false -auto-approve \
            -var "vm_name=${TF_VM_NAME}-${BUILD_NUMBER}" \
            -var "image_name=${TF_IMAGE_NAME}" \
            -var "flavor_name=${TF_FLAVOR_NAME}" \
            -var "network_name=${TF_NETWORK_NAME}" \
            -var "keypair_name=${TF_KEYPAIR_NAME}" \
            -var "secgroup_name=${TF_SECGROUP_NAME}"

          echo "==> Outputs:"
          terraform output || true
        '''
      }
    }

    stage('Ansible deploy') {
      when { expression { params.ACTION == 'apply' } }
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: params.SSH_CRED_ID,
          keyFileVariable: 'SSH_KEY_FILE',
          usernameVariable: 'SSH_USER'
        )]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail

            chmod 600 "$SSH_KEY_FILE"

            VM_IP="$(terraform output -raw vm_ip)"
            echo "Target VM IP: ${VM_IP}"

            WHEEL="$(ls -1 deploy_art/**/*.whl deploy_art/*.whl 2>/dev/null | head -n 1 || true)"
            TGZ="$(ls -1 deploy_art/app-restoringvalues.tgz 2>/dev/null | head -n 1 || true)"

            if [ -z "$WHEEL" ]; then
              echo "No wheel found"; find deploy_art -maxdepth 4 -type f -print || true; exit 1
            fi
            if [ -z "$TGZ" ]; then
              echo "No app-restoringvalues.tgz found"; find deploy_art -maxdepth 4 -type f -print || true; exit 1
            fi

            echo "Wheel: $WHEEL"
            echo "TGZ:   $TGZ"

            cat > inventory.ini <<EOF
[app]
${VM_IP} ansible_user=${SSH_USER} ansible_ssh_private_key_file=${SSH_KEY_FILE} ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
EOF

            # playbook файл у тебя в корне, поправь имя если не playbook.yml
            ansible-playbook -i inventory.ini playbook.yml \
              --extra-vars "wheel_path=${WHEEL} app_tgz_path=${TGZ} release_id=${BUILD_NUMBER}"
          '''
        }
      }
    }

    stage('Terraform destroy') {
      when { expression { params.ACTION == 'destroy' } }
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          . "${OPENRC_PATH}"
          terraform destroy -input=false -auto-approve \
            -var "vm_name=${TF_VM_NAME}-${BUILD_NUMBER}" \
            -var "image_name=${TF_IMAGE_NAME}" \
            -var "flavor_name=${TF_FLAVOR_NAME}" \
            -var "network_name=${TF_NETWORK_NAME}" \
            -var "keypair_name=${TF_KEYPAIR_NAME}" \
            -var "secgroup_name=${TF_SECGROUP_NAME}"
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'deploy_art/**, terraform.tfstate*, inventory.ini', allowEmptyArchive: true
      cleanWs()
    }
  }
}
