pipeline {
  environment {
    registry = "thedeepsyadav/devsecops-training"
    registryCredential = "DockerHub"
    dockerImage = ''
  }
  
  agent any
  
  stages {
    stage('Check for Secrets'){
      steps {
        sh "rm -rf trufflehog.json || true"
        sh "docker run dxa4481/trufflehog:latest --json https://github.com/deepshankaryadav/CyberFRAT-DevSecOps-Training-Sample-Flask-App.git > trufflehog.json || true"
        sh "cat trufflehog.json"
      }
    }
    
    stage('SCA'){
      steps {
        sh "pip3 install safety"
        sh "rm -rf safety.json || true"
        sh "safety check -r requirements.txt --json > safety.json || true"
        sh "cat safety.json"
      }
    }
    
    stage('SAST'){
      steps {
        sh "rm -rf bandit.json || true"
        sh "bandit -r -f=json -o=bandit.json . || true"
        sh "cat bandit.json"
      }
    }
    
    stage('Build Docker Image') {
      steps {
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    
    stage('Push to DockerHub') {
      steps {
        script {
          docker.withRegistry('', registryCredential ) {
            dockerImage.push('latest')
          }
        }
      }
    }
    
    stage('Deploy Test Application') {
      steps {
        sh 'docker stop flaskr && docker rm flaskr || true'
        sh 'docker pull thedeepsyadav/devsecops-training:latest'
        sh 'docker run -d -p 5000:5000 --name flaskr thedeepsyadav/devsecops-training:latest'
      }
    }
    
    stage("Docker Security"){
      parallel{
        stage("Docker Benchmarking"){
          steps{
            sh '''
            docker pull docker/docker-bench-security
            docker run --net host --pid host --cap-add audit_control -v /var/lib:/var/lib -v /var/run/docker.sock:/var/run/docker.sock -v /usr/lib/systemd:/usr/lib/systemd -v /etc:/etc --label docker_bench_security docker/docker-bench-security || true
            '''
          }
        }
        
        stage("Image Scanning"){
          steps{
            sh '''
            DOCKER_GATEWAY=$(docker network inspect bridge --format "{{range .IPAM.Config}}{{.Gateway}}{{end}}")
            wget -qO clair-scanner https://github.com/arminc/clair-scanner/releases/download/v8/clair-scanner_linux_amd64 && chmod +x clair-scanner
            ./clair-scanner --ip="$DOCKER_GATEWAY" thedeepsyadav/devsecops-training:latest || exit 0
            '''
          }
        }
      }
    }
    
    stage("Compliance as Code"){
      steps{
        sh '''
        curl https://omnitruck.chef.io/install.sh | sudo bash -s -- -P inspec || true
        inspec exec https://github.com/dev-sec/cis-docker-benchmark || true
        inspec exec https://github.com/dev-sec/linux-baseline || true
        '''
      }
    }
    
    stage("Deploy to PROD"){
      steps{
        script{
          input message: 'Do you want to deploy in production?', ok: "OK"
        }
      }
    }
   
    stage('Deploy to Application Server') {
      steps {
        sshagent(['AppSec']) {
          sh 'ssh -o StrictHostKeyChecking=no root@159.65.157.103 "uptime && docker pull thedeepsyadav/devsecops-training:latest && docker stop devsecops-training && docker rm devsecops-training && docker run -d -p 5000:5000 --name devsecops-training thedeepsyadav/devsecops-training:latest"'
          sh 'ssh -o StrictHostKeyChecking=no root@159.65.157.103 "inspec exec https://github.com/dev-sec/linux-baseline || true"'
        }
      }
    }
    
    stage("Push Data to DefectDojo"){
      steps{
        sh '''
        DATE= date +%Y-%m-%d
        curl -i -F 'file=@trufflehog.json' -H 'Authorization: ApiKey admin:af88fe9e8ab6524b3497d10c201fdf564d4eaff3' -F 'scan_type=Trufflehog Scan' -F 'tags=apicurl' -F 'verified=true' -F 'active=true' -F 'scan_date=2020-11-08' -F 'engagement=/api/v1/engagements/1/' http://159.65.157.103:8080/api/v1/importscan/
        '''
      }
    }
  } 
}
