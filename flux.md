```shell
#
# Set these environment variables
#
export CLUSTER_NAME=vivek
export GITHUB_TOKEN=XXXXX
export GITHUB_USER=vivekr

#
# Bootstrapping the cluster with Flux
# The bootstrap process will automatically create a GitRepository custom resource that points to the given repository
# The GitRepository resource is named after the namespace where Flux GitOps ToolKit is installed. In this case, it is 'flux-system'
# The bootstrap process will configure the repository with an SSH key for read-only access
#
kubectl create ns flux-system
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --namespace=flux-system \
  --repository=eks-gitops-crossplane-flux \
  --branch=main \
  --path=clusters/$CLUSTER_NAME \
  --personal

#
# In order to authenticate with the external provider API such as AWS, the Crossplane AWS provider controller need to have access to credentials.
# An AWS user with Administrative privileges is needed to enable Crossplane to create the required resources
# We wil have to first create a configuration file, secrets.conf, with credeantials of an AWS account in the following format.
#
# [default]
# aws_access_key_id = XXXXXXXXXXXXXXXXXXXXXXX
# aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXX
#
# Then using this file, a YAML file that defines a Kubernetes Secret is created as follows
#
kubectl -n crossplane-system create secret generic aws-credentials --from-file=credentials=./secrets.conf --dry-run=client -o yaml > aws-credentials.yaml

#
# Next, deploy the Bitnami's Sealed Secrets controller in the 'sealed-secrets' namespace
# Then, generate a SealedSecret corresponding to the 'aws-credentials' Secret created above. This is done as shown below using the 'kubeseal' companion utility for Bitnami's Sealed Secrets.
# The file 'aws-credentials-sealed.yaml' resulting from the operation below is the one to deploy to the management cluster in the GitOps workflow.
# Push this file to the './deploy/crossplane-composition' directory of the GitHub repo that Flux is pointing to
#
kubeseal --controller-namespace sealed-secrets --format yaml < aws-credentials.yaml > aws-credentials-sealed.yaml

#
# Important! Extract the master sealing key from the Sealed Secrets controller into a YAML file.
# After extracting the master key, the Sealed Secrets controller may be termintaed.
# The controller per se will get deployed as part of the GitOps workflow.
# But you must make sure that the sealing key is first deployed using this file so that all SealedSecrets that were created using this master could be unsealed
#
kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealing-master.yaml


#
# Now, you are ready to initiate the GitOps worflow.
# Create a Kustomization resource under 'cluster/$CLUSTER_NAME' that points to the 'crossplane' directory in the config repo.
# Pushing this file to the Git repository will trigger a Flux reconcilliation loop which will install the following:
# 1. Crossplane core components
# 2. Crossplane AWS provider-specific components
# 3. Crossplane Configuration package for creating EKS cluster and other AWS resources
# 4. Composite resource to create an EKS cluster
# When this reconcilliation loop is completed, Crossplane will start provisioning the EKS cluster.
# It will take about 10 minutes for the cluster to be ready and operational
#
mkdir -p ./clusters/${CLUSTER_NAME}
flux create kustomization crossplane \
  --source=flux-system \
  --namespace=flux-system \
  --path=./crossplane \
  --prune=true \
  --validation=client \
  --interval=30s \
  --export > ./clusters/$CLUSTER_NAME/crossplane.yaml

#
# To deploy workloads to the remote cluster using the credentials of the cluster creator, continue with the following step.
# To deploy using the credentials of a service account with appropriate set of RBAC permissions, refer to ./remote/remote-cluster-setup.sh before proceeding further.
# Create a Kustomization resource under 'cluster/$CLUSTER_NAME' that points to the 'applications' directory 
# Pushing this file to the Git repository will trigger a Flux reconcilliation loop which will install the following:
# 1. Sample web application that exposes Prometheus metrics
# 2. Prometheus server which scrapes the metrics from the sample application and sends it to an AMP workspace
#
mkdir -p ./clusters/${CLUSTER_NAME}
flux create kustomization applications \
  --source=flux-system \
  --namespace=flux-system \
  --path=./applications \
  --prune=true \
  --validation=client \
  --interval=30s \
  --export > ./clusters/$CLUSTER_NAME/applications.yaml


  ```
