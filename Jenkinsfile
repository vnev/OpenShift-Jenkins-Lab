def appName = "birthday-paradox"
def replicas = "1"
def devProject = "vnev-dev"
def testProject = "vnev-test"
def prodProject = "vnev-prod"

def skopeoToken
def imageTag

def getVersionFromPom() {
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    matcher ? matcher[0][1] : null
}

def skopeoCopy(def skopeoToken, def srcProject, def destProject, def appName, def imageTag) {
    sh """skopeo copy --src-tls-verify=false --src-creds=jenkins:${skopeoToken} \
    --dest-tls-verify=false --dest-creds=jenkins:${skopeoToken} \
    docker://image-registry.openshift-image-registry.svc:5000/${srcProject}/${appName}:${imageTag} \
    docker://image-registry.openshift-image-registry.svc:5000/${destProject}/${appName}:${imageTag}"""
}

def deployApplication(def appName, def imageTag, def project, def replicas) {
    openshift.withCluster() {
        openshift.withProject(project) {
            dir("openshift") {
                def result = openshift.process(readFile(file:"deploy.yaml"), "-p", "APPLICATION_NAME=${appName}", "-p", "IMAGE_TAG=${imageTag}", "-p", "APPLICATION_PROJECT=${project}")
                openshift.apply(result)
            }
            openshift.selector("deployment", appName).scale("--replicas=${replicas}")
        }
    }
}

pipeline {
    agent { label "maven" }
    stages {
        stage("Setup") {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(devProject) {
                            skopeoToken = openshift.raw("sa get-token jenkins").out.trim()
                        }
                    }
                    imageTag = getVersionFromPom()
                }
            }
        }
        stage("Build & Test") {
            steps {
                // TODO: Build, Test, and Package birthday-paradox using Maven
                sh "mvn clean install"
            }
        }
        stage("Create Image") {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(devProject) {
                            dir("openshift") {
                                /* TODO: Process and Apply the build.yaml OpenShift template. 
                                **       This template will create the birthday-paradox BuildConfig and ImageStream
                                **       There is a similar example for this in the deployApplication() function at the top of this file. Reference that function but write your implementation here.
                                **       Be sure to look at the openshift/build.yaml file to check what parameters the template requires
                                */
                                def result = openshift.process(readFile(file:"build.yaml"), "-p", "APPLICATION_NAME=${appName}", "-p", "IMAGE_TAG=${imageTag}")
                                openshift.apply(result)
                            }
                            dir("target") {
                                openshift.selector("bc", appName).startBuild("--from-file=${appName}-${imageTag}.jar").logs("-f")
                            }
                        }
                    }
                }
            }
        }
        stage("Deploy Application to Dev") {
            steps {
                script {
                    /*
                    ** TODO: Use the deployApplication() function, defined above, to deploy birthday-paradox to Dev
                    **       Be sure to use the parameters that have already been defined in the pipeline.
                    */
                    deployApplication(appName, imageTag, devProject, 1)
                }
            }
        }
        stage("Copy Image to Test") {
            agent { label "jenkins-agent-skopeo" }
            steps {
                script {
                    skopeoCopy(skopeoToken, devProject, testProject, appName, imageTag)
                }
            }
        }
        stage("Deploy Application to Test") {
            steps {
                script {
                    /*
                    ** TODO: Use the deployApplication() function, defined above, to deploy birthday-paradox to Test
                    **       Be sure to use the parameters that have already been defined in the pipeline.
                    */
                    deployApplication(appName, imageTag, testProject, 1)
                }
            }
        }
        stage("Prompt for Prod Approval") {
            steps {
                input "Deploy to prod?"
            }
        }
        stage("Copy image to Prod") {
            agent { label "jenkins-agent-skopeo" }
            steps {
                script {
                    skopeoCopy(skopeoToken, devProject, prodProject, appName, imageTag)
                }
            }
        }
        stage("Deploy Application to Prod") {
            steps {
                script {
                    /*
                    ** TODO: Use the deployApplication() function, defined above, to deploy birthday-paradox to Prod
                    ** Be sure to use the parameters that have already been defined in the pipeline.
                    */
                    deployApplication(appName, imageTag, prodProject, 1)
                }
            }
        }
    }
}
