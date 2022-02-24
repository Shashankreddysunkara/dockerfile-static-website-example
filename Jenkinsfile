def customImage = ""
  node("linux") {

    stage("source") {
    git 'https://github.com/Shashankreddysunkara/dockerfile-static-website-example.git'
    }
    stage("build docker") {
    customImage = docker.build("dock101/helloinsta")
    }
    stage("verify image") {
        try {
    sh '''
        docker run --rm -d -p 8000:8000/tcp --name phonebook dock101/helloinsta
        sleep 20s
        curl_response=$(curl -s -o /dev/null -w "%{http_code}" 'http://localhost:8000')
        if [ $curl_response -eq 200 ]
        then
            exit 0
        else
            exit 1
        fi
    ''' 
    } catch (Exception e) {
    sh '''
        docker stop phonebook
        docker rm phonebook
    '''
    }
    stage("cleanup container") {
    sh '''
        docker rm phonebook -f
    '''
    }
    }
    stage("push to DockerHub") {
        echo "Push to Dockerhub"
    withDockerRegistry(credentialsId: 'dockerhub.creds') {
    customImage.push("${env.BUILD_NUMBER}")
    customImage.push("latest")
    }
    }
    
    stage("deploy to EKS") {
    sh '''              
        kubectl apply -f deployment.yml
        kubectl set image deployment/phonebook phonebook=dock101/helloinsta:"${BUILD_NUMBER}" --record
        kubectl apply -f service.yml
        kubectl apply -f loadbalancer.yml
        sleep 20s
        kubectl get svc phonebook-lb -o jsonpath="{.status.loadBalancer.ingress[*]['ip', 'hostname']}" > appUrl.txt
    '''
    }
    stage("apply HPA") {
        try {
    sh '''
        export KUBECONFIG=/home/ubuntu/kubeconfig_ops-eks
        kubectl autoscale deployment phonebook --cpu-percent=70 --min=1 --max=4
    '''
     } catch (Exception e) {
    sh '''
        export KUBECONFIG=/home/ubuntu/kubeconfig_ops-eks
        kubectl delete hpa phonebook
        kubectl autoscale deployment phonebook --cpu-percent=70 --min=1 --max=4
    '''
        }
    }
    stage("performance test") {
    sh '''
        export KUBECONFIG=/home/ubuntu/kubeconfig_ops-eks
        APP_URL=$(kubectl get svc phonebook-lb -o jsonpath="{.status.loadBalancer.ingress[*]['ip', 'hostname']}")
        sed 's/@SERVER@/'$APP_URL'/g' load-test.jmx > load-test-act.jmx
        jmeter -n -t load-test-act.jmx -l /home/ubuntu/load-test-"${BUILD_NUMBER}".jtl 
    '''
    }
    stage("slack message"){
        APP_URL = readFile('appUrl.txt').trim()
        slackSend color: "good", message: "Build  #${env.BUILD_NUMBER} Finished Successfully. App URL: ${APP_URL}"
    }
}
