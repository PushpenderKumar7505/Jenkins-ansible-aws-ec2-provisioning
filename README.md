# ☁️ Automated AWS EC2 Provisioning using Jenkins & Ansible

> A GitHub push triggers a **Jenkins** pipeline, which ships the code to an **Ansible** server, which runs a playbook that creates a **Security Group and an EC2 instance on AWS**. No manual clicks in the AWS Console.

|               | Details                                                              |
|:-------------:|:--------------------------------------------------------------------:|
| **Goal**      | Provision AWS EC2 infrastructure automatically from code             |
| **Trigger**   | GitHub Webhook (push event)                                          |
| **Key Tools** | Jenkins + Ansible + AWS EC2                                          |
| **Result**    | ✅ Pipeline `Aws-project`: 6 builds, #2 to #6 successful (#1 failed)  |

---

## 🏗️ Overall Architecture

```mermaid
flowchart TD
    A[("🐙 GitHub Repository")] -->|"Webhook on push"| B
    B["⚙️ Jenkins Server\nt3.micro · ap-south-1\njob: Aws-project"] -->|"SSH + SCP\n(sshagent credential)"| D
    D["📦 Ansible Server\nt3.micro · ap-south-1"] -->|"ansible-playbook ec2.yml"| E
    E["☁️ AWS EC2 API"] --> F["🔐 Security Group\nmy-sg1 · SSH 22"]
    E --> G["🖥️ EC2 Instance\nt3.micro"]

    style A fill:#2d333b,color:#adbac7,stroke:#444c56
    style B fill:#d33833,color:#ffffff,stroke:#d33833
    style D fill:#e3a008,color:#000000,stroke:#e3a008
    style E fill:#ff9900,color:#000000,stroke:#ff9900
    style F fill:#0e8a16,color:#ffffff,stroke:#0e8a16
    style G fill:#0e8a16,color:#ffffff,stroke:#0e8a16
```

A static copy of this diagram is also available: [`architecture-diagram.png`](architecture-diagram.png)

---

## 🔧 Tools Used

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat&logo=jenkins&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat&logo=ansible&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=flat&logo=amazonaws&logoColor=white)
![GitHub Webhooks](https://img.shields.io/badge/GitHub_Webhooks-181717?style=flat&logo=github&logoColor=white)
![Groovy](https://img.shields.io/badge/Groovy-4298B8?style=flat&logo=apachegroovy&logoColor=white)
![YAML](https://img.shields.io/badge/YAML-CB171E?style=flat&logo=yaml&logoColor=white)
![Linux](https://img.shields.io/badge/Linux_Ubuntu-E95420?style=flat&logo=ubuntu&logoColor=white)
![Python](https://img.shields.io/badge/python3--boto-3776AB?style=flat&logo=python&logoColor=white)

---

## 🔄 Pipeline Flow

```mermaid
flowchart LR
    A["🐙 GitHub\nCode Push"] -->|"Webhook trigger"| B
    B["⚙️ Jenkins\nPipeline"] --> C
    C["Stage 1\n📥 Github connect"] --> D
    D["Stage 2\n📤 ansible-server\nSSH + SCP"] --> E
    E["Stage 3\n🚀 ansible deploy\nansible-playbook ec2.yml"] --> F
    F["☁️ Security Group +\nEC2 instance ✅"]

    style A fill:#2d333b,color:#adbac7,stroke:#444c56
    style B fill:#d33833,color:#ffffff,stroke:#d33833
    style C fill:#0075ca,color:#ffffff,stroke:#0075ca
    style D fill:#0075ca,color:#ffffff,stroke:#0075ca
    style E fill:#e3a008,color:#000000,stroke:#e3a008
    style F fill:#0e8a16,color:#ffffff,stroke:#0e8a16
```

---

## 📝 Step-by-Step Breakdown

**① Jenkins Server Setup**
Jenkins installed on an EC2 Ubuntu instance, unlocked with the initial admin password, suggested plugins installed. Managed over SSH using MobaXterm. *(P1-01 to P1-04)*

**② Credentials for Ansible Access**
Added a Global credential of type *SSH Username with private key* (ID `ansible`, user `ubuntu`). The pipeline uses it through the `sshagent` step, so no password is typed or stored in the script. *(P1-05, P1-06)*

**③ Ansible Server Verified**
Three EC2 instances are running (Jenkins, Ansible, kubernetes). From the Ansible server, `ansible k8s -m ping` returns `pong`, confirming passwordless SSH and Ansible connectivity. The Kubernetes node is used only for this connectivity check and is **not** part of this pipeline. *(P1-07)*

**④ GitHub → Jenkins Integration**
Created a Pipeline job, generated the `git` step with the Pipeline Syntax snippet generator, pushed a test commit to `main`, generated a Jenkins API token, and added a GitHub webhook (`/github-webhook/`, `application/json`, push event). GitHub reports **"Last delivery was successful"**. *(P1-08 to P1-14)*

**⑤ Ansible Playbook + Final 3-Stage Pipeline**
`ec2.yml` creates a security group and an EC2 instance. The final pipeline checks out the code, copies it to the Ansible server and runs the playbook remotely. *(P1-15, P1-16)*

**⑥ Build Results**
Job `Aws-project` is green. Build history shows #1 failed and #2 to #6 successful. *(P1-17, P1-18)*

---

## 🧩 Jenkins Pipeline Code

File: [`Jenkinsfile`](Jenkinsfile)

```groovy
node {
  stage('Github connect') {
    git branch: 'main',
        url: 'https://github.com/PushpenderKumar7505/2pm-wd-project.git'
  }
  stage('ansible-server') {
    sshagent(['ansible']) {
      sh 'ssh -o StrictHostKeyChecking=no ubuntu@<ansible-private-ip>'
      sh 'scp /var/lib/jenkins/workspace/Aws-project/* ubuntu@<ansible-private-ip>:/home/ubuntu'
    }
  }
  stage('ansible deploy') {
    sshagent(['ansible']) {
      sh 'ssh -o StrictHostKeyChecking=no ubuntu@<ansible-private-ip> cd /home/ubuntu'
      sh 'ssh -o StrictHostKeyChecking=no ubuntu@<ansible-private-ip> ansible-playbook ec2.yml'
    }
  }
}
```

| Stage | What it does |
|---|---|
| `Github connect` | Checks out branch `main` into the Jenkins workspace |
| `ansible-server` | Connects to the Ansible server with `sshagent` and copies the workspace files using `scp` |
| `ansible deploy` | Runs `ansible-playbook ec2.yml` on the Ansible server over SSH |

---

## 📜 Ansible Playbook: `ec2.yml`

File: [`ec2.yml`](ec2.yml)

```yaml
---
- name: Create EC2
  hosts: localhost
  gather_facts: no

  vars:
    region: "us-west-2"
    ami_id: "ami-0cc96c4cd98401dae"
    instance_type: "t3.micro"
    key_name: "Pushpender"
    security_group_name: "my-sg1"
    subnet_id: "subnet-091d3132bcf1a71a0"

  tasks:
    - name: Create a security group
      amazon.aws.ec2_group:
        name: "{{ security_group_name }}"
        description: "Security group for EC2 instance"
        region: "{{ region }}"
        rules:
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: "0.0.0.0/0"
        state: present
      register: sg

    - name: Create an EC2 instance
      amazon.aws.ec2_instance:
        name: "Sharma ji "
        image_id: "{{ ami_id }}"
        instance_type: "{{ instance_type }}"
        key_name: "{{ key_name }}"
        region: "{{ region }}"
        security_groups:
          - "{{ sg.group_name }}"
        subnet_id: "{{ subnet_id }}"
        count: 1
        wait: yes
      register: ec2_instance
```

**Needs on the Ansible server:** Ansible, the `amazon.aws` collection, `python3-boto3`/`botocore`, and AWS credentials with EC2 permissions.

---

> [!TIP]
> ✅ **Result**: Jenkins job `Aws-project` ran **6 builds: #1 failed, #2 to #6 successful** (last build about 18 seconds) · GitHub webhook configured with a successful delivery · Pipeline copies the project to the Ansible server and runs `ec2.yml` remotely · Passwordless SSH from Jenkins to Ansible using `sshagent` · `ansible -m ping` returned `pong`

---

## ⚠️ Known Limitations (Lab Setup)

- The security group allows SSH (22) from `0.0.0.0/0`. Restrict it to a trusted CIDR in real use.
- `StrictHostKeyChecking=no` skips host key verification. Use a managed `known_hosts` instead.
- Jenkins runs on plain HTTP (port 8080).
- Pipeline script is stored in the Jenkins job; moving it to a `Jenkinsfile` in the repo (Pipeline from SCM) is the next step.
- Playbook region is `us-west-2`, while the Jenkins and Ansible servers run in `ap-south-1`.
- Screenshots P1-01 to P1-14 and P1-15 to P1-18 were taken on two different Jenkins servers (different IPs). Secrets visible in the original screenshots are redacted here.

---

## ⚙️ Complete Tech Stack

| Category                 | Tools |
|--------------------------|-------|
| CI/CD & Automation       | ![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat&logo=jenkins&logoColor=white) ![Webhooks](https://img.shields.io/badge/GitHub_Webhooks-181717?style=flat&logo=github&logoColor=white) Pipeline as Code (Groovy) |
| Configuration Management | ![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat&logo=ansible&logoColor=white) Playbooks · `amazon.aws` collection |
| Cloud Platform           | ![AWS](https://img.shields.io/badge/AWS-FF9900?style=flat&logo=amazonaws&logoColor=white) EC2 · Security Groups · Key Pair · Subnet |
| OS & Scripting           | ![Linux](https://img.shields.io/badge/Linux_Ubuntu-E95420?style=flat&logo=ubuntu&logoColor=white) ![YAML](https://img.shields.io/badge/YAML-CB171E?style=flat&logo=yaml&logoColor=white) |
| Version Control          | ![Git](https://img.shields.io/badge/Git-F05032?style=flat&logo=git&logoColor=white) ![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white) |
| Networking               | SSH · SCP · `sshagent` · Passwordless Authentication |
| Tools                    | MobaXterm · python3-boto |

---

## 📸 Project Screenshots

| Screenshot | Description |
|---|---|
| `P1-01-ec2-jenkins-server-ssh-session.png` | SSH session to the Jenkins EC2 server |
| `P1-02-jenkins-unlock-initial-admin-password.png` | Unlock Jenkins (password redacted) |
| `P1-03-jenkins-install-suggested-plugins.png` | Install suggested plugins |
| `P1-04-jenkins-dashboard-ready.png` | Jenkins dashboard ready |
| `P1-05-jenkins-global-credentials-ansible.png` | Global credentials with `ansible` entry |
| `P1-06-jenkins-add-ssh-private-key-credential.png` | Add SSH Username with private key |
| `P1-07-ansible-server-ping-and-ec2-instances.png` | `ansible -m ping` → pong, 3 EC2 instances running |
| `P1-08-jenkins-new-pipeline-item.png` | Create Pipeline item |
| `P1-09-jenkins-pipeline-syntax-git-snippet.png` | Generate `git` step with Pipeline Syntax |
| `P1-10-git-clone-commit-push.png` | Clone, commit and push to `main` |
| `P1-11-pipeline-script-github-connect-stage.png` | First pipeline script (Github connect) |
| `P1-12-jenkins-api-token-generate.png` | Generate Jenkins API token |
| `P1-13-github-add-webhook.png` | Add GitHub webhook (secret redacted) |
| `P1-14-github-webhook-delivery-success.png` | Webhook last delivery successful |
| `P1-15-ansible-playbook-ec2-yml.png` | `ec2.yml` playbook |
| `P1-16-final-pipeline-script-3-stages.png` | Final 3-stage pipeline script |
| `P1-17-jenkins-dashboard-aws-project-success.png` | Dashboard: `Aws-project` green |
| `P1-18-jenkins-build-history-6-builds.png` | Build history #1 to #6 |

<details>
<summary><b>🖼️ View all screenshots</b></summary>

### Jenkins setup
![P1-01](P1-01-ec2-jenkins-server-ssh-session.png)
![P1-02](P1-02-jenkins-unlock-initial-admin-password.png)
![P1-03](P1-03-jenkins-install-suggested-plugins.png)
![P1-04](P1-04-jenkins-dashboard-ready.png)

### Credentials and Ansible access
![P1-05](P1-05-jenkins-global-credentials-ansible.png)
![P1-06](P1-06-jenkins-add-ssh-private-key-credential.png)
![P1-07](P1-07-ansible-server-ping-and-ec2-instances.png)

### Pipeline job and GitHub webhook
![P1-08](P1-08-jenkins-new-pipeline-item.png)
![P1-09](P1-09-jenkins-pipeline-syntax-git-snippet.png)
![P1-10](P1-10-git-clone-commit-push.png)
![P1-11](P1-11-pipeline-script-github-connect-stage.png)
![P1-12](P1-12-jenkins-api-token-generate.png)
![P1-13](P1-13-github-add-webhook.png)
![P1-14](P1-14-github-webhook-delivery-success.png)

### Playbook, final pipeline and results
![P1-15](P1-15-ansible-playbook-ec2-yml.png)
![P1-16](P1-16-final-pipeline-script-3-stages.png)
![P1-17](P1-17-jenkins-dashboard-aws-project-success.png)
![P1-18](P1-18-jenkins-build-history-6-builds.png)

</details>

---

## 👨‍💻 Author

**Pushpender Kumar** · B.Tech CSE, GLA University 2024

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/pushpender-kumar-5280b7226/)
[![GitHub](https://img.shields.io/badge/GitHub-100000?style=flat&logo=github&logoColor=white)](https://github.com/PushpenderKumar7505)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat&logo=gmail&logoColor=white)](mailto:pushpender7505@gmail.com)
