pipeline {
  agent any

  parameters {
    string(name: 'AMI_ID', defaultValue: 'ami-0e001c9271cf7f3b9', description: 'Ubuntu 22.04 AMI (us-east-1)')
    string(name: 'REGION', defaultValue: 'us-east-1', description: 'AWS region')
    booleanParam(name: 'AUTO_TERMINATE', defaultValue: false, description: 'Terminate demo instances after run?')
  }

  environment {
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    SSH_KEY_CRED          = 'jenkins-ssh-key'
    DEMO_TAG_NAME         = 'jenkins-demo-ec2'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prepare Python venv & Tools') {
      steps {
        sh '''
          set -euo pipefail
          echo "Python info: $(python3 --version 2>/dev/null || echo 'python3 missing')"

          # create venv (bootstraps pip even on PEP-668 systems)
          if [ ! -d ".venv" ]; then
            python3 -m venv .venv
          fi

          . .venv/bin/activate

          pip install --upgrade pip setuptools wheel
          pip install ansible boto3 botocore awscli

          export PATH="$PWD/.venv/bin:$PATH"

          # install amazon.aws collection
          ansible-galaxy collection install amazon.aws --force

          echo "Venv ready: $(ansible --version | head -n1 || echo 'ansible n/a')"
        '''
      }
    }

    stage('Run Ansible playbook') {
      steps {
        sshagent (credentials: [env.SSH_KEY_CRED]) {
          withEnv(["AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}", "AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}", "PATH=$PWD/.venv/bin:$PATH"]) {
            sh '''
              set -euo pipefail
              . .venv/bin/activate
              echo "Running playbook (region=${REGION}, ami=${AMI_ID})"
              ansible-playbook jenkins_ansible_ec2_apache.yml -e "region=${REGION} ami_id=${AMI_ID}"

              # short wait for AWS eventual consistency
              sleep 5

              echo "Instances with tag Name=${DEMO_TAG_NAME}:"
              aws ec2 describe-instances \
                --filters "Name=tag:Name,Values=${DEMO_TAG_NAME}" "Name=instance-state-name,Values=pending,running" \
                --query 'Reservations[].Instances[].[InstanceId,PublicIpAddress,State.Name]' \
                --output table --region ${REGION} || true
            '''
          }
        }
      }
    }

    stage('Smoke test (optional)') {
      when { expression { return !params.AUTO_TERMINATE } }
      steps {
        withEnv(["AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}", "AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}", "PATH=$PWD/.venv/bin:$PATH"]) {
          sh '''
            set -euo pipefail
            . .venv/bin/activate

            PUBIP=$(aws ec2 describe-instances \
              --filters "Name=tag:Name,Values=${DEMO_TAG_NAME}" "Name=instance-state-name,Values=running" \
              --query 'Reservations[].Instances[].PublicIpAddress' --output text --region ${REGION} | awk 'NR==1{print $1}')

            if [ -z "$PUBIP" ]; then
              echo "No running instance with public IP found."
            else
              echo "Attempting HTTP GET on http://$PUBIP ..."
              if command -v curl >/dev/null 2>&1; then
                curl -I --max-time 10 http://$PUBIP || true
              else
                echo "curl not available; skip HTTP check."
              fi
            fi
          '''
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished."
    }
    cleanup {
      script {
        if (params.AUTO_TERMINATE.toBoolean()) {
          withEnv(["AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}", "AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}", "PATH=$PWD/.venv/bin:$PATH"]) {
            sh '''
              set -euo pipefail
              . .venv/bin/activate

              IIDS=$(aws ec2 describe-instances \
                --filters "Name=tag:Name,Values=${DEMO_TAG_NAME}" \
                --query 'Reservations[].Instances[].InstanceId' --output text --region ${REGION} || true)

              if [ -n "$IIDS" ]; then
                echo "Terminating instances: $IIDS"
                aws ec2 terminate-instances --instance-ids $IIDS --region ${REGION}
              else
                echo "No instances to terminate."
              fi
            '''
          }
        } else {
          echo "AUTO_TERMINATE is false — leaving instances running."
        }
      }
    }
  }
}
