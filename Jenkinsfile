def LABEL_ID = "jenkins-slave-node-${UUID.randomUUID().toString()}"
def REPOS
def IMAGE_VERSION
def IMAGE_POSFIX = ""
def IMAGE_NAME = "frontend"
def APP_NAME = "questcode-frontend"
def KUBE_NAMESPACE
def ENVIRONMENT
def GIT_BRANCH 
def HELM_CHART_NAME = "questcode/questcode-frontend"
def HELM_DEPLOY_NAME
def CHARTMUSEUM_URL = "http://helm-repo-chartmuseum:8080"
def INGRESS_HOST = "questcode.org"

podTemplate(
    name: 'jenkins-slave-node',
    namespace: 'devops',
    label: 'LABEL_ID',
    containers: [
      containerTemplate(args: 'cat', command: '/bin/sh -c', image: 'docker', livenessProbe: containerLivenessProbe(execArgs: '', failureThreshold: 0, initialDelaySeconds: 0, periodSeconds: 0, successThreshold: 0, timeoutSeconds: 0), name: 'docker-container', resourceLimitCpu: '', resourceLimitMemory: '', resourceRequestCpu: '', resourceRequestMemory: '', ttyEnabled: true),
      containerTemplate(args: 'cat', command: '/bin/sh -c', image: 'lachlanevenson/k8s-helm:v2.11.0', name: 'helm-container', ttyEnabled: true)
    ],
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')],
)
{
    node('LABEL_ID') {
        stage('Checkout') {
            echo 'INICIANDO O CLONE DO REPOSITORIO'
            REPOS = checkout scm
            GIT_BRANCH = REPOS.GIT_BRANCH

            if(GIT_BRANCH.equals("master")){
                KUBE_NAMESPACE = "prod"
                ENVIRONMENT = "production"
            } else if (GIT_BRANCH.equals("develop")) {
                KUBE_NAMESPACE = "staging"
                ENVIRONMENT = "staging"
                IMAGE_POSFIX = "-RC"
                INGRESS_HOST = ENVIRONMENT + "." + INGRESS_HOST
            } else if (GIT_BRANCH.equals("qa")) {
                KUBE_NAMESPACE = "qa"
                ENVIRONMENT = "staging"
                IMAGE_POSFIX = "-QA"
                INGRESS_HOST = ENVIRONMENT + "." + INGRESS_HOST
            } else {
                def error = "NAO EXISTE PIPELINE PARA A BRANCH: ${GIT_BRANCH}"
                echo error
                throw new Exception(error)
            }
            HELM_DEPLOY_NAME = APP_NAME + "-" + KUBE_NAMESPACE
            IMAGE_VERSION = sh returnStdout: true, script: 'sh read-package-version.sh'
            IMAGE_VERSION = IMAGE_VERSION.trim() + IMAGE_POSFIX
        }
        stage('Package'){
            container('docker-container') {
                echo 'INICIANDO O EMPACOTAMENTO COM O DOCKER'
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                    sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}"
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION} . --build-arg NPM_ENV='${ENVIRONMENT}'"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION}"
                }
            }
        }
    }

    if(GIT_BRANCH.equals("master")){
        timeout(time: 15, unit: 'MINUTES'){
            input message: "Efetuar o Deploy do ${APP_NAME}:${IMAGE_VERSION}?", ok: 'Continue'
        }
    }

    node('LABEL_ID') {
        stage('Deploy') {
            container('helm-container'){
                echo "INICIANDO O DEPLOY COM O HELM"
                sh """
                    helm init --client-only
                    helm repo add questcode ${CHARTMUSEUM_URL}
                    helm repo update
                """
                try {
                    sh "helm upgrade --namespace=${KUBE_NAMESPACE} ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set image.tag=${IMAGE_VERSION} --set ingress.hosts[0]=${INGRESS_HOST}"
                } catch(Exception e) {
                    sh "helm install --namespace=${KUBE_NAMESPACE} --name ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set image.tag=${IMAGE_VERSION} --set ingress.hosts[0]=${INGRESS_HOST}"
                }
            }
        }
    }
}