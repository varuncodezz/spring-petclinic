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
        stage('Performance Tests') {
            steps {
                container('jmeter'){
                    sh "jmeter -n -t ${env.WORKSPACE}/src/test/jmeter/petclinic_test_plan.jmx -l result.jtl -e -o result"
                    perfReport filterRegex: '', sourceDataFiles: '**/*.jtl'
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'result', reportFiles: 'index.html', reportName: 'JMeter-Report', reportTitles: ''])
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
                        sh "trivy image --quiet --vuln-type os,library --exit-code 1 --severity CRITICAL brainupgrade/spring-petclinic:${env.BUILD_ID}"
                    }
                }
            }
        }        
    }
}
