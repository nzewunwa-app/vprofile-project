def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]

pipeline {
    agent any

    tools {
        maven "MAVEN3.9.9"
        jdk "JDK17"
    }
    environment {
        APP_NAME = "vprofile"
        DEPLOY_ENV = "dev"

        SNAP_REPO = 'vprofile-snapshot'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUS_GRP_REPO = 'vpro-maven-group'

        NEXUSIP = '10.0.26.97'
        NEXUSPORT = '8081'
        NEXUS_LOGIN = 'nexuslogin'

        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
    }

    options {
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                splunkinsSend(
                    message: "Checkout completed",
                    metadata: [
                        stage: "checkout",
                        branch: env.GIT_BRANCH,
                        commit: env.GIT_COMMIT
                    ]
                )
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -s settings.xml clean package -DskipTests'

                archiveArtifacts artifacts: '**/*.war'

                splunkinsSend(
                    message: "Build completed",
                    metadata: [
                        stage: "build",
                        artifact: "vprofile.war"
                    ]
                )
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn -s settings.xml test'
                junit 'target/surefire-reports/*.xml'

                splunkinsSend(
                    message: "Unit tests executed",
                    metadata: [
                        stage: "unit_test",
                        test_results: "surefire"
                    ]
                )
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'

                splunkinsSend(
                    message: "Checkstyle completed",
                    metadata: [
                        stage: "checkstyle"
                    ]
                )
            }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh '''${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                splunkinsSend(
                    message: "SonarQube scan completed",
                    metadata: [
                        stage: "sonarqube",
                        sonar_server: SONARSERVER
                    ]
                )
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }

                splunkinsSend(
                    message: "Quality gate passed",
                    metadata: [
                        stage: "quality_gate"
                    ]
                )
            }
        }

        stage("Upload Artifact to Nexus") {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_NUMBER}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [
                        [
                            artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war'
                        ]
                    ]
                )

                splunkinsSend(
                    message: "Artifact uploaded to Nexus",
                    metadata: [
                        stage: "nexus_upload",
                        repository: RELEASE_REPO
                    ]
                )
            }
        }

        stage('Deploy to Dev') {
            steps {
                sh '''
                    echo "Deploying to DEV..."
                    # Deployment script goes here
                '''

                splunkinsSend(
                    message: "Deployment to DEV completed",
                    metadata: [
                        stage: "deploy",
                        environment: env.DEPLOY_ENV
                    ]
                )
            }
        }
    }

    post {
        always {
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\nMore info: ${env.BUILD_URL}"

            splunkinsSend(
                includeConsoleLog: true,
                metadata: [
                    build_status: currentBuild.currentResult,
                    app: env.APP_NAME,
                    environment: env.DEPLOY_ENV
                ]
            )
        }
    }
}