pipeline {
    agent any
    environment{
        sonarScanner = tool 'SONAR6.2'
        registryCredential = 'ecr:us-east-1:jenkinsecrcreds'
        imageName = "596517178703.dkr.ecr.us-east-1.amazonaws.com/atomecrrepo"
        ecrRegistry = "https://596517178703.dkr.ecr.us-east-1.amazonaws.com"
    }
    tools{
        maven "MAVEN3.9"
        jdk "JDK17"
    }
    stages {
        stage('Fetch Code'){
            steps{
                git branch: 'main', url: 'https://github.com/Shadab909/atom-project-app.git'
            }
        }
        stage('Build'){
            steps{
                sh 'mvn clean install -DskipTests'
            }
            post{
                success {
                    echo 'Archiving'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }
        stage('Unit Test'){
            steps{
                sh 'mvn test'
            }
        }
        stage('Checkstyle Analysis'){
            steps{
                sh 'mvn checkstyle:checkstyle'
            }
        }
        stage('SonarQube Code Analysis'){
            steps{
                withSonarQubeEnv('SONAR-SERVER'){
                    sh '''${sonarScanner}/bin/sonar-scanner -Dsonar.projectKey=atom-project \
                   -Dsonar.projectName=atom-project \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }
        stage('Build App Image with Docker'){
            steps{
                script{
                    dockerImage = docker.build(imageName + ":$BUILD_NUMBER", ".")
                }
            }
        }
        stage('Upload Image to ECR'){
            steps{
                script{
                    docker.withRegistry(ecrRegistry,registryCredential){
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }
        stage('Remove Container from Local'){
            steps{
                sh 'docker rmi -f $(docker images -aq)'
            }
        }
    }
}