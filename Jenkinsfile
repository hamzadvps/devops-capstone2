pipeline {

    agent any

    stages {

        stage('Build') {

            steps {

                echo "Building Docker Image"

                sh 'docker build -t website-app .'

            }
        }

        stage('Test') {

            steps {

                echo "Testing"

                sh 'docker run --rm website-app ls /var/www/html'

            }
        }

        stage('Prod') {

            when {

                branch 'master'
            }

            steps {

                echo "Deploying to Production"

                sh '''
                docker save website-app > website.tar
                scp -o StrictHostKeyChecking=no website.tar ubuntu@172.31.23.253:/home/ubuntu

                ssh ubuntu@172.31.23.253"
                docker load < website.tar
                docker rm -f web || true
                docker run -d --name web -p 80:80 website-app
                "
                '''
            }
        }
    }
}
