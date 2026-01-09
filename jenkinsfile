pipeline{
    agent {
        node {
            label "windows"
        }
    }
    stages{
        stage('Get Code') {
            steps {
                echo 'Access repo by private key ssh'
                git branch : 'master',
                    credentialsId : 'github-ssh',
                    url: 'git@github.com:r4kogama/Caso-Practico-1-CP1.2.git'
                echo WORKSPACE
                bat 'dir'
                stash name:'code', includes:'**/*'
            }
        }
        stage('Static') {
            steps{
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    bat """
                        flake8 --format=pylint --exit-zero app >flake8.out
                    """
                    recordIssues qualityGates:  [
                        [integerThreshold: 8, threshold: 8.0, type: 'TOTAL'], 
                        [criticality: 'FAILURE', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']
                    ],
                    sourceCodeRetention: 'LAST_BUILD', tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], unhealthy: 10
                }
            }
        }
        stage('Test'){
            parallel{
                stage('Unit_test'){
                    steps{
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            deleteDir()
                            unstash name:'code'
                            bat """
                                set PYTHONPATH=${WORKSPACE}
                                python -m coverage run --branch --source=app --omit=app/__init__.py,app/api.py -m pytest test\\unit --junitxml=report\\unit\\result-unit.xml --html-output=report\\unit\\unit-test
                                python -m coverage xml -o report\\unit\\coverage\\coverage.xml
                                python -m coverage html -d report\\unit\\coverage
                                del report\\unit\\report.html
                                dir
                                """
                            junit testResults: '**/result-*.xml'
                        }
                        stash name:'report-unit', includes:'report/unit/**', useDefaultExcludes: false
                        stash name:'files-py', includes: 'app/calc.py, app/util.py', useDefaultExcludes: false
                    }
                }
                stage('Service'){
                    steps{
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            deleteDir()
                            bat """
                                set FLASK_APP=app\\api.py
                                start /B flask run
                                ping -n 5 127.0.0.1 > nul
                                start /B java -jar C:\\unir\\tareas\\wiremock\\wiremock-standalone-3.13.2.jar --port 9090 --root-dir test\\wiremock
                                ping -n 5 127.0.0.1 > nul
                                set PYTHONPATH=${WORKSPACE}
                                pytest test\\rest --junitxml=report\\rest\\result-rest.xml --html-output=report\\rest\\rest-test
                                del report\\rest\\report.html
                                for /f "tokens=2" %%a in ('tasklist /v ^| findstr /i "wiremock-standalone-3.13.2.jar"') do taskkill /F /PID %%a 
                                ping -n 2 127.0.0.1 > nul
                            """ 
                        }
                        stash name:'report-rest', includes:'report/rest/**', useDefaultExcludes: false
                    }
                }
            }
        }
        stage('Cobertura') {
            steps{
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    unstash name:'report-unit'
                    unstash name:'files-py'
                    recordCoverage qualityGates: [
                        [criticality: 'ERROR', integerThreshold: 84, metric: 'LINE',   threshold: 84.0],
                        [criticality: 'NOTE',  integerThreshold: 95, metric: 'LINE',   threshold: 95.0],
                        [criticality: 'ERROR', integerThreshold: 79, metric: 'BRANCH', threshold: 79.0],
                        [criticality: 'NOTE',  integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]
                    ],
                    sourceCodeRetention: 'LAST_BUILD', tools: [[parser: 'COBERTURA', pattern: '**/coverage.xml']]
                }
            }
        }
        stage('Security') {
            steps{
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    bat """
                        bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                        """
                    recordIssues qualityGates: [
                            [integerThreshold: 2, threshold: 2.0, type: 'TOTAL'],
                            [criticality: 'FAILURE', integerThreshold: 4, threshold: 4.0, type: 'TOTAL']
                        ], 
                    sourceCodeRetention: 'LAST_BUILD', tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')], unhealthy: 4
                }
            }
        }
        stage('Perfomance') {
            steps{
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    bat """
                        C:\\unir\\tareas\\apache-jmeter-5.6.3\\bin\\jmeter -n -t test\\jmeter\\flask-reto1.jmx -f -l flask.jtl
                        """
                    perfReport sourceDataFiles: 'flask.jtl'
                    cleanWs(notFailBuild: true)
                }
            }
        }
    }
}