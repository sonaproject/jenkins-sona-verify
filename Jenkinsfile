pipeline {
     agent { label 'master' }
     stages {
         stage ('Fetch-SONA') {
          steps {
                  notifyBuild('STARTED', params.NOTIFY_BUILD)
                  cleanWs()
                  sh 'ssh centos@${ONOS_IP} "sudo docker pull opensona/onos-sona-repo-build || true"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker stop onos-build || true"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker rm onos-build || true"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker run --rm -itd --name onos-build opensona/onos-sona-repo-build"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'mkdir -p /src/\'"'

                  sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && repo init -u https://github.com/sonaproject/onos-sona-repo.git -b \"${ONOS_VERSION}\"\'"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && repo sync\'"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'echo \"export ONOS_ROOT=/src\" > ~/.bash_profile\'"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'. ~/.bash_profile\'"'
              }
          }

          stage ('Pull-Review-Repo') {
              when {
                  expression {
                      return params.GERRIT_BUILD
                  }
              }
              steps {
                  sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && rm -rf 02-master-onos\'"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src && git clone https://gerrit.onosproject.org/onos 02-master-onos\'"'
                  sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'cd /src/02-master-onos && git review -d \'${REVIEW_NO}\' && git log -5\'"'
              }
          }

          stage ('Patch-ONOS') {
              steps {
                sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && cd /src && ./patch.sh \"${ONOS_VERSION}\" \'"'
              }
          }

          stage ('Build-SONA') {
              steps {
                sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && cd /src && ./build.sh\'"'
              }
          }

          stage ('Test-SONA') {
              steps {
                sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && cd /src && ./verify.sh\'"'
              }
          }

         stage ('Deploy-ONOS') {
             steps {
                 sh 'ssh centos@${ONOS_IP} "sudo docker pull opensona/onos-sona-nightly-docker:stable || true"'
                 sh 'ssh centos@${ONOS_IP} "sudo docker stop onos || true"'
                 sh 'ssh centos@${ONOS_IP} "sudo docker rm onos || true"'
                 sh 'ssh centos@${ONOS_IP} "sudo docker run --rm -itd --network host --name onos opensona/onos-sona-nightly-docker:stable"'
                 retry(10) {
                     sleep 30
                     sh 'curl --silent --show-error --fail --user onos:rocks -X GET --header \"Accept: application/json\" http://${ONOS_IP}:8181/onos/v1/mastership'
                 }
             }
         }

         stage ('Deploy-SONA') {
             steps {
               sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && /src/tools/package/runtime/bin/onos-app ${ONOS_IP} reinstall! /src/sona-out/openstacknode.oar\'"'
               sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && /src/tools/package/runtime/bin/onos-app ${ONOS_IP} reinstall! /src/sona-out/openstacknetworking.oar\'"'
               sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && /src/tools/package/runtime/bin/onos-app ${ONOS_IP} reinstall! /src/sona-out/openstacknetworking.oar\'"'
               sh 'ssh centos@${ONOS_IP} "sudo docker exec -i onos-build /bin/bash -c \'export ONOS_ROOT=/src && /src/tools/package/runtime/bin/onos-app ${ONOS_IP} reinstall! /src/sona-out/openstacknetworkingui.oar\'"'
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
           }
         }

         stage ('Configure-SONA') {
             steps {
               sh 'curl --silent --show-error --fail --user onos:rocks -X POST -H \"Content-Type: application/json\" http://${ONOS_IP}:8181/onos/openstacknode/configure -d @/var/jenkins_home/network-cfg.json'
               sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/sync/states'
               sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/sync/rules'
               sh 'curl --silent --show-error --fail --user onos:rocks -X GET http://${ONOS_IP}:8181/onos/openstacknetworking/management/config/securityGroup/enable'

               sh 'ssh centos@${ONOS_IP} "sudo docker stop onos-build || true"'
               sh 'ssh centos@${ONOS_IP} "sudo docker rm onos-build || true"'
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
               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --pattern network --skip-list sona-skip-list.yaml --detail --concurrency 1\'" | tee tempest_out.log'
               sh 'cat tempest_out.log | grep -c " Failures: 0" || if [ $? -ne 0 ]; then rm -rf tempest_out.log && ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify rerun --failed --detail --concurrency 1\'" | tee tempest_out.log; fi'
               sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'

               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify add-verifier-ext --source https://github.com/openstack/neutron-tempest-plugin.git'"'
               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --pattern neutron_tempest_plugin --skip-list sona-skip-list.yaml --detail --concurrency 1\'" | tee tempest_out.log'
               sh 'cat tempest_out.log | grep -c " Failures: 0" || if [ $? -ne 0 ]; then rm -rf tempest_out.log && ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify rerun --failed --detail --concurrency 1\'" | tee tempest_out.log; fi'
               sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'

               sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify delete-verifier-ext --name neutron_tests'"'
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
                sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --pattern network --skip-list sona-skip-list.yaml --detail --concurrency 1\'" | tee tempest_out.log'
                sh 'cat tempest_out.log | grep -c " Failures: 0" || if [ $? -ne 0 ]; then rm -rf tempest_out.log && ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify rerun --failed --detail --concurrency 1\'" | tee tempest_out.log; fi'
                sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'

                sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify add-verifier-ext --source https://github.com/openstack/neutron-tempest-plugin.git'"'
                sh 'ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify start --pattern neutron_tempest_plugin --skip-list sona-skip-list.yaml --detail --concurrency 1\'" | tee tempest_out.log'
                sh 'cat tempest_out.log | grep -c " Failures: 0" || if [ $? -ne 0 ]; then rm -rf tempest_out.log && ssh centos@${TEMPEST_IP} "sudo docker exec -i router /bin/bash -c \'source admin-openrc.sh && OS_AUTH_URL=${KEYSTONE_EP} && rally verify rerun --failed --detail --concurrency 1\'" | tee tempest_out.log; fi'
                sh 'cat tempest_out.log | grep -c " Failures: 0" || (EC=$?; exit $EC)'
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
             sh 'ssh centos@${ONOS_IP} "sudo docker stop onos"'
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
