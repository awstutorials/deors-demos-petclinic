pipeline {
    agent {
        docker {
            image 'maven:3.6.3-jdk-8-slim'
            args '-v /root/.m2:/root/.m2 --network ci -v /var/run/docker.sock:/var/run/docker.sock'        
        }
    }
    environment {
        ORG_NAME = "deors"
        APP_NAME = "deors-demos-petclinic"
        APP_CONTEXT_ROOT = "petclinic"
        TEST_CONTAINER_NAME = "ci-${APP_NAME}-${BUILD_NUMBER}"
        DOCKER_HUB = credentials("${ORG_NAME}-docker-hub")
    }
    stages {
        stage('Compile') {
            steps {
                echo "-=- compiling project -=-"
                sh "mvn clean compile"
            }
        }

        stage('Package') {
            steps {
                echo "-=- packaging project -=-"
                sh "mvn package -DskipTests"
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }

        stage('Build Docker image') {
            steps {
                echo "-=- build Docker image -=-"
                sh "mvn docker:build"
            }
        }

        stage('Run Docker image') {
            steps {
                echo "-=- run Docker image -=-"
                echo "${PATH}"
                sh "docker run --name ${TEST_CONTAINER_NAME} --detach --rm --network ci --expose 6300 --env JAVA_OPTS='-javaagent:/usr/local/tomcat/jacocoagent.jar=output=tcpserver,address=*,port=6300' ravisankar/petclinic:latest"
            }
        }

        stage('Integration tests') {
            steps {
                echo "-=- execute integration tests -=-"
                sh "mvn failsafe:integration-test failsafe:verify -DargLine=\"-Dtest.selenium.hub.url=http://selenium-hub:4444/wd/hub -Dtest.target.server.url=http://${TEST_CONTAINER_NAME}:8080/${APP_CONTEXT_ROOT}\""
                sh "java -jar target/dependency/jacococli.jar dump --address ${TEST_CONTAINER_NAME} --port 6300 --destfile target/jacoco-it.exec"
                junit 'target/failsafe-reports/*.xml'
                step( [ $class: 'JacocoPublisher' ] )
            }
        }

        stage('Performance tests') {
            steps {
                echo "-=- execute performance tests -=-"
                sh "mvn jmeter:jmeter jmeter:results -Djmeter.target.host=${TEST_CONTAINER_NAME} -Djmeter.target.port=8080 -Djmeter.target.root=${APP_CONTEXT_ROOT}"
                perfReport sourceDataFiles: 'target/jmeter/results/*.csv', errorUnstableThreshold: 0, errorFailedThreshold: 5, errorUnstableResponseTimeThreshold: 'petclinic.jtl:100'
            }
        }

        stage('Dependency vulnerability tests') {
            steps {
                echo "-=- run dependency vulnerability tests -=-"
                sh "mvn dependency-check:check"
                dependencyCheckPublisher failedTotalHigh: 30, unstableTotalHigh: 25
            }
        }

        stage('Code inspection & quality gate') {
            steps {
                echo "-=- run code inspection & check quality gate -=-"
                withSonarQubeEnv('ci-sonarqube') {
                    sh "mvn sonar:sonar"
                }
                timeout(time: 10, unit: 'MINUTES') {
                    //waitForQualityGate abortPipeline: true
                    script  {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK' && qg.status != 'WARN') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

    }

    post {
        always {
            echo "-=- remove deployment -=-"
            sh "docker stop ${TEST_CONTAINER_NAME}"
        }
    }
}
