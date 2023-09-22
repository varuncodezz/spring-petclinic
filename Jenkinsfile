pipeline {
    agent {
        label 'jenkins-k8s-agent'
    }
    options {
      disableConcurrentBuilds()
      buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        SCANNER_HOME=tool 'sonarqube'
    }

    stages {
        stage('Compile') {
            steps {
                container('maven') {
                  sh 'mvn clean compile  -Dcheckstyle.skip'
                }
            }
        }
        stage('JUnit Tests') {
            steps {
                container('maven') {
                    sh 'mvn test  -Dcheckstyle.skip jacoco:report'
                    junit 'target/surefire-reports/**/*.xml'
                }
            }
        }        
        stage('Code Coverage') {
            steps {
                container('maven') {
                    jacoco(
                        execPattern: '**/target/*.exec',
                        classPattern: '**/target/classes',
                        sourcePattern: '**/src/main'
                    )
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'target/site/jacoco', reportFiles: 'index.html', reportName: 'CodeCoverage-Report', reportTitles: ''])
                }
            }
        }        
        stage('Build') {
            steps {
                container('maven') {
                  sh 'mvn install  -Dcheckstyle.skip -DskipTests'
                }
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonarqube') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Petclinic \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=Petclinic '''
    
                }
            }
        }
        
        // stage("OWASP Dependency Check"){
        //     steps{
        //         dependencyCheck additionalArguments: '--scan ./ --format HTML ', odcInstallation: 'owasp-dc'
        //         dependencyCheckPublisher pattern: '**/dependency-check-report.html'
        //     }
        // }

        stage("Docker Build & Push"){
            steps{
                container('docker') {
                    withDockerRegistry(credentialsId: 'docker-hub-credentials', url: 'https://index.docker.io/v1/') {                        
                        sh "docker build -t brainupgrade/spring-petclinic:${env.BUILD_ID} ."
                        sh "docker push brainupgrade/spring-petclinic:${env.BUILD_ID} "
                    }
                }
            }
        }        
        stage('Image Scan'){
            steps{
                script{
                    container('docker'){
                        sh "wget https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl"
                    }
                    container('trivy'){
                        sh "trivy image --severity HIGH,CRITICAL --format template --template '@html.tpl'   --output image-cve.html brainupgrade/spring-petclinic:${env.BUILD_ID}"
                        publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '.', reportFiles: 'image-cve.html', reportName: 'ImageScan-CVE-Trivy-Report', reportTitles: 'Trivy Image Scan'])
                        // sh "trivy image --quiet --vuln-type os,library --exit-code 1 --severity CRITICAL brainupgrade/spring-petclinic:${env.BUILD_ID}"
                    }
                }
            }
        }     
        stage('Deploy E2E') {
            when {
                branch 'main'
            }      
            environment {
                GIT_CREDS = credentials('bu-github-credentials')
            }
            steps {
                input message:'Approve deployment to E2E?'
                container('tools') {
                sh "git clone https://$GIT_CREDS_USR:$GIT_CREDS_PSW@github.com/brainupgrade-in/gitops-k8s-apps.git"
                sh "git config --global user.email 'ci@ci.com'"

                dir("gitops-k8s-apps") {
                    sh "cd ./petclinic/e2e && kustomize edit set image brainupgrade/petclinic:${env.GIT_COMMIT}"
                    sh "git commit -am 'Publish new version' && git push || echo 'no changes'"
                }
                }
            }
        }

        stage('Performance Tests') {
            container('tools'){
                sh "/bin/bash -c echo -n 'Waiting for petclinic to be up http://petclinic.e2e.svc.cluster.local ...' && until nc -z petclinic.e2e.svc.cluster.local 80; do sleep 1 && echo -n .; done && echo 'http://petclinic.e2e.svc.cluster.local is up now!'"
            }

            container('jmeter'){
                sh "jmeter -n -t ${env.WORKSPACE}/src/test/jmeter/petclinic_test_plan.jmx -l result.jtl -e -o result"
                perfReport filterRegex: '', sourceDataFiles: '**/*.jtl'
                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'result', reportFiles: 'index.html', reportName: 'JMeter-Report', reportTitles: ''])
            }
        }        

        stage('Deploy to UAT') {
            when {
                branch 'main'
            }      
            steps {
                input message:'Approve deployment to UAT?'
                container('tools') {
                dir("gitops-k8s-apps") {
                    sh "cd ./petclinic/uat && kustomize edit set image brainupgrade/petclinic:${env.GIT_COMMIT}"
                    sh "git commit -am 'Publish new version' && git push || echo 'no changes'"
                }
                }
            }
        }

        stage('Deploy to Prod') {
            when {
                branch 'main'
            }      
            steps {
                input message:'Approve deployment to PROD?'
                container('tools') {
                dir("gitops-k8s-apps") {
                    sh "cd ./petclinic/prod && kustomize edit set image brainupgrade/petclinic:${env.GIT_COMMIT}"
                    sh "git commit -am 'Publish new version' && git push || echo 'no changes'"
                }
                }
            }
        }
        stage('final') {
            when {
                branch 'main'
            }      
            steps {
                container('docker') {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh "docker login -u ${USERNAME} -p ${PASSWORD}"
                    sh "docker tag brainupgrade/petclinic:${env.GIT_COMMIT} brainupgrade/petclinic:latest"
                    sh "docker push brainupgrade/petclinic:latest"
                }
                }
            }
        }            
    }
}
