def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]

pipeline {
    agent any
    environment {
        WORKSPACE = "${env.WORKSPACE}"
        NEXUS_CREDENTIAL_ID = 'Nexus-Credential'
        ANSIBLE_CREDENTIAL_ID = 'Ansible-Credential'
        WORKSPACE_PATH = "/var/lib/jenkins/workspace/Jenkins-Complete-CICD-Pipeline"
    }
    tools {
        maven 'localMaven'
        jdk 'localJdk'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
            post {
                success {
                    echo ' now Archiving '
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }
        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Integration Test') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
        stage('Checkstyle Code Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        stage('SonarQube Inspection') {
            steps {
                withSonarQubeEnv('SonarQube') { 
                    withCredentials([string(credentialsId: 'SonarQube-Token', variable: 'SONAR_TOKEN')]) {
                        sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=JavaWebApp-Project \
                        -Dsonar.host.url=http://172.31.30.12:9000 \
                        -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }
        stage('SonarQube GateKeeper') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage("Nexus Artifact Uploader") {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '172.31.22.103:8081',
                    groupId: 'webapp',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'maven-project-releases',
                    credentialsId: "${NEXUS_CREDENTIAL_ID}",
                    artifacts: [
                        [artifactId: 'webapp',
                        classifier: '',
                        file: "${WORKSPACE}/webapp/target/webapp.war",
                        type: 'war']
                    ]
                )
            }
        }
        stage('Deploy to Development Env') {
            environment {
                HOSTS = 'dev'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "${ANSIBLE_CREDENTIAL_ID}", passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    script {
                        echo "Running Ansible Playbook for Development Environment"
                        echo "Workspace path: ${WORKSPACE_PATH}"
                        echo "Hosts: tag_Environment_${HOSTS}"
                        sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
                    }
                }
            }
        }
        stage('Deploy to Staging Env') {
            environment {
                HOSTS = 'stage'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "${ANSIBLE_CREDENTIAL_ID}", passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    script {
                        echo "Running Ansible Playbook for Staging Environment"
                        echo "Workspace path: ${WORKSPACE_PATH}"
                        echo "Hosts: tag_Environment_${HOSTS}"
                        sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
                    }
                }
            }
        }
        stage('Quality Assurance Approval') {
            steps {
                input('Do you want to proceed?')
            }
        }
        stage('Deploy to Production Env') {
            environment {
                HOSTS = 'prod'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "${ANSIBLE_CREDENTIAL_ID}", passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    script {
                        echo "Running Ansible Playbook for Production Environment"
                        echo "Workspace path: ${WORKSPACE_PATH}"
                        echo "Hosts: tag_Environment_${HOSTS}"
                        sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#george-my-jenkins-cicd-pipeline-alerts',
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job Name '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \n Build Timestamp: ${env.BUILD_TIMESTAMP} \n Project Workspace: ${env.WORKSPACE} \n More info at: ${env.BUILD_URL}"
        }
    }
}
