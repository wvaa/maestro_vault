pipeline {
    
    // 1. AGENT (Directiva principal)
    agent {
        label 'maestro-pod' 
    }
    
    // 2. ENVIRONMENT (Directiva principal)
    environment {
        REPO = 'https://github.com/wvaa/maestro_vault.git'
        BRANCH = 'main'
        BROWSERSTACK_LOCAL_BINARY = "./BrowserStackLocal"
    }

    // 3. PARAMETERS (Directiva principal)
    parameters {
        string(name: 'APP_ID', defaultValue: 'bs://d059b204e6af41fd4a0dfe54ad29c9bb058176e8', description: 'ID de apk cargado en Browserstack')
        
    }

    
    // Para simplificar, lo ponemos en la primera etapa, envolviendo todos los pasos.
    stages {
        stage('Full Pipeline Execution with Vault Secrets') {
            steps {
                // withVault es un STEP, por lo tanto, debe ir dentro de steps {}
                withVault(vaultSecrets: [ // Ya quitamos 'credentialsId'/'configuration'
                    [secretPath: 'secret/data/ci/browserstack', engineVersion: 2, secretValues: [
                        [key: 'username', envVar: 'BROWSERSTACK_USERNAME'],
                        [key: 'access_key', envVar: 'BROWSERSTACK_ACCESS_KEY']
                    ]]
                ]) {
                    // AHORA, todos sus pasos originales van aquí, dentro del scope de withVault

                    // === Install tools ===
                    sh "apt-get update && apt-get install -y curl unzip git zip"

                    // === Clone Test Suite Repo ===
                    sh """
                    echo "=== Clonando repositorio de Test Suite ==="
                    git clone --branch ${BRANCH} ${REPO} suite-repo
                    ls -la suite-repo
                    """
                    
                    // === Zip Test Suite ===
                    sh """
                    echo "=== Comprimiendo carpeta TEST_SUITE ==="
                    cd suite-repo
                    zip -r TEST_SUITE.zip flows
                    ls -lh TEST_SUITE.zip
                    """
                    
                    // === Upload Test Suite to BrowserStack ===
                    script {
                        echo "=== Subiendo Test Suite a BrowserStack ==="
                        // Aquí BROWSERSTACK_USERNAME y BROWSERSTACK_ACCESS_KEY son visibles.
                        def uploadResponse = sh(
                            script: """curl -u "${BROWSERSTACK_USERNAME}:${BROWSERSTACK_ACCESS_KEY}" \\
                            -X POST "https://api-cloud.browserstack.com/app-automate/maestro/v2/test-suite" \\
                            -F "file=@suite-repo/TEST_SUITE.zip" \\
                            -F "custom_id=SampleTest" """,
                            returnStdout: true
                        ).trim()

                        def jsonUpload = readJSON text: uploadResponse
                        env.TEST_SUITE_ID = jsonUpload.test_suite_id
                        echo "✅ Test Suite ID obtenido: ${env.TEST_SUITE_ID}"
                    }
                    
                    // === Download BrowserStackLocal binary ===
                    sh """
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

                    // === Start BrowserStack Local ===
                    sh """
                    echo "=== Iniciando BrowserStack Local ==="
                    ${BROWSERSTACK_LOCAL_BINARY} --key ${BROWSERSTACK_ACCESS_KEY} --daemon start
                    echo "Esperando 10 segundos para estabilizar..."
                    sleep 10
                    """
                    
                    // === Build Steps Linux (Ejecución de las pruebas) ===
                    script {
                        def appParam = params.APP_ID?.trim() ?: 'default'
                        echo "Usando Test Suite ID dinámico: ${env.TEST_SUITE_ID}"

                        // Aquí se usan BROWSERSTACK_USERNAME y BROWSERSTACK_ACCESS_KEY de forma segura
                        
                        // ... (Su lógica de curl con las variables de Vault) ...
                        
                        // Su lógica de ejecución de pruebas (curl 1)
                        def response = sh(
                            script: """curl -u "${BROWSERSTACK_USERNAME}:${BROWSERSTACK_ACCESS_KEY}" -X POST https://api-cloud.browserstack.com/app-automate/maestro/v2/android/build -d '{
                            "local": "true",
                            "devices": ["Samsung Galaxy S21-12.0"],
                            "app": "${appParam}",
                            "testSuite": "bs://${TEST_SUITE_ID}",
                            "project": "DevOsp",
                            "buildTag": "Vault",
                            "execute": ["flows/"],
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
                        
                       

                        // Su lógica de monitoreo de estado
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
                    } // Cierre del script
                } // Cierre del withVault
            } // Cierre del steps
        } // Cierre del stage
    } // Cierre del stages

    // 5. POST (Directiva principal)
    post {
        always {
            echo "=== Cerrando túnel BrowserStack Local ==="
            sh """
            ${BROWSERSTACK_LOCAL_BINARY} --daemon stop || true
            """
        }
        success { echo 'El pipeline se ejecutó exitosamente' }
        failure { echo 'El pipeline ha fallado. Revisar y corregir.' }
    }
}