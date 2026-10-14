pipeline {
    agent {
        node {
            label "roboshop"
        }
    }
    environment {
        version = ""
        app_name = "catalogue"
        region = "us-east-1"
        id = "220719767845"
    }
    options {
        // disableConcurrentBuilds()
        timeout(time: 7, unit: 'MINUTES')
    } 
    // parameters {
    //     string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    //     text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')
    //     booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Deploy the application')
    //     choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')
    //     password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    // }
    stages {
        stage("read the version in package.json") {
            steps {
                script {
                    def packagejson = readJSON file: 'package.json'

                    version = packagejson.version
                    echo "Version is ${version}"
                }
            }
        }

        stage("install dependencies") {
            steps {
                script{
                    sh """
                        npm install
                    """
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool name: 'sonar-8'
                    withSonarQubeEnv('sonarqube-server') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        // stage('Check Dependabot Alerts') {
        //     steps {
        //         withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
        //             sh '''
        //                 set -e

        //                 REPO="krishna-33s/catalogue"

        //                 curl -s -L \
        //                 -H "Accept: application/vnd.github+json" \
        //                 -H "Authorization: Bearer ${GH_TOKEN}" \
        //                 -H "X-GitHub-Api-Version: 2026-03-10" \
        //                 "https://api.github.com/repos/${REPO}/dependabot/alerts?state=open" \
        //                 -o alerts.json

        //                 echo "---- Open Dependabot Alerts ----"
        //                 jq -r '.[] | "\\(.number)\\t\\(.security_vulnerability.severity)\\t\\(.dependency.package.name)\\t\\(.security_advisory.ghsa_id)"' alerts.json

        //                 HIGH_CRITICAL_COUNT=$(jq '[.[] | select(.security_vulnerability.severity == "high" or .security_vulnerability.severity == "critical")] | length' alerts.json)

        //                 echo "High/Critical alert count: ${HIGH_CRITICAL_COUNT}"

        //                 if [ "$HIGH_CRITICAL_COUNT" -gt 0 ]; then
        //                     echo "❌ Found ${HIGH_CRITICAL_COUNT} High/Critical severity dependency alert(s). Failing build."
        //                     exit 1
        //                 else
        //                     echo "✅ No High/Critical dependency alerts found."
        //                 fi
        //             '''
        //         }
        //     }
        // }
           
        stage("build docker image") {
            steps {
                script{
                    withAWS(credentials: 'aws-creds', region: "${region}") {

                    sh """
                        aws ecr get-login-password --region ${region} | docker login --username AWS --password-stdin ${id}.dkr.ecr.${region}.amazonaws.com
                        docker build -t ${id}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${version} .
                        docker push ${id}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${version}

                    """
                    }
                }
            }
        }
        // stage('Trivy Scan') {
        //     steps {
        //         script {
        //             def dockerfileScan = sh(
        //                 script: """
        //                     trivy config --exit-code 1 \
        //                     --severity HIGH,CRITICAL \
        //                     --format table ./Dockerfile
        //                 """,
        //                 returnStatus: true
        //             )

        //             def imageScan = sh(
        //                 script: """
        //                     trivy image --scanners vuln \
        //                     --pkg-types os \
        //                     --exit-code 1 \
        //                     --severity HIGH,CRITICAL \
        //                     --format table ${id}.dkr.ecr.us-east-1.amazonaws.com/roboshop/catalogue:${version}
        //                 """,
        //                 returnStatus: true
        //             )

        //             if (dockerfileScan != 0 || imageScan != 0) {
        //                 error "Trivy found HIGH/CRITICAL issues in Dockerfile and/or OS packages. Failing pipeline."
        //             }
        //         }
        //     }
        // }

        stage('ECR Image push') {
            steps {
                script {
                    // in this block we get aws authentication
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${id}.dkr.ecr.us-east-1.amazonaws.com
                            docker push ${id}.dkr.ecr.us-east-1.amazonaws.com/roboshop/catalogue:${version}
                        """
                    }
                }
            }
        }
    }    
    post {
        always {
            echo "docker image is built"
        }
        success {
            echo "docker image built image successfully completed"
        }
        failure {
            echo "building image failed"
        }
    }
}