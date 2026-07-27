##terraform to install 4VMs: control node, Kubernetes master node, kubernetes worker1,2 nodes

pipeline {

    agent any

    environment {
        IMAGE_NAME = "hamzadvps/devops-capstone2:latest"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/hamzadvps/devops-capstone2.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push $IMAGE_NAME
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
            }
        }

        stage('Verify') {
            steps {
                sh 'kubectl get pods'
                sh 'kubectl get svc'
            }
        }
    }
}


## vi hosts- to define on which nodes(prib ip add), reqd installations must be done.

[k8s_master]
172.31.27.239

[k8s_workers]
172.31.31.251
172.31.31.10

[k8s_cluster:children]
k8s_master
k8s_workers

## vi ansible.cfg
[defaults]
host_key_checking = False


## ansible playbook: defining what all need to be installed in the nodes mentioned in vi hosts:

##ubuntu@ip-172-31-20-140:~$ cat install.yaml
---
- name: Configure Kubernetes Nodes
  hosts: k8s_cluster
  become: yes

  tasks:

    - name: Update packages
      apt:
        update_cache: yes

    - name: Install Docker
      apt:
        name: docker.io
        state: present

    - name: Enable Docker
      service:
        name: docker
        state: started
        enabled: yes

    - name: Install required packages
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
        state: present

    - name: Create keyring directory
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Download Kubernetes GPG key
      shell: |
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
        gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      args:
        creates: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    - name: Add Kubernetes repository
      copy:
        dest: /etc/apt/sources.list.d/kubernetes.list
        content: |
          deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /

    - name: Update apt
      apt:
        update_cache: yes

    - name: Install Kubernetes packages
      apt:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present

    - name: Hold Kubernetes packages
      dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop:
        - kubelet
        - kubeadm
        - kubectl
