pipeline {
     agent { label 'master' }
     stages {
         stage ('Fetch-SONA') {
          steps {
                  notifyBuild('STARTED', params.NOTIFY_BUILD)
                  cleanWs()
                  sh 'ssh centos@${BUILD_IP} "sudo docker pull opensona/onos-sona-repo-build || true"'
                  sh 'ssh centos@${BUILD_IP} "sudo docker stop onos-build || true"'
                  sh 'ssh centos@${BUILD_IP} "sudo docker rm onos-build || true"'

                  sleep 10

                  sh 'ssh centos@${BUILD_IP} "sudo docker run --rm -itd --name onos-build -v root_home:/root opensona/onos-sona-repo-build"'
                  sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'mkdir -p /src/\'"'

                  sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && repo init -u https://github.com/sonaproject/onos-sona-repo.git -b \"${ONOS_VERSION}\"\'"'
                  sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && repo sync\'"'
                  sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'echo \"export ONOS_ROOT=/src\" > ~/.bash_profile\'"'
                  sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'. ~/.bash_profile\'"'
              }
          }

          stage ('Pull-Review-Repo') {
              when {
                  expression {
                      return params.GERRIT_BUILD
                  }
              }
              steps {
                  sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && rm -rf 02-master-onos\'"'
                  sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && git clone https://gerrit.onosproject.org/onos 02-master-onos\'"'
                  sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src/02-master-onos && git review -d \'${REVIEW_NO}\' && git log -5\'"'
              }
          }

          stage ('Patch-ONOS') {
              steps {
                sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && cd /src && ./patch.sh \"${ONOS_VERSION}\" \'"'
              }
          }

          stage ('Build-SONA') {
              steps {
                sh 'ssh centos@${BUILD_IP} "sudo docker stop onos-deps-repo || true"'
                sleep 10
                sh 'ssh centos@${BUILD_IP} "sudo docker pull opensona/onos-deps-repo || true"'
                sh 'ssh centos@${BUILD_IP} "sudo docker run --rm -d -p 8080:80 --name onos-deps-repo opensona/onos-deps-repo"'
                sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && sed -i \"s/https/http/g\" ./bucklets/node.bucklet\'"'
                sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && sed -i \"s/nodejs.org/${BUILD_IP}:8080/g\" ./bucklets/node.bucklet\'"'
                sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && cd /src && ./build.sh\'"'
              }
          }

          stage ('Test-SONA') {
              steps {
                sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && cd /src && ./verify.sh\'"'
              }
          }

         stage ('Deploy-ONOS') {
             steps {
               script {
                 sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'git clone -b stable https://github.com/sonaproject/onos-docker-tool.git\'"'
                 sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'mkdir -p onos-docker-tool/site/sona && echo \"export ODC1=${ONOS_IP}\" > onos-docker-tool/site/sona/cell\'"'

                 if (env.ONOS2_IP) {
                   sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'mkdir -p onos-docker-tool/site/sona && echo \"export ODC2=${ONOS2_IP}\" >> onos-docker-tool/site/sona/cell\'"'
                 }

                 sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd onos-docker-tool && source bash_profile && onos-docker-site sona && ./stop.sh\'"'
                 sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd onos-docker-tool && source bash_profile && onos-docker-site sona && ./start.sh\'"'
                 retry(10) {
                     sleep 15
                     sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/floatingips/all'
                     sleep 15
                 }
               }
             }
         }

         stage ('Deploy-SONA') {
             steps {
               sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && /src/tools/package/runtime/bin/onos-app ${ONOS_IP} reinstall! /src/sona-out/openstacknode.oar\'"'
               sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && /src/tools/package/runtime/bin/onos-app ${ONOS_IP} reinstall! /src/sona-out/openstacknetworking.oar\'"'
               sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && /src/tools/package/runtime/bin/onos-app ${ONOS_IP} reinstall! /src/sona-out/openstacknetworking.oar\'"'
               sh 'ssh centos@${BUILD_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && /src/tools/package/runtime/bin/onos-app ${ONOS_IP} reinstall! /src/sona-out/openstacknetworkingui.oar\'"'
             }
         }
         stage ('Prepare-Tempest') {
             steps {
               sh 'git clone https://github.com/sonaproject/tempest-sona-conf.git'
               sh 'ssh centos@${TEMPEST_IP} "sudo rm -rf /var/lib/rally_container || true"'
               sh 'ssh centos@${TEMPEST_IP} "sudo mkdir -p /var/lib/rally_container || true"'
               sh 'ssh centos@${TEMPEST_IP} "sudo chown 65500 /var/lib/rally_container"'
               sh 'ssh centos@${TEMPEST_IP} "sudo rm -rf /home/centos/tempest-sona-conf || true"'
               sh 'ssh centos@${TEMPEST_IP} "mkdir -p /home/centos/tempest-sona-conf || true"'
               sh 'scp tempest-sona-conf/* centos@${TEMPEST_IP}:/home/centos/tempest-sona-conf'
               sh 'ssh centos@${TEMPEST_IP} "sudo cp /home/centos/tempest-sona-conf/* /var/lib/rally_container"'

               sh 'ssh centos@${TEMPEST_IP} "cd ~/sona-setup-tempest && ./createExternalRouter.sh"'
               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'mkdir -p data\'"'
               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'rally db recreate\'"'
               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally deployment create --fromenv --name sona-test\'"'
               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify create-verifier --type tempest --name tempest-verifier --source https://github.com/sonaproject/tempest.git --version ${OPENSTACK_VERSION}\'"'
               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify configure-verifier --extend sona-tempest.conf\'"'
               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify add-verifier-ext --source https://github.com/openstack/neutron-tempest-plugin.git\'"'
           }
         }

         stage ('Configure-SONA') {
             steps {
               sh 'curl --silent --show-error --fail --user onos:rocks -X POST -H \"Content-Type: application/json\" http://${ONOS_IP}:8181/onos/openstacknode/configure -d @/var/jenkins_home/network-cfg.json'
               sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/sync/states'
               sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/sync/rules'
               sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/config/securityGroup/enable'

               sh 'ssh centos@${BUILD_IP} "sudo docker stop onos-build || true"'
               sh 'ssh centos@${BUILD_IP} "sudo docker rm onos-build || true"'
             }
         }

         stage ('Verify-Broadcast-Mode') {
             when {
                 expression {
                     return params.ARP_MODE != "proxy"
                 }
             }
             steps {
               sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/config/arpmode/broadcast'

               script {
                 if (env.VERIFY_TARGET != "scenario") {
                   sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --pattern tempest.api.network --skip-list sona-skip-list.yaml --detail --concurrency ${CONCURRENCY}\'" | tee tempest_out.log'
                   sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'

                   sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --pattern neutron_tempest_plugin.api --skip-list sona-skip-list.yaml --detail --concurrency ${CONCURRENCY}\'" | tee tempest_out.log'
                   sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'
                 }

                 if (env.VERIFY_TARGET != "api") {
                   sleep 60
                   sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --load-list sona-load-list.yaml --detail --concurrency ${CONCURRENCY}\'" | tee tempest_out.log'
                   sh 'cat tempest_out.log | grep -c " Failures: 0" || if [ $? -ne 0 ]; then rm -rf tempest_out.log && ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify rerun --failed --detail --concurrency ${CONCURRENCY}\'" | tee tempest_out.log; fi'
                   sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'
                 }
               }
             }
         }

         stage ('Verify-Proxy-Mode') {
              when {
                 expression {
                      return params.ARP_MODE != "broadcast"
                 }
              }
              steps {
                sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/config/arpmode/proxy'

                script {
                  if (env.VERIFY_TARGET != "scenario") {
                    sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --pattern tempest.api.network --skip-list sona-skip-list.yaml --detail --concurrency ${CONCURRENCY}\'" | tee tempest_out.log'
                    sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'

                    sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --pattern neutron_tempest_plugin.api --skip-list sona-skip-list.yaml --detail --concurrency ${CONCURRENCY}\'" | tee tempest_out.log'
                    sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'
                  }

                  if (env.VERIFY_TARGET != "api") {
                    sleep 60
                    sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --load-list sona-load-list.yaml --detail --concurrency ${CONCURRENCY}\'" | tee tempest_out.log'
                    sh 'cat tempest_out.log | grep -c " Failures: 0" || if [ $? -ne 0 ]; then rm -rf tempest_out.log && ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify rerun --failed --detail --concurrency ${CONCURRENCY}\'" | tee tempest_out.log; fi'
                    sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'
                  }
                }
              }
         }

         stage ('Deliver-ONOS-SONA') {
             when {
                  expression {
                      return params.DOCKER_DELIVERY
                  }
              }
              steps {
                  sh 'test $(cat tempest_out.log | grep -c " Failures: 0") -eq 1 && curl -H "Content-Type: application/json" --data \'{"source_type": "Branch", "source_name": "master"}\' -X POST ${TRIGGER_URL}'

                  sleep 60

                  sh 'test $(cat tempest_out.log | grep -c " Failures: 0") -eq 1 && curl -H "Content-Type: application/json" --data \'{"source_type": "Branch", "source_name": "stable"}\' -X POST ${TRIGGER_URL}'

                  sleep 60

                  sh 'test $(cat tempest_out.log | grep -c " Failures: 0") -eq 1 && curl -H "Content-Type: application/json" --data \'{"source_type": "Branch", "source_name": "dev"}\' -X POST ${TRIGGER_URL}'
              }
         }
     }
     post {
         always {
             sh 'ssh centos@${TEMPEST_IP} "sudo docker stop router"'
             sh 'ssh centos@${TEMPEST_IP} "sudo rm -rf /home/centos/tempest-sona-conf"'
             sh 'ssh centos@${TEMPEST_IP} "sudo rm -rf /var/lib/rally_container"'
             cleanWs()
             notifyBuild(currentBuild.result, params.NOTIFY_BUILD)
         }
     }
}

// notify build status to third-party application like email and slack
def notifyBuild(String buildStatus = 'STARTED', Boolean sendFlag = false) {

    if (sendFlag == true) {
        // build status of null means successful
        buildStatus =  buildStatus ?: 'SUCCESSFUL'

        // Default values
        def colorName = 'RED'
        def colorCode = '#FF0000'
        def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
        def summary = "${subject} (${env.BUILD_URL})"
        def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""

        // Override default values based on build status
        if (buildStatus == 'STARTED') {
            color = 'YELLOW'
            colorCode = '#FFFF00'
        } else if (buildStatus == 'SUCCESSFUL') {
            color = 'GREEN'
            colorCode = '#00FF00'
        } else {
            color = 'RED'
            colorCode = '#FF0000'
        }

        // Send notifications
        slackSend (color: colorCode, message: summary)
    }
}
