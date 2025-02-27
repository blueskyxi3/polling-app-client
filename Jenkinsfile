def label = "slave-${UUID.randomUUID().toString()}"

podTemplate(label: label, containers: [
  containerTemplate(name: 'node', image: 'node:alpine', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'cnych/kubectl', command: 'cat', ttyEnabled: true)
], volumes: [
  hostPathVolume(mountPath: '/root/.kube', hostPath: '/root/.kube'),
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
  node(label) {
    def myRepo = checkout scm
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH
    def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    def dockerRegistryUrl = "registry.citictel.com"
    def imageEndpoint = "demo/polling-ui"
    def image = "${dockerRegistryUrl}/${imageEndpoint}"
    def branch = gitBranch.substring(gitBranch.indexOf("/")+1)
    

    stage('版本檢查') {
      echo "版本檢查"
      sh """
         echo " ----------------------- "
         echo "gitBranch-->${gitBranch}"
         echo "set BranchOrTag is ${BranchOrTag}"
         echo "BUILD_NUMBER:${env.BUILD_NUMBER}"
         echo "BUILD_ID:${env.BUILD_ID}"
         echo "JOB_NAME:${env.JOB_NAME}"
         echo "BUILD_TAG:${env.BUILD_TAG}"
         echo "branch:${branch}"
         echo "imageTag--->${imageTag}"
         echo " ----------------------- "
         """   
      if(gitBranch.indexOf("/")>0){
        imageTag = branch +"-"+ imageTag
      }else{
        echo "----else---"
        imageTag = BranchOrTag
      }
      echo "imageTag--->${imageTag}"
    }
      
    stage('单元测试') {
      echo "1.测试阶段"
      echo "set BranchOrTag is ${BranchOrTag}"  
    }
    
    stage('编译打包') {
      echo "2.编译打包"
      container('node') {
        sh """
          npm install
          npm run build
          ls -la
        """
      }
    }
    stage('构建 Docker 镜像') {
      withCredentials([[$class: 'UsernamePasswordMultiBinding',
        credentialsId: 'dockerhub',
        usernameVariable: 'DOCKER_HUB_USER',
        passwordVariable: 'DOCKER_HUB_PASSWORD']]) {
          container('docker') {
            echo "3. 构建 Docker 镜像阶段"
            sh """
              ls -la
              docker login ${dockerRegistryUrl} -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}
              docker build -t ${image}:${imageTag} .
              docker push ${image}:${imageTag}
              """
          }
      }
    }
     stage('运行 Kubectl') {
      
      container('kubectl') {          
       echo "查看 K8S 集群 Pod 列表"
        sh "kubectl get pods -n demo"   
        sh """
          sed -i "s/<BUILD_TAG>/${imageTag}/" manifests/k8s.yaml
          sed -i "s/<CI_ENV>/${branch}/" manifests/k8s.yaml
          kubectl apply -f manifests/k8s.yaml
          """
         sh "kubectl get pods -n demo"   
      }
    }
  }
}
