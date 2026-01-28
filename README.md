Lambda-based RDS Password Rotation with EKS Blue–Green Deployment
1. Overview
This document describes an AWS Lambda function that performs secure RDS password rotation and application rollout on Amazon EKS using a Blue–Green deployment strategy.
The Lambda function: - Rotates Amazon RDS (Aurora) master passwords - Updates AWS SSM Parameter Store with new secrets - Updates Kubernetes workloads using Helm - Scales deployments down and back up to avoid runtime failures - Supports mandatory BLUE and optional GREEN environments
This design is suitable for production-safe secret rotation with minimal downtime.
________________________________________
2. Architecture Flow
1.	Lambda is triggered (manual / scheduled / event-based)
2.	Lambda authenticates to AWS and EKS
3.	Kubernetes kubeconfig is generated dynamically
4.	For each environment (BLUE → GREEN):
o	Scale deployment to 0 replicas
o	Rotate RDS master password
o	Store new password in SSM Parameter Store
o	Update Helm release with new secret
o	Scale deployment back to desired replicas
________________________________________
3. Blue–Green Strategy
Environment	Purpose	Behavior
BLUE	Live / Production	Always processed
GREEN	Testing / Staging	Processed only if namespace exists
GREEN is optional and can be removed after validation.
________________________________________
4. Environment Variables
Required (Global)
Variable	Description
AWS_REGION	AWS region
EKS_CLUSTER_NAME	Target EKS cluster
HELM_RELEASE	Helm release name
HELM_CHART	Helm chart reference
HELM_REPO_NAME	Helm repo name
HELM_REPO_URL	Helm repo URL
DEPLOYMENT_NAME	Kubernetes deployment
REPLICA_COUNT	Desired replicas after rollout
BLUE (Mandatory)
Variable	Description
BLUE_DB_CLUSTER_ID	RDS cluster identifier
BLUE_SSM_PARAMETER_NAME	SSM parameter for DB password
BLUE_NAMESPACE	Kubernetes namespace
GREEN (Optional)
Variable	Description
GREEN_DB_CLUSTER_ID	RDS cluster identifier
GREEN_SSM_PARAMETER_NAME	SSM parameter for DB password
GREEN_NAMESPACE	Kubernetes namespace
________________________________________
5. Key Components
5.1 Helm Setup
Helm cache and config paths are redirected to /tmp because Lambda has a read-only filesystem except /tmp.
/tmp/helm/cache
/tmp/helm/config
/tmp/helm/data
________________________________________
5.2 EKS Authentication
The function uses STS RequestSigner to generate a short-lived Kubernetes authentication token.
Benefits: - No kubeconfig stored in Lambda - IAM-based authentication - Fully ephemeral credentials
________________________________________
5.3 Kubernetes Access
A temporary kubeconfig is written to:
/tmp/kubeconfig
This allows kubectl and helm to operate inside Lambda.
________________________________________
5.4 Password Generation
Passwords are: - Cryptographically secure (secrets module) - 32 characters long - Include symbols, numbers, and mixed case
________________________________________
6. Environment Processing Logic
Each environment (BLUE / GREEN) follows the same controlled workflow:
Step 1: Scale Down
kubectl scale deployment <deployment> -n <namespace> --replicas=0
Prevents DB auth failures during rotation.
________________________________________
Step 2: Rotate RDS Password
rds.modify_db_cluster(
  MasterUserPassword=new_password,
  ApplyImmediately=True
)
Lambda waits until the cluster becomes available.
________________________________________
Step 3: Update SSM Parameter
ssm.put_parameter(
  Type="SecureString",
  Overwrite=True
)
SSM acts as the source of truth for secrets.
________________________________________
Step 4: Helm Upgrade
helm upgrade <release> <chart> \
  --reuse-values \
  --set-string iq_server.database.password=<password>
Updates the application with the new DB password.
________________________________________
Step 5: Scale Up
kubectl scale deployment <deployment> --replicas=<count>
Restores application traffic.
________________________________________
7. Lambda Handler Logic
•	Logs caller identity
•	Prepares kubeconfig
•	Adds Helm repo
•	Processes BLUE first
•	Processes GREEN only if namespace exists
•	Returns structured execution result
Sample Response
{
  "status": "SUCCESS",
  "updated_environments": ["BLUE", "GREEN"],
  "message": "Successfully updated new password for BLUE and GREEN environment(s)."
}
________________________________________
8. IAM Permissions Required
Lambda execution role must include:
•	rds:DescribeDBClusters
•	rds:ModifyDBCluster
•	ssm:PutParameter
•	eks:DescribeCluster
•	sts:GetCallerIdentity
Plus EKS RBAC access to: - Scale deployments - Perform Helm upgrades
________________________________________
9. Security Best Practices
✔ No hardcoded credentials
✔ Secrets stored only in SSM SecureString
✔ Temporary tokens for EKS access
✔ Blue–Green isolation
✔ Controlled downtime
________________________________________
10. Use Cases
•	Automated DB credential rotation
•	Zero-trust Kubernetes deployments
•	Compliance-driven secret management
•	Production-safe application updates
________________________________________
11. Notes & Recommendations
•	GREEN environment can be safely removed after validation
•	Schedule via EventBridge for regular rotation
•	Add Slack / SNS notifications if needed
•	Consider readiness probes for smoother rollout
________________________________________
End of Document
