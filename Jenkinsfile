pipeline {

    agent {
        label 'Agent1'
    }

    environment {
        appVersion = ''
        REGION = 'us-east-1'
        ACC_ID ='989088456804'
        PROJECT = 'roboshop'
        COMPONENT = 'user'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    parameters {
        string ( name: 'appVersion', description: 'image version of the application')
        choice (name: 'deploy_to', choices:['dev', 'qa', 'prod'], description: 'Pick the Environment')
    }
    stages {
         stage('Check Status') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        sh """
                            aws eks update-kubeconfig --region ${REGION} --name "${PROJECT}-${params.deploy_to}"
                        """

                        def deploymentStatus = sh(
                            returnStdout: true,
                            script: """
                                kubectl rollout status deployment/${env.COMPONENT} --timeout=60s -n ${PROJECT} || echo FAILED
                            """
                        ).trim()

                        if (deploymentStatus.contains("successfully rolled out")) {
                            echo "✅ Deployment successful!"
                        } else {
                            echo "⚠️ Deployment failed. Checking if rollback is possible..."
                            // Check if a previous Helm revision exists
                            def revision = sh(returnStdout: true, script: "helm history ${COMPONENT} -n ${PROJECT} | tail -n +2 | wc -l").trim().toInteger()

                            if (revision > 1) {
                                echo "🔁 Rolling back to previous revision..."
                                sh """
                                    helm rollback ${COMPONENT} -n ${PROJECT}
                                    sleep 20
                                    kubectl rollout status deployment/${COMPONENT} --timeout=60s -n ${PROJECT} || echo FAILED
                                """
                                echo "deployment failed but rollback success"
                            } else {
                                echo "🚫 No previous Helm release found — skipping rollback."
                                error "Deployment failed (no rollback possible)."
                            }
                        }
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws eks update-kubeconfig --region $REGION --name "$PROJECT-${params.deploy_to}"
                            kubectl get nodes
                            kubectl apply -f 01-namespace.yaml
                            sed -i "s/IMAGE_VERSION/${params.appVersion}/g" values-${params.deploy_to}.yaml
                            helm upgrade --install $COMPONENT -f values-${params.deploy_to}.yaml -n $PROJECT .
                        """
                    }
                }
            }
        }
       

    //Api testing
    stage('functional testing') {
        when {
            expression { params.deploy_to == "dev"}
         }
            steps {
                script {
                   echo "Functional testing started" 
                 }    
            }
       
    }
     //All components testing
    stage('integration testing') {
        when {
            expression { params.deploy_to == "qa"}
         }
            steps {
                script {
                    echo "integration testing started"  
                 }    
            }
       
    }
     stage('prod-deploy') {
         when {
            expression { params.deploy_to == "prod"}
         }  
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            echo "get cr number"
                            echo "check with in the deployment_window"
                            echo "is CR approved"
                            echo "trigger PROD deploy"
                        """
                    }
                }
            }
       
    }
}

    post {
        always {
            echo 'Cleaning workspace'
            deleteDir()
        }

        success {
            echo 'Pipeline Success'
        }

        failure {
            echo 'Pipeline Failed'
        }
    }
}