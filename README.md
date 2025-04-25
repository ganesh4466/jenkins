pipeline {
    agent any
    stages {
        stage('Download File') {
            steps {
                script {
                    def fileUrl = "https://example.com/path/to/file.zip"
                    def saveDir = "/var/lib/jenkins/shared-files"
                    
                    // Ensure directory exists
                    sh "mkdir -p ${saveDir}"

                    // Run Groovy script
                    sh """
                    groovy -e \"
                    import java.net.HttpURLConnection
                    import java.net.URL
                    import java.io.File

                    def downloadFile(String fileUrl, String saveDir) {
                        URL url = new URL(fileUrl)
                        HttpURLConnection connection = (HttpURLConnection) url.openConnection()
                        connection.setRequestMethod('GET')

                        if (connection.responseCode == HttpURLConnection.HTTP_OK) {
                            File saveDirectory = new File(saveDir)
                            if (!saveDirectory.exists()) {
                                saveDirectory.mkdirs()
                            }

                            String fileName = url.getPath().tokenize('/').last()
                            File outputFile = new File(saveDirectory, fileName)

                            outputFile.withOutputStream { fos ->
                                connection.inputStream.withStream { input ->
                                    input.transferTo(fos)
                                }
                            }

                            println \\"File downloaded to: ${outputFile.absolutePath}\\"
                        } else {
                            throw new RuntimeException(\\"Failed to download file. HTTP response code: ${connection.responseCode}\\")
                        }
                    }

                    downloadFile('${fileUrl}', '${saveDir}')
                    \"
                    """
                }
            }
        }
    }
}


pipeline {
    agent any
    stages {
        stage('List Credentials') {
            steps {
                script {
                    def allCredentials = com.cloudbees.plugins.credentials.CredentialsProvider
                        .lookupCredentials(
                            com.cloudbees.plugins.credentials.Credentials.class,
                            Jenkins.instance,
                            null,
                            com.cloudbees.plugins.credentials.domains.Domain.global()
                        )
                    
                    echo "=== Global Credentials ==="
                    allCredentials.each { cred ->
                        echo "ID: ${cred.id} | Type: ${cred.getClass().simpleName} | Description: ${cred.description ?: 'None'}"
                    }
                    echo "Total: ${allCredentials.size()}"
                }
            }
        }
    }
}

As per the simplification utility method, while executing a command, we are using the script:
cmd > commandOutput.txt 2> errorOutput.txt
This saves standard output (such as 'info' or 'success' logs) to commandOutput.txt, and saves errors, warnings, and anonymous logs to errorOutput.txt.
Because of this, npm feeds (which are categorized as anonymous) were missing from commandOutput.txt.
I made changes to display the npm logs directly in the console.
