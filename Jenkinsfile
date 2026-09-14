pipeline {
    agent any
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Select target Layer7 environment'
        )
        string(
            name: 'RELEASE',
            defaultValue: 'R1',
            description: 'Release to run: R1, R2, R3, R4, or ALL. Case-insensitive.'
        )
        string(
            name: 'APP_FILTER',
            defaultValue: '',
            description: 'Optional: comma/space-separated app names to limit within the release (e.g. "findFid.xml,generateFid.xml"). Blank = all apps in the release.'
        )
    }
    environment {
        GMU_HOME = 'C:\\gmu'
        JAVA_HOME = 'C:\\Program Files\\Java\\jdk-17.0.18'
        PATH = "${env.JAVA_HOME}\\bin;${env.PATH}"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Load Environment & Manifest Config') {
            steps {
                script {
                    // Load basic Environment configurations
                    def envConfig = readYaml file: "config/${params.ENVIRONMENT}.yaml"
                    env.GATEWAY_HOST = envConfig.gateway.host
                    env.GATEWAY_PORT = envConfig.gateway.port.toString()
                    env.GATEWAY_PROTOCOL = envConfig.gateway.protocol
                    echo "Target Environment : ${params.ENVIRONMENT}"
                    echo "Gateway Host       : ${env.GATEWAY_HOST}"
                    echo "Gateway Port       : ${env.GATEWAY_PORT}"
 
                    // ── Determine releases ────────────────────────────────────────
                    def releaseInput = params.RELEASE?.trim()?.toUpperCase() ?: 'R1'
                    def releasesToRun = []

                    if (releaseInput == 'ALL') {
                        releasesToRun = ['R1', 'R2', 'R3', 'R4'].findAll { fileExists("releases/${it}/manifest.yaml") }
                    } else {
                        releasesToRun = releaseInput.split('[,\\s]+')
                                            .collect { it.trim() }
                                            .findAll { it && fileExists("releases/${it}/manifest.yaml") }
                    }
 
                    // ── Optional app-level filter ──────────────────────────────
                    def appFilter = params.APP_FILTER?.trim()
                        ? params.APP_FILTER.split('[,\\s]+').collect { it.trim() }.findAll { it }
                        : []
 
                    // ── Map tracking database generation ────────────────────────
                    // Tracks releases to their service objects list
                    logReleaseApiMap = [:] 
                    releasesToRun.each { release ->
                        def manifestPath = "releases/${release}/manifest.yaml"
                        if (!fileExists(manifestPath)) {
                            error "Release ${release}: manifest not found at ${manifestPath}"
                        }
                        def manifest = readYaml file: manifestPath
                        
                        // Parse complex service maps from manifest layout
                        def rawServices = manifest.services ?: []
                        def processedServices = []

                        rawServices.each { svc ->
                            def svcName = svc.name?.toString()
                            def svcArtifact = svc.artifact?.toString()
                            def svcTargetFolder = svc.target_folder?.toString()

                            if (svcName && svcArtifact && svcTargetFolder) {
                                processedServices.add([
                                    name: svcName,
                                    artifact: svcArtifact,
                                    targetFolder: svcTargetFolder
                                ])
                            }
                        }
 
                        // If user defined an APP_FILTER, restrict the execution pool based on the name key
                        if (appFilter) {
                            def filterLower = appFilter.collect { it.toLowerCase() }
                            processedServices = processedServices.findAll { filterLower.contains(it.name.toLowerCase()) }
                            echo "Release ${release}: filtered to [${processedServices.collect { it.name }.join(', ')}]"
                        }

                        if (processedServices.isEmpty()) {
                            error "No valid bundles found to process for Release ${release} with current APP_FILTER."
                        }
 
                        logReleaseApiMap[release] = processedServices
                        def displayedNames = processedServices.collect { it.name }
                        echo "Final apps to process for Release ${release}: [${displayedNames.join(', ')}]"
                    }
                }
            }
        }
        stage('Validate Bundles') {
            steps {
                script {
                    logReleaseApiMap.each { release, services ->
                        services.each { svc ->
                            if (!fileExists(svc.artifact)) {
                                error "Bundle file not found in workspace: ${svc.artifact}"
                            }
                            echo "Validated physical bundle existence: ${svc.artifact}"
                        }
                    }
                }
            }
        }
        stage('Validate GMU') {
            steps {
                bat '''
                    @echo off
                    echo Checking GMU installation...
                    if not exist "%GMU_HOME%\\GatewayMigrationUtility.bat" (
                        echo GMU not found at %GMU_HOME%
                        exit /b 1
                    )
                    echo GMU installation found.
                '''
            }
        }
 
        stage('Test Layer7 Connectivity') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'layer7-gateway-credentials',
                        usernameVariable: 'GATEWAY_USERNAME',
                        passwordVariable: 'GATEWAY_PASSWORD'
                    )
                ]) {
                    script {
                        logReleaseApiMap.each { release, services ->
                            services.each { svc ->
                                echo "========================================"
                                echo "Testing connectivity using bundle: ${svc.artifact}"
                                echo "Targeting Gateway Folder: ${svc.targetFolder}"
                                echo "========================================"
                                bat """
                                    set "PATH=%JAVA_HOME%\\bin;%PATH%"
                                    "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                                        -h "%GATEWAY_HOST%" ^
                                        -p "%GATEWAY_PORT%" ^
                                        -u "%GATEWAY_USERNAME%" ^
                                        --plaintextPassword "%GATEWAY_PASSWORD%" ^
                                        --bundle "${svc.artifact}" ^
                                        --plaintextEncryptionPassphrase "%GATEWAY_PASSWORD%" ^
                                        --destFolder "${svc.targetFolder}" ^
                                        --results "results-${svc.name}" ^
                                        --trustCertificate ^
                                        --trustHostname ^
                                        --test
                                """
                            }
                        }
                    }
                }
            }
        }
        stage('Deploy to Layer7') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'layer7-gateway-credentials',
                        usernameVariable: 'GATEWAY_USERNAME',
                        passwordVariable: 'GATEWAY_PASSWORD'
                    )
                ]) {
                    script {
                        logReleaseApiMap.each { release, services ->
                            services.each { svc ->
                                echo "Deploying bundle: ${svc.artifact} to folder ${svc.targetFolder} on Layer7 Gateway..."
                                int exitCode = bat(
                                    returnStatus: true,
                                    script: """
                                        set "PATH=%JAVA_HOME%\\bin;%PATH%"
                                        "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                                            -h "%GATEWAY_HOST%" ^
                                            -p "%GATEWAY_PORT%" ^
                                            -u "%GATEWAY_USERNAME%" ^
                                            --plaintextPassword "%GATEWAY_PASSWORD%" ^
                                            --bundle "${svc.artifact}" ^
                                            --plaintextEncryptionPassphrase "%GATEWAY_PASSWORD%" ^
                                            --destFolder "${svc.targetFolder}" ^
                                            --results "results-${svc.name}" ^
                                            --trustCertificate ^
                                            --trustHostname
                                    """
                                )
                                if (exitCode != 0) {
                                    error "Deployment failed for bundle ${svc.name} with error code ${exitCode}."
									}
								echo "Successfully deployed: ${svc.name}"
                            }
                        }
                    }
                }
            }
        }
		stage('Deployment Verification') {
            steps {
                echo "All specified GMU deployments completed successfully."
            }
        }
    }
	post {
        success {
            script {
                echo """==========================================Layer7 Deployment SUCCESSEnvironment : ${params.ENVIRONMENT}Processed Deployments:"""logReleaseApiMap.each {
                    release, services ->  def displayedNames = services.collect {
                        it.name
                    }echo "Release ${release}: ${displayedNames.join(', ')}"
                }echo "=========================================="
            }
        }failure {
            echo """==========================================Layer7 Deployment FAILEDEnvironment : ${params.ENVIRONMENT}=========================================="""
        }always {
            archiveArtifacts artifacts: 'results-*.xml', allowEmptyArchive: truecleanWs()
        }
    }
}
