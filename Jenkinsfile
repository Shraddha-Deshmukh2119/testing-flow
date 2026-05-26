pipeline {
    agent any
    tools {
        maven 'maven'
        jdk   'JDK25' 
    }

    parameters {
        choice(name: 'REPO_MODE', choices: ['SEPARATE_REPOS', 'SINGLE_REPO'], description: 'Choose your repository structure')
        string(name: 'COMBINED_REPO_URL', defaultValue: '', description: 'URL for single repo')
        string(name: 'JAVA_REPO_URL', defaultValue: 'https://github.com/Shraddha-Deshmukh2119/java-repo.git', description: 'Java repo URL')
        string(name: 'CPP_REPO_URL', defaultValue: 'https://github.com/Shraddha-Deshmukh2119/Cpp-repo.git', description: 'C++ repo URL')
        string(name: 'SCRIPT_PATH', defaultValue: 'C:\\jenkins-scripts\\unify_reports.py', description: 'Path to Python script')
        string(name: 'SONAR_ORG_KEY', defaultValue: 'loosely-coupled-key', description: 'SonarCloud Org')
        string(name: 'SONAR_PROJECT_KEY', defaultValue: 'loosely-coupled-key_unified-system-analysis', description: 'SonarCloud Project')
    }

    environment {
        SONAR_TOKEN = credentials('SonarCloud-token')
        SONAR_PROJECT_KEY = "${params.SONAR_PROJECT_KEY}"
        SONAR_ORG_KEY = "${params.SONAR_ORG_KEY}"
    }

    stages {
        stage('CheckoutRepo') {
            steps {
                script {
                    if (params.REPO_MODE == 'SINGLE_REPO') {
                        checkout scm: [$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: params.COMBINED_REPO_URL]]]
                        env.JAVA_BASE = 'Java/JavaFullstackEcommerce'
                        env.CPP_BASE = 'Cpp'
                    } else {
                        dir('java-app') { checkout scm: [$class: 'GitSCM', branches: [[name: '**']], userRemoteConfigs: [[url: params.JAVA_REPO_URL]]] }
                        dir('cpp-app') { checkout scm: [$class: 'GitSCM', branches: [[name: '**']], userRemoteConfigs: [[url: params.CPP_REPO_URL]]] }
                        
                        env.JAVA_BASE = 'java-app/Java/JavaFullstackEcommerce'
                        env.CPP_BASE = 'cpp-app'
                    }
                    env.RUN_JAVA = 'true'
                    env.RUN_CPP = 'true'
                }
            }
        }

        stage('Build & Test') {
            parallel {
                stage('Java') {
                    steps {
                        script {
                            if (env.RUN_JAVA == 'true') {
                                dir(env.JAVA_BASE) {
                                    def javaToolHome = tool name: 'JDK25', type: 'jdk'
                                    def mavenToolHome = tool name: 'maven', type: 'maven'
                                    
                                    withEnv([
                                        "JAVA_HOME=${javaToolHome}",
                                        "M2_HOME=${mavenToolHome}",
                                        "PATH+MAVEN=${mavenToolHome}\\bin",
                                        "PATH+JAVA=${javaToolHome}\\bin"
                                    ]) {
                                        bat 'mvn clean verify -T 1C -Dmaven.test.failure.ignore=true'
                                    }
                                }
                            }
                        }
                    }
                }

                stage('C++') {
                    steps {
                        script {
                            if (env.RUN_CPP == 'true') {
                                dir(env.CPP_BASE) {
                                    withEnv([
                                        "PATH+MINGW=C:\\mingw64\\mingw64\\bin", 
                                        "PATH+CMAKE_WIN=C:\\Program Files\\CMake\\bin",
                                        "PATH+PYTHON=C:\\Python313\\;C:\\Python313\\Scripts\\",
                                        "PYTHONUNBUFFERED=1"
                                    ]) {
                                        bat """
                                        @echo off
                                        if not exist build mkdir build
                                        
                                        echo [INFO] Step 1: Initializing CMake with Coverage Instrumentation...
                                        cmake -B build -G "MinGW Makefiles" -DCMAKE_SH="MUTED" -DCMAKE_CXX_COMPILER=C:/mingw64/mingw64/bin/g++.exe -DCMAKE_C_COMPILER=C:/mingw64/mingw64/bin/gcc.exe -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTS=ON -DENABLE_COVERAGE=ON -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
                                        if errorlevel 1 exit /b %errorlevel%
                                        
                                        echo [INFO] Step 2: Running Compilations...
                                        cmake --build build --parallel %NUMBER_OF_PROCESSORS%
                                        if errorlevel 1 exit /b %errorlevel%

                                        echo [INFO] Step 3: Executing Test Binary Modules...
                                        ctest --test-dir build --verbose --output-on-failure -j %NUMBER_OF_PROCESSORS% > ctest_output.log 2>&1
                                        type ctest_output.log
                                        
                                        echo [INFO] Step 4: Generating Live Cobertura XML Coverage via gcovr...
                                        gcovr -r . --object-directory build --xml -o build/coverage.xml
                                        if errorlevel 1 (
                                            echo [WARN] Base gcovr build extraction missing root files. Forcing directory fallback filters...
                                            gcovr -r . --object-directory build --filter "Server/Headers" --filter "Client/Headers" --xml -o build/coverage.xml
                                        )
                                        
                                        echo [INFO] Step 5: Normalizing Workspace Validation Logs...
                                        if exist build\\coverage.xml (
                                            echo [SUCCESS] Real structural coverage.xml generated successfully.
                                        ) else (
                                            echo [ERROR] coverage.xml was not created. Verify system pathing configurations.
                                            exit /b 1
                                        )
                                        """
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Sonar Scan') {
            steps {
                script {
                    def scanner = tool name: 'sonar_scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                    
                    def srcList = []
                    if (env.RUN_JAVA == 'true') { srcList << "${env.WORKSPACE}/${env.JAVA_BASE}/src/main/java" }
                    if (env.RUN_CPP == 'true')  { srcList << "${env.WORKSPACE}/${env.CPP_BASE}" }
                    
                    def srcPaths = srcList.join(',')
                    def testPaths = (env.RUN_JAVA == 'true' && fileExists("${env.JAVA_BASE}/src/test/java")) ? "${env.WORKSPACE}/${env.JAVA_BASE}/src/test/java" : ""
                    def sonarTestsProperty = !testPaths.isEmpty() ? "-Dsonar.tests=\"${testPaths}\"" : ""

                    withSonarQubeEnv('SonarCloud-token') {
                        withEnv(["SONAR_USER_HOME=${WORKSPACE}\\.sonar"]) {
                            bat """
                            "${scanner}\\bin\\sonar-scanner" ^
                            -Dsonar.token=%SONAR_TOKEN% ^
                            -Dsonar.projectKey="${SONAR_PROJECT_KEY}" ^
                            -Dsonar.organization="${SONAR_ORG_KEY}" ^
                            -Dsonar.sources="${srcPaths}" ^
                            ${sonarTestsProperty} ^
                            -Dsonar.java.binaries="**/target/classes" ^
                            -Dsonar.coverage.jacoco.xmlReportPaths="**/target/site/jacoco/jacoco.xml" ^
                            -Dsonar.cfamily.compile-commands="${env.WORKSPACE}\\${env.CPP_BASE}\\build\\compile_commands.json" ^
                            -Dsonar.cfamily.gcov.reportsPath="${env.WORKSPACE}\\${env.CPP_BASE}\\build" ^
                            -Dsonar.scanner.skipJreProvisioning=true ^
                            -Dsonar.scm.disabled=true ^
                            -Dsonar.cache.enabled=true ^
                            -Dsonar.exclusions="**/node_modules/**,**/venv/**,**/target/**,**/build/**" ^
                            -Dsonar.c.file.suffixes=".c" ^
                            -Dsonar.cpp.file.suffixes=".cpp,.h,.hpp"
                            """
                        }
                    }
                }
            }
        }

        stage('Final Reporting') {
            steps {
                script {
                    def jacocoReports = findFiles(glob: '**/target/site/jacoco/jacoco.xml')
                    def gcovrReports = findFiles(glob: '**/build/coverage.xml')
                    
                    env.JAVA_XML_PATH = jacocoReports.length > 0 ? jacocoReports[0].path : ""
                    env.CPP_XML_PATH  = gcovrReports.length > 0 ? gcovrReports[0].path : ""
                    
                    withEnv([
                        "JAVA_XML_PATH=${env.JAVA_XML_PATH}",
                        "CPP_XML_PATH=${env.CPP_XML_PATH}"
                    ]) {
                        bat "python \"${params.SCRIPT_PATH}\""
                    }
                    
                    if (fileExists('unified_master_report.json')) {
                        archiveArtifacts 'unified_master_report.json'
                    }
                }
            }
        }

        stage('Upload To Coverage Dashboard') {
            steps {
                script {
                    bat """
                    echo Uploading unified report to dashboard...

                    curl -v ^
                      -X POST ^
                      -H "Content-Type: application/json" ^
                      --data-binary "@unified_master_report.json" ^
                      http://10.33.7.113:8000/api/builds/upload

                    echo Upload completed.
                    """
                }
            }
        }
    }
}
