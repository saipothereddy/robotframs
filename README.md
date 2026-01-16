AWS_REGION=ap-south-1

DB_INSTANCE_ID=nexusiq-db
SECRET_ID=nexusiq/rds

EKS_CLUSTER_NAME=my-eks

HELM_RELEASE=nexus-iq
HELM_CHART=sonatype/nexus-iq-server
HELM_NAMESPACE=nexusiq

DEPLOYMENT_NAME=nexus-iq-server
REPLICA_COUNT=2




mkdir -p layer/bin
cd layer/bin

# kubectl
curl -LO https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl
chmod +x kubectl

# helm
curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
tar -xzf helm-v3.14.0-linux-amd64.tar.gz
mv linux-amd64/helm .
chmod +x helm

cd ..
zip -r kubectl-helm-layer.zip .
