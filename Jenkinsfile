#!groovy
def prefix = "172d"

podTemplate(
  label: "skopeo-pod",
  cloud: "openshift",
  inheritFrom: "maven",
  containers: [
    containerTemplate(
      name: "jnlp",
      image: "docker-registry.default.svc:5000/${prefix}-jenkins/jenkins-agent-appdev",
      resourceRequestMemory: "1Gi",
      resourceLimitMemory: "2Gi",
      resourceRequestCpu: "1",
      resourceLimitCpu: "2"
    )
  ]
) {
  node('skopeo-pod') {
    // Define Maven Command to point to the correct
    // settings for our Nexus installation
    def mvnCmd = "mvn -s ../nexus_settings.xml"

    // Checkout Source Code.
    stage('Checkout Source') {
        git 'https://github.com/mjhall77/Avd-Apps-Class.git'
    }

    // Build the Tasks Service
    dir('openshift-tasks') {
      // The following variables need to be defined at the top level
      // and not inside the scope of a stage - otherwise they would not
      // be accessible from other stages.
      // Extract version from the pom.xml
      def version = getVersionFromPom("pom.xml")

      // Set the tag for the development image: version + build number
      def devTag = "${version}-" + currentBuild.number

      // Set the tag for the production image: version
      def prodTag = "${version}"

      // Set devProject 
      def devProject  = "${prefix}-tasks-dev"

      // Set prodProject
      def prodProject = "${prefix}-tasks-prod"

      // Using Maven build the war file
      // Do not run tests in this step
      stage('Build war') {
        echo "Building version ${devTag}"
        sh "${mvnCmd} clean package -DskipTests=true" 

        // TBD: Execute Maven Build
      }

      // TBD: The next two stages should run in parallel

      // Using Maven run the unit tests
      stage('Unit Tests') {
        echo "Running Unit Tests"
        sh "${mvnCmd} test"
        // TBD: Execute Unit Tests
      }

      // Using Maven to call SonarQube for Code Analysis
      stage('Code Analysis') {
        echo "Running Code Analysis"
        sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-gpte-hw-cicd.apps.na311.openshift.opentlc.com -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
        // TBD: Execute Sonarqube Tests
      }

      // Publish the built war file to Nexus
      stage('Publish to Nexus') {
        echo "Publish to Nexus"
        sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3-gpte-hw-cicd.apps.na311.openshift.opentlc.com/repository/releases"
        // TBD: Publish to Nexus
      }

      // Build the OpenShift Image in OpenShift and tag it.
      stage('Build and Tag OpenShift Image') {
        echo "Building OpenShift container image tasks:${devTag}"

        // TBD: Build Image, tag Image

      script {
        openshift.withCluster() {
          openshift.withProject("${devProject}") {
            openshift.selector("bc", "tasks").startBuild("--from-file=http://nexus3-gpte-hw-cicd.apps.na311.openshift.opentlc.com/repository/releases/org/jboss/quickstarts/eap/tasks/${version}/tasks-${version}.war", "--wait=true")

          // OR use the file you just published into Nexus:
          // "--from-file=http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${version}/tasks-${version}.war"
            openshift.tag("tasks:latest", "tasks:${devTag}")
          }
        }
      }
      }

      // Deploy the built image to the Development Environment.
      stage('Deploy to Dev') {
        echo "Deploying container image to Development Project"
        script {
        // Update the Image on the Development Deployment Config
          openshift.withCluster() {
            openshift.withProject("${devProject}") {
              openshift.set("env", "dc/tasks", "VERSION='0.0 (tasks-dev)'")
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

      // Copy Image to Nexus container registry
      stage('Copy Image to Nexus container registry') {
        echo "Copy image to Nexus container registry"
        script {
          sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:redhat docker://docker-registry.default.svc.cluster.local:5000/${devProject}/tasks:${devTag} docker://nexus-registry-gpte-hw-cicd.apps.na311.openshift.opentlc.com/tasks:${devTag}"

          // Tag the built image with the production tag.
          openshift.withCluster() {
            openshift.withProject("${prodProject}") {
              openshift.tag("${devProject}/tasks:${devTag}", "${devProject}/tasks:${prodTag}")
            }
          }
        }
      }

      // Blue/Green Deployment into Production
      // -------------------------------------
      def destApp   = "tasks-green"
      def activeApp = ""

      stage('Blue/Green Production Deployment') {
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

              openshift.set("env", "dc/${destApp}", "--overwrite VERSION='${prdTag} (${destApp})'")

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
        // TBD: Determine which application is active
        //      Set Image, Set VERSION
        //      Deploy into the other application
        //      Make sure the application is running and ready before proceeding
      }

      stage('Switch over to new Version') {
        echo "Switching Production application to ${destApp}."
        script {
          openshift.withCluster() {
            openshift.withProject("${prodProject}") {
              def route = openshift.selector("route/tasks").object()
              route.spec.to.name="${destApp}"
              openshift.apply(route)
            }
          }
        }
      }
    }
  }
}

// Convenience Functions to read version from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
