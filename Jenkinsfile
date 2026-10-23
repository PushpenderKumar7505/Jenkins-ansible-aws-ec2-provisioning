// Jenkins Pipeline (Pipeline script) used in job "Aws-project"
node {
    stage ('Github connect') {
        git branch: 'main', url: 'https://github.com/PushpenderKumar7505/2pm-wd-project.git'
    }
    stage ('ansible-server') {
        sshagent(['ansible']) {
            sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.35.92'
            sh 'scp /var/lib/jenkins/workspace/Aws-project/* ubuntu@172.31.35.92:/home/ubuntu'
        }
    }
    stage ('ansible deploy') {
        sshagent(['ansible']) {
            sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.35.92 cd /home/ubuntu'
            sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.35.92 ansible-playbook ec2.yml'
        }
    }
}
