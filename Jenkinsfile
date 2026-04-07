@Library('sharedlibrary')_

pipeline {
    environment {
        ecrRegistry = "070797855002.dkr.ecr.ap-south-1.amazonaws.com"
        backendImage = "${ecrRegistry}/backend"
        frontendImage = "${ecrRegistry}/frontend"
        modelImage = "${ecrRegistry}/model"
        branchName = sh(script: 'echo $BRANCH_NAME | sed "s#/#-#"', returnStdout: true).trim()
        gitCommit = "${GIT_COMMIT[0..6]}"
        snykOrg = "14141617-a2e0-4a4f-b558-dce1ea5cad2d"
        dockerTag = "${branchName}-${gitCommit}-${env.BUILD_NUMBER}"
        gitRepoURL = "https://github.com/artisantek/sept2025.git"
    }
    agent {
        label 'agent'
    }
    stages {
        stage('Git Checkout') {
            steps {
                gitCheckout("$gitRepoURL", "$BRANCH_NAME", "githubCred")
            }
        }

        // stage("Sonarqube Analysis ") {
        //     steps {
        //         withSonarQubeEnv('sonar-server') {
        //             dir('movie-analyzer-app/frontend') { 
        //                 sh '''
        //                 $SCANNER_HOME/bin/sonar-scanner \
        //                 -Dsonar.projectName="movie-analyzer" \
        //                 -Dsonar.projectKey="movie-analyzer"
        //                 '''
        //             }
        //         }
        //     }
        // }

        // stage("quality gate"){
        //    steps {
        //         script {
        //             waitForQualityGate abortPipeline: false, credentialsId: 'sonarToken' 
        //         }
        //     } 
        // }


        stage('Docker Build') {
            parallel {
                stage('Build Backend') {
                    steps {
                        dir('backend') {
                            dockerImageBuild('$backendImage', '$dockerTag')
                        }
                    }
                }
                stage('Build Frontend') {
                    steps {
                        dir('frontend') {
                            dockerImageBuild('$frontendImage', '$dockerTag')
                        }
                    }
                }
                stage('Build Model') {
                    steps {
                        dir('model') {
                            dockerImageBuild('$modelImage', '$dockerTag')
                        }
                    }
                }
            }
        }

        stage('Security Scans') {
            parallel {
                stage('Snyk Scan Backend') {
                    steps {
                        snykImageScan('$backendImage', '$dockerTag', 'snykCred', '$snykOrg')
                    }
                }
                stage('Snyk Scan Frontend') {
                    steps {
                        snykImageScan('$frontendImage', '$dockerTag', 'snykCred', '$snykOrg')
                    }
                }
                stage('Snyk Scan Model') {
                    steps {
                        snykImageScan('$modelImage', '$dockerTag', 'snykCred', '$snykOrg')
                    }
                }
            }
        }

        stage('Trivy Scans') {
            parallel {
                stage('Trivy Scan Backend') {
                    steps {
                        sh "trivy image -f json -o backend-results-${BUILD_NUMBER}.json ${backendImage}:${dockerTag}"
                    }
                }
                stage('Trivy Scan Frontend') {
                    steps {
                        sh "trivy image -f json -o frontend-results-${BUILD_NUMBER}.json ${frontendImage}:${dockerTag}"
                    }
                }
                stage('Trivy Scan Model') {
                    steps {
                        sh "trivy image -f json -o model-results-${BUILD_NUMBER}.json ${modelImage}:${dockerTag}"
                    }
                }
            }
        }

        stage('Docker Push') {
            parallel {
                stage('Push Backend') {
                    steps {
                        dockerECRImagePush('$backendImage', '$dockerTag', 'ap-south-1')
                    }
                }
                stage('Push Frontend') {
                    steps {
                        dockerECRImagePush('$frontendImage', '$dockerTag', 'ap-south-1')
                    }
                }
                stage('Push Model') {
                    steps {
                        dockerECRImagePush('$modelImage', '$dockerTag', 'ap-south-1')
                    }
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                dir('helm') {
                    sh """
                        sed -i 's|__BACKEND_IMAGE_REPOSITORY__|${backendImage}|g' values.yaml
                        sed -i 's|__FRONTEND_IMAGE_REPOSITORY__|${frontendImage}|g' values.yaml
                        sed -i 's|__MODEL_IMAGE_REPOSITORY__|${modelImage}|g' values.yaml
                        sed -i 's|__BACKEND_IMAGE_TAG__|${dockerTag}|g' values.yaml
                        sed -i 's|__FRONTEND_IMAGE_TAG__|${dockerTag}|g' values.yaml
                        sed -i 's|__MODEL_IMAGE_TAG__|${dockerTag}|g' values.yaml
                    """
                }
            }
        }

        stage('Kubernetes Deploy - DEV') {
            when {
                branch 'dev'
            }
            steps {
                kubernetesEKSHelmDeploy('movie-analyzer', 'dev')
            }
        }

        stage('Kubernetes Deploy - STAGING') {        
            when {
                branch 'staging'
            }
            steps {
                kubernetesEKSHelmDeploy('movie-analyzer', 'staging')
            }
        }

        stage('Kubernetes Deploy - PROD') {
            when {
                branch 'master'
            }
            steps {
                kubernetesEKSHelmDeploy('movie-analyzer', 'prod')
            }
        }

    }
}
