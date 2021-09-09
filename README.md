# Introduction

This is an update to my deployment [jenkins-deploy-eks-via-terraform](https://github.com/spicysomtam/jenkins-deploy-eks-via-terraform). Technology moves on, and there is now an easier way to deploy an EKS cluster using `eksctl`, which is now officially supported by AWS (docs [here](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)). Both deploys have the same functionality so I have moved most of the documentation to this repo. 

Comparing the `terraform` and `eksctl` deploys:
* `eksctl` uses AWS Cloudformation to deploy EKS clusters, and a single command creates the vpc, subnets, nodegroups and everything else needed. 
* Since `eksctl` uses Coundformation stacks this might be regarded as slow and inferior to `terraform`; however Cloudformation is an official technology from AWS.
* `terraform` is from Hashicorp and is a tool for building infrastructure for different providers like AWS, GCP, Azure, etc.
* `eksctl` is a much simpler way to build EKS clusters than `terraform`.
* `eksctl` deploy is much slower than `terraform`; this was not noticable on earlier versions of `eksctl` but its very apparent now. 24 mins vs 13 mins.
* Its interesting how technologies leap frog each other; this was a reasonable way to deploy EKS but its now notibly slow.

I might also add that I don't like EKS much; it has too many issues and edge cases; anyhow I digress (use k3s if you want a fast light k8s!). However I understand why someone might want an officially supported service from AWS, and over time I am sure EKS will get better as AWS develops it.

There are lots of options for `eksctl`; most of this is documented at [eksctl.io](https://eksctl.io), although you might want to issue `--help` against the latest binary to see what options are available.

# EKS notes
## Fargate

Fargate is not supported in this repo. I test Fargate and its really slow to spin up a pod (~ 60s) and its not very elegant; a Fargate worker node is deployed for each pod since Fargate uses VMs. See [this issue](https://github.com/aws/containers-roadmap/issues/649) for a discussion on EKS Fargate slowness. At the time of writing Fargate is not a realistic option (although it may get better in time). My recommendation: stick with EC2 node groups.

## EKS has very low max pods per node

This was a surprise to me. Max pods per node is the maximum pods a worker node can accomodate. We get a max of 29 pods on a `m5.large` instance type (2 vcpu 8Gb)! Traditionally this was based on cpu and memory resources available on a node, but there are other limits such the cidr range, the container network driver, and the node configuration (node allocatable which can limit pods, memory, cpu, etc). 

The default for Google Kubernetes Enginer (GKE) is 110 pods on a fairly small node (e2-medium 2 vcpu 4Gb). On Rancher k3s, which is a light weight but fully functional k8s distribution, on a AWS `t3a.medium` instance (2 vcpu 4Gb) I can get 100 pods per node with good performance across 2 masters and 2 workers, giving 400 pods on a fairly minimal setup. My test pods were using the default `helm` chart helm3 creates for you (see [Testing the cluster autoscaler](#testing-the-cluster-autoscaler)); this creates a fairly light nginx deployment which is great for creating lots of pods. 

You can see my deployment for k3s clusters [here](https://github.com/spicysomtam/k3s-aws-cluster); I have done alot of work on testing k3s and its pretty solid and scaleable, and its backed by Rancher Labs, a commercial k8s vendor. The Rancher GUI makes setting up logging and monitoring a breeze.

So the pods per node for EKS is limited by the Container Network Interface (CNI) driver used by AWS for VPCs and the cidr range. There is a list [here](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt). From this list we can see its 29 pods for a `m5.large` instance, which is small. This would be fine if your pods use alot of resources, but the idea behind k8s and containerisation in general is to have lots of light weight, fast to deploy/scale, pods. Thus it could turn out quite expensive to run a EKS cluster. There are a couple of guides you can find via a Google to replace the AWS CNI on EKS, which would allow you to increase the pods per node.

I also observe the pod startup and creation is much faster on k3s and gke than eks, and might attribute that to the CNI allocation of the IP for the pod. The cluster creation is also much quicker with google cloud platform and my terraform k3s deploy.

Of all the cloud native kubernetes services, IMHO GCP is the best; not only are the clusters fast to deploy, many things such as ingress have been made easy to setup, and there are lots of guides for doing common tasks within the gcp console. It feels slick and polished. Google Cloud also has a command line tool and api interface.

## Accessing the cluster

Ensure your aws cli is up to date as the new way to access the cluster is via the `aws` cli rather than the `aws-iam-authenticator`, which you used to need to download and install in your path somewhere. Now you can just issue the aws cli to update your kube config:
```
$ aws eks update-kubeconfig --name demo --region eu-west-1
```

You would use `kubectl` to access the cluster (install latest or >= v1.21 at the time of this update). 

Once you can access the cluster via  a test command (eg `kubectl get all -A`), you can add access for other aws users; see official EKS docs [here](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html). 

## Kubernetes api and worker nodes are on the public internet

Just something to be aware of. My view on this is its not very secure, even though all traffic is encypted using TLS and ports are limited. Ideally these should only be accessible on the vpc, and then you need to get access to the vpc via a bastion host or vpn. However this example is intended to be a simple example you can spin up, and maybe enhance to fit your needs.


# Jenkins pipeline

You will need some aws credentials adding to Jenkins to allow access to your aws account and deploy the stack.

Add the git repo where the code is, and tell it to run [Jenkinsfile](Jenkinsfile) as the pipeline.

`create` creates an eks cluster stack and `destroy` destroys it.

When running the Jenkins job, you will need to confirm the `create` or `destroy`.

You can create multiple eks clusters/stacks by simply specifying a different cluster name.

If a `create` goes wrong, simply re-run it for the same cluster name, but choose `destroy`, which will clean it down. Conversly you do the `destroy` when you want to tear down the stack.

## Summary of features and options available via the Jenkins pipeline

We can automatically install these features:
* Cloudwatch logging: all cluster backplane logging goes into cloudwatch. Enabled by default.
* Cloudwatch metrics and container insights. This can cost alot of money in terms of aws bills. Thus its default is disabled. Use metrics-server and prometheus instead (and these are better imho).
* Kubernetes dashboard. Some people like this, especially if you are new to k8s, or don't have access to the command line. I would recommend k8s Lens instead, which is a client side program. By default its disabled.
* Prometheus metrics scraper. This is used by various monitoring software (including Lens). By default its enabled
* nginx-ingress. This is discussed elsewhere. Enable an ingress controller, which is extremly useful and thus enabled by default.
* Cluster autoscaler. Spin up and down nodes depending on whether pods are not scheduled (eg the cluster runs out of resources). Only enable for prod deploys and thus disabled by default.
* cert-manager. Automatically manage TLS certs in k8s. Very useful for free Letsencrypt certs (but also works for others such as Godaddy, etc). Disabled by default.

## Kubernetes version can be specified

You can choose all the versions currently offered in the AWS console.

## Automatic setting up of CloudWatch logging, metrics and Container Insights

EKS allows all k8s logging to be sent to CloudWatch logs, which is really useful for investigating issues. I have added an option for this.

In addition, CloudWatch metrics are also gathered from EKS clusters, and these are fed into the recently released Container Insights, which allows you to see graphs on performance, etc. These are not setup automatically in EKS and thus I added this as an option, with the default being disabled. The reason its disabled is because costs can mount on the metrics, while the logging costs are reasonable. Thus you might enable metrics on prod clusters but turn them off on dev clusters.

Note that Container Insights can become expensive to operate; consider installing metrics-server and then some form of scaper and presenter (Prometheus, Kibana, Lens, etc).

## Automatic setup of an ingress controller and a load balancer

The idea here is to set an ingress controller and then just use a single Layer4/TCP style load balancer for all inbound traffic. This is in preference to creating a load balancer for each service, which will create multiple load balancers, incurr additional cost and complexity, and is not necessary! It also makes everything simpler in terms of multiple dns names, certificates, etc; everything is managed in kubernetes rather than a mixed setup of AWS and kubernetes. Trust me; this is the way to go (and I have seen it badly done and then difficult to unravel once setup).

In essence you create a DNS cname record for each service, which points to the load balancer. On ingress the nginx ingress determines the DNS name requested and then directs traffic to the correct service. You can Google kubernetes ingress to discover more about it. Note this setup also supports TLS HTTPS ingress (see TLS in kubernetes documentation on ingress controllers; you can also use wildcard certs, set a default ingress cert, use lets encrypt, etc).

There are multiple ingress controllers available. Most people use the free open source `kubernetes/ingress-nginx`, while confusing ther is another free nginx ingress from Nginx Inc called `nginxinc/kubernetes-ingress`; I used the latter as it has an official AWS Solution documented [here](https://aws.amazon.com/premiumsupport/knowledge-center/eks-access-kubernetes-services/), which has a deployment specifically for AWS. I have simplified the install using the official Helm chart.

Nginx ingress is setup by default, otherwise the EKS cluster has no ingress (just a load balancer for the k8s API). Its a good default if you are not sure what you want and anyway nginx ingress is the way to go with modern k8s!

## cert-manager install

I added the setup of cert-manager, plus installing the ClusterIssuers for Letsencrypt (LE) staging and production using http01 validation method. This means you can use cert-manager to automatically generate LE certificates. 

All you then need to do is point a dns cname record at the load balancer, create a staging cert for it, and then let cert-manager get a cert for you. Sometimes the staging cert does not work first time; if so delete the cert and try again. Once you have a staging cert working, you can replace it with a prod cert, and cert-manager will then manage the cert renewals, etc.

Note: Staging certs won't pass validation on most web browsers while prod ones do. Be aware that prod ones are throttled and controlled more stringely than staging ones; thus why you need to get the staging working first, which will allow you to troubleshoot issues, etc.

See cert manager docs for full details.

## Monitoring and metrics to allow horizontal and vertical pod autoscaling

The horizontal and vertical pod autoscalers need cpu and memory metrics to allow these to operate so you need the metrics-server to be installed for these, which is an option in the Jenkins pipeline.

Prometheus is a common performance scraper used by various monitoring tools (my favourite is k8s Lens); this can now be setup via the Jenkins pipeline.

## kubernetes dashboard

This is a popular gui for those new to k8s or those without access to the command line and kubectl. This can now be installed via the Jenkins pipeline.

## Populating the aws-auth configmap with admin users

After an EKS cluster is created, only the AWS credentials that created the cluster can access it. This is problematic as you may not have the credentials Jenkins used to create the cluster.

Thus you can specify a comma delimited list of IAM users who will be given full admin rights on the cluster.

## Kubernetes Cluster Autoscaler install

I added this as an option (default not enabled). What is it? The kubernetes [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) (CA) allows k8s to scale worker nodes up and down, dependant on load. Since EKS implements worker nodes via node groups and the autoscaling groups (ASG), a deployment within the cluster monitors pod load, and scales up nodes via the ASG when pods are not scheduled (not enough resources available to run them), and scales nodes down again when they are not used (typically after 10 minutes of idle).

Note that there is a max_workers Jenkins parameter; ensure this is large enough as this is the limit that CA can scale to!

Also be aware that the minimum number of nodes will be the same as the desired (num_workers Jenkins parameter). Originally I set the minimum to 1 but the CA will scale down to this with no load which I don't think is a good idea; thus I set the minimum to be the same as desired. You don't want low fault tolerance (one worker) or your cluster to be starved of resources. At a minimum you should have 3 workers, if only to spread them across 3 availability zones, provide some decent capacity and ability to handle some load without waiting for CA to scale more nodes.

Note that you should also consider the Horizontal Pod Autoscaler, which will scale pods up/down based on cpu load (it requires `metrics-server` to aquire metrics to determine load).

### Testing the Cluster Autoscaler

For a complete test we need to test that nodes are scaled up and then down.

#### Scale up

The easiest way to do this is deploy a cluster with a limited number of worker nodes, and then overload them with pods. A simple way to do this is deploy a helm3 chart, and then scale up the number of replicas (pods). 

I found that based on a `m5.large` instance type, using a `nginx` deployment, I could deploy approx 30 pods per worker (pods per node). Lets run through this:

```
$ kubectl get nodes # We only have one node as I deliberatly set the cluster up like this
NAME                                      STATUS     ROLES    AGE   VERSION
ip-10-0-1-74.eu-west-1.compute.internal   Ready      <none>   46m   v1.17.11-eks-cfdc40

$ kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler # we can check ca logging
$ cd /tmp
$ helm create nginx # This creates a local chart based on nginx
$ helm upgrade ng0 nginx/ --set replicaCount=30 # 1 node overloaded
$ helm upgrade ng0 nginx/ --set replicaCount=60 # 2 nodes overloaded
$ kubeclt get po # Do we have any pending pods?
$ kubectl get nodes # See if we are scaling up; could check the AWS EC2 Console?
```

You should see nodes being added as pods are in pending state. In AWS it can take a couple of minutes to deploy another node. I would increase the number of pods such that you force the addition of 2 nodes.

#### Scale down

We can also check the scale down:
* Reduce the number of pods to 10.
* Wait 10 mins; the ca should start terminating nodes via the auto scaling group.
* If there are pods on nodes that are terminated, kubernetes we kill and restart these on active nodes.
* Needless to say your application should be able to safely restart pods without any side effects!

#### Working round the delay in spinning up another worker

You might ask how can be get round the delay in scaling up another worker? Pods have a priority. You could create a deployment with pods of a lower priority than your default (0); lets call these placeholder pods. Then k8s will kill these placeholder pods and replace them with your regular pods when needed. So you could autoprovision a single node full of these placeholder pods. Your placeholder pods will then become pending after being killed and CA will then spin up another worker for them; since these placeholder pods arn't doing anything useful we don't mind the delay. This is just an idea; I probably need to google a solution where this has been implemented, and provide a link to it (I am sure someone has written something for this).

Another solution: use fargate. However this is not a realistic solution (see my notes above).
