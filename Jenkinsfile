#!/usr/bin/env groovy

pipeline {
    agent { label 'master' }

    stages {

        stage('Build Container') {

            steps {
                sh "docker build -t jgaafromnorth/nginx-swarm-proxy:v${env.BUILD_ID} ."
                sh "docker tag jgaafromnorth/nginx-swarm-proxy:v${env.BUILD_ID} jgaafromnorth/nginx-swarm-proxy:latest"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withDockerRegistry([ credentialsId: "8cb91394-2af2-4477-8db8-8c0e13605900", url: "" ]) {
                    sh "docker push jgaafromnorth/nginx-swarm-proxy:v${env.BUILD_ID}"
                    sh 'docker push jgaafromnorth/nginx-swarm-proxy:latest'
                }
            }
        }
    }
}
