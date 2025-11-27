pipeline {
  agent any

  parameters {
    string(name: 'AMI_ID', defaultValue: 'ami-0e001c9271cf7f3b9', description: 'Ubuntu 22.04 AMI (us-east-1)')
    string(name: 'REGION', defaultValue: 'us-east-1', description: 'AWS region')
    booleanParam(name: 'AUTO_TERMINATE', defaultValue: false, description: 'Terminate demo instances after run?')
  }

  environment {
    DEMO_TAG_NAME = 'jenkins-demo-ec2'
    KEY_NAME      = 'Jenkins_master.pem'   // keep the exact name you used in playbook
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Prepare Python venv & Tools') {
      steps {
        sh '''
          bash -lc '
set -euo pipefail
if [ ! -d ".venv" ]; then python3 -m venv .venv; fi
. .venv/bin/activate
pip install --upgrade pip setuptools wheel
pip install ansible boto3 botocore awscli
export PATH="$PWD/.venv/bin:$PATH"
ansible-galaxy collection install amazon.aws --force
'
        '''
      }
    }

    stage('Ensure AWS KeyPair exists (import from PEM)') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
          file(credentialsId: 'jenkins-ssh-file', variable: 'SSH_KEY_FILE')
        ]) {
          sh '''
            bash -lc '
set -euo pipefail
. .venv/bin/activate
export PATH="$PWD/.venv/bin:$PATH"
export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY"

chmod 600 "$SSH_KEY_FILE"

PUBKEY_FILE="$(mktemp)"
if ! ssh-keygen -y -f "$SSH_KEY_FILE" > "$PUBKEY_FILE" 2>/dev/null; then
  echo "Failed to derive public key from private key. Ensure the file is a valid OpenSSH private key."
  exit 1
fi

echo "Checking if key-pair named ${KEY_NAME} exists in ${REGION}..."
if aws ec2 describe-key-pairs --key-names "${KEY_NAME}" --region ${REGION} >/dev/null 2>&1; then
  echo "Key pair ${KEY_NAME} already exists."
else
  echo "Key pair ${KEY_NAME} not found — importing using provided PEM-derived public key..."
  aws ec2 import-key-pair --key-name "${KEY_NAME}" --public-key-material "$(cat ${PUBKEY_FILE})" --region ${REGION}
  echo "Imported key pair ${KEY_NAME}."
fi

rm -f "$PUBKEY_FILE"
'
          '''
        }
      }
    }

    stage('Run Playbook') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
          file(credentialsId: 'jenkins-ssh-file', variable: 'SSH_KEY_FILE')
        ]) {
          sh '''
            bash -lc '
set -euo pipefail
. .venv/bin/activate
export PATH="$PWD/.venv/bin:$PATH"
export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY"

chmod 600 "$SSH_KEY_FILE"

ansible-playbook jenkins_ansible_ec2_apache.yml -e "region=${REGION} ami_id=${AMI_ID} key_name=${KEY_NAME}" --private-key "$SSH_KEY_FILE"

sleep 5

if [ -f "instance_ips.txt" ]; then
  echo "---- instance_ips.txt ----"
  cat instance_ips.txt
else
  echo "instance_ips.txt not found"
fi

aws ec2 describe-instances --filters "Name=tag:Name,Values=${DEMO_TAG_NAME}" "Name=instance-state-name,Values=pending,running" \
  --query "Reservations[].Instances[].[InstanceId,PublicIpAddress,State.Name]" --output table --region ${REGION} || true
'
          '''
        }
      }
    }

    stage('Smoke test (optional)') {
      when { expression { return !params.AUTO_TERMINATE } }
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          sh '''
            bash -lc '
set -euo pipefail
. .venv/bin/activate
export PATH="$PWD/.venv/bin:$PATH"
export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY"

PUBIP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=${DEMO_TAG_NAME}" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].PublicIpAddress" --output text --region ${REGION} | awk "NR==1{print \$1}")

if [ -z "$PUBIP" ]; then
  echo "No running instance with public IP found for smoke test."
else
  echo "Attempting HTTP GET on http://$PUBIP ..."
  if command -v curl >/dev/null 2>&1; then
    curl -I --max-time 10 http://$PUBIP || true
  else
    echo "curl not available; skipping HTTP check."
  fi
fi
'
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: "instance_ips.txt", allowEmptyArchive: true
      echo "Archived instance_ips.txt (if present)."
    }
    cleanup {
      script {
        if (params.AUTO_TERMINATE.toBoolean()) {
          withCredentials([
            string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
            string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
          ]) {
            sh '''
              bash -lc '
set -euo pipefail
. .venv/bin/activate
export PATH="$PWD/.venv/bin:$PATH"
export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY"

IIDS=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=${DEMO_TAG_NAME}" \
  --query "Reservations[].Instances[].InstanceId" --output text --region ${REGION} || true)

if [ -n "$IIDS" ]; then
  echo "Terminating instances: $IIDS"
  aws ec2 terminate-instances --instance-ids $IIDS --region ${REGION}
else
  echo "No instances to terminate."
fi
'
            '''
          }
        } else {
          echo "AUTO_TERMINATE is false — leaving instances running."
        }
      }
    }
  }
}
