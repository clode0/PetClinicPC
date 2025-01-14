pipeline {
    
    agent any
    
    tools {
        // Install the Maven version configured as "maven3" and add it to the path.
        maven "M3"
    }
    
    environment {
        // This can be nexus3 or nexus2
        NEXUS_VERSION = "nexus3"
        // This can be http or https
        NEXUS_PROTOCOL = "https"
        // Where your Nexus is running
        NEXUS_URL = "nexus-fitec.sof3way.fr"
        // Repository where we will upload the artifact
        NEXUS_REPOSITORY = "petclinic-repo/"
        // Jenkins credential id to authenticate to Nexus OSS
        NEXUS_CREDENTIAL_ID = "nexus-user-credentials"
    }
    
    stages {
        
        stage('Clone Project') {
            steps {
              echo 'Compilation in progress ...'
              sh "pwd && ls -la"
              git 'https://github.com/Taki12/PetClinicPC'
            }
        }
        
        stage('Maven Build') {
            steps {
              // Run Maven on a Unix agent.
              sh "mvn clean install -P MySQL -DskipTests"
            }
        }
        
        stage('SonarQube Analysis') {
	    steps {
                withSonarQubeEnv('sonarqube') {
		     sh "mvn clean package sonar:sonar"
		}
             }
        } 
        
        stage("Quality Gate"){
	      steps {
                   echo 'quality gate check'
                   waitForQualityGate abortPipeline: true 
              }
	}       
        
        stage("Publish to Nexus") {
            steps {
                script {
                    // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
                    pom = readMavenPom file: "pom.xml";
                    // Find built artifact under target folder
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    // Print some info from the artifact found
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    // Extract the path from the File found
                    artifactPath = filesByGlob[0].path;
                    // Assign to a boolean response verifying If the artifact name exists
                    artifactExists = fileExists artifactPath;

                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";

                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                // Artifact generated such as .jar, .ear and .war files.
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],

                                // Lets upload the pom.xml file for additional information for Transitive dependencies
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );

                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }
        
        stage('Deploy in local VM before Azure deployment') {
            steps {
                sh "docker compose up -d"
            }
        }
        
        stage('Test Petclinic App in local VM') {
            steps {
                sleep(time:60,unit:"SECONDS")
                sh 'curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8083/'
            }
        }
            
        stage('Docker compose down') {
            steps {
               sh "docker compose down"
            }
        }
    
    }
}
