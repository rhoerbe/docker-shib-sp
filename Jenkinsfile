// Stand-alone test (cleanup before and after; does not retain container or persistent volumes)
pipeline {
    agent any
    options { disableConcurrentBuilds() }
    parameters {
        string(defaultValue: '', description: 'Force "docker build --nocache" (blank or 1)', name: 'nocache')
        string(description: 'push docker image after build (blank or 1)', name: 'pushimage')
        string(description: 'overwrite default docker registry user', name: 'docker_registry_user')
        string(description: 'overwrite default docker registry host', name: 'docker_registry_host')
    }

    stages {
        stage('Cleanup container, volumes') {
            steps {
                sh '''
                    rm conf.sh 2> /dev/null || true
                    ln -sf conf.sh.default conf.sh
                    ./dscripts/manage.sh rm 2>/dev/null || true
                    ./dscripts/manage.sh rmvol 2>/dev/null || true
                '''
            }
        }
        stage('Build parent image') {
            steps {
                echo 'Job d-shibspbase: Building parent image ..'
                build job: 'd-shibspbase'
            }
        }
        stage('Build') {
            steps {
                echo "==========================="
                sh 'set +x; source ./conf.sh; echo "Building $IMAGENAME"'
                echo "Pipeline args: nocache=$nocache; pushimage=$pushimage; docker_registry_user=$docker_registry_user; docker_registry_host=$docker_registry_host"
                echo "==========================="
                sh '''
                    set +x
                    echo [[ "$docker_registry_user" ]] && echo "DOCKER_REGISTRY_USER $docker_registry_user"  > local.conf
                    echo [[ "$docker_registry_host" ]] && echo "DOCKER_REGISTRY_HOST $docker_registry_host"  >> local.conf
                    [[ "$pushimage" ]] && pushopt='-P'
                    [[ "$nocache" ]] && nocacheopt='-c'
                    ./dscripts/build.sh -p $nocacheopt $pushopt
                '''
                     sh '''
                    echo "generate run script"
                    ./dscripts/run.sh -w
                '''
       }
        }
        stage('Setup + Run') {
            steps {
                sh '''#!/bin/bash
                    echo "Setup persistent volumes unless already setup"
                    ./dscripts/manage.sh statcode
                    is_running=$?
                    ./dscripts/exec.sh -iR /opt/bin/is_initialized.sh
                    is_init=$?
                    if (( $is_init != 0 )); then
                        >&2 echo "setup test config"
                        ./dscripts/run.sh -iC 'cp /opt/install/config/express_setup_citest.yaml \
                                                  /opt/etc/express_setup_citest.yaml'
                        ./dscripts/run.sh -iC /opt/install/scripts/express_setup.sh -s express_setup_citest.yaml
                        >&2 echo "start server"
                        ./dscripts/run.sh
                        ./dscripts/manage.sh logs
                    else
                        >&2 echo 'skipping setup - already done'
                        if (( $is_running > 0 )); then
                            >&2 echo "start server"
                            ./dscripts/run.sh
                        fi
                    fi
                '''
            }
        }
        stage('Test') {
            steps {
                sh '''
                    sleep 1
                    ./dscripts/exec.sh -i /opt/install/tests/test_sp.sh
                '''
            }
        }
    }
    post {
        always {
            echo 'Remove container, volumes'
            sh '''
                ./dscripts/manage.sh rm 2>/dev/null || true
                ./dscripts/manage.sh rmvol 2>/dev/null || true
            '''
        }
    }
}