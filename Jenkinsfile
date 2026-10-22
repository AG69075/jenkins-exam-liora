pipeline {
    agent any
    environment {
        DOCKER_ID  = 'ag69075'
        DOCKER_TAG = "v${env.BUILD_ID}"
    }
    stages {
        stage('Build') {
            parallel {
                stage('movie') { steps { sh 'docker build -t $DOCKER_ID/movie-service:$DOCKER_TAG ./movie-service' } }
                stage('cast')  { steps { sh 'docker build -t $DOCKER_ID/cast-service:$DOCKER_TAG ./cast-service' } }
                stage('nginx') { steps { sh 'docker build -t $DOCKER_ID/nginx-gateway:$DOCKER_TAG ./nginx' } }
            }
        }
        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                        usernameVariable: 'DU', passwordVariable: 'DP')]) {
                    sh '''
                        echo "$DP" | docker login -u "$DU" --password-stdin
                        docker push $DOCKER_ID/movie-service:$DOCKER_TAG
                        docker push $DOCKER_ID/cast-service:$DOCKER_TAG
                        docker push $DOCKER_ID/nginx-gateway:$DOCKER_TAG
                    '''
                }
            }
        }
        stage('Deploy dev')     { steps { deployAll('dev',     '30080') } }
        stage('Deploy qa')      { steps { deployAll('qa',      '30081') } }
        stage('Deploy staging') { steps { deployAll('staging', '30082') } }
        stage('Deploy prod') {
            when { branch 'master' }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Déployer en production ?', ok: 'Deploy'
                }
                deployAll('prod', '30083')
            }
        }
    }
}

def deployAll(String ns, String nodePort) {
    withEnv(["NS=${ns}", "NODE_PORT=${nodePort}"]) {
        withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG')]) {
            sh '''
                export PATH=$PATH:/snap/bin
                H="helm upgrade --install --create-namespace -n $NS"
                $H movie-db  ./charts -f helm-values/movie-db.yaml
                $H cast-db   ./charts -f helm-values/cast-db.yaml
                $H movie     ./charts -f helm-values/movie.yaml --set image.tag=$DOCKER_TAG
                $H cast      ./charts -f helm-values/cast.yaml  --set image.tag=$DOCKER_TAG
                $H nginx     ./charts -f helm-values/nginx.yaml --set image.tag=$DOCKER_TAG --set service.nodePort=$NODE_PORT
            '''
        }
    }
}
