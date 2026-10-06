pipeline {

    agent {
        label 'DevOpsMaster'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(
            logRotator(
                numToKeepStr: '10'
            )
        )
        skipDefaultCheckout(true)
    }

    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    stages {

        // ==========================================================
        // 1. CHECKOUT
        // ==========================================================

        stage('Checkout') {
            steps {
                script {
                    try {

                        echo "=========================================="
                        echo "CHECKOUT"
                        echo "=========================================="

                        checkout scm

                        echo "Source code checkout completed."

                    } catch (Exception e) {

                        echo "Checkout failed."
                        echo "Error: ${e.getMessage()}"

                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }


        // ==========================================================
        // 2. CHECK JENKINS AGENT PREREQUISITES
        // ==========================================================

        stage('Check Jenkins Prerequisites') {
            steps {
                script {
                    try {

                        sh '''
                            set -e

                            echo "=========================================="
                            echo "JENKINS AGENT INFORMATION"
                            echo "=========================================="

                            echo "Hostname:"
                            hostname

                            echo ""
                            echo "IP Address:"
                            hostname -I

                            echo ""
                            echo "Operating System:"
                            cat /etc/os-release

                            echo ""
                            echo "=========================================="
                            echo "CHECKING PACKAGE MANAGER"
                            echo "=========================================="

                            if command -v apt-get; then
                                PACKAGE_MANAGER="apt"

                            elif command -v dnf; then
                                PACKAGE_MANAGER="dnf"

                            elif command -v yum; then
                                PACKAGE_MANAGER="yum"

                            elif command -v zypper; then
                                PACKAGE_MANAGER="zypper"

                            elif command -v apk; then
                                PACKAGE_MANAGER="apk"

                            else
                                echo "ERROR: No supported package manager found."
                                exit 1
                            fi

                            echo "Package manager: ${PACKAGE_MANAGER}"


                            # ==================================================
                            # PYTHON
                            # ==================================================

                            echo ""
                            echo "Checking Python3..."

                            if command -v python3; then
                                echo "Python3 already installed."
                            else
                                echo "Python3 not found. Installing..."

                                case "${PACKAGE_MANAGER}" in

                                    apt)
                                        sudo apt-get update
                                        sudo apt-get install -y python3
                                        ;;

                                    dnf)
                                        sudo dnf install -y python3
                                        ;;

                                    yum)
                                        sudo yum install -y python3
                                        ;;

                                    zypper)
                                        sudo zypper --non-interactive install python3
                                        ;;

                                    apk)
                                        sudo apk add python3
                                        ;;

                                esac
                            fi

                            python3 --version


                            # ==================================================
                            # PIP
                            # ==================================================

                            echo ""
                            echo "Checking pip3..."

                            if command -v pip3; then
                                echo "pip3 already installed."
                            else
                                echo "pip3 not found. Installing..."

                                case "${PACKAGE_MANAGER}" in

                                    apt)
                                        sudo apt-get install -y python3-pip
                                        ;;

                                    dnf)
                                        sudo dnf install -y python3-pip
                                        ;;

                                    yum)
                                        sudo yum install -y python3-pip
                                        ;;

                                    zypper)
                                        sudo zypper --non-interactive install python3-pip
                                        ;;

                                    apk)
                                        sudo apk add py3-pip
                                        ;;

                                esac
                            fi

                            pip3 --version


                            # ==================================================
                            # ANSIBLE
                            # ==================================================

                            echo ""
                            echo "Checking Ansible..."

                            if command -v ansible; then

                                echo "Ansible already installed."

                            else

                                echo "Ansible not found. Installing..."

                                python3 -m pip install --user ansible \
                                    --disable-pip-version-check

                            fi


                            # Add user-local binaries to current shell PATH.
                            export PATH="${HOME}/.local/bin:${PATH}"

                            if ! command -v ansible; then
                                echo "ERROR: Ansible installation failed."
                                exit 1
                            fi

                            ansible --version


                            # ==================================================
                            # ANSIBLE GALAXY
                            # ==================================================

                            echo ""
                            echo "Checking ansible-galaxy..."

                            if ! command -v ansible-galaxy; then
                                echo "ERROR: ansible-galaxy not available."
                                exit 1
                            fi

                            ansible-galaxy --version


                            # ==================================================
                            # DOCKER
                            # ==================================================

                            echo ""
                            echo "Checking Docker..."

                            if command -v docker; then

                                echo "Docker already installed."

                            else

                                echo "Docker not found. Installing..."

                                case "${PACKAGE_MANAGER}" in

                                    apt)
                                        sudo apt-get update
                                        sudo apt-get install -y docker.io
                                        ;;

                                    dnf)
                                        sudo dnf install -y docker
                                        ;;

                                    yum)
                                        sudo yum install -y docker
                                        ;;

                                    zypper)
                                        sudo zypper --non-interactive install docker
                                        ;;

                                    apk)
                                        sudo apk add docker
                                        ;;

                                esac

                            fi


                            echo ""
                            echo "Starting Docker..."

                            sudo systemctl enable --now docker


                            if ! docker info >/dev/null 2>&1; then

                                echo "Docker command exists but current user cannot access Docker."

                                sudo usermod -aG docker "$(whoami)"

                                echo "Docker access requires refreshed group membership."
                                echo "The pipeline will use sudo docker where required."

                            fi

                            sudo docker --version


                            # ==================================================
                            # PYTHON DOCKER SDK
                            # ==================================================

                            echo ""
                            echo "Checking Python Docker SDK..."

                            python3 -c "import docker" 2>/dev/null || {

                                echo "Python Docker SDK not found."
                                echo "Installing Docker SDK..."

                                python3 -m pip install --user docker \
                                    --disable-pip-version-check
                            }

                            python3 -c "import docker"

                            echo "Python Docker SDK is available."


                            # ==================================================
                            # ANSIBLE COLLECTION
                            # ==================================================

                            echo ""
                            echo "Installing Ansible collections..."

                            if [ -f requirements.yml ]; then

                                ansible-galaxy collection install \
                                    -r requirements.yml

                            else

                                echo "requirements.yml not found."

                                echo "Installing community.docker directly..."

                                ansible-galaxy collection install \
                                    community.docker

                            fi


                            echo ""
                            echo "=========================================="
                            echo "JENKINS AGENT PREREQUISITES READY"
                            echo "=========================================="
                        '''

                    } catch (Exception e) {

                        echo "Jenkins agent prerequisite installation failed."
                        echo "Error: ${e.getMessage()}"

                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }


        // ==========================================================
        // 3. CHECK WHETHER JENKINS AGENT AND ANSIBLE TARGET
        //    ARE THE SAME MACHINE
        // ==========================================================

        stage('Check Jenkins Agent and Ansible Target') {
            steps {
                script {
                    try {

                        sh '''
                            set -e

                            echo "=========================================="
                            echo "CHECKING JENKINS AND ANSIBLE TARGET"
                            echo "=========================================="

                            # Jenkins agent IP
                            JENKINS_IP=$(hostname -I | awk '{print $1}')

                            echo "Jenkins Agent IP : ${JENKINS_IP}"


                            # Get first host from webservers group
                            ANSIBLE_TARGET=$(ansible-inventory \
                                -i inventory/hosts.yml \
                                --list \
                                | python3 -c '
import sys
import json

data = json.load(sys.stdin)

hosts = data.get("webservers", {}).get("hosts", [])

if not hosts:
    print("ERROR: No hosts found in webservers group", file=sys.stderr)
    sys.exit(1)

print(hosts[0])
                            ')

                            echo "Ansible Inventory Host: ${ANSIBLE_TARGET}"


                            # Get ansible_host from inventory
                            ANSIBLE_IP=$(ansible-inventory \
                                -i inventory/hosts.yml \
                                --host "${ANSIBLE_TARGET}" \
                                | python3 -c '
import sys
import json

data = json.load(sys.stdin)

ip = data.get("ansible_host", "")

if not ip:
    print("ERROR: ansible_host is not defined", file=sys.stderr)
    sys.exit(1)

print(ip)
                            ')

                            echo "Ansible Target IP: ${ANSIBLE_IP}"


                            # Save values for later stages
                            echo "ANSIBLE_TARGET=${ANSIBLE_TARGET}" > target-info.env
                            echo "ANSIBLE_IP=${ANSIBLE_IP}" >> target-info.env


                            # ==================================================
                            # COMPARE IP ADDRESSES
                            # ==================================================

                            if [ "${JENKINS_IP}" = "${ANSIBLE_IP}" ]; then

                                echo ""
                                echo "=========================================="
                                echo "SAME MACHINE"
                                echo "=========================================="

                                echo "Jenkins Agent IP : ${JENKINS_IP}"
                                echo "Ansible Target IP: ${ANSIBLE_IP}"

                                echo ""
                                echo "Jenkins agent and Ansible target"
                                echo "are the same machine."

                                echo ""
                                echo "SSH will be SKIPPED."

                                echo "SAME_MACHINE=true" >> target-info.env

                            else

                                echo ""
                                echo "=========================================="
                                echo "DIFFERENT MACHINES"
                                echo "=========================================="

                                echo "Jenkins Agent IP : ${JENKINS_IP}"
                                echo "Ansible Target IP: ${ANSIBLE_IP}"

                                echo ""
                                echo "Jenkins agent and Ansible target"
                                echo "are different machines."

                                echo ""
                                echo "SSH will be REQUIRED."

                                echo "SAME_MACHINE=false" >> target-info.env

                            fi
                        '''

                        // Read values for subsequent stages
                        def targetInfo = readFile('target-info.env').readLines()

                        targetInfo.each { line ->

                            def parts = line.split('=', 2)

                            if (parts.size() == 2) {
                                env."${parts[0]}" = parts[1]
                            }
                        }

                        echo "SAME_MACHINE  = ${env.SAME_MACHINE}"
                        echo "ANSIBLE_TARGET = ${env.ANSIBLE_TARGET}"
                        echo "ANSIBLE_IP     = ${env.ANSIBLE_IP}"

                    } catch (Exception e) {

                        echo "Failed to determine whether Jenkins and Ansible target are the same."

                        echo "Error: ${e.getMessage()}"

                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }


        // ==========================================================
        // 4. PREPARE ANSIBLE TARGET
        // ==========================================================

        stage('Prepare Ansible Target') {
            steps {
                script {

                    try {

                        if (env.SAME_MACHINE == 'true') {

                            // ==================================================
                            // SAME MACHINE
                            // ==================================================

                            echo "=========================================="
                            echo "LOCAL ANSIBLE TARGET"
                            echo "=========================================="

                            echo "Target IP: ${env.ANSIBLE_IP}"
                            echo "No SSH required."

                            sh '''
                                set -e

                                echo "=========================================="
                                echo "PREPARING LOCAL ANSIBLE TARGET"
                                echo "=========================================="

                                echo "Hostname:"
                                hostname

                                echo ""
                                echo "IP:"
                                hostname -I


                                # ==================================================
                                # PYTHON
                                # ==================================================

                                echo ""
                                echo "Checking Python3..."

                                if ! command -v python3; then

                                    echo "Python3 not found."

                                    if command -v apt-get; then
                                        sudo apt-get update
                                        sudo apt-get install -y python3

                                    elif command -v dnf; then
                                        sudo dnf install -y python3

                                    elif command -v yum; then
                                        sudo yum install -y python3

                                    else
                                        echo "ERROR: Unsupported package manager."
                                        exit 1
                                    fi

                                else

                                    echo "Python3 already installed."

                                fi

                                python3 --version


                                # ==================================================
                                # PIP
                                # ==================================================

                                echo ""
                                echo "Checking pip3..."

                                if ! command -v pip3; then

                                    echo "pip3 not found."

                                    if command -v apt-get; then
                                        sudo apt-get install -y python3-pip

                                    elif command -v dnf; then
                                        sudo dnf install -y python3-pip

                                    elif command -v yum; then
                                        sudo yum install -y python3-pip

                                    else
                                        echo "ERROR: Unable to install pip3."
                                        exit 1
                                    fi

                                else

                                    echo "pip3 already installed."

                                fi

                                pip3 --version


                                # ==================================================
                                # DOCKER
                                # ==================================================

                                echo ""
                                echo "Checking Docker..."

                                if ! command -v docker; then

                                    echo "Docker not found."

                                    if command -v apt-get; then

                                        sudo apt-get update
                                        sudo apt-get install -y docker.io

                                    elif command -v dnf; then

                                        sudo dnf install -y docker

                                    elif command -v yum; then

                                        sudo yum install -y docker

                                    else

                                        echo "ERROR: Unable to install Docker."
                                        exit 1

                                    fi

                                else

                                    echo "Docker already installed."

                                fi


                                echo ""
                                echo "Starting Docker..."

                                sudo systemctl enable --now docker

                                sudo docker --version


                                # ==================================================
                                # PYTHON DOCKER SDK
                                # ==================================================

                                echo ""
                                echo "Checking Python Docker SDK..."

                                python3 -c "import docker" 2>/dev/null || {

                                    echo "Docker Python SDK not found."
                                    echo "Installing..."

                                    python3 -m pip install --user docker \
                                        --disable-pip-version-check
                                }

                                python3 -c "import docker"

                                echo "Docker Python SDK available."


                                echo ""
                                echo "=========================================="
                                echo "LOCAL TARGET READY"
                                echo "=========================================="
                            '''


                        } else {

                            // ==================================================
                            // REMOTE MACHINE
                            // ==================================================

                            echo "=========================================="
                            echo "REMOTE ANSIBLE TARGET"
                            echo "=========================================="

                            echo "Target IP: ${env.ANSIBLE_IP}"
                            echo "SSH required."

                            sshagent(['DevOpsMaster']) {

                                sh '''
                                    set -e

                                    echo "=========================================="
                                    echo "CHECKING SSH CONNECTIVITY"
                                    echo "=========================================="

                                    ssh \
                                        -o StrictHostKeyChecking=no \
                                        -o ConnectTimeout=10 \
                                        "${ANSIBLE_IP}" \
                                        "hostname"

                                    echo ""
                                    echo "SSH connectivity successful."


                                    # ==================================================
                                    # TARGET PREREQUISITES
                                    # ==================================================

                                    ssh \
                                        -o StrictHostKeyChecking=no \
                                        -o ConnectTimeout=10 \
                                        "${ANSIBLE_IP}" \
                                        '
                                            set -e

                                            echo "=========================================="
                                            echo "PREPARING REMOTE ANSIBLE TARGET"
                                            echo "=========================================="

                                            echo "Hostname:"
                                            hostname

                                            echo ""
                                            echo "IP:"
                                            hostname -I


                                            # ==========================================
                                            # PYTHON
                                            # ==========================================

                                            echo ""
                                            echo "Checking Python3..."

                                            if ! command -v python3; then

                                                echo "Python3 not found. Installing..."

                                                if command -v apt-get; then

                                                    sudo apt-get update
                                                    sudo apt-get install -y python3

                                                elif command -v dnf; then

                                                    sudo dnf install -y python3

                                                elif command -v yum; then

                                                    sudo yum install -y python3

                                                else

                                                    echo "ERROR: Unsupported package manager."
                                                    exit 1

                                                fi

                                            else

                                                echo "Python3 already installed."

                                            fi

                                            python3 --version


                                            # ==========================================
                                            # PIP
                                            # ==========================================

                                            echo ""
                                            echo "Checking pip3..."

                                            if ! command -v pip3; then

                                                echo "pip3 not found. Installing..."

                                                if command -v apt-get; then

                                                    sudo apt-get install -y python3-pip

                                                elif command -v dnf; then

                                                    sudo dnf install -y python3-pip

                                                elif command -v yum; then

                                                    sudo yum install -y python3-pip

                                                else

                                                    echo "ERROR: Unable to install pip3."
                                                    exit 1

                                                fi

                                            else

                                                echo "pip3 already installed."

                                            fi

                                            pip3 --version


                                            # ==========================================
                                            # DOCKER
                                            # ==========================================

                                            echo ""
                                            echo "Checking Docker..."

                                            if ! command -v docker; then

                                                echo "Docker not found. Installing..."

                                                if command -v apt-get; then

                                                    sudo apt-get update
                                                    sudo apt-get install -y docker.io

                                                elif command -v dnf; then

                                                    sudo dnf install -y docker

                                                elif command -v yum; then

                                                    sudo yum install -y docker

                                                else

                                                    echo "ERROR: Unable to install Docker."
                                                    exit 1

                                                fi

                                            else

                                                echo "Docker already installed."

                                            fi


                                            echo ""
                                            echo "Starting Docker..."

                                            sudo systemctl enable --now docker

                                            sudo docker --version


                                            # ==========================================
                                            # PYTHON DOCKER SDK
                                            # ==========================================

                                            echo ""
                                            echo "Checking Python Docker SDK..."

                                            python3 -c "import docker" 2>/dev/null || {

                                                echo "Docker Python SDK not found."
                                                echo "Installing..."

                                                python3 -m pip install --user docker \
                                                    --disable-pip-version-check
                                            }

                                            python3 -c "import docker"

                                            echo "Docker Python SDK available."


                                            echo ""
                                            echo "=========================================="
                                            echo "REMOTE TARGET READY"
                                            echo "=========================================="
                                        '
                                '''
                            }
                        }

                    } catch (Exception e) {

                        echo "Ansible target preparation failed."
                        echo "Error: ${e.getMessage()}"

                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }


        // ==========================================================
        // 5. VALIDATE INVENTORY
        // ==========================================================

        stage('Validate Ansible Inventory') {
            steps {
                script {
                    try {

                        sh '''
                            set -e

                            echo "=========================================="
                            echo "VALIDATING ANSIBLE INVENTORY"
                            echo "=========================================="

                            ansible-inventory \
                                -i inventory/hosts.yml \
                                --list

                            echo ""
                            echo "Inventory validation successful."
                        '''

                    } catch (Exception e) {

                        echo "Ansible inventory validation failed."
                        echo "Error: ${e.getMessage()}"

                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }


        // ==========================================================
        // 6. ANSIBLE SYNTAX CHECK
        // ==========================================================

        stage('Ansible Syntax Check') {
            steps {
                script {
                    try {

                        sh '''
                            set -e

                            echo "=========================================="
                            echo "ANSIBLE SYNTAX CHECK"
                            echo "=========================================="

                            ansible-playbook \
                                -i inventory/hosts.yml \
                                deploy_nginx_container.yml \
                                --syntax-check

                            echo ""
                            echo "Ansible syntax check successful."
                        '''

                    } catch (Exception e) {

                        echo "Ansible syntax check failed."
                        echo "Error: ${e.getMessage()}"

                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }


        // ==========================================================
        // 7. DEPLOY NGINX
        // ==========================================================

        stage('Deploy Nginx') {
            steps {
                script {

                    try {

                        if (env.SAME_MACHINE == 'true') {

                            // ==================================================
                            // LOCAL ANSIBLE
                            // ==================================================

                            echo "=========================================="
                            echo "DEPLOY NGINX LOCALLY"
                            echo "=========================================="

                            echo "Jenkins agent = Ansible target."
                            echo "SSH will be skipped."

                            sh '''
                                set -e

                                ansible-playbook \
                                    -i inventory/hosts.yml \
                                    deploy_nginx_container.yml \
                                    -e ansible_connection=local
                            '''

                        } else {

                            // ==================================================
                            // REMOTE ANSIBLE
                            // ==================================================

                            echo "=========================================="
                            echo "DEPLOY NGINX REMOTELY"
                            echo "=========================================="

                            echo "Jenkins agent != Ansible target."
                            echo "Using SSH."

                            sshagent(['DevOpsMaster']) {

                                sh '''
                                    set -e

                                    ansible-playbook \
                                        -i inventory/hosts.yml \
                                        deploy_nginx_container.yml
                                '''
                            }
                        }

                    } catch (Exception e) {

                        echo "Nginx deployment failed."
                        echo "Error: ${e.getMessage()}"

                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }
    }


    // ==========================================================
    // PIPELINE POST ACTIONS
    // ==========================================================

    post {

        success {
            echo "=========================================="
            echo "PIPELINE SUCCESSFUL"
            echo "=========================================="
            echo "Nginx deployment completed successfully."
        }

        failure {
            echo "=========================================="
            echo "PIPELINE FAILED"
            echo "=========================================="
            echo "Check the stage logs for the failure."
        }

        aborted {
            echo "=========================================="
            echo "PIPELINE ABORTED"
            echo "=========================================="
        }

        always {
            echo "=========================================="
            echo "PIPELINE COMPLETED"
            echo "=========================================="

            sh '''
                rm -f target-info.env || true
            '''
        }
    }
}
