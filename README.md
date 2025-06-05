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


=============================================================


MSBuild and Node Integration in Jenkins Builds
Combination 1: Triggering Node Project Build from MSBuild Stage

In this configuration, the Node project build is triggered as part of the MSBuild pipeline in Jenkins. This behavior is controlled using the runnodebeforeMsbuild flag.

If runnodebeforeMsbuild is set to true:
The Node project (specified using the nodeproject parameter) is built before the MSBuild execution. The resulting Node packages are stored in the Binaries folder. Then, the MSBuild process runs and places its output in the same Binaries location. The packaging stage uses this folder to collect all the build artifacts and publish them to the Artifact repository.

If runnodebeforeMsbuild is set to false:
The MSBuild project runs first, followed by the Node project build. The rest of the process—placing outputs in the Binaries folder and packaging—remains the same.

This setup replicates the logic of the legacy framework in a simplified and more maintainable way. Importantly, the number of build stages in Jenkins remains consistent. For example, if the original MSBuild pipeline had 10 stages, this combination will also result in 10 stages after integration.

We enable this behavior by passing the appropriate Node parameters through the Jenkins MSBuild job parameters.


--

Combination 2: Sequential MSBuild and Node Builds Using Separate Pipeline Plugins
In this setup, MSBuild and Node builds are triggered as separate stages using different Jenkins pipeline plugins, defined sequentially within the same Jenkinsfile. This approach is not present in either the legacy framework or the simplified integration described in Combination 1.

Example:
If this combination is executed, the resulting pipeline will include:

2 Node stages: build and unit test

10 MSBuild stages

Drawback:
If the MSBuild plugin is defined before the Node plugin in the Jenkinsfile, the pipeline will execute all MSBuild stages—from SCM checkout through to deployment—before it begins the Node build and unit test stages.

This sequence introduces a logical issue: the Node build and unit test stages are expected to run before MSBuild's package, publish, and deploy stages. Running them afterward can break the expected build flow and artifact integrity.

To avoid this, the Node stages should be placed earlier in the Jenkinsfile or integrated into the MSBuild stages
--
To avoid this issue, the Node pipeline plugin should be placed earlier in the Jenkinsfile so that the Node-related stages are executed before the critical MSBuild packaging and deployment stages.

There are two possible sequencing approaches:

Node (2) + MSBuild (10) – ✅ Recommended Approach

Node build and unit test stages run first, followed by all MSBuild stages.

Ensures that any dependencies or outputs from the Node project are available before MSBuild's package, publish, and deploy steps.

MSBuild (10) + Node (2) – ❌ Not Recommended

MSBuild completes all its stages, including deployment, before Node stages begin.

This breaks the expected build flow, as Node outputs are not available when MSBuild needs them.

Adopting the first approach ensures a logical and reliable build process aligned with both dependency order and deployment requirements.



Subject: Request for PR Approval – Standalone Pipeline Changes

Hi [Leads' Names or Team],

I’ve raised a PR that includes standalone pipeline changes. As part of this update, the combination changes (related to MSBuild and Node integration) have been removed for now. These will be addressed in the next release as part of the pipeline chaining implementation.

Kindly review and approve the PR at your earliest convenience.

Please let me know if any further clarification is needed.

Best regards,
