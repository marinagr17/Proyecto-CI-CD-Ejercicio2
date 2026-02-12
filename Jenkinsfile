pipeline {
    agent none
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        IMAGEN = "marinagr17/dockerci-cd"
        USUARIO = credentials('USER_DOCKERHUB')
    }
    
    stages {
        stage("Test Django") {
            agent {
                docker {
                    image 'python:3.12-slim'
                    args '-u root:root'
                }
            }
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], userRemoteConfigs: [[url: 'https://github.com/marinagr17/Guestbook-Tutorial.git']]])
                
                dir("build/app") {
                    sh '''
                    apt-get update && apt-get install -y --no-install-recommends gcc default-libmysqlclient-dev build-essential libssl-dev pkg-config
                    pip install --quiet -r requirements.txt
                    python3 manage.py test --settings=django_tutorial.settings_test --verbosity=2
                    '''
                }
            }
        }
        
        stage("Build & Push") {
            agent any
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], userRemoteConfigs: [[url: 'https://github.com/marinagr17/Guestbook-Tutorial.git']]])
                
                script {
                    dir('build') {
                        docker.build("${IMAGEN}:${BUILD_NUMBER}")
                        docker.image("${IMAGEN}:${BUILD_NUMBER}").inside('-u root') {
                            sh 'python3 -c "import django; print(django.get_version())"'
                        }
                    }
                    
                    sh '''
                    echo ${USUARIO_PSW} | docker login -u ${USUARIO_USR} --password-stdin
                    docker push ${IMAGEN}:${BUILD_NUMBER}
                    docker tag ${IMAGEN}:${BUILD_NUMBER} ${IMAGEN}:latest
                    docker push ${IMAGEN}:latest
                    docker logout
                    '''
                }
            }
        }

        stage ("Clean up") {
            agent any
            steps {
                sh '''
                docker rmi ${IMAGEN}:${BUILD_NUMBER} 2>/dev/null || true
                docker rmi ${IMAGEN}:latest 2>/dev/null || true
                '''
            }
        }

        stage("Deployment") {
            agent any
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: 'https://github.com/marinagr17/Proyecto-CI-CD-Ejercicio2.git']]])
                
                script {

                    sh '''
                    ssh -v -i ~/.ssh/jenkins -o StrictHostKeyChecking=no debian@nymeria.pingamarina.site << 'EOF'
                    cd /home/debian/Proyecto-CI-CD-Ejercicio2

                    docker pull marinagr17/django_tutorial:latest

                    docker compose up -d
                    sleep 10

                    # Solo clonar si no existe
                    if [ ! -d Proyecto-CI-CD-Ejercicio2 ]; then
                    git clone https://github.com/marinagr17/Proyecto-CI-CD-Ejercicio2.git
                    fi

                    # Copiar config nginx solo si no existe
                    if [ ! -f /etc/nginx/sites-available/jenkins ]; then
                        sudo cp /home/debian/Proyecto-CI-CD-Ejercicio2/jenkins /etc/nginx/sites-available/jenkins
                    fi

                    # Crear enlace solo si no existe
                    if [ ! -L /etc/nginx/sites-enabled/jenkins ]; then
                        sudo ln -s /etc/nginx/sites-available/jenkins /etc/nginx/sites-enabled/jenkins
                    fi

                    sudo systemctl reload nginx

                    docker compose up -d
                    EOF
                    '''    
                 }               
            }
        }
    }
}
