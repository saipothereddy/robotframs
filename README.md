import os
import json
import base64
import boto3
import secrets
import string
import subprocess
from botocore.signers import RequestSigner
from botocore.session import Session

# ================= ENV =================
REGION = os.environ["AWS_REGION"]
DB_INSTANCE_ID = os.environ["DB_INSTANCE_ID"]
SECRET_ID = os.environ["SECRET_ID"]
EKS_CLUSTER_NAME = os.environ["EKS_CLUSTER_NAME"]

HELM_RELEASE = os.environ["HELM_RELEASE"]
HELM_CHART = os.environ["HELM_CHART"]
HELM_NAMESPACE = os.environ["HELM_NAMESPACE"]

DEPLOYMENT_NAME = os.environ["DEPLOYMENT_NAME"]
REPLICA_COUNT = os.environ["REPLICA_COUNT"]

# ================= AWS CLIENTS =================
rds = boto3.client("rds", region_name=REGION)
secretsmanager = boto3.client("secretsmanager", region_name=REGION)
eks = boto3.client("eks", region_name=REGION)

# ================= HELPERS =================

def generate_password(length=32):
    chars = string.ascii_letters + string.digits + "!@#$%^&*"
    return ''.join(secrets.choice(chars) for _ in range(length))


def write_kubeconfig():
    cluster = eks.describe_cluster(name=EKS_CLUSTER_NAME)["cluster"]

    session = Session()
    signer = RequestSigner(
        service_id="sts",
        region_name=REGION,
        signing_name="sts",
        signature_version="v4",
        credentials=session.get_credentials(),
        event_emitter=session.events,
    )

    params = {
        "method": "GET",
        "url": "https://sts.amazonaws.com/?Action=GetCallerIdentity&Version=2011-06-15",
        "body": {},
        "headers": {"x-k8s-aws-id": EKS_CLUSTER_NAME},
        "context": {},
    }

    signed_url = signer.generate_presigned_url(
        params, region_name=REGION, expires_in=60
    )

    token = "k8s-aws-v1." + base64.urlsafe_b64encode(
        signed_url.encode()
    ).decode().rstrip("=")

    kubeconfig = f"""
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: {cluster['endpoint']}
    certificate-authority-data: {cluster['certificateAuthority']['data']}
  name: eks
contexts:
- context:
    cluster: eks
    user: eks-user
  name: eks
current-context: eks
users:
- name: eks-user
  user:
    token: {token}
"""

    with open("/tmp/kubeconfig", "w") as f:
        f.write(kubeconfig)

    os.environ["KUBECONFIG"] = "/tmp/kubeconfig"


def run(cmd):
    subprocess.check_call(cmd, shell=True)


# ================= LAMBDA HANDLER =================

def lambda_handler(event, context):

    # 1️⃣ Read existing secret
    secret = json.loads(
        secretsmanager.get_secret_value(
            SecretId=SECRET_ID
        )["SecretString"]
    )

    # 2️⃣ Generate new password
    new_password = generate_password()

    # 3️⃣ Rotate password in RDS
    rds.modify_db_instance(
        DBInstanceIdentifier=DB_INSTANCE_ID,
        MasterUserPassword=new_password,
        ApplyImmediately=True
    )

    # 4️⃣ Rotate password in Secrets Manager
    secret["password"] = new_password
    secretsmanager.put_secret_value(
        SecretId=SECRET_ID,
        SecretString=json.dumps(secret)
    )

    # 5️⃣ Prepare kubeconfig
    write_kubeconfig()

    # 6️⃣ Helm upgrade (update DB password in values.yaml)
    run(f"""
helm upgrade {HELM_RELEASE} {HELM_CHART} \
  -n {HELM_NAMESPACE} \
  --reuse-values \
  --set database.password='{new_password}'
""")

    # 7️⃣ Scale DOWN Nexus IQ
    run(f"""
kubectl scale deployment {DEPLOYMENT_NAME} \
  -n {HELM_NAMESPACE} \
  --replicas=0
""")

    # 8️⃣ Scale UP Nexus IQ
    run(f"""
kubectl scale deployment {DEPLOYMENT_NAME} \
  -n {HELM_NAMESPACE} \
  --replicas={REPLICA_COUNT}
""")

    return {
        "status": "SUCCESS",
        "message": "RDS password rotated, secret updated, Helm upgraded, Nexus IQ restarted via scaling"
    }
