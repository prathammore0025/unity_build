## Jenkins Setup and Unity Build Configuration

### Jenkins Installation on EC2 Ubuntu Linux

1. **Install Java** (Jenkins requires Java):
    ```bash
    sudo apt update
    sudo apt install openjdk-11-jdk
    ```

2. **Add Jenkins repository**:
    ```bash
    wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
    sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
    ```

3. **Install Jenkins**:
    ```bash
    sudo apt update
    sudo apt install jenkins
    ```

4. **Start Jenkins**:
    ```bash
    sudo systemctl start jenkins
    sudo systemctl enable jenkins
    ```

5. **Unlock Jenkins**:
    - Open Jenkins in your browser (`http://localhost:8080`).
    - Get the initial admin password:
      ```bash
      sudo cat /var/lib/jenkins/secrets/initialAdminPassword
      ```
    - Follow the instructions to unlock Jenkins and install suggested plugins.

6. **Create an Admin User**.

### Configure Local Windows Server as Jenkins Agent

1. **Install Java**:
    - Download and install Java JDK from [Oracle Java](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html) or [AdoptOpenJDK](https://adoptopenjdk.net/).

2. **Configure Jenkins Agent on Windows**:

   #### Option 1: Launch Agent via JNLP
    - **Create a New Node** in Jenkins:
      - Go to Jenkins Dashboard > Manage Jenkins > Manage Nodes and Clouds > New Node.
      - Select **Permanent Agent**, name it (e.g., `windows-agent`), and configure:
        - **Remote Root Directory**: `C:\Jenkins`
        - **Labels**: Add `unity`
        - **Launch Method**: Select **Launch agent by connecting it to the master using Java Web Start (JNLP)**.
    - **Download Agent JAR** from Jenkins and run the following command:
      ```bash
      java -jar agent.jar -jnlpUrl http://<your-jenkins-url>/computer/windows-agent/slave-agent.jnlp -secret <secret-key>
      ```

   #### Option 2: Launch Agent via SSH
    - Install an SSH server on Windows.
    - **Create a New Node** in Jenkins:
      - Go to Jenkins Dashboard > Manage Jenkins > Manage Nodes and Clouds > New Node.
      - Select **Permanent Agent**, configure SSH launch method.

### Unity Setup on Windows Server

1. **Install Unity**:
    - Download Unity Hub from [Unity's website](https://unity3d.com/get-unity/download).
    - Install Unity and ensure **WebGL Build Support** is added.

2. **Set Unity Path in Environment Variables**:
    - Go to **Control Panel** > **System** > **Advanced system settings** > **Environment Variables**.
    - Add a new system variable:
      - **Variable name**: `UNITY_PATH`
      - **Variable value**: Path to Unity executable (e.g., `C:\Program Files\Unity\Hub\Editor\2020.3.0f1\Editor\Unity.exe`).

### Jenkins Pipeline Configuration

1. **Create a Pipeline Job**:
    - Go to Jenkins Dashboard > New Item > Pipeline and name it (e.g., `Unity-WebGL-Build`).

2. **Configure Pipeline Script**:
    ```groovy
    pipeline {
        agent { label 'unity' }  // Target the agent with the 'unity' label

        environment {
            UNITY_PATH = "${UNITY_PATH}"  // Unity path from the environment variable
            BUILD_METHOD = "BuildScript.PerformWebGLBuild"  // Method to run in Unity for the build
            PROJECT_PATH = "${WORKSPACE}/"  // Jenkins workspace path
            BUILD_OUTPUT = "${WORKSPACE}/Build/WebGL"  // Output directory for the WebGL build
        }

        stages {
            stage('Checkout') {
                steps {
                    git branch: 'main', url: 'https://github.com/your-repo/unity-project.git'
                }
            }

            stage('Build WebGL') {
                steps {
                    bat """
                    ${UNITY_PATH} -batchmode -quit \
                    -projectPath ${PROJECT_PATH} \
                    -executeMethod ${BUILD_METHOD} \
                    -buildTarget WebGL \
                    -logFile ${WORKSPACE}/unity_build.log \
                    -output ${BUILD_OUTPUT}
                    """
                }
            }
        }

        post {
            always {
                archiveArtifacts artifacts: '**/unity_build.log', allowEmptyArchive: true
                archiveArtifacts artifacts: 'Build/WebGL/**/*', allowEmptyArchive: true
            }

            failure {
                echo 'Build failed!'
            }
        }
    }
    ```

### Additional Notes
- Ensure the Windows agent can communicate with the Jenkins master.
- Configure necessary network and firewall settings to allow the Jenkins server and agent to interact.
- Verify that Unity is correctly installed and licensed on the Windows build agent.

