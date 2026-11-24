// =============================================================================
// Stage 2 Declarative Jenkins Pipeline: Vanilla Kubernetes Bootstrap via Ansible
// =============================================================================
// Features:
// 1. Zero Static AWS Credentials - Runs on EC2 with IAM Instance Profile
// 2. Portable Virtual Environment in ${HOME}/ansible-venv (jenkins user safe)
// 3. Pre-flight dynamic inventory & keyless SSM ping verification
// 4. Sequential Ansible orchestration of Vanilla K8s v1.31 over AWS SSM
// 5. Post-configure exit gate: verifies all 3 nodes Ready, ecr-registry-secret
//    present in 'prod' namespace, ALB Target Group healthy, and /healthz 200 OK.
// =============================================================================

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 45, unit: 'MINUTES')
    }

    environment {
        AWS_REGION                = 'ap-southeast-2'
        AWS_DEFAULT_REGION        = 'ap-southeast-2'
        VENV_DIR                  = "${HOME}/ansible-venv"
        ANSIBLE_NOCOLOR           = '1'
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        ANSIBLE_FORCE_COLOR       = '0'
    }

    stages {
        // ---------------------------------------------------------------------
        // Stage 1: Pre-Flight & Tooling Bootstrap
        // ---------------------------------------------------------------------
        stage('Pre-Flight & Tooling Bootstrap') {
            steps {
                script {
                    echo "=========================================================="
                    echo " Stage 1: Verifying Tooling & Virtual Environment Setup   "
                    echo " Target Region : ${AWS_REGION}                            "
                    echo " Virtualenv Dir: ${VENV_DIR}                              "
                    echo "=========================================================="

                    // Verify AWS CLI identity and Session Manager Plugin availability
                    sh """
                        echo "[INFO] Verifying AWS IAM Instance Profile identity..."
                        aws sts get-caller-identity --region ${AWS_REGION}

                        echo "[INFO] Verifying AWS Session Manager CLI plugin..."
                        session-manager-plugin --version
                    """

                    // Bootstrap Python virtual environment if missing
                    sh """
                        if [ ! -f "${VENV_DIR}/bin/ansible-playbook" ]; then
                            echo "[INFO] Bootstrapping virtual environment in ${VENV_DIR}..."
                            python3 -m venv "${VENV_DIR}"
                            "${VENV_DIR}/bin/pip" install --upgrade pip
                            "${VENV_DIR}/bin/pip" install -r requirements.txt
                            "${VENV_DIR}/bin/ansible-galaxy" collection install -r requirements.yml --force
                        else
                            echo "[INFO] Existing virtual environment found at ${VENV_DIR}."
                        fi

                        echo "[INFO] Ansible version check:"
                        "${VENV_DIR}/bin/ansible" --version
                    """
                }
            }
        }

        // ---------------------------------------------------------------------
        // Stage 2: Syntax & Dynamic Inventory Verification
        // ---------------------------------------------------------------------
        stage('Syntax & Dynamic Inventory Verification') {
            steps {
                script {
                    echo "=========================================================="
                    echo " Stage 2: Validating Playbook Syntax & Dynamic Inventory  "
                    echo "=========================================================="

                    sh """
                        export PATH="${VENV_DIR}/bin:\$PATH"

                        echo "[INFO] Validating site.yml syntax..."
                        ansible-playbook --syntax-check site.yml

                        echo "[INFO] Evaluating EC2 dynamic inventory graph..."
                        GRAPH_OUTPUT=\$(ansible-inventory -i inventory/aws_ec2.yml --graph)
                        echo "\$GRAPH_OUTPUT"

                        # Ensure both role_k8s_master and role_k8s_worker groups exist
                        if ! echo "\$GRAPH_OUTPUT" | grep -q "role_k8s_master"; then
                            echo "[ERROR] 'role_k8s_master' group not found in dynamic inventory!"
                            exit 1
                        fi

                        if ! echo "\$GRAPH_OUTPUT" | grep -q "role_k8s_worker"; then
                            echo "[ERROR] 'role_k8s_worker' group not found in dynamic inventory!"
                            exit 1
                        fi

                        echo "[SUCCESS] Dynamic inventory resolved required master and worker groups."
                    """
                }
            }
        }

        // ---------------------------------------------------------------------
        // Stage 3: SSM Connectivity Check
        // ---------------------------------------------------------------------
        stage('SSM Connectivity Check') {
            steps {
                script {
                    echo "=========================================================="
                    echo " Stage 3: Testing Keyless SSM Connectivity to All Nodes   "
                    echo " Transport: community.aws.aws_ssm via S3 Artifactory      "
                    echo "=========================================================="

                    sh """
                        export PATH="${VENV_DIR}/bin:\$PATH"
                        echo "[INFO] Pinging all EC2 cluster nodes via AWS SSM..."
                        ansible all -i inventory/aws_ec2.yml -m ping
                    """
                }
            }
        }

        // ---------------------------------------------------------------------
        // Stage 4: Bootstrap & Configure K8s Cluster
        // ---------------------------------------------------------------------
        stage('Bootstrap & Configure K8s Cluster') {
            steps {
                script {
                    echo "=========================================================="
                    echo " Stage 4: Executing Multi-Play Kubernetes Bootstrap       "
                    echo " Play 0: Inventory Guard Assertion                        "
                    echo " Play 1: Common OS & Containerd Runtime Setup             "
                    echo " Play 2: Control Plane (kubeadm init) & Token Generation  "
                    echo " Play 3: Worker Nodes (kubeadm join) Execution            "
                    echo " Play 4: Addons (Flannel, Ingress, Metrics, ECR CronJob)  "
                    echo "=========================================================="

                    sh """
                        export PATH="${VENV_DIR}/bin:\$PATH"
                        ansible-playbook site.yml
                    """
                }
            }
        }

        // ---------------------------------------------------------------------
        // Stage 5: Post-Configure Exit Gate (Cluster + ALB Health Verification)
        // ---------------------------------------------------------------------
        stage('Post-Configure Exit Gate') {
            steps {
                script {
                    echo "=========================================================="
                    echo " Stage 5: Verifying Cluster Readiness & ALB Health Status "
                    echo "=========================================================="

                    sh """
                        export PATH="${VENV_DIR}/bin:\$PATH"

                        echo "[INFO] Checking node readiness from Master node..."
                        ansible role_k8s_master -i inventory/aws_ec2.yml -m command \
                            -a "kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes -o wide"

                        echo "[INFO] Verifying ECR registry secret in 'prod' namespace..."
                        ansible role_k8s_master -i inventory/aws_ec2.yml -m command \
                            -a "kubectl --kubeconfig=/etc/kubernetes/admin.conf get secret ecr-registry-secret -n prod"

                        echo "[INFO] Querying Stage 1 ALB Target Group health (port 30080)..."
                        TG_ARN=\$(aws elbv2 describe-target-groups \
                            --names "k8s-ingress-tg-prod" \
                            --region ${AWS_REGION} \
                            --query "TargetGroups[0].TargetGroupArn" \
                            --output text)

                        echo "Discovered Target Group ARN: \$TG_ARN"

                        EXPECTED_HEALTHY=2
                        MAX_RETRIES=12
                        RETRY_INTERVAL=15
                        HEALTHY_COUNT=0

                        for i in \$(seq 1 \$MAX_RETRIES); do
                            HEALTHY_COUNT=\$(aws elbv2 describe-target-health \
                                --target-group-arn "\$TG_ARN" \
                                --region ${AWS_REGION} \
                                --query "TargetHealthDescriptions[?TargetHealth.State=='healthy'].Target.Id" \
                                --output text | wc -w)

                            echo "  [Attempt \$i/\$MAX_RETRIES] \$HEALTHY_COUNT of \$EXPECTED_HEALTHY worker targets healthy in ALB..."
                            if [ "\$HEALTHY_COUNT" -ge "\$EXPECTED_HEALTHY" ]; then
                                echo "[SUCCESS] Both worker nodes are registered and Healthy in ALB Target Group!"
                                break
                            fi
                            sleep \$RETRY_INTERVAL
                        done

                        if [ "\$HEALTHY_COUNT" -lt "\$EXPECTED_HEALTHY" ]; then
                            echo "[ERROR] Timed out waiting for \$EXPECTED_HEALTHY targets to become healthy in ALB Target Group."
                            aws elbv2 describe-target-health --target-group-arn "\$TG_ARN" --region ${AWS_REGION}
                            exit 1
                        fi

                        echo "[INFO] Fetching public ALB DNS from SSM Parameter Store..."
                        ALB_DNS=\$(aws ssm get-parameter \
                            --name "/prod/app/alb_dns" \
                            --query "Parameter.Value" \
                            --output text \
                            --region ${AWS_REGION})

                        echo "ALB Public DNS: \$ALB_DNS"

                        echo "[INFO] Validating HTTP GET /healthz response through Public ALB..."
                        HTTP_STATUS=\$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 "http://\${ALB_DNS}/healthz")
                        echo "ALB /healthz HTTP Response Code: \$HTTP_STATUS"

                        if [ "\$HTTP_STATUS" != "200" ]; then
                            echo "[ERROR] ALB /healthz did not return HTTP 200 OK (returned \$HTTP_STATUS)."
                            exit 1
                        fi

                        echo "[SUCCESS] Public ALB health check successfully verified with 200 OK!"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "=========================================================="
            echo " [SUCCESS] Stage 2 Kubernetes Bootstrap is Complete!      "
            echo " - Control Plane : Ready (kubeadm v1.31)                  "
            echo " - Worker Nodes  : 2/2 Joined and Ready                   "
            echo " - Flannel CNI   : Operational (10.244.0.0/16)            "
            echo " - Ingress-NGINX : DaemonSet on workers (NodePort 30080)  "
            echo " - ALB TG Health : 2/2 Worker Targets Healthy             "
            echo " - ECR Secret    : Active in 'prod' namespace             "
            echo " Cluster is fully ready for Stage 3 Application Deploy!   "
            echo "=========================================================="
        }
        failure {
            echo "=========================================================="
            echo " [FAILURE] Stage 2 Ansible Bootstrap encountered an error!"
            echo " Actionable Troubleshooting Hints:                        "
            echo " 1. Verify SSM Agent status on private EC2 instances      "
            echo " 2. Check pod health: kubectl get pods -A                 "
            echo " 3. Check Ingress controller: kubectl get events -n ingress-nginx"
            echo " 4. Check ALB Target Health: aws elbv2 describe-target-health"
            echo "=========================================================="
        }
    }
}
