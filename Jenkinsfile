#!groovy

pipeline {
    tools {
        maven "M3"
    }

    agent any

    stages {

        stage ('Checkout') {
            steps {
                checkout scm
                script {
                    def v = version()
                    if (v) { echo "Building version ${v}" }
                }
                stash includes: '*/**', name: 'allCheckout'
            }
        }

        stage ('Compile') {
            steps {
                unstash 'allCheckout'
                sh "${tool 'M3'}/bin/mvn compile"

                stash includes: '*/**', name: 'allAfterCompile'
            }
        }

        stage ('Sonar') {
            steps {
                unstash 'allAfterCompile'
                sh "${tool 'M3'}/bin/mvn sonar:sonar -Dsonar.branch=${env.BRANCH_NAME} -Dsonar.exclusions='**/vendors/**,**/tests/**,**/test/**' -Dsonar.host.url='http://localhost:9000'"
            }
        }

        stage ('Unit Test') {
            steps {
                runTests()
            }
        }

        stage ('Package') {
            steps {

                unstash 'allAfterCompile'
                sh "${tool 'M3'}/bin/mvn package -DskipTests=true"

                // Sauvegarde du war
                stash includes: '**/*.war', name: 'war'
            }
        }

        stage ('Deploy Tomcat') {
            steps {
                script {
                    def workspacePwd = pwd()

                    unstash 'allAfterCompile'
                    unstash 'war'

                    // Deploy de l'application
                    sh "${tool 'M3'}/bin/mvn tomcat7:deploy-only -Dmaven.tomcat.charset='UTF-8' -Dmaven.tomcat.update=true -Dmaven.tomcat.url='http://localhost:9966/manager/text' -DwarFile='${workspacePwd}/target/petclinic.war'"

                    input message:'Ok ?'

                    // Undeploy de l'application
                    sh "${tool 'M3'}/bin/mvn tomcat7:undeploy -Dmaven.tomcat.path='/petclinic' -Dmaven.tomcat.url='http://localhost:9966/manager/text'"
                }
            }
        }
    }
    post {
        success {
            unstash 'war'
            archiveArtifacts artifacts: '**/*.war', fingerprint: true
        }
    }
}

// Joue les tests de maniere parallele
def runTests() {

    def mvnHome = tool 'M3'
    def splits = splitTests count(2)
    def testGroups = [:]
    for (int i = 0; i < splits.size(); i++) {

        def index = i
        testGroups["split${i}"] = {
            node {
                unstash 'allAfterCompile'
                def exclusions = splits.get(index);
                writeFile file: 'exclusions.txt', text: exclusions.join("\n")

                def mavenTest = 'test -DMaven.test.failure.ignore=true -Pexclusions'
                sh "${tool 'M3'}/bin/mvn ${mavenTest}"

                junit '**/target/surefire-reports/TEST-*.xml'
            }
        }
    }
    parallel testGroups
}


// Methodes utilitaires pour Maven
def version() {
  def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
