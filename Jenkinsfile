pipeline {
  agent { label 'tool' }

  options {
    timestamps()
    timeout(time: 25, unit: 'MINUTES')
    disableConcurrentBuilds()
  }

  parameters {
    choice(name: 'ACTION', choices: ['apply', 'destroy'], description: 'apply=create/update VM + deploy, destroy=remove VM')
    string(name: 'BUILD_JOB', defaultValue: 'AndreyIL/AndreyLAb2', description: 'L2 job name that produces dist/*.whl')
    string(name: 'SSH_CRED_ID', defaultValue: 'Andrey-heatvm', description: 'SSH key to connect to created VM (user ubuntu)')
    string(name: 'OPENRC_PATH', defaultValue: '/home/ubuntu/openrc-jenkins.sh', description: 'Path to OpenStack openrc on Jenkins node')
  }

  environment {
    // ВАЖНО: TF_CLI_CONFIG_FILE должен указывать на terraform.rc в workspace (мы его создаём в init)
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

          // если tgz нужен — оставь, если нет — можешь удалить этот блок
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

          terraform -version

          # Если у тебя "зеркала" и registry отрезан — terraform.rc должен быть в репо.
          # Если terraform.rc уже лежит в репо, этот блок можно удалить.
          if [ -f "./terraform.rc" ]; then
            echo "Using existing terraform.rc from repo"
          else
            echo "No terraform.rc found in repo."
            echo "Create terraform.rc (provider_installation) to use mirrors, or Terraform won't download providers."
            exit 1
          fi

          export TF_CLI_CONFIG_FILE="$PWD/terraform.rc"

          terraform fmt -check || true
          terraform init -input=false
        '''
      }
    }

    stage('Terraform apply') {
      when { expression { params.ACTION == 'apply' } }
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          export TF_CLI_CONFIG_FILE="$PWD/terraform.rc"

          if [ ! -f "${OPENRC_PATH}" ]; then
            echo "OpenRC not found at: ${OPENRC_PATH}"
            exit 1
          fi

          # OpenStack creds from local file on Jenkins node
          . "${OPENRC_PATH}"

          # Быстрая проверка, что креды живые
          openstack token issue >/dev/null

          terraform apply -auto-approve -input=false

          # IP VM нам нужен для ansible
          terraform output -raw vm_ip | tee vm_ip.txt
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

            VM_IP="$(cat vm_ip.txt | tr -d ' \\n\\r')"
            if [ -z "$VM_IP" ]; then
              echo "vm_ip is empty (terraform output failed?)"
              exit 1
            fi

            # wheel может лежать как deploy_art/dist/*.whl или deploy_art/*.whl
            WHEEL="$(ls -1 deploy_art/**/*.whl deploy_art/*.whl 2>/dev/null | head -n 1 || true)"
            if [ -z "$WHEEL" ]; then
              echo "No .whl found in deploy_art. Listing:"
              find deploy_art -maxdepth 4 -type f -print || true
              exit 1
            fi

            echo "VM_IP=$VM_IP"
            echo "Using wheel: $WHEEL"

            # Инвентарь на лету
            cat > inventory.ini <<EOF
[app]
$VM_IP ansible_user=$SSH_USER ansible_ssh_private_key_file=$SSH_KEY_FILE ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
EOF

            # ВАЖНО: если VM только что создана — SSH может подняться не мгновенно
            echo "==> Wait for SSH..."
            for i in $(seq 1 30); do
              if ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null "$SSH_USER@$VM_IP" "echo ok" >/dev/null 2>&1; then
                break
              fi
              sleep 2
            done

            # Запуск плейбука
            ansible-playbook -i inventory.ini playbook.yml --extra-vars "wheel_path=$WHEEL"

            echo "==> Check service on VM"
            ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null "$SSH_USER@$VM_IP" "sudo systemctl --no-pager --full status restoringvalues.service || true"
          '''
        }
      }
    }

    stage('Terraform destroy') {
      when { expression { params.ACTION == 'destroy' } }
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          export TF_CLI_CONFIG_FILE="$PWD/terraform.rc"

          if [ ! -f "${OPENRC_PATH}" ]; then
            echo "OpenRC not found at: ${OPENRC_PATH}"
            exit 1
          fi

          . "${OPENRC_PATH}"
          openstack token issue >/dev/null

          terraform destroy -auto-approve -input=false
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'deploy_art/**, *.txt, inventory.ini, .terraform.lock.hcl', allowEmptyArchive: true
      cleanWs()
    }
  }
}
