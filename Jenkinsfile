pipeline {
    agent any
    
    environment {
        GRAFANA_URL = credentials('grafana-url')  // Optional: if stored as secret
        GRAFANA_TOKEN = credentials('grafana-api-token')
    }
    
    parameters {
        choice(
            name: 'ACTION',
            choices: ['UPDATE', 'VALIDATE'],
            description: 'Choose action: UPDATE to sync, VALIDATE to check only'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                git(
                    url: 'https://github.com/your-username/your-repo.git',
                    credentialsId: 'github-credentials',
                    branch: 'main'
                )
            }
        }
        
        stage('Validate JSON') {
            steps {
                script {
                    def jsonFiles = findFiles(glob: '**/*.json')
                    if (jsonFiles.isEmpty()) {
                        error('No JSON files found!')
                    }
                    
                    jsonFiles.each { file ->
                        echo "Validating: ${file.path}"
                        // Basic JSON validation
                        def jsonContent = readFile file: file.path
                        try {
                            new groovy.json.JsonSlurper().parseText(jsonContent)
                            echo "✓ ${file.name} is valid JSON"
                        } catch (Exception e) {
                            error("✗ ${file.name} contains invalid JSON: ${e.message}")
                        }
                    }
                }
            }
        }
        
        stage('Sync to Grafana') {
            when {
                expression { params.ACTION == 'UPDATE' }
            }
            steps {
                script {
                    def jsonFiles = findFiles(glob: '**/*.json')
                    
                    jsonFiles.each { file ->
                        echo "Processing: ${file.path}"
                        
                        // Read JSON content
                        def jsonContent = readFile file: file.path
                        def json = new groovy.json.JsonSlurper().parseText(jsonContent)
                        
                        // Extract dashboard UID or title
                        def dashboardUid = json.uid ?: json.dashboard?.uid
                        def dashboardTitle = json.title ?: json.dashboard?.title
                        
                        if (!dashboardUid && dashboardTitle) {
                            // Create UID from title if not present
                            dashboardUid = dashboardTitle.toLowerCase().replaceAll('\\s+', '-').replaceAll('[^a-z0-9-]', '')
                        }
                        
                        // Prepare API payload
                        def apiPayload = [
                            dashboard: json.dashboard ?: json,
                            overwrite: true,
                            message: "Updated via Jenkins ${new Date()}"
                        ]
                        
                        // Call Grafana API
                        def response = httpRequest(
                            acceptType: 'APPLICATION_JSON',
                            contentType: 'APPLICATION_JSON',
                            httpMode: 'POST',
                            customHeaders: [[
                                name: 'Authorization',
                                value: "Bearer ${GRAFANA_TOKEN}"
                            ]],
                            requestBody: groovy.json.JsonOutput.toJson(apiPayload),
                            url: "${env.GRAFANA_URL ?: 'http://localhost:3000'}/api/dashboards/db",
                            validResponseCodes: '200,400,404'
                        )
                        
                        if (response.status == 200) {
                            echo "✓ Successfully updated: ${dashboardTitle}"
                        } else {
                            echo "⚠ Response: ${response.content}"
                            // Try creating new dashboard if update fails
                            if (response.status == 404) {
                                echo "Dashboard not found, attempting to create new..."
                            }
                        }
                    }
                }
            }
        }
        
        stage('Notification') {
            steps {
                script {
                    echo "Dashboard sync completed!"
                    // Add email/slack notification here if needed
                    // emailext subject: "Grafana Dashboard Sync Completed",
                    //     body: "The dashboard sync job has completed.",
                    //     to: 'team@example.com'
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
