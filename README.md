package com.gseven

import net.sf.json.groovy.JsonSlurper

public class SamOne {
    static void main(String[] args) {
        def configText= "pipelineOne{\n" +
                "   app = \"name\"\n" +
                "   dev = true\n" +
                "   lst = [\"gty\"]\n" +
                "   map = [sd:\"sdf\"]\n" +
                "   //apa = \"qa\"\n" +
                "   \n" +
                "   /* \n" +
                "    app = \"test\"\n" +
                "*/\n" +
                "}\n" +
                "\n" +
                "pipelineTwo{\n" +
                "   app = \"name\"\n" +
                "   dev = true\n" +
                "   lst = [\"gty\"]\n" +
                "   map = [sd:\"sdf\"]\n" +
                "   //apa = \"qa\"\n" +
                "   \n" +
                "   /* \n" +
                "    app = \"test\"\n" +
                "*/\n" +
                "}\n" +
                "\n"
        // Remove all comments (single-line and multi-line)
        def cleaned = configText.replaceAll(/\/\/.*?(\n|$)/, '\n')  // Single-line
                .replaceAll(/\/\*.*?\*\//, '')       // Multi-line
                .trim()

        // Parse into map of pipeline configurations
        def result = [:]
        def currentPipeline = null
        def pipelineContent = [:]

        cleaned.eachLine { line ->
            line = line.trim()
            if (line.isEmpty()) return

            // Check for pipeline declaration (e.g., "pipelineOne{")
            def pipelineMatch = line =~ /^(\w+)\s*\{$/
            if (pipelineMatch) {
                currentPipeline = pipelineMatch[0][1]
                pipelineContent = [:]
                return
            }

            // Check for closing brace
            if (line == '}' && currentPipeline) {
                result[currentPipeline] = pipelineContent
                currentPipeline = null
                return
            }

            // Parse key-value pairs if inside a pipeline block
            if (currentPipeline) {
                // Handle different value types
                def valueMatch = line =~ /^(\w+)\s*=\s*(.+)$/
                if (valueMatch) {
                    def key = valueMatch[0][1]
                    def valueStr = valueMatch[0][2].trim()

                    // Evaluate the value string safely
                    pipelineContent[key] = evaluateValue(valueStr)

                }
            }

            System.out.println(pipelineContent)
        }
    }

    // Helper method to safely evaluate different value types
    def evaluateValue(String valueStr) {
        try {
            // Handle lists
            if (valueStr.startsWith('[') && valueStr.endsWith(']')) {
                return new JsonSlurper().parseText(valueStr)
            }
            // Handle maps
            else if (valueStr.startsWith('[') && valueStr.contains(':') && valueStr.endsWith(']')) {
                return new JsonSlurper().parseText(valueStr)
            }
            // Handle booleans
            else if (valueStr == 'true') return true
            else if (valueStr == 'false') return false
            // Handle numbers
            else if (valueStr.isNumber()) return valueStr.toInteger()
            // Default to string (remove quotes if present)
            else return valueStr.replaceAll(/^"(.*)"$/, '$1')
        } catch (Exception e) {
            return valueStr // Return as string if parsing fails
        }
    }


}




=========================================================================================================================================================
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


Subject: Request to Add Valid Standalone Binary Files (ZIP) with Versions

Hi [Recipient’s Name],

I noticed that the JSON from the DML URL you shared previously does not contain validated standalone binaries. To support our setup and enable unit testing, I request you to add standalone ZIP binary files for the following tools, with proper versions included, for both Windows and Linux:

Maven

Python

Gradle

Node.js

Having validated and versioned standalone ZIP binaries in the DML URL will help ensure consistency across environments and make our unit testing process more reliable.

Kindly update the JSON with the correct ZIP binary versions in the DML URL, and please let me know once this is available.


Please add stable Gradle versions from the EARC portal for our testing. As per our design, support for both Windows and Linux is required, but for now, adding ZIP binaries for any one environment will help us proceed with the work.
Please find below the Minutes of Meeting (MoM) regarding the discussion on handling .exe type binaries and associated installation scripts.

MoM – Q&A Format

Q1. Do we get the installation scripts along with the binaries in Artifactory?
A: No, the installation scripts are not bundled with binaries in Artifactory. They are maintained only in GitHub.

Q2. Are the scripts available for all four pipelines (Maven, Python, Node, and Gradle) on both Linux and Windows?
A: No, the scripts are not available for all four pipelines. They are currently available only for Python and Node on Windows.

Q3. Are the scripts available in Artifactory?
A: No, the scripts are not stored in Artifactory. They are available only in GitHub and require guest access to use.

Q4. How do the installation scripts work?
A: The scripts are written in PowerShell (.ps1) for each tool. They are designed to handle .exe type binaries, take the version and Artifactory API key as inputs, and allow installation of all supported versions.

Please find below the Minutes of Meeting (MoM) in Q&A format for our discussion on handling .exe type binaries and installation scripts.


Subject: Missing Binaries in Artifactory – Request to Update Source Links in tools.json

Hi [Team/Name],

We have identified that certain binaries are missing in Artifactory, which appears to be caused by outdated source links defined in the tools.json file.

Kindly review and update the source links in tools.json with the latest available sources. This issue is currently blocking our ongoing release, so your prompt action will be highly appreciated.

Please confirm once the update has been completed or let us know if any further information is required from our side.

Thank you for your support.

==============================

Slide 1: Faster Runtime - What & Why
(Presenter: Start with confidence and eye contact)

"Good morning everyone. Today I'm going to show you a game-changer for our pipeline execution - Faster Runtime Tool Management.

Let me paint a picture we've all experienced: You trigger a pipeline, and it fails because the specific Maven or Python version you need isn't installed on the Jenkins agent. You then spend hours coordinating with DevOps teams to get tools installed.

This feature solves exactly that problem. It automatically downloads and installs tools during pipeline execution - but here's the crucial part: only tools that are EARC-approved. This means we get both speed and compliance in one solution.

In just a moment, I'll show you a live demo of how this works. But first, let me explain the three scenarios you'll encounter."

Slide 2: Usage Scenarios
(Presenter: Use hand gestures to indicate the three scenarios)

"Now, when you use this feature, you'll typically encounter three scenarios:

First - when you're using a tool for the very first time. The pipeline checks, doesn't find it, verifies it's EARC-approved, and installs it automatically.

Second - the happy path. The tool is already there from a previous run, so we skip the download and use it immediately. This is where we see the real speed benefit.

Third - what happens if someone requests a tool that's not approved or doesn't exist. The system checks, finds it's not in our EARC approval list, and fails the pipeline with a clear error message.

This ensures we never compromise on security while gaining flexibility."

Slide 3: Live Demo
(Presenter: Switch to demo screen)

"Now let me show you this in action. I have a Jenkins pipeline configured to use WINDOWS_PYTHON_3.11.1.

(Start demo/video)

Watch the console output here... The system first checks if this version exists on the agent... It doesn't.

Now it's verifying with EARC's approval system... You can see the green check mark - it's approved.

It's now downloading from our SDI Artifactory... And here - installation completes in about 45 seconds.

Notice the format here - WINDOWS_PYTHON_3.11.1 - this strict format is critical. It must be OS, then tool name, then exact version in X.X.X format.

(Show successful build completion)

And there we have it - the pipeline completes successfully without any pre-installation."

Slide 4: Key Advantages
(Presenter: Move away from screen, engage audience)

"So what does this mean for us? Let me highlight three key advantages:

First - Efficiency. We eliminate days of waiting for tool installations. Teams can specify exactly what they need, when they need it.

Second - Compliance. This isn't a wild-west tool download system. Every single tool must be EARC-approved. Our security standards are baked right into the process.

Third - Flexibility. Different projects can use different Python or Node versions on the same Jenkins agent. No more 'one-size-fits-all' tool installations that satisfy nobody.

The result? Developers spend more time coding, less time waiting for environments."

Slide 5: Summary & Resources
(Presenter: Summarize clearly, slower pace)

"To wrap up, here's what you need to know:

First - always use the correct format: OS, underscore, tool name, underscore, and the exact version.

Second - before you use any tool, check our EARC approval portal at the URL shown here. This will save you from pipeline failures.

Third - we have dedicated teams to support you. For technical issues, contact the SDI Artifactory team. For new tool approvals, reach out to the EARC team.

Finally - start small. Test this in your development pipelines first. Get comfortable with the format and process.

Any questions about what we've covered or about the demo you just saw?"

Presenter Notes:

Timing: Aim for 2-3 minutes per slide

Engagement: Ask rhetorical questions like "How many of you have faced this problem?"

Transitions: Use phrases like "Now, let's look at...", "Moving to the next point...", "What does this mean for you?"

Demo: Practice the demo flow beforehand, have a backup screenshot ready

Q&A: End with "I'm happy to take any questions" and pause for 5 seconds

give it simple english
Slide 1: What This Feature Does
"Hello everyone. Today I'll show you a new feature that makes pipelines run faster.

We all know this problem: Your pipeline needs a specific tool, but it's not installed on the Jenkins agent. So the pipeline fails, and you have to wait for someone to install it.

This feature fixes that. When your pipeline needs a tool, it automatically downloads and installs it. But only if the tool is approved by our security team (EARC).

This means: No more waiting for tool installations, but still safe and approved tools."

Slide 2: How You Use It
"You'll see three situations when using this:

First time use - The tool isn't on the agent, so the system downloads and installs it.

Tool already there - If you used it before, it's already installed. The system skips the download.

Wrong or not approved - If you ask for a tool that's not approved, the pipeline will fail.

Simple rule: Always use this format: OS_TOOLNAME_VERSION
Like: WINDOWS_PYTHON_3.11.1"

Slide 3: Let Me Show You
(Switch to demo screen)

"Watch this. I have a pipeline that needs Python 3.11.1.

First, it checks if Python is already installed... It's not.

Then it checks if this version is approved... Yes, it is.

Now it downloads and installs... Takes about 45 seconds.

And now the pipeline runs successfully!

See the format here: WINDOWS_PYTHON_3.11.1
That's important - use exactly this format."

Slide 4: Why This Is Good
"This feature helps us in three ways:

Saves time - No waiting for tool installations. You get what you need right away.

Keeps us safe - Only approved tools can be installed. Our security team controls the list.

More flexible - Different teams can use different tool versions on the same Jenkins agent.

Result: Less waiting, more working."

Slide 5: What You Need to Do
"Here's what you should remember:

Check first - Before using a tool, check if it's approved on our EARC website.

Use right format - Always use: OS_TOOLNAME_VERSION.VERSION.VERSION

Test first - Try it in your development pipelines before using in production.

Ask for help - If you need a new tool approved, contact the EARC team. If you have technical problems, contact the SDI team.

Questions? I'm happy to answer them."

