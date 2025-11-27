pipeline {
  agent any

  environment {
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    SSH_KEY_CRED          = 'jenkins-ssh-key'   // Add your Jenkins_master.pem here
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Setup Ansible') {
      steps {
        sh '''
          python3 -m pip install --user ansible boto3 botocore
          ~/.local/bin/ansible-galaxy collection install amazon.aws
        '''
      }
    }

    stage('Run Playbook') {
      steps {
        sshagent (credentials: [env.SSH_KEY_CRED]) {
          sh '''
            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

            ~/.local/bin/ansible-playbook jenkins_ansible_ec2_apache.yml
          '''
        }
      }
    }
  }
}
