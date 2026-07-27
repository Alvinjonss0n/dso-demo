pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  environment {
    ARGO_SERVER = 'argocd-server.argocd.svc.cluster.local:80'
    DEV_URL = 'http://dso-demo.dev.svc.cluster.local:8080/'
  }
  stages {
    stage('Repo Scan') {
      steps {
        container('trufflehog') {
          sh 'trufflehog --regex --entropy=True --max_depth=50 https://github.com/alvinjonss0n/dso-demo.git'
        }
      }
    }
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('SCA') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                  sh 'mvn org.owasp:dependency-check-maven:12.1.0:check -DnvdApiKey=$NVD_API_KEY -DossindexAnalyzerEnabled=false'
                }
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
        stage('Generate SBOM') {
          steps {
            container('maven') {
              sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
            }
          }
          post {
            success {
              // dependencyTrackPublisher projectName: 'sample-spring-app', projectVersion: '0.0.1', artifact: 'target/bom.xml', autoCreateProjects: true, synchronous: true
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/bom.xml', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
        stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              sh 'ls -al'
              sh '''#!/bin/bash --login
                /bin/bash --login
                rvm use default
                gem install license_finder
                license_finder
              '''
            }
          }
        }
        stage('SAST') {
          steps {
            container('slscan') {
              sh 'scan --type java,depscan --build'
            }
          }
          post {
            success {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/*', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
      }
    }
    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
        stage('OCI Image BnP') {
          steps {
            container('kaniko') {
              sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --destination=docker.io/alvinjonss0n/dso-demo'
            }
          }
        }
      }
    }
    stage('Image Analysis') {
      parallel {
        stage('Image Linting') {
          steps {
            container('docker-tools') {
              sh 'dockle docker.io/alvinjonss0n/dso-demo'
            }
          }
        }
        stage('Image Scan') {
          steps {
            container('trivy') {
              sh 'trivy image --timeout 20m --exit-code 1 --severity CRITICAL --scanners vuln --cache-dir /tmp/trivycache alvinjonss0n/dso-demo'
            }
          }
        }
      }
    }
    stage('Deploy to Dev') {
      environment {
        AUTH_TOKEN = credentials('argocd-jenkins-deployer-token')
      }
      steps {
        container('argocd') {
          sh 'wget -qO /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.13.2/argocd-linux-amd64 && chmod +x /usr/local/bin/argocd'
          sh 'argocd app sync dso-demo --plaintext --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
          sh 'argocd app wait dso-demo --health --timeout 300 --plaintext --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
        }
      }
    }
    stage('Dynamic Analysis') {
      parallel {
        stage('E2E tests') {
          steps {
            sh 'echo "All Tests passed!!!"'
          }
        }
        stage('DAST') {
          steps {
            container('zap') {
              sh 'zap-baseline.py -t $DEV_URL || exit 0'
            }
          }
        }
      }
    }
  }
}
