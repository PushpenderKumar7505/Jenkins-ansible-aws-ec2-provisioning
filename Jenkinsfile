// Jenkins Pipeline: job "Aws-project" (Pipeline script from SCM)
node {
    def ANSIBLE_HOST = 'ubuntu@172.31.35.92'   // Ansible server private IP

    stage('Github connect') {
        checkout scm
    }
    stage('ansible-server') {
        sshagent(['ansible']) {
            sh "scp -o StrictHostKeyChecking=no ec2.yml ${ANSIBLE_HOST}:/home/ubuntu/"
        }
    }
    stage('ansible deploy') {
        sshagent(['ansible']) {
            sh "ssh -o StrictHostKeyChecking=no ${ANSIBLE_HOST} 'cd /home/ubuntu && ansible-playbook ec2.yml'"
        }
    }
}
