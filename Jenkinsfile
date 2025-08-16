@Library('iti-sharedlib')_

pipeline {
    agent any
    
    options {
        disableConcurrentBuilds()
    }

    environment {
        JAVA_HOME = tool name: 'java-11', type: 'jdk'
        MAVEN_HOME = tool name: 'mvn-3-5-4', type: 'maven'
        DOCKER_USER = credentials('docker-username')
        DOCKER_PASS = credentials('docker-password')
        PATH = "${JAVA_HOME}/bin:${MAVEN_HOME}/bin:${env.PATH}"
        DOCKER_IMAGE = "ahmedasaadrashad/iti-java"  // Add your Docker Hub username if needed
    }

    stages {
        stage("Get code") {
            steps {
                checkout scmGit(
                    branches: [[name: '*/master']], 
                    extensions: [], 
                    userRemoteConfigs: [[url: 'https://github.com/ahmedasaadrashad/java.git']]
                )
            }
        }
        
        stage("Build app") {
            steps {
                sh 'java -version'
                sh 'mvn -version'
                script {
                    def mavenBuild = new org.iti.mvn()
                    mavenBuild.javaBuild("package install")
                }
            }
        }
        
        stage("Archive app") {
            steps {
                archiveArtifacts artifacts: '**/*.jar', followSymlinks: false
            }
        }
        
        stage("Docker build") {
            steps {
                script {
                    def docker = new com.iti.docker()
                    docker.build("${env.DOCKER_IMAGE}", "${BUILD_NUMBER}")
                }
            }
        }
        
        stage("Push java app image") {
            steps {
                script {
                    def docker = new com.iti.docker()
                    // More secure login method
                    sh "echo ${env.DOCKER_PASS} | docker login -u ${env.DOCKER_USER} --password-stdin"
                    // Push with your Docker Hub username in the tag
                    docker.push("${env.DOCKER_IMAGE}", "${BUILD_NUMBER}")
                }
            }
        }
        
       stage("Update ArgoCD manifest") {
           steps {
                script {
                // Clone the ArgoCD repo
                sh "mkdir -p argocd"
                dir('argocd') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/main']],
                        extensions: [],
                        userRemoteConfigs: [[
                            url: 'https://github.com/ahmedasaadrashad/argocd.git',
                            credentialsId: 'git-cred'  // Jenkins credential for GitHub access
                        ]]
                    ])

                    // Update the image tag in deployment.yaml
                    sh "sed -i 's#        image: .*#        image: ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER}#' iti-dev/deployment.yaml"
                    sh "cat iti-dev/deployment.yaml"
                    sh "ls"
                    sh "git add ."
                    sh "git commit -m 'update Image'"
                // Commit and push changes
                    withCredentials([usernamePassword(
                        credentialsId: 'git-cred',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                )]) {
                        sh """
                            git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/ahmedasaadrashad/argocd.git
                            git push origin HEAD:main
                            """
                }
                }
            }
    }
}
    }
}
