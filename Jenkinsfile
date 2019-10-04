// Set your project Prefix using your GUID
def prefix      = "414e"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -s ./nexus_openshift_settings.xml"
// Set Development and Production Project Names
def devProject  = "${prefix}-tasks-dev"
def prodProject = "${prefix}-tasks-prod"
// Set the tag for the development image: version + build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""

pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "maven-appdev"
  }
  stages {
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
      steps {
        // TBD: Get code from protected Git repository
       git url: 'https://github.com/Tutj/advdev_homework_template.git'
       script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version

          // TBD: Set the tag for the development image: version + build number.
          // Example: def devTag  = "0.0-0"
          devTag  = "${version}-" + currentBuild.number

          // TBD: Set the tag for the production image: version
          // Example: def prodTag = "0.0"
          prodTag = "${version}"

        }
      }
    }
    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build War File') {
      steps {
        echo "Building version ${devTag}"
        sh "${mvnCmd} clean package -DskipTests=true"

      }
    }

    // Using Maven run the unit tests
    parallel {
      stage('Unit Tests') {
        steps {
          echo "Running Unit Tests"
          sh "${mvnCmd} test"
          // TBD
        }
      }

      //Using Maven call SonarQube for Code Analysis
      stage('Code Analysis') {
        steps {
          echo "Running Code Analysis"
          sh "${mvnCmd} admin:redhat -Dsonar.host.url=http://sonarqube-gpte-hw-cicd.apps.na311.openshift.opentlc.com -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
          // TBD

        }
      }
    }

    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps {
        echo "Publish to Nexus"

        // TBD
        sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.gpte-hw-cicd.svc.cluster.local:8081/repository/maven-all-public"

      }
    }

    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps {
        echo "Building OpenShift container image tasks:${devTag}"

        // TBD: Start binary build in OpenShift using the file we just published.
        // Either use the file from your
        // workspace (filename to pass into the binary build
        // is openshift-tasks.war in the 'target' directory of
        // your current Jenkins workspace).
        // OR use the file you just published into Nexus:
        // "--from-file=http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war"


        // TBD: Tag the image using the devTag.
        script {
            openshift.withCluster() {
                openshift.withProject("${devProject}") {
                    openshift.selector("bc", "tasks").startBuild("--from-file=./target/openshift-tasks.war", "--wait=true")
                    openshift.tag("tasks:latest", "tasks:${devTag}")
                }
            }
        }

      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      steps {
        echo "Deploy container image to Development Project"
        script {
          // Update the Image on the Development Deployment Config
          openshift.withCluster() {
            openshift.withProject("${devProject}") {
              
             // For OpenShift 3 use this:
              openshift.set("image", "dc/tasks", "tasks=docker-registry.default.svc:5000/${devProject}/tasks:${devTag}")
    
              // Update the Config Map which contains the users for the Tasks application
              // (just in case the properties files changed in the latest commit)
              openshift.selector('configmap', 'tasks-config').delete()
              def configmap = openshift.create('configmap', 'tasks-config', '--from-file=./configuration/application-users.properties', '--from-file=./configuration/application-roles.properties' )
    
              // Deploy the development application.
              openshift.selector("dc", "tasks").rollout().latest();
    
              // Wait for application to be deployed
              def dc = openshift.selector("dc", "tasks").object()
              def dc_version = dc.status.latestVersion
              def rc = openshift.selector("rc", "tasks-${dc_version}").object()
    
              echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
              while (rc.spec.replicas != rc.status.readyReplicas) {
                sleep 5
                rc = openshift.selector("rc", "tasks-${dc_version}").object()
              }
            }
          }
        }
      }
    }

    // Copy Image to Nexus Docker Registry
    stage('Copy Image to Nexus Docker Registry') {
      steps {
        echo "Copy image to Nexus Docker Registry"
        script {

          sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:redhat docker://docker-registry.default.svc.cluster.local:5000/${devProject}/tasks:${devTag} docker://nexus3-registry.gpte-hw-cicd.svc.cluster.local:5000/tasks:${devTag}"
    
          // Tag the built image with the production tag.
          openshift.withCluster() {
            openshift.withProject("${prodProject}") {
              openshift.tag("${devProject}/tasks:${devTag}", "${devProject}/tasks:${prodTag}")
            }
          }
        }
      }
    }

    // Blue/Green Deployment into Production
    // -------------------------------------
    // Do not activate the new version yet.
    stage('Blue/Green Production Deployment') {
      steps {
        echo "Blue/Green Deployment"
        script {
          openshift.withCluster() {
            openshift.withProject("${prodProject}") {
              activeApp = openshift.selector("route", "tasks").object().spec.to.name
              if (activeApp == "tasks-green") {
                destApp = "tasks-blue"
              }
              echo "Active Application:      " + activeApp
              echo "Destination Application: " + destApp
    
              // Update the Image on the Production Deployment Config
              def dc = openshift.selector("dc/${destApp}").object()
    
              // OpenShift 3
              dc.spec.template.spec.containers[0].image="docker-registry.default.svc:5000/${devProject}/tasks:${prodTag}"
    
              openshift.apply(dc)
    
              // Update Config Map in change config files changed in the source
              openshift.selector("configmap", "${destApp}-config").delete()
              def configmap = openshift.create("configmap", "${destApp}-config", "--from-file=./configuration/application-users.properties", "--from-file=./configuration/application-roles.properties" )
    
              // Deploy the inactive application.
              openshift.selector("dc", "${destApp}").rollout().latest();
    
              // Wait for application to be deployed
              def dc_prod = openshift.selector("dc", "${destApp}").object()
              def dc_version = dc_prod.status.latestVersion
              def rc_prod = openshift.selector("rc", "${destApp}-${dc_version}").object()
              echo "Waiting for ${destApp} to be ready"
              while (rc_prod.spec.replicas != rc_prod.status.readyReplicas) {
                sleep 5
                rc_prod = openshift.selector("rc", "${destApp}-${dc_version}").object()
              }
            }
          }
        }
      }
    }
  }
}

