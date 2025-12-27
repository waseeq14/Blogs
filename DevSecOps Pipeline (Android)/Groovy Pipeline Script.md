```groovy
pipeline {
    agent any
    
    environment {
        SEMGREP_APP_TOKEN = credentials('SEMGREP_APP_TOKEN')
        SEMGREP_PR_ID = "${env.CHANGE_ID}"
        SCANNER_HOME = tool 'sonar-scanner'
        DEFECTDOJO_API = credentials('DEFECTDOJO_API_KEY')
        DEFECTDOJO_URL = 'http://<url>'
        MobSF_API = credentials('MobSFAPI')
        MobSF_URL = 'http://<url>'
        
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/satishpatnayak/AndroGoat.git'
            }
        }

        stage('Static Code Analysis with Semgrep') {
            steps {
                sh '''
                REPORT_DIR="reports"
                mkdir -p "$REPORT_DIR"

                # Run Semgrep via Docker
                docker run --rm \
                    -e SEMGREP_APP_TOKEN="$SEMGREP_APP_TOKEN" \
                    -v "${WORKSPACE}:/src" \
                    semgrep/semgrep:latest semgrep ci \
                    --json \
                    --output /src/$REPORT_DIR/semgrep-report.json
                '''
            }
        }
        
        stage('GitLeaks Scan') {
            steps {
                sh '''
                REPORT_DIR="reports"
                mkdir -p "$REPORT_DIR"
        
                gitleaks detect --source="$WORKSPACE" --report-format=json --report-path="$REPORT_DIR/gitleaks-report.json" || true
                '''
            }
        }
        
        stage('SonarQube Analysis'){
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=AndroGoat-Proj \
                    -Dsonar.projectKey=AndroGoat-Proj \
                    -Dsonar.log.report=true \
                    -Dsonar.report.export.path=reports/sonarqube-report.json
                    '''
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-qube'
                }
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh '''
                REPORT_DIR="reports"
                mkdir -p "$REPORT_DIR"
        
                # Run Trivy and save JSON report
                trivy fs --format json -o "$REPORT_DIR/trivy-report.json" .
                '''
            }
        }
        
        stage('Upload Semgrep Report to DefectDojo') {
            steps {
                defectDojoPublisher(
                    artifact: 'reports/semgrep-report.json',
                    scanType: 'Semgrep JSON Report',
                    defectDojoUrl: "${DEFECTDOJO_URL}",
                    defectDojoCredentialsId: 'DEFECTDOJO_API_KEY',
                    productId: '1',
                    engagementId: '1',
                    engagementName: 'Jenkins-CI',
                    autoCreateProducts: false,
                    autoCreateEngagements: false
                )
            }
        }
        
        stage('Upload Gitleaks Report to DefectDojo') {
            steps {
                defectDojoPublisher(
                    artifact: 'reports/gitleaks-report.json',
                    autoCreateEngagements: false,
                    autoCreateProducts: false,
                    branchTag: '',
                    commitHash: '',
                    defectDojoCredentialsId: 'DEFECTDOJO_API_KEY',
                    defectDojoUrl: "${DEFECTDOJO_URL}",
                    engagementId: '1',
                    engagementName: 'Jenkins-CI',
                    productId: '1',
                    scanType: 'Gitleaks Scan',
                    sourceCodeUrl: ''
                )
            }
        }

        stage('Upload Trivy Report to DefectDojo') {
            steps {
                defectDojoPublisher(
                    artifact: 'reports/trivy-report.json',
                    autoCreateEngagements: false,
                    autoCreateProducts: false,
                    branchTag: '',
                    commitHash: '',
                    defectDojoCredentialsId: 'DEFECTDOJO_API_KEY',
                    defectDojoUrl: "${DEFECTDOJO_URL}",
                    engagementId: '1',
                    engagementName: 'Jenkins-CI',
                    productId: '1',
                    scanType: 'Trivy Scan',
                    sourceCodeUrl: ''
                )
            }
        }

        stage('Build APK') {
            steps {
                sh '''
                chmod +x gradlew
                ./gradlew assembleDebug
                '''
            }
        }
        

        stage("MobSF Scan") {
            steps {
                script {
                    echo "Uploading APK to MobSF..."
                    def apkPath = "${env.WORKSPACE}/app/build/outputs/apk/debug/app-debug.apk"
        
                    // Upload APK
                    def uploadCmd = """curl -s -F 'file=@${apkPath}' \\
                        ${MobSF_URL}/api/v1/upload \\
                        -H "Authorization: ${MobSF_API}" -o response.json"""
                    sh uploadCmd
                    echo "APK Uploaded. Parsing hash..."
        
                    // Get hash from response
                    def hash = sh(script: "jq -r '.hash' response.json", returnStdout: true).trim()
                    echo "File hash: ${hash}"
        
                    echo "Starting MobSF Scan..."
                    def scanCmd = """curl -s -X POST \\
                        --url ${MobSF_URL}/api/v1/scan \\
                        -H "Authorization: ${MobSF_API}" \\
                        --data "hash=${hash}" -o scan_response.json"""
                    sh scanCmd
                    echo "Scan Started."
        
                    echo "Fetching JSON Report..."
                    def reportCmd = """curl -s -X POST \\
                        --url ${MobSF_URL}/api/v1/report_json \\
                        -H "Authorization: ${MobSF_API}" \\
                        --data "hash=${hash}" -o mobsf_report.json"""
                    sh reportCmd
                    echo "JSON report saved as mobsf_report.json"
                }
            }
        }

        stage('Upload MobSF Report to DefectDojo') {
            steps {
                defectDojoPublisher(
                    artifact: "${env.WORKSPACE}/mobsf_report.json",
                    autoCreateEngagements: false,
                    autoCreateProducts: false,
                    branchTag: '',
                    commitHash: '',
                    defectDojoCredentialsId: 'DEFECTDOJO_API_KEY',
                    defectDojoUrl: "${DEFECTDOJO_URL}",
                    engagementId: '1',
                    engagementName: 'Jenkins-CI',
                    productId: '1',
                    scanType: 'MobSF Scan',
                    sourceCodeUrl: ''
                )
            }
        }

    }
}

```