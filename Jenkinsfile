pipeline {
   agent any
   environment {
     DOCKER_REGISTRY = "registry.devops:5000"
     VAULT_ADDR = "http://vault.devops:8200"
     VAULT_PATH_MYSQL="kv/mysql/db"
     VAULT_TOKEN_MYSQL="s.S2VM3C7WGWKMIKfgXPLyJt65"
     MYSQL_STAGING_URL="staging.devops:3306"
     MYSQL_PROD_URL="production.devops:3306"
     MYSQL_DB_NAME="test"
     MYSQL_DB_PASSWORD="test"
     MYSQL_DB_USER="test"
     MYSQL_DB_ROOT="tooor"
   }
   stages {
      stage('Build') {
         steps {
            sh '''
               docker images -f dangling=true -q | xargs docker rmi || true
               mvn -f backend/pom.xml clean package
               npm --prefix frontend install
            '''   
         }
      }
         stage('Code Security') {
         steps {
            parallel(
               dependency: {
                  sh '''
                     mkdir -p report
                     chmod 777 report
                     docker run --rm -v "$PWD":/code -v "$PWD"/report:/report  registry.gitlab.com/gitlab-ci-utils/docker-dependency-check:1.9.0 -f JSON  -s /code/frontend -o /report/frontend_dependency_check_result.json  --project devops
                     docker run --rm -v "$PWD":/code -v "$PWD"/report:/report  registry.gitlab.com/gitlab-ci-utils/docker-dependency-check:1.9.0 -f JSON  -s /code/backend -o /report/backend_dependency_check_result.json  --project devops

                     echo "Scan Report Creted Successfully" 
                     
                  '''
               },
               // sast: {
               //    sh 'pip3 install semgrep'
               //    sh 'semgrep ci'
                  
               // }
               sast: {
                  sh '''
                     mkdir -p semgrep
                     chmod 777 semgrep
                     docker run --rm -v "$PWD"/semgrep:/semgrep -v "$PWD":/src returntocorp/semgrep --config auto --output /semgrep/semgrep_result.json --json
                  '''
               }
            )
         }
      }
      stage('Archive') {
         steps {
            parallel(
               ui: { 
                  sh '''
                        tar czf ${GIT_COMMIT}.tar.gz frontend/
                  '''            
                  script {
                     def remote = [:]
                     remote.name = 'archiver'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'archiver.devops'
                     remote.identityFile = '~/.ssh/archiver.key'
                     sshPut remote: remote, filterRegex: '.tar.gz$',from: '.' ,into: '/home/vagrant/archiver/frontend'
                  }
               },
               api:{
                  sh '''
                        mv ${WORKSPACE}/backend/target/*.jar ${WORKSPACE}/backend/target/${GIT_COMMIT}.jar
                  '''            
                  script {
                     def remote = [:]
                     remote.name = 'archiver'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'archiver.devops'
                     remote.identityFile = '~/.ssh/archiver.key'
                     sshPut remote: remote, filterRegex: '.jar$',from: './backend/target' ,into: '/home/vagrant/archiver/backend'
                  }
               }       
            )
         }              
      }
      stage('Staging Setup') {
         steps {
            parallel(
               ui: { // Prepare the Docker image for the staging ui
                  sh '''
                     mv frontend/nginx-staging.conf frontend/nginx.conf
                     docker build --no-cache --build-arg STAGE=staging -t "devops/ui:staging" -f frontend/Dockerfile .
                     docker tag "devops/ui:staging" "${DOCKER_REGISTRY}/devops/ui:staging"
                     docker push "${DOCKER_REGISTRY}/devops/ui:staging"
                     docker rmi "${DOCKER_REGISTRY}/devops/ui:staging"
                     docker rmi "devops/ui:staging"
                  '''
               },
               api: {
                  sh '''
                     docker build --no-cache --build-arg FILE_NAME=${GIT_COMMIT} -t "devops/api:staging" -f backend/Dockerfile .
                     docker tag "devops/api:staging" "${DOCKER_REGISTRY}/devops/api:staging"
                     docker push "${DOCKER_REGISTRY}/devops/api:staging"
                     docker rmi "${DOCKER_REGISTRY}/devops/api:staging"
                     docker rmi "devops/api:staging"
                  '''
               },
               db: { // Parallely start the MySQL Daemon in the staging server first stop if already running then start
                  script {
                     def remote = [:]
                     remote.name = 'staging'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'staging.devops'
                     remote.identityFile = '~/.ssh/staging.key'
                     sshCommand remote: remote, command: "docker stop mysqldb backend frontend || true"
                     sshCommand remote: remote, command: "docker rm backend mysqldb frontend || true"
                     sshCommand remote: remote, command: "docker rmi ${DOCKER_REGISTRY}/devops/api:staging ${DOCKER_REGISTRY}/devops/ui:staging || true"
                     sshCommand remote: remote, command: "docker run -d -p 3306:3306 \
                        -e MYSQL_DATABASE=${MYSQL_DB_NAME} -e MYSQL_ROOT_PASSWORD=${MYSQL_DB_ROOT} -e MYSQL_USER=${MYSQL_DB_USER} -e MYSQL_PASSWORD=${MYSQL_DB_PASSWORD} \
                        -v /home/vagrant/mysql:/var/lib/mysql \
                        --name mysqldb mysql \
                        --default-authentication-plugin=mysql_native_password"
                  }               
               }
            )
         }
      }
      stage('Staging Deploy') {//providing delay for mysql to start
         steps {   
            sh '''
		            sleep 5
            '''
            script {
                def remote = [:]
                remote.name = 'staging'
                remote.user = 'vagrant'
                remote.allowAnyHosts = true
                remote.host = 'staging.devops'
                remote.identityFile = '~/.ssh/staging.key'
                sshCommand remote: remote, command: "docker run -d -p 8080:8080 --link mysqldb -e MYSQL_DB_USER=${MYSQL_DB_USER} \
                  -e MYSQL_DB_PASSWORD=${MYSQL_DB_PASSWORD} -e MYSQL_JDBC_URL=${MYSQL_STAGING_URL} -e MYSQL_DB_NAME=${MYSQL_DB_NAME} \
                  -v /home/vagrant/logs:/home/boot/logs/ --name backend ${DOCKER_REGISTRY}/devops/api:staging"
                sshCommand remote: remote, command: "docker run -d -p 80:80 --link backend \
                  --name frontend ${DOCKER_REGISTRY}/devops/ui:staging"                  
            }
         }
      }
      stage('UAT Tests') {
         steps {   
               echo 'UAT Tests'
         }
      }
      stage('DAST') {
         steps {
               sh '''
                  # remove wrk folder
                  rm -rf wrk

                  # create wrk folder
                  mkdir wrk

                  chmod 777 wrk
                  docker run \
                     --volume $(pwd)/wrk:/output:rw \
                     --volume $(pwd)/wrk:/zap/wrk:rw \
                     registry.gitlab.com/gitlab-org/security-products/dast:latest /analyze -t http://staging.devops -x report.xml

   

                 echo "Scan Report Created"
               '''
         }
      }      
      stage('Production Setup') {
         steps {
            parallel(
               ui: { // Prepare the Docker image for the staging ui
                  sh '''
                     mv frontend/nginx-prod.conf frontend/nginx.conf
                     docker build --no-cache --build-arg STAGE=prod -t "devops/ui:prod" -f frontend/Dockerfile .
                     docker tag "devops/ui:prod" "${DOCKER_REGISTRY}/devops/ui:prod"
                     docker push "${DOCKER_REGISTRY}/devops/ui:prod"
                     docker rmi "${DOCKER_REGISTRY}/devops/ui:prod"
                     docker rmi "devops/ui:prod"
                  '''
               },
               api: {
                  sh '''
                     docker build --no-cache --build-arg FILE_NAME=${GIT_COMMIT} -t "devops/api:prod" -f backend/Dockerfile .
                     docker tag "devops/api:prod" "${DOCKER_REGISTRY}/devops/api:prod"
                     docker push "${DOCKER_REGISTRY}/devops/api:prod"
                     docker rmi "${DOCKER_REGISTRY}/devops/api:prod"
                     docker rmi "devops/api:prod"
                  '''
               },
               db: { 
                  // Parallely start the MySQL Daemon in the staging server first stop if already running then start
                  withCredentials([string(credentialsId: 'mysqlroot', variable: 'mysqlroot'), string(credentialsId: 'mysqldbpw', variable: 'mysqldbpw')]){
                     script {
                        def remote = [:]
                        remote.name = 'production'
                        remote.user = 'vagrant'
                        remote.allowAnyHosts = true
                        remote.host = 'production.devops'
                        remote.identityFile = '~/.ssh/production.key'
                        sshCommand remote: remote, command: "docker stop mysqldb backend frontend || true"
                           sshCommand remote: remote, command: "docker rm backend mysqldb frontend || true"
                           sshCommand remote: remote, command: "docker rmi ${DOCKER_REGISTRY}/devops/api:prod ${DOCKER_REGISTRY}/devops/ui:prod || true"
                           sshCommand remote: remote, command: "docker run -d -p 3306:3306 \
                           -e MYSQL_DATABASE=${MYSQL_DB_NAME} -e MYSQL_ROOT_PASSWORD=${mysqlroot} \
                           -e MYSQL_USER=${MYSQL_DB_NAME} -e MYSQL_PASSWORD=${mysqldbpw} \
                           -v /home/vagrant/mysql:/var/lib/mysql \
                           --name mysqldb mysql \
                           --default-authentication-plugin=mysql_native_password"
                     }
                  }               
               }
            )
         }
      }
      stage('Production Deploy Approval') {
         steps {
            script {
                  input message: 'Do you approve Deployment ?', ok: 'OK'
            }
         }
      }      
      stage('Production Deploy') {       
         steps {   
            script {
                def remote = [:]
                remote.name = 'production'
                remote.user = 'vagrant'
                remote.allowAnyHosts = true
                remote.host = 'production.devops'
                remote.identityFile = '~/.ssh/production.key'
                sshCommand remote: remote, command: "docker run -d -p 8080:8080 --link mysqldb \
                   -e VAULT_TOKEN_MYSQL=${VAULT_TOKEN_MYSQL} -e MYSQL_DB_NAME=${MYSQL_DB_NAME} \
                   -e VAULT_PATH_MYSQL=${VAULT_PATH_MYSQL} -e MYSQL_JDBC_URL=${MYSQL_PROD_URL} -e VAULT_ADDR=${VAULT_ADDR} \
                   -v /home/vagrant/logs:/home/boot/logs/ --name backend ${DOCKER_REGISTRY}/devops/api:prod"
                sshCommand remote: remote, command: "docker run -d -p 80:80 --link backend \
                  --name frontend ${DOCKER_REGISTRY}/devops/ui:prod"                  
            }
         }
      }                       
   }
   post {
    failure {
      script {
        currentBuild.result = 'FAILURE'
      }
    }
    always {
      step([$class: 'Mailer', notifyEveryUnstableBuild: true,recipients: "build-failed@devops.local",sendToIndividuals: true])
      step([$class: 'WsCleanup'])
    }
  }
}
