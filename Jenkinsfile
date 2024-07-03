#!groovy
@Library(["pipelineuk@2.8.5", "pipelineuk-sec@5.0.1"]) _

import java.text.SimpleDateFormat


def service_account_credID = "S0503254_USRPWD"
def project_name = "alm-api"
def registry_url = "registry-ukdev01a.paas.santanderuk.dev.corp"

pipeline {
    agent { 
        label 'python3' 
    }
    
    
    //environment {
    //    project = 'alm-api'
    //    app = 'alm-core-api'
    //    configProfile = 'alm-core-api'
    //    routePath = 'alm-core-api'   
    //}

    
    options {
        skipDefaultCheckout() // Needed to avoid Git checkout for Docker slave
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(daysToKeepStr: '2')) 
        disableResume() //Do not allow the pipeline to resume if the Jenkins master restarts
    }
    
    stages {
        stage ('Checkout from GIT') {
            steps {
                checkout scm
            }
        }
        
        stage('Environment for BRANCH Builds') {
            when {
                branch 'development'
            }
            steps {
                hlSetEnv('APP_NAME', 'alm-core-api')
                hlSetEnv('IMAGE_TAG', 'latest')
                hlSetEnv('CONFIG_LABEL', 'develop')
                print "Image Tag - ${IMAGE_TAG}"
            }
        }
        stage('Environment for TAG Builds') {
            when {
                buildingTag()
            }
            steps {
                script {
                    def image_tag = sh(script: "python setup.py --version", returnStdout: true).trim()
                    hlSetEnv('APP_NAME', 'alm-core-api')
                    hlSetEnv('IMAGE_TAG', image_tag)
                    hlSetEnv('CONFIG_LABEL', 'master')
                    print "Image Tag is ${IMAGE_TAG}"
                    if (IMAGE_TAG != env.BRANCH_NAME) {
                        error "Version in setup.py (${IMAGE_TAG}) does not match Tag in source code GitLab (${env.BRANCH_NAME})"
                    }
                    sleep 180 //sleep 3 mins to avoid race with develop branch builds.
                }
            }
        }

        stage('Build source') {
            steps {
                stash name: 'app'
            }
        }

        stage ('Unit Test') {
            // TODO: Remove once Jenkins gets updated with this limit applied to the Maven slave directly.
            environment {
                SPRING_CONFIG_LOCATION = 'http://config-service:8080' // Java heap memory size - limit set to 1/4 available memory
            }
            steps {
                //sh 'pip install  --index-url=https://nexus-proxy.almuk.santanderuk.corp/repository/pypi-repo/simple/ --trusted-host=nexus-proxy.almuk.santanderuk.corp --upgrade setuptools'
                sh 'echo "python setup.py test"'
            }
        }
        //stage('Code Coverage') {
        //    steps {
        //        sh "flake8 almapi/"
        //    }
        //}
        //stage('Vulnerabilities Checks') {
        //    steps {
        //        sh "bandit -r almapi/"
        //    }
        //}
        //stage('Dependency Checks') {
        //    steps {
        //        sh "safety check --proxy-host=b2bproxy.santanderuk.corp --proxy-port=8080 --proxy-protocol=https"
        //    }

        //}

            
        
        
        
//        stage ('Vulnerability/Dependency Check') {
//            environment {
//               SONAR_SCANNER_OPTS='-Xmx256m' // Java heap memory size - limit set to 1/4 available memory
//            }
//            steps {
//                hlsecSAST(
//                    'java',
//                    'alm-core-api',
//                    'alm-core-api',
//                    [:],
//                    [:]
//                )
//            }
//        }
//
        
        stage('Build Image') {
            agent {
                label 'docker'
            }
            steps {
                unstash 'app'
                hlBuildImage('DockerImage', [
                    image             : "${registry_url}/${project_name}/${APP_NAME}:${IMAGE_TAG}",
                    dockerCredentialId: service_account_credID
                ])
            }
        }

        
        
        stage ('Kraken Test') {
            steps {
                hlKrakenScan([
                    maxRetry    :  20,     // helps when harbor scan takes time
                    registry    : registry_url,
                    project     : project_name,
                    imageName   : APP_NAME,
                    tag         : IMAGE_TAG,
                    serviceAccount: service_account_credID
                ])
            }
        } 

        stage('Check Project Quota') {
            steps {
                hlCheckQuota([
                    memoryAlert: 4,  // GB
                    logLevel   : "WARNING"
                ])
            }
        }

        stage('Deploy to DEV') {
            steps {
                hlDeployImage('DockerImage', [
                    appName        : APP_NAME, //Deployment Name in PaaS
                    dockerImage    : "${registry_url}/${project_name}/${APP_NAME}:${IMAGE_TAG}",
                    imagePullPolicy: "Always",
                    triggers       : [[type: "ConfigChange"]],
                    ports          : [[containerPort: 8080, protocol: "TCP"]],
                    createRoute    : true, //internal-route
                    routePath      : "${APP_NAME}",
                    routeHost: "alm-api-dev.santanderuk.pre.corp",
                    labels         : [
                        image_tag: "${IMAGE_TAG}",
                        app      : "${APP_NAME}"
                    ],
                    annotations    : [
                        deployed_jenkins_job_name_id: "${APP_NAME}-${env.BUILD_NUMBER}"
                    ],
                    resources: [
                        requests: [cpu: "300m", memory: "512M"],
                        limits: [cpu: "900m", memory: "1G"]
                    ],
                    livenessProbe:[
                        httpGet:[path:"/alm-core-api/v1/version",port:8080,scheme:"HTTP"],
                        initialDelaySeconds:80,timeoutSeconds:10,periodSeconds:10,
                        successThreshold:1,failureThreshold:3
                    ],
                    readinessProbe:[
                        httpGet:[path:"/alm-core-api/v1/version",port:8080,scheme:"HTTP"],
                        initialDelaySeconds:30,timeoutSeconds:10,periodSeconds:10,
                        successThreshold:1,failureThreshold:3
                    ],
                    envs           : [
                        [name: "APP_NAME", valueFrom: [fieldRef: [apiVersion: "v1", fieldPath: "metadata.name"]]],
                        [name: "TZ", value: "Europe/London"],
                        [name: "LDAP_SECURITY", value: "True"],
                        [name: "SPRING_CONFIG_LOCATION", value: 'http://config-service:8080'],
                        [name: "SPRING_CONFIG_ENVIRONMENT", value: "Dev"],
                        [name: "SPRING_CONFIG_ENABLED", value: "true"],
                        [name: "SPRING_CONFIG_LABEL", value: 'master'],
                        [name: "SPRING_CONFIG_PROFILE", value: 'alm-core-api'],
                        [name: "LOG_LEVEL", value: '2']
                    ]
                ])
            }
        }

        
        
        stage('Verify Service') {
            steps {
                openshiftVerifyService(svcName: APP_NAME, verbose: 'false')
            }
        }

        stage('Create F5 Route') {
            steps {
                hlCreateOcpRoute([
                    annotations: [live: "false"], // Set to 'true' if route is going to be live.
                    routeName  : "${APP_NAME}-f5",
                    service    : [name: APP_NAME],
                    targetPort : 8080,
                    routeHost  : "${project_name}-dev.santanderuk.pre.corp",
                    routePath  : "${APP_NAME}/",
                    labels     : [internet: "false"],   //Set true or false depending on your application.
                ])
            }
        }

        stage('Trigger Pre Release') {
            when {
                buildingTag()
            }
            steps {
                hlBuildJenkinsRemoteJob([
                    jenkinsUrl    : "https://cd-jenkins-pre.alm.santanderuk.pre.corp",
                    jobName       : "alm-api-pre/job/alm-core-api/job/alm-core-api-release/",
                    token         : "alm-core-api-token",
                    urlJobParams  : "VERSION=${IMAGE_TAG}",
                    ServiceAccount: service_account_credID
                ])              
            }
        }

    }
    
    post {
        always {
            cleanWs()
            echo "All stages finished running"
        }
        success {
            echo "Job finished successfully"
        }
        failure {
            emailext(
                to: '$DEFAULT_RECIPIENTS',
                subject: "Jenkins FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "<a href='${env.BUILD_URL}'>Fix me!</a>",
                recipientProviders: [
                    [$class: 'DevelopersRecipientProvider'], 
                    [$class: 'CulpritsRecipientProvider'],
                    [$class: 'RequesterRecipientProvider'], 
                    [$class: 'FirstFailingBuildSuspectsRecipientProvider']
                ]
            )
        }
    }
}
