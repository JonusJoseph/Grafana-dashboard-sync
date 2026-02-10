pipeline {
    agent any
    
    environment {
        GRAFANA_URL = 'http://192.168.226.139:3000'  // Local Grafana Docker
        GRAFANA_TOKEN = credentials('grafana-api-token')
    }
    
    parameters {
        choice(
            name: 'ACTION',
            choices: ['SYNC_ALL', 'SYNC_CHANGED'],
            description: 'Sync all dashboards or only changed files'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    extensions: [[
                        $class: 'PathRestriction',
                        includedRegions: '.*\\.json$'
                    ]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/YOUR_USER/YOUR_REPO.git',
                        credentialsId: 'github-credentials'
                    ]]
                ])
            }
        }
        
        stage('Validate JSON') {
            steps {
                script {
                    def jsonFiles = findFiles(glob: '**/*.json')
                    if (jsonFiles.isEmpty()) {
                        echo 'No JSON files found!'
                        currentBuild.result = 'FAILURE'
                        error('No dashboard JSON files found')
                    }
                    
                    jsonFiles.each { file ->
                        echo "Validating: ${file.path}"
                        try {
                            readJSON file: file.path
                            echo "✅ ${file.name} is valid JSON"
                        } catch (Exception e) {
                            echo "❌ ${file.name} contains invalid JSON: ${e.message}"
                            currentBuild.result = 'FAILURE'
                            error('Invalid JSON found')
                        }
                    }
                }
            }
        }
        
        stage('Sync to Grafana') {
            steps {
                script {
                    def jsonFiles = findFiles(glob: '**/*.json')
                    int successCount = 0
                    int failureCount = 0
                    
                    jsonFiles.each { file ->
                        echo "Processing: ${file.name}"
                        
                        try {
                            // Read and prepare dashboard
                            def dashboardContent = readFile file: file.path
                            def json = readJSON text: dashboardContent
                            
                            // Ensure proper structure
                            def payload = [:]
                            if (json.dashboard) {
                                payload = json
                            } else {
                                payload = [
                                    dashboard: json,
                                    overwrite: true
                                ]
                            }
                            
                            // Add metadata if not present
                            if (!payload.overwrite) payload.overwrite = true
                            if (!payload.message) payload.message = "Updated via Jenkins ${new Date()}"
                            
                            // Send to Grafana
                            def response = httpRequest(
                                acceptType: 'APPLICATION_JSON',
                                contentType: 'APPLICATION_JSON',
                                httpMode: 'POST',
                                customHeaders: [[
                                    name: 'Authorization',
                                    value: "Bearer ${env.GRAFANA_TOKEN}"
                                ]],
                                requestBody: writeJSON returnText: true, json: payload,
                                url: "${env.GRAFANA_URL}/api/dashboards/db",
                                validResponseCodes: '200,400,404'
                            )
                            
                            if (response.status == 200) {
                                echo "✅ Successfully synced: ${file.name}"
                                successCount++
                            } else {
                                echo "⚠️ Failed to sync ${file.name}: ${response.content}"
                                failureCount++
                            }
                            
                        } catch (Exception e) {
                            echo "❌ Error syncing ${file.name}: ${e.message}"
                            failureCount++
                        }
                    }
                    
                    echo "\n📊 Sync Summary:"
                    echo "✅ Successfully synced: ${successCount}"
                    echo "❌ Failed: ${failureCount}"
                    
                    if (failureCount > 0) {
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '🎉 Dashboard sync completed successfully!'
        }
        failure {
            echo '❌ Dashboard sync failed!'
        }
        unstable {
            echo '⚠️ Some dashboards failed to sync'
        }
    }
}
