apiVersion: v1
kind: Template
labels:
  template: cicd
  group: cicd
metadata:
  annotations:
    iconClass: icon-jenkins
    tags: instant-app,jenkins,gogs,nexus,cicd
  name: cicd
message: "Use the following credentials for login:\nJenkins: use your OpenShift credentials\nNexus: admin/admin123\nSonarQube: admin/admin\nGogs Git Server: gogs/gogs"
parameters:
- displayName: DEV project name
  value: dev
  name: DEV_PROJECT
  required: true
- displayName: STAGE project name
  value: stage
  name: STAGE_PROJECT
  required: true
- displayName: Ephemeral
  description: Use no persistent storage for Gogs and Nexus
  value: "true"
  name: EPHEMERAL
  required: true
- description: Webhook secret
  from: '[a-zA-Z0-9]{8}'
  generate: expression
  name: WEBHOOK_SECRET
  required: true
- displayName: Integrate Quay.io
  description: Integrate image build and deployment with Quay.io 
  value: "false"
  name: ENABLE_QUAY
  required: true
- displayName: Quay.io Username
  description: Quay.io username to push the images to tasks-sample-app repository on your Quay.io account
  name: QUAY_USERNAME
- displayName: Quay.io Password
  description: Quay.io password to push the images to tasks-sample-app repository on your Quay.io account
  name: QUAY_PASSWORD
- displayName: Quay.io Image Repository
  description: Quay.io repository for pushing Tasks container images
  name: QUAY_REPOSITORY
  required: true
  value: tasks-app
objects:
- apiVersion: v1
  groupNames: null
  kind: RoleBinding
  metadata:
    name: default_admin
  roleRef:
    name: admin
  subjects:
  - kind: ServiceAccount
    name: default
# Pipeline
- apiVersion: v1
  kind: BuildConfig
  metadata:
    annotations:
      pipeline.alpha.openshift.io/uses: '[{"name": "jenkins", "namespace": "", "kind": "DeploymentConfig"}]'
    labels:
      app: cicd-pipeline
      name: cicd-pipeline
    name: tasks-pipeline
  spec:
    triggers:
      - type: GitHub
        github:
          secret: ${WEBHOOK_SECRET}
      - type: Generic
        generic:
          secret: ${WEBHOOK_SECRET}
    runPolicy: Serial
    source:
      type: None
    strategy:
      jenkinsPipelineStrategy:
        env:
        - name: DEV_PROJECT
          value: ${DEV_PROJECT}
        - name: STAGE_PROJECT
          value: ${STAGE_PROJECT}
        - name: ENABLE_QUAY
          value: ${ENABLE_QUAY}
        jenkinsfile: |-
          def mvnCmd = "mvn -s configuration/cicd-settings-nexus3.xml"

          pipeline {
            agent {
              label 'maven'
            }
            stages {
              stage('Build App') {
                steps {
                  git branch: 'eap-7', url: 'http://gogs:3000/gogs/openshift-tasks.git'
                  sh "${mvnCmd} install -DskipTests=true"
                }
              }
              stage('Test') {
                steps {
                  sh "${mvnCmd} test"
                  step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
                }
              }
              stage('Code Analysis') {
                steps {
                  script {
                    sh "${mvnCmd} install sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
                  }
                }
              }
              stage('Archive App') {
                steps {
                  sh "${mvnCmd} deploy -DskipTests=true -P nexus3"
                }
              }
              stage('Build Image') {
                steps {
                  sh "cp target/openshift-tasks.war target/ROOT.war"
                  script {
                    openshift.withCluster() {
                      openshift.withProject(env.DEV_PROJECT) {
                        openshift.selector("bc", "tasks").startBuild("--from-file=target/ROOT.war", "--wait=true")
                      }
                    }
                  }
                }
              }
              stage('Deploy DEV') {
                steps {
                  script {
                    openshift.withCluster() {
                      openshift.withProject(env.DEV_PROJECT) {
                        openshift.selector("dc", "tasks").rollout().latest();
                      }
                    }
                  }
                }
              }
              stage('Promote to STAGE?') {
                agent {
                  label 'skopeo'
                }
                steps {
                  timeout(time:15, unit:'MINUTES') {
                      input message: "Promote to STAGE?", ok: "Promote"
                  }

                  script {
                    openshift.withCluster() {
                      if (env.ENABLE_QUAY.toBoolean()) {
                        withCredentials([usernamePassword(credentialsId: "${openshift.project()}-quay-cicd-secret", usernameVariable: "QUAY_USER", passwordVariable: "QUAY_PWD")]) {
                          sh "skopeo copy docker://quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:latest docker://quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:stage --src-creds \"$QUAY_USER:$QUAY_PWD\" --dest-creds \"$QUAY_USER:$QUAY_PWD\" --src-tls-verify=false --dest-tls-verify=false"
                        }
                      } else {
                        openshift.tag("${env.DEV_PROJECT}/tasks:latest", "${env.STAGE_PROJECT}/tasks:stage")
                      }
                    }
                  }
                }
              }
              stage('Deploy STAGE') {
                steps {
                  script {
                    openshift.withCluster() {
                      openshift.withProject(env.STAGE_PROJECT) {
                        openshift.selector("dc", "tasks").rollout().latest();
                      }
                    }
                  }
                }
              }
            }
          }
      type: JenkinsPipeline
- apiVersion: v1
  kind: ConfigMap
  metadata:
    labels:
      app: cicd-pipeline
      role: jenkins-slave
    name: jenkins-slaves
  data:
    maven-template: |-
      <org.csanchez.jenkins.plugins.kubernetes.PodTemplate>
        <inheritFrom></inheritFrom>
        <name>maven</name>
        <privileged>false</privileged>
        <alwaysPullImage>false</alwaysPullImage>
        <instanceCap>2147483647</instanceCap>
        <idleMinutes>0</idleMinutes>
        <label>maven</label>
        <serviceAccount>jenkins</serviceAccount>
        <nodeSelector></nodeSelector>
        <customWorkspaceVolumeEnabled>false</customWorkspaceVolumeEnabled>
        <workspaceVolume class="org.csanchez.jenkins.plugins.kubernetes.volumes.workspace.EmptyDirWorkspaceVolume">
          <memory>false</memory>
        </workspaceVolume>
        <volumes />
        <containers>
          <org.csanchez.jenkins.plugins.kubernetes.ContainerTemplate>
            <name>jnlp</name>
            <image>quay.io/siamaksade/jenkins-agent-maven-node:4.6</image>
            <privileged>false</privileged>
            <alwaysPullImage>false</alwaysPullImage>
            <workingDir>/tmp</workingDir>
            <command></command>
            <args>${computer.jnlpmac} ${computer.name}</args>
            <ttyEnabled>false</ttyEnabled>
            <resourceRequestCpu>200m</resourceRequestCpu>
            <resourceRequestMemory>512Mi</resourceRequestMemory>
            <resourceLimitCpu>2</resourceLimitCpu>
            <resourceLimitMemory>4Gi</resourceLimitMemory>
            <envVars/>
          </org.csanchez.jenkins.plugins.kubernetes.ContainerTemplate>
        </containers>
        <envVars/>
        <annotations/>
        <imagePullSecrets/>
      </org.csanchez.jenkins.plugins.kubernetes.PodTemplate>
    skopeo-template: |-
      <org.csanchez.jenkins.plugins.kubernetes.PodTemplate>
        <inheritFrom></inheritFrom>
        <name>skopeo</name>
        <privileged>false</privileged>
        <alwaysPullImage>false</alwaysPullImage>
        <instanceCap>2147483647</instanceCap>
        <idleMinutes>0</idleMinutes>
        <label>skopeo</label>
        <serviceAccount>jenkins</serviceAccount>
        <nodeSelector></nodeSelector>
        <customWorkspaceVolumeEnabled>false</customWorkspaceVolumeEnabled>
        <workspaceVolume class="org.csanchez.jenkins.plugins.kubernetes.volumes.workspace.EmptyDirWorkspaceVolume">
          <memory>false</memory>
        </workspaceVolume>
        <volumes />
        <containers>
          <org.csanchez.jenkins.plugins.kubernetes.ContainerTemplate>
            <name>jnlp</name>
            <image>docker.io/siamaksade/jenkins-slave-skopeo-centos7</image>
            <privileged>false</privileged>
            <alwaysPullImage>false</alwaysPullImage>
            <workingDir>/tmp</workingDir>
            <command></command>
            <args>${computer.jnlpmac} ${computer.name}</args>
            <ttyEnabled>false</ttyEnabled>
            <envVars/>
          </org.csanchez.jenkins.plugins.kubernetes.ContainerTemplate>
        </containers>
        <envVars/>
        <annotations/>
        <imagePullSecrets/>
      </org.csanchez.jenkins.plugins.kubernetes.PodTemplate>
# Setup Demo
- apiVersion: batch/v1
  kind: Job
  metadata:
    name: cicd-demo-installer
  spec:
    activeDeadlineSeconds: 400
    completions: 1
    parallelism: 1
    template:
      spec:
        containers:
        - env:
          - name: CICD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          command:
          - /bin/bash
          - -x
          - -c
          - |           
            # adjust jenkins 
            oc set resources dc/jenkins --limits=cpu=2,memory=2Gi --requests=cpu=100m,memory=512Mi 
            oc label dc jenkins app=jenkins --overwrite 

            # setup dev env
            oc import-image wildfly --from=openshift/wildfly-120-centos7 --confirm -n ${DEV_PROJECT} 
            
            if [ "${ENABLE_QUAY}" == "true" ] ; then
              # cicd
              oc create secret generic quay-cicd-secret --from-literal="username=${QUAY_USERNAME}" --from-literal="password=${QUAY_PASSWORD}" -n ${CICD_NAMESPACE}
              oc label secret quay-cicd-secret credential.sync.jenkins.openshift.io=true -n ${CICD_NAMESPACE}
              
              # dev
              oc create secret docker-registry quay-cicd-secret --docker-server=quay.io --docker-username="${QUAY_USERNAME}" --docker-password="${QUAY_PASSWORD}" --docker-email=cicd@redhat.com -n ${DEV_PROJECT}
              oc new-build --name=tasks --image-stream=wildfly:latest --binary=true --push-secret=quay-cicd-secret --to-docker --to='quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:latest' -n ${DEV_PROJECT}
              oc new-app --name=tasks --docker-image=quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:latest --allow-missing-images --as-deployment-config -n ${DEV_PROJECT}
              oc set triggers dc tasks --remove-all -n ${DEV_PROJECT}
              oc patch dc tasks -p '{"spec": {"template": {"spec": {"containers": [{"name": "tasks", "imagePullPolicy": "Always"}]}}}}' -n ${DEV_PROJECT}
              oc delete is tasks -n ${DEV_PROJECT}
              oc secrets link default quay-cicd-secret --for=pull -n ${DEV_PROJECT}
              
              # stage
              oc create secret docker-registry quay-cicd-secret --docker-server=quay.io --docker-username="${QUAY_USERNAME}" --docker-password="${QUAY_PASSWORD}" --docker-email=cicd@redhat.com -n ${STAGE_PROJECT}
              oc new-app --name=tasks --docker-image=quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:stage --allow-missing-images --as-deployment-config -n ${STAGE_PROJECT}
              oc set triggers dc tasks --remove-all -n ${STAGE_PROJECT}
              oc patch dc tasks -p '{"spec": {"template": {"spec": {"containers": [{"name": "tasks", "imagePullPolicy": "Always"}]}}}}' -n ${STAGE_PROJECT}
              oc delete is tasks -n ${STAGE_PROJECT}
              oc secrets link default quay-cicd-secret --for=pull -n ${STAGE_PROJECT}
            else
              # dev
              oc new-build --name=tasks --image-stream=wildfly:latest --binary=true -n ${DEV_PROJECT}
              oc new-app tasks:latest --allow-missing-images --as-deployment-config -n ${DEV_PROJECT}
              oc set triggers dc -l app=tasks --containers=tasks --from-image=tasks:latest --manual -n ${DEV_PROJECT}
              
              # stage
              oc new-app tasks:stage --allow-missing-images --as-deployment-config -n ${STAGE_PROJECT}
              oc set triggers dc -l app=tasks --containers=tasks --from-image=tasks:stage --manual -n ${STAGE_PROJECT}
            fi
            
            # dev project
            oc expose dc/tasks --port=8080 -n ${DEV_PROJECT}
            oc expose svc/tasks -n ${DEV_PROJECT}
            oc set probe dc/tasks --readiness --get-url=http://:8080/ws/demo/healthcheck --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10 -n ${DEV_PROJECT}
            oc set probe dc/tasks --liveness  --get-url=http://:8080/ws/demo/healthcheck --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10 -n ${DEV_PROJECT}
            oc rollout cancel dc/tasks -n ${STAGE_PROJECT}

            # stage project
            oc expose dc/tasks --port=8080 -n ${STAGE_PROJECT}
            oc expose svc/tasks -n ${STAGE_PROJECT}
            oc set probe dc/tasks --readiness --get-url=http://:8080/ws/demo/healthcheck --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10 -n ${STAGE_PROJECT}
            oc set probe dc/tasks --liveness  --get-url=http://:8080/ws/demo/healthcheck --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10 -n ${STAGE_PROJECT}
            oc rollout cancel dc/tasks -n ${DEV_PROJECT}

            # deploy gogs
            HOSTNAME=$(oc get route jenkins -o template --template='{{.spec.host}}' | sed "s/jenkins-${CICD_NAMESPACE}.//g")
            GOGS_HOSTNAME="gogs-$CICD_NAMESPACE.$HOSTNAME"

            if [ "${EPHEMERAL}" == "true" ] ; then
              oc new-app -f https://raw.githubusercontent.com/siamaksade/gogs/master/gogs-template-ephemeral.yaml \
                  --param=GOGS_VERSION=0.11.34 \
                  --param=HOSTNAME=$GOGS_HOSTNAME \
                  --param=SKIP_TLS_VERIFY=true
            else
              oc new-app -f https://raw.githubusercontent.com/siamaksade/gogs/master/gogs-template.yaml \
                  --param=GOGS_VERSION=0.11.34 \
                  --param=HOSTNAME=$GOGS_HOSTNAME \
                  --param=SKIP_TLS_VERIFY=true
            fi
            
            sleep 5

            if [ "${EPHEMERAL}" == "true" ] ; then
              oc process -f https://raw.githubusercontent.com/siamaksade/sonarqube/8/sonarqube-template.yaml | oc create -f -
            else
              oc process -f https://raw.githubusercontent.com/siamaksade/sonarqube/8/sonarqube-persistent-template.yaml | oc create -f -
            fi

            oc set resources dc/sonarqube --limits=cpu=1,memory=2.5Gi --requests=cpu=200m,memory=512Mi

            if [ "${EPHEMERAL}" == "true" ] ; then
              oc new-app -f https://raw.githubusercontent.com/OpenShiftDemos/nexus/master/nexus3-template.yaml --param=NEXUS_VERSION=3.13.0 --param=MAX_MEMORY=2Gi
            else
              oc new-app -f https://raw.githubusercontent.com/OpenShiftDemos/nexus/master/nexus3-persistent-template.yaml --param=NEXUS_VERSION=3.13.0 --param=MAX_MEMORY=2Gi
            fi

            oc set resources dc/nexus --requests=cpu=200m --limits=cpu=2

            GOGS_SVC=$(oc get svc gogs -o template --template='{{.spec.clusterIP}}')
            GOGS_USER=gogs
            GOGS_PWD=gogs

            oc rollout status dc gogs

            # Even though the rollout is complete gogs isn't always ready to create the admin user
            sleep 10

            # Try 10 times to create the admin user. Fail after that.
            for i in {1..10};
            do

              _RETURN=$(curl -o /tmp/curl.log -sL --post302 -w "%{http_code}" http://$GOGS_SVC:3000/user/sign_up \
                --form user_name=$GOGS_USER \
                --form password=$GOGS_PWD \
                --form retype=$GOGS_PWD \
                --form email=admin@gogs.com)

              if [ $_RETURN == "200" ] || [ $_RETURN == "302" ]
              then
                echo "SUCCESS: Created gogs admin user"
                break
              elif [ $_RETURN != "200" ] && [ $_RETURN != "302" ] && [ $i == 10 ]; then
                echo "ERROR: Failed to create Gogs admin"
                cat /tmp/curl.log
                exit 255
              fi

              # Sleep between each attempt
              sleep 10

            done


            cat <<EOF > /tmp/data.json
            {
              "clone_addr": "https://github.com/OpenShiftDemos/openshift-tasks.git",
              "uid": 1,
              "repo_name": "openshift-tasks"
            }
            EOF

            _RETURN=$(curl -o /tmp/curl.log -sL -w "%{http_code}" -H "Content-Type: application/json" \
            -u $GOGS_USER:$GOGS_PWD -X POST http://$GOGS_SVC:3000/api/v1/repos/migrate -d @/tmp/data.json)

            if [ $_RETURN != "201" ] ;then
              echo "ERROR: Failed to import openshift-tasks GitHub repo"
              cat /tmp/curl.log
              exit 255
            fi

            sleep 5

            cat <<EOF > /tmp/data.json
            {
              "type": "gogs",
              "config": {
                "url": "https://openshift.default.svc.cluster.local/apis/build.openshift.io/v1/namespaces/$CICD_NAMESPACE/buildconfigs/tasks-pipeline/webhooks/${WEBHOOK_SECRET}/generic",
                "content_type": "json"
              },
              "events": [
                "push"
              ],
              "active": true
            }
            EOF

            _RETURN=$(curl -o /tmp/curl.log -sL -w "%{http_code}" -H "Content-Type: application/json" \
            -u $GOGS_USER:$GOGS_PWD -X POST http://$GOGS_SVC:3000/api/v1/repos/gogs/openshift-tasks/hooks -d @/tmp/data.json)

            if [ $_RETURN != "201" ] ; then
              echo "ERROR: Failed to set webhook"
              cat /tmp/curl.log
              exit 255
            fi

            oc label dc jenkins "app.kubernetes.io/part-of"="jenkins" --overwrite
            oc label dc nexus "app.kubernetes.io/part-of"="nexus" --overwrite
          image: image-registry.openshift-image-registry.svc:5000/openshift/cli:latest
          name: cicd-demo-installer-job
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        restartPolicy: Never

