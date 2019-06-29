# jenkins-pipeline

## Include in the Jenkinsfile
`@Library('github.com/vinothzomato/jenkins-pipeline@master')`

## Define the pipeline
`def pipeline = new com.zomato.Pipeline()`

## Example
```
#!/usr/bin/groovy

// load pipeline functions
// Requires pipeline-github-lib plugin to load library from github
@Library('github.com/vinothzomato/jenkins-pipeline@master')
def pipeline = new com.zomato.Pipeline()

podTemplate(label: 'jenkins-pipeline', containers: [
    containerTemplate(name: 'jnlp', image: 'jenkinsci/jnlp-slave:3.29-1-alpine', args: '${computer.jnlpmac} ${computer.name}', workingDir: '/home/jenkins', resourceRequestCpu: '500m', resourceLimitCpu: '500m', resourceRequestMemory: '1024Mi', resourceLimitMemory: '1024Mi'),
    containerTemplate(name: 'docker', image: 'docker:18.06.0', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'helm', image: 'vinothzomato/k8s-helm:latest', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'vinothzomato/k8s-kubectl:latest', command: 'cat', ttyEnabled: true)
],
volumes:[
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
]){

  node ('jenkins-pipeline') {

    def pwd = pwd()
    def chart_dir = "${pwd}/charts/yourchart"
    def tags = [env.BUILD_TAG, 'latest']
    def docker_registry_url = "registry-intl-vpc.ap-south-1.aliyuncs.com"
    def app_hostname = "yourhostname";
    def docker_repo = "repo"
    def docker_acct = "namespace"
    def jenkins_registry_cred_id = "registry-intl-vpc.ap-south-1.aliyuncs.com"

    // checkout sources
    checkout scm

    // set additional git envvars for image tagging
    pipeline.gitEnvVars()

    // Test Helm deployment (dry-run)
    stage ('Helm lint') {

      container('helm') {

        // run helm chart linter
        pipeline.helmLint(chart_dir)
      }
    }

    // Build and push the Docker image
    stage ('Build & Push Docker image') {

      container('docker') {
        println "build & push"

        // perform docker login
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: jenkins_registry_cred_id, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
          sh "docker login -u ${env.USERNAME} -p ${env.PASSWORD} ${docker_registry_url}"
        }

        // build and publish container
        pipeline.containerBuildPub(
            dockerfile: "./",
            host      : docker_registry_url,
            acct      : docker_acct,
            repo      : docker_repo,
            tags      : tags,
            auth_id   : jenkins_registry_cred_id
        )
      }
    }
    
    // Deploy the new version to Kubernetes
    stage ('Deploy to Kubernetes') {
        container('helm') {

          // Deploy using Helm chart
           pipeline.helmDeploy(
            dry_run       : false,
            name          : "hello-java",
            namespace     : "hello-java",
            version_tag   : tags.get(0),
            chart_dir     : chart_dir,
            replicas      : 2,
            cpu           : "10m",
            memory        : "128Mi",
            hostname      : app_hostname
          )
        }
      }
  }
}
```
