pipeline {
    agent {
        label 'docker'
    }
    tools {
        maven 'maven' // Use the name configured in Jenkins
    }
    environment {
        WORKSPACE_DIR = '/var/lib/jenkins/workspace/adq-java-app'
        GCS_BUCKET = 'gs://adq-java-app'
        PROJECT_ID = 'gcp-adq-pocproject-dev'
        ZONE = 'us-central1-c'
        INSTANCE_NAME = 'get-ubuntudesktop'
        TARGET_HOST_PATH = '/opt/tomcat/apache-tomcat-10.1.25'
        SONARQUBE_SERVER = 'SonarQube' // Name configured in Jenkins for SonarQube server
        SONARQUBE_PROJECT_KEY = 'adq-java-app'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    def gitInfo = checkout([$class: 'GitSCM',
                        branches: [[name: "*/${BRANCH_NAME}"]],
                        userRemoteConfigs: [[url: 'https://github.com/SaravanaNani/jenkins-java-project.git']]
                    ])
                    def branchName = gitInfo.GIT_BRANCH.tokenize('/')[1]
                    echo "Branch name: ${branchName}"
                }
            }
        }

        stage('Package') {
            steps {
                sh '''
                mvn clean compile
                mvn test
                mvn package
                '''                
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn sonar:sonar -Dsonar.projectKey=${SONARQUBE_PROJECT_KEY} -Dsonar.login=${SONAR_AUTH_TOKEN} -Dsonar.branch.name=${BRANCH_NAME}'
                    }
                }
            }
        }

        stage('Upload Artifact') {
            steps {
                script {
                    // Rename the WAR file
                    sh '''
                    cd ${WORKSPACE_DIR}/target/
                    mv JAVA_APP-1.2.*.war JAVA_APP-1.2.${BUILD_NUMBER}.war
                    '''

                    // Set the path of the artifact and upload path
                    def artifactPath = "${WORKSPACE_DIR}/target/JAVA_APP-1.2.${BUILD_NUMBER}.war"
                    def uploadPath = "${BRANCH_NAME}/target/JAVA_APP-1.2.${BUILD_NUMBER}.war"

                    // Upload to Google Cloud Storage using gsutil
                    sh """
                    /google-cloud-sdk/bin/gsutil cp ${artifactPath} ${GCS_BUCKET}/${uploadPath}
                    """
                }
            }
        }

        stage('Artifact Quality Check') {
            steps {
                script {
                    // Assuming you have a tool or script to check the artifact quality
                    sh '''
                    # Example: running a security scan or quality check on the artifact
                    security-scan-tool ${WORKSPACE_DIR}/target/JAVA_APP-1.2.${BUILD_NUMBER}.war
                    '''
                }
            }
        }

        stage('Confirmation') {
            steps {
                script {
                    input message: 'Are you sure you want to proceed with the deployment?', ok: 'Yes'
                    
                    def instanceStatus = sh(script: "gcloud compute instances describe ${INSTANCE_NAME} --project=${PROJECT_ID} --zone=${ZONE} --format='get(status)'", returnStdout: true).trim()
                    
                    if (instanceStatus != 'RUNNING') {
                        error "VM instance is not running. Deployment stopped."
                    }

                    env.PRIVATE_IP = sh(script: '''
                        gcloud compute instances list --filter="labels.adq_ubuntudesktop=app" --format="value(networkInterfaces[0].networkIP)" --limit=1
                    ''', returnStdout: true).trim()

                    echo "Private IP: ${env.PRIVATE_IP}"
                }
            }
        }

        stage('Deployment') {
            steps {
                script {
                    sh '''
                    # Remove all .war files in the target directory
                    rm -f ${WORKSPACE_DIR}/target/*.war
                    
                    # Get WAR from Artifactory
                    /google-cloud-sdk/bin/gsutil cp ${GCS_BUCKET}/${BRANCH_NAME}/target/JAVA_APP-1.2.${BUILD_NUMBER}.war ${WORKSPACE_DIR}/target/
                    ls -al ${WORKSPACE_DIR}/target/
                    
                    # Shutdown Tomcat
                    ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/id_rsa root@${PRIVATE_IP} "${TARGET_HOST_PATH}/bin/shutdown.sh"

                    # Remove old WAR files and directories
                    ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/id_rsa root@${PRIVATE_IP} "find ${TARGET_HOST_PATH}/webapps/ -type d -name 'JAVA_APP-1.2.*' -exec rm -rf {} +"
                    ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/id_rsa root@${PRIVATE_IP} "find ${TARGET_HOST_PATH}/webapps/ -type f -
