// Unfortunatly eks is fiddly and messy. Consider gcp gke if you want easy to deply k8s clusters with greater performance!

pipeline {

   parameters {
    choice(name: 'action', choices: 'create\ndestroy', description: 'Create/update or destroy the eks cluster.')
    string(name: 'cluster', defaultValue : 'demo', description: "EKS cluster name.")
    choice(name: 'k8s_version', choices: '1.21\n1.20\n1.19\n1.18\n1.17\n1.16', description: 'K8s version to install.')
    string(name: 'instance_type', defaultValue : 'm5.large', description: "k8s worker node instance type.")
    string(name: 'num_workers', defaultValue : '3', description: "k8s number of worker instances.")
    string(name: 'max_workers', defaultValue : '10', description: "k8s maximum number of worker instances that can be scaled.")
    string(name: 'admin_users', defaultValue : '', description: "Comma delimited list of IAM users to add to the aws-auth config map.")
    string(name: 'credential', defaultValue : 'jenkins', description: "Jenkins credential that provides the AWS access key and secret.")
    booleanParam(name: 'cw_logs', defaultValue : true, description: "Setup Cloudwatch logging?")
    booleanParam(name: 'cw_metrics', defaultValue : false, description: "Setup Cloudwatch metrics and Container Insights?")
    booleanParam(name: 'nginx_ingress', defaultValue : true, description: "Setup nginx ingress and load balancer?")
    booleanParam(name: 'ca', defaultValue : false, description: "Setup k8s Cluster Autoscaler?")
    booleanParam(name: 'cert_manager', defaultValue : false, description: "Setup cert-manager for certificate handling?")
    string(name: 'region', defaultValue : 'eu-west-1', description: "AWS region.")
    string(name: 'key_pair', defaultValue : 'spicysomtam-aws7', description: "EC2 instance ssh keypair.")
  }

  options {
    disableConcurrentBuilds()
    timeout(time: 1, unit: 'HOURS')
    withAWS(credentials: params.credential, region: params.region)
    ansiColor('xterm')
  }

  agent { label 'master' }

  environment {
    // Set path to workspace bin dir
    PATH = "${env.WORKSPACE}/bin:${env.PATH}"
  }

  stages {

    stage('Setup') {
      steps {
        script {
          currentBuild.displayName = "#" + env.BUILD_NUMBER + " " + params.action + " " + params.cluster

          println "Getting the kubectl, eksctl and helm binaries..."
          (major, minor) = params.k8s_version.split(/\./)
          sh """
            mkdir bin
            ( cd bin
            curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_\$(uname -s)_amd64.tar.gz" | tar xzf -
            # 'latest' kubectl is backward compatible with older api versions
            curl --silent -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
            curl -fsSL -o - https://get.helm.sh/helm-v3.6.3-linux-amd64.tar.gz | tar -xzf - linux-amd64/helm
            mv linux-amd64/helm .
            rm -rf linux-amd64
            chmod u+x eksctl kubectl helm
            ls -l eksctl kubectl helm )
            [ ${major} -ge 1 -a ${minor} -ge 18 ] && {
              echo "k8s >= 1.18"
            }

          """
        }
      }
    }

    stage('Create') {
      when {
        expression { params.action == 'create' }
      }
      steps {
        script {
          input "Create EKS cluster ${params.cluster} in aws?" 

          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
          credentialsId: params.credential, 
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',  
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            
            ca_args=""

            if (params.ca == true) {
                ca_args="--asg-access" 
            }

            sh """
              eksctl create cluster \
                --name ${params.cluster} \
                --version ${params.k8s_version} \
                --region  ${params.region} \
                --nodegroup-name ${params.cluster}-0 \
                --nodes ${params.num_workers} \
                --nodes-min ${params.num_workers} \
                --nodes-max ${params.max_workers} \
                --node-type ${params.instance_type} \
                --with-oidc \
                --ssh-access \
                ${ca_args} \
                --ssh-public-key ${params.key_pair} \
                --managed
            """

            // Oddly there isn't a eksctl arg to enable cw logs, although it can be passed as config.
            if (params.cw_logs == true) {
              echo "Setting up Cloudwatch logs."
              sh """
                eksctl utils update-cluster-logging --enable-types all --approve --cluster ${params.cluster}
              """
            }
          }
        }
      }
    }

    stage('Cluster setup') {
      when {
        expression { params.action == 'create' }
      }
      steps {
        script {
          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
          credentialsId: params.credential, 
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',  
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            
            sh """
              aws eks update-kubeconfig --name ${params.cluster} --region ${params.region}
            """

            // If admin_users specified
            if (params.admin_users != '') {
              echo "Adding admin_users to configmap aws-auth."
              sh "./generate-aws-auth-admins.sh ${params.admin_users} | kubectl apply -f -"
            }

            // The recently introduced CW Metrics and Container Insights setup
            // https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-prerequisites.html
            // https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-quickstart.html
            // eksctl does not support CW metrics currently, so we need to tweak the EC2 instance role to allow it to work.
            // See this for the issue: https://github.com/weaveworks/eksctl/issues/811#issuecomment-731266712
            if (params.cw_metrics == true) {
              echo "Setting up Cloudwatch metrics and Container Insights."

              sh """
                curl --silent https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | \\
                  sed "s/{{cluster_name}}/${params.cluster}/;s/{{region_name}}/${params.region}/" | \\
                  kubectl apply -f -
              """

              // All this complexity to get the EC2 instance role and then attach the policy for CW Metrics
              roleArn = sh(returnStdout: true, 
                script: """
                aws eks describe-nodegroup --nodegroup-name ${params.cluster}-0 --cluster-name ${params.cluster} --query nodegroup.nodeRole --output text
                """).trim()

              role = roleArn.split('/')[1]

              sh """
                aws iam attach-role-policy --role-name ${role} --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
              """
            }

            if (params.ca == true) {
              echo "Setting up k8s Cluster Autoscaler."

              // Keep the google region logic simple; us or eu
              gregion='us'

              if (params.region =~ '^eu') {
                gregion='eu'
              }

              // CA image tag, which is k8s major version plus CA minor version.
              // See for latest versions: https://github.com/kubernetes/autoscaler/releases
              switch (params.k8s_version) {
                case '1.21':
                  tag='0'
                    break;
                case '1.20':
                  tag='0'
                    break;
                case '1.19':
                  tag='1'
                    break;
                case '1.18':
                  tag='3'
                    break;
                case '1.17':
                  tag='4'
                    break;
                case '1.16':
                  tag='7'
                    break;
              }

              // Setup documented here: https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html
              sh """
                kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
                kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
                kubectl -n kube-system get deployment.apps/cluster-autoscaler -o json | \\
                  jq | \\
                  sed 's/<YOUR CLUSTER NAME>/${params.cluster}/g' | \\
                  jq '.spec.template.spec.containers[0].command += ["--balance-similar-node-groups","--skip-nodes-with-system-pods=false"]' | \\
                  kubectl apply -f -
                kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=${gregion}.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v${params.k8s_version}.${tag}
              """
            }

            // See: https://aws.amazon.com/premiumsupport/knowledge-center/eks-access-kubernetes-services/
            if (params.nginx_ingress == true) {
              echo "Setting up nginx ingress and load balancer."
              (major, minor) = params.k8s_version.split(/\./)
              sh """
                [ -d kubernetes-ingress ] && rm -rf kubernetes-ingress
                git clone https://github.com/nginxinc/kubernetes-ingress.git
                cd kubernetes-ingress/deployments/
                kubectl apply -f common/ns-and-sa.yaml
                kubectl apply -f rbac/rbac.yaml
                kubectl apply -f common/default-server-secret.yaml
                kubectl apply -f common/nginx-config.yaml

                # k8s >= 1.18 
                [ ${major} -ge 1 -a ${minor} -ge 18 ] && {
                  kubectl apply -f common/ingress-class.yaml
                }

                kubectl apply -f deployment/nginx-ingress.yaml
                sleep 5
                kubectl apply -f service/loadbalancer-aws-elb.yaml
                cd -
                rm -rf kubernetes-ingress
                kubectl apply -f nginx-ingress-proxy.yaml
                kubectl get svc --namespace=nginx-ingress
              """
            }

            if (params.cert_manager == true) {
              echo "Setting up cert-manager."
              sh """
                helm repo add jetstack https://charts.jetstack.io || true
                helm repo update
                kubectl create ns cert-manager
                helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.1.0 --set installCRDs=true
                sleep 30 # allow cert-manager setup in the cluster
                kubectl apply -f cluster-issuer-le-staging.yaml
                kubectl apply -f cluster-issuer-le-prod.yaml
              """
            }
          }
        }
      }
    }

    stage('Destroy') {
      when {
        expression { params.action == 'destroy' }
      }
      steps {
        script {
          input "Destroy EKS cluster ${params.cluster} in aws?" 

          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
            credentialsId: params.credential, 
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',  
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            // All this complexity to get the EC2 instance role and then dettach the policy for CW Metrics
            // We need to detach before running eksctl otherwise eksctl will fail to delete
            roleArn = sh(returnStdout: true, 
              script: """
              aws eks describe-nodegroup --nodegroup-name ${params.cluster}-0 --cluster-name ${params.cluster} --query nodegroup.nodeRole --output text
              """).trim()

            role = roleArn.split('/')[1]

            sh """
              aws iam detach-role-policy --role-name ${role} --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy || true

              [ -d kubernetes-ingress ] && rm -rf kubernetes-ingress
              git clone https://github.com/nginxinc/kubernetes-ingress.git

              # Need to clean this up otherwise the vpc can't be deleted
              kubectl delete -f kubernetes-ingress/deployments/service/loadbalancer-aws-elb.yaml || true
              [ -d kubernetes-ingress ] && rm -rf kubernetes-ingress
              sleep 20
            """

            sh """
              eksctl delete cluster \
                --name ${params.cluster} \
                --region ${params.region} \
                --wait
            """
          }
        }
      }
    }

  }

  post {
    always{
      deleteDir()
    }
  }
}
