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