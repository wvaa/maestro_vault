// NOTA: Necesitas el plugin 'Pipeline Utility Steps' para 'readJSON'
// También debes tener el 'HashiCorp Vault Plugin' instalado y configurado en Jenkins

pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: Maestro-Container
    image: "eclipse-temurin:17-jdk"
    imagePullPolicy: "Always"
    resources:
      limits:
        memory: "2Gi"
        cpu: "1000m"
      requests:
        memory: "2Gi"
        cpu: "1000m"
    tty: true
    command:
    - sleep
    args:
    - infinity
'''
            defaultContainer 'Maestro-Container'
        }
    }
        
        environment {
            // Variables NO secretas
            REPO = 'https://github.com/wvaa/maestro_vault.git'
            BRANCH = 'main'
            BROWSERSTACK_LOCAL_BINARY = "./BrowserStackLocal"
            
            // Las variables BROWSERSTACK_USERNAME y BROWSERSTACK_ACCESS_KEY 
            // serán inyectadas por el bloque withVault.
        }
        
        parameters {
            string(name: 'APP_ID', defaultValue: 'bs://d059b204e6af41fd4a0dfe54ad29c9bb058176e8', description: 'ID de apk cargado en Browserstack')
            
        }
        
        stages {
            stage('Install tools') {
                steps {
                    sh """
                    apt-get update && apt-get install -y curl unzip git zip
                    """
                }
            }

            stage('Clone Test Suite Repo') {
                steps {
                    sh """
                    echo "=== Clonando repositorio de Test Suite ==="
                    git clone --branch ${BRANCH} ${REPO} suite-repo
                    ls -la suite-repo
                    """
                }
            }

            stage('Zip Test Suite') {
                steps {
                    sh """
                    echo "=== Comprimiendo carpeta TEST_SUITE ==="
                    cd suite-repo
                    zip -r TEST_SUITE.zip flows tests
                    ls -lh TEST_SUITE.zip
                    """
                }
            }
            
            
            stage('Upload Test Suite to BrowserStack') {
                steps {
                     withVault(configuration: 'BROWSERSTACK_VAULT_CONFIG', vaultSecrets: [
                            [secretPath: 'secret/data/ci/browserstack', engineVersion: 2, secretValues: [
                            [key: 'username', envVar: 'BROWSERSTACK_USERNAME'],
                            [key: 'access_key', envVar: 'BROWSERSTACK_ACCESS_KEY']
                        ]]
                ]) {
                    script {
                        echo "=== Subiendo Test Suite a BrowserStack ==="
                        // Aquí BROWSERSTACK_USERNAME y BROWSERSTACK_ACCESS_KEY son usados de forma segura
                        def uploadResponse = sh(
                            script: """curl -u "${BROWSERSTACK_USERNAME}:${BROWSERSTACK_ACCESS_KEY}" \\
                            -X POST "https://api-cloud.browserstack.com/app-automate/maestro/v2/test-suite" \\
                            -F "file=@suite-repo/TEST_SUITE.zip" \\
                            -F "custom_id=SampleTest" """,
                            returnStdout: true
                        ).trim()

                        def jsonUpload = readJSON text: uploadResponse
                        def testSuiteId = jsonUpload.test_suite_id
                        echo "✅ Test Suite ID obtenido: ${testSuiteId}"

                        // Guardamos en variable global para usar en siguiente stage
                        env.TEST_SUITE_ID = testSuiteId
                    }
                }
            }
            }

            stage('Download BrowserStackLocal binary') {
                steps {
                    sh """
                    echo "=== Verificando binario de BrowserStack Local ==="
                    if [ ! -f BrowserStackLocal ]; then
                        echo "Descargando BrowserStackLocal..."
                        curl -sS https://www.browserstack.com/browserstack-local/BrowserStackLocal-linux-x64.zip -o bs_local.zip
                        unzip -o bs_local.zip
                        chmod +x BrowserStackLocal
                        echo "Binario descargado correctamente."
                    else
                        echo "Binario ya existe, continuando..."
                    fi
                    """
                }
            }

            stage('Start BrowserStack Local') {
                steps {
                    
                     withVault(configuration: 'BROWSERSTACK_VAULT_CONFIG', vaultSecrets: [
                    [secretPath: 'secret/data/ci/browserstack', engineVersion: 2, secretValues: [
                     [key: 'username', envVar: 'BROWSERSTACK_USERNAME'],
                     [key: 'access_key', envVar: 'BROWSERSTACK_ACCESS_KEY']
                    ]]
             ]) {

                    sh """
                    echo "=== Iniciando BrowserStack Local ==="
                    // Uso seguro de BROWSERSTACK_ACCESS_KEY
                    ${BROWSERSTACK_LOCAL_BINARY} --key ${BROWSERSTACK_ACCESS_KEY} --daemon start
                    echo "Esperando 10 segundos para estabilizar..."
                    sleep 10
                    """
                }
                }

            }

            stage('Build Steps Linux') {
                steps {

                    withVault(configuration: 'BROWSERSTACK_VAULT_CONFIG', vaultSecrets: [
        [secretPath: 'secret/data/ci/browserstack', engineVersion: 2, secretValues: [
            [key: 'username', envVar: 'BROWSERSTACK_USERNAME'],
            [key: 'access_key', envVar: 'BROWSERSTACK_ACCESS_KEY']
        ]]
    ]) { 
                    script {
                        def appParam = params.APP_ID?.trim() ?: 'default'
                        def testTagParam = params.Test_tag?.trim() ?: 'regression-test'

                        echo "Usando Test Suite ID dinámico: ${env.TEST_SUITE_ID}"

                        // Uso seguro de BROWSERSTACK_USERNAME y BROWSERSTACK_ACCESS_KEY
                        def response = sh(
                            script: """curl -u "${BROWSERSTACK_USERNAME}:${BROWSERSTACK_ACCESS_KEY}" -X POST https://api-cloud.browserstack.com/app-automate/maestro/v2/android/build -d '{
                            "local": "true",
                            "devices": ["Samsung Galaxy S21-12.0"],
                            "app": "${appParam}",
                            "testSuite": "bs://${TEST_SUITE_ID}",
                            "project": "APP GUATEMALA",
                            "buildTag": "APPGT",
                            "execute": [
                            "flows/"
                            ],
                            "video": "true",
                            "networkLogs": "true",
                            "deviceLogs": "true"
                            }' -H 'Content-Type: application/json'
                            """,
                            returnStdout: true
                        ).trim()

                        def json = readJSON text: response
                        def buildId = json.build_id
                        echo "📦 Build ID obtenido: ${buildId}"

                        // Se repite la logica para la segunda llamada a la API...
                        def response_ach = sh(
                            script: """curl -u "${BROWSERSTACK_USERNAME}:${BROWSERSTACK_ACCESS_KEY}" -X POST https://api-cloud.browserstack.com/app-automate/maestro/v2/android/build -d '{
                            "local": "true",
                            "devices": ["Samsung Galaxy S23-13.0"],
                            "app": "${appParam}",
                            "testSuite": "bs://${TEST_SUITE_ID}",
                            "project": "APP GUATEMALA",
                            "buildTag": "APPGT-ACH",
                            "execute": [
                            "ACH_tests/"
                            ],
                            "video": "true",
                            "networkLogs": "true",
                            "deviceLogs": "true",
                            "tags": {
                            "includeTags": ["${testTagParam}"],
                            "excludeTags": ["fullregression-test"]
                            }
                            
                            }' -H 'Content-Type: application/json'
                            """,
                            returnStdout: true
                        ).trim()

                        def json_ach = readJSON text: response_ach
                        def buildId_ach = json_ach.build_id
                        echo "📦 Build ID para pruebas ACH: ${buildId_ach}"
                        

                        def status = "running"
                        while (status == "running") {
                            echo "⏳ Build aún en ejecución. Esperando 180 segundos..."
                            sleep 180
                            def response2 = sh(script: """ curl -u "${BROWSERSTACK_USERNAME}:${BROWSERSTACK_ACCESS_KEY}" \\
                                -X GET \"https://api-cloud.browserstack.com/app-automate/maestro/v2/builds/${buildId}\" """,
                            returnStdout: true
                            ).trim()

                            def json2 = readJSON text: response2
                            status = json2.status
                        }
                        
                        // Evaluar resultado final
                        if (status == "passed") {
                            echo "✅ Build finalizada con éxito en BrowserStack"
                        } else {
                            error("❌ Build fallida en BrowserStack con estado: ${status}")
                        }
                    }
                
                }
            }
        }
        }

        post {
            always {
                echo "=== Cerrando túnel BrowserStack Local ==="
                sh """
                ${BROWSERSTACK_LOCAL_BINARY} --daemon stop || true
                """
            }
            success {
                echo 'El pipeline se ejecutó exitosamente'
            }
            failure {
                echo 'El pipeline ha fallado. Revisar y corregir.'
            }
        }
        
}