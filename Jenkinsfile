pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: gradle
            image: gradle:jdk11-alpine
            imagePullPolicy: IfNotPresent
            command:
            - sleep
            args:
            - 99d
          - name: kaniko
            image: gcr.io/kaniko-project/executor:v1.6.0-debug
            imagePullPolicy: IfNotPresent
            command:
            - sleep
            args:
            - 99d
            volumeMounts:
              - name: jenkins-docker-cfg
                mountPath: /kaniko/.docker
          - name: grype
            image: alpine:latest
            imagePullPolicy: IfNotPresent
            command:
            - sleep
            args:
            - 99d          
            env:
              - name: DOCKER_CONFIG
                value: /config
            volumeMounts:
              - name: jenkins-docker-cfg
                mountPath: /config
          - name: zap-dast
            image: owasp/zap2docker-stable:2.11.0
            imagePullPolicy: IfNotPresent
            command:
            - sleep
            args:
            - 99d
          volumes:
          - name: jenkins-docker-cfg
            projected:
              sources:
              - secret:
                  name: artifact-registry
                  items:
                    - key: .dockerconfigjson
                      path: config.json
        '''
    }
  }

    environment {
      PROJETO = 'gradle-test'
      NOME = 'walber'
      EMAIL = 'walber.silva@sysmap.com.br'
      NAMESPACE = 'bc-renan-medina'
      PORTA = '8080'


      REGISTRY = 'us-central1-docker.pkg.dev/sysmap-coe-tecnologia'
      REGISTRY_DIR = 'coe-images/gradle'

      // credenciais  
      MVN_SET = credentials('maven_settings')
      CRED_VAULT_SONAR = 'sonar-login'
      CRED_GITLAB = 'gitlab-w'
      CRED_KUBERNETES = 'sa-renan-internal'
      
    }
    
        stages {
            
              stage('Checkout sources') {
                steps {
                  checkout scm
                  sh 'chmod +x ./gradlew'
                }
              }
 
              stage('Gradle') {
                steps {
                  container('gradle') {
                    sh './gradlew clean build'
                  }
                }
              }
        
              stage('Unit Test') {
                steps {
                  container('gradle') {
                    sh './gradlew test'
                  }
                }
              }
            
              stage('Sast') {
                steps {
                  container('gradle') {

                    withCredentials([[$class: 'VaultUsernamePasswordCredentialBinding', credentialsId: env.CRED_VAULT_SONAR, usernameVariable: 'SONAR_LOGIN']]) {
                      sh './gradlew sonarqube -Dsonar.host.url=http://sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 -Dsonar.login=$SONAR_LOGIN'

                    }
                  }
                }
              }

              stage('Kaniko') {
                steps {
                    container('kaniko') {
                      sh '/kaniko/executor --context `pwd` --destination $REGISTRY/$REGISTRY_DIR/$NOME/$PROJETO:$BUILD_NUMBER'
                  }
                }
              }

                stage('Grype Security Scan') {
                  steps{
                    container('grype') {
                      sh 'apk add curl'
                      sh 'curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin'
                      sh 'grype $REGISTRY/$REGISTRY_DIR/$NOME/$PROJETO:$BUILD_NUMBER'
                    }
                  }
                }

                stage('Commit') {
                  steps {
                    withCredentials([gitUsernamePassword(credentialsId: env.CRED_GITLAB, gitToolName: 'Default')]) {
                      sh """
                      git config --global user.name $NOME
                      git config --global user.email $EMAIL
                      sed -i 's,$REGISTRY.*,$REGISTRY/$REGISTRY_DIR/$NOME/$PROJETO,' helm/values.yaml
                      sed -i 's,tag:.*",tag: \"$BUILD_NUMBER\",' helm/values.yaml
                      sed -i 's,namespace:.*,namespace: $NAMESPACE,' helm/values.yaml
                      sed -i 's,name:.*,name: $NOME,' helm/Chart.yaml
                      git add helm/values.yaml helm/Chart.yaml
                      git commit -am 'Alterando a versao do projeto por $NOME'
                      git push origin HEAD:master
                      """
                    }
                  }
              }
              
/*
                stage('Deploy with helm') {
                  steps{
                    container('helm') {
                        withKubeConfig([credentialsId: env.CRED_KUBERNETES,
                                        namespace: env.NAMESPACE,
                                ]) {
                          sh 'helm upgrade --install $PROJETO helm/'
                        }
                      }
                  }
                }

                stage('ZAP - DAST') {
                  steps{
                    container('zap-dast') {
                      sh 'zap-api-scan.py -f openapi -t http://$PROJETO-$NOME.$NAMESPACE.svc.cluster.local:$PORTA'
                    }
                  }
                }
*/
        }
}

