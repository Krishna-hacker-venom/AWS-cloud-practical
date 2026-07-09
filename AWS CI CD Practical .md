# AWS CI/CD Practical: Flask Deployment via S3 -> CodePipeline -> Elastic Beanstalk

## Definition

- A CI/CD (Continuous Integration / Continuous Deployment) pipeline automates the path from "code change" to "code running in production" without manual redeployment steps.
- This practical builds a pipeline where:
  - **Source**: An S3 bucket holds a zipped Flask application (`app.zip`)
  - **Build**: Skipped (no compilation/testing step needed for this simple app)
  - **Deploy**: AWS Elastic Beanstalk automatically receives and deploys the new version
- Trigger mechanism: S3 **bucket versioning** + **CloudWatch Events** — every time `app.zip` is overwritten, a new object version is created, which fires an event that starts the pipeline.

### Analogy: The Restaurant Kitchen Pass

Think of this pipeline like a restaurant kitchen pass system:

- **S3 bucket** = the order ticket rail where new orders (code) get pinned
- **CodePipeline** = the expediter who watches the rail and the moment a new ticket appears, walks it to the right station
- **Elastic Beanstalk** = the cooking station that takes the ticket and prepares the dish (runs the app) using a consistent recipe (platform config) every time
- **IAM roles** = the kitchen's ID badges — they determine who is allowed to touch which equipment. If a badge doesn't have "line cook" clearance, the expediter can't even *pass* the ticket, let alone the cook prepare it

---

## How It Works

```
Local/EC2 code changes
        |
        v
   zip app.zip
        |
        v
 aws s3 cp app.zip s3://bucket/app.zip   <-- new S3 object VERSION created
        |
        v
CloudWatch Event fires (S3 object created/updated)
        |
        v
CodePipeline: Source stage pulls new version --> SUCCESS/FAIL
        |
        v
CodePipeline: Build stage (skipped in this setup)
        |
        v
CodePipeline: Deploy stage --> pushes zip to Elastic Beanstalk
        |
        v
Elastic Beanstalk creates a new application version
        |
        v
EB updates the running EC2 environment with new code
        |
        v
App live at: <env-name>.<hash>.<region>.elasticbeanstalk.com
```

Key mechanical detail: **CodePipeline does not "watch a folder."** It watches for a specific S3 **object key** to receive a **new version ID**. Versioning must be turned on for the bucket, or nothing ever triggers, no matter how many times you re-upload the same-named file.

---

## Methodology (Step-by-Step, With Errors Actually Hit)

### Step 1 — Write the Flask app

```python
# application.py
from flask import Flask

application = Flask(__name__)

@application.route('/')
def hello():
    return '<h1>Hello Batch 26 this is the demonstration of python deployment</h1>'

if __name__ == '__main__':
    application.run(host='0.0.0.0', port=8080)
```

```
# requirements.txt
Flask==3.0.0
```

**Critical naming rule for Elastic Beanstalk's Python platform:**
- File must be named `application.py`
- Flask instance variable must be named `application`, not `app`
- Beanstalk's underlying WSGI server looks for this exact name — using `app` instead silently breaks the deployment (502 errors with no obvious cause in the code itself)

### Step 2 — Zip and prepare on EC2

```bash
cd ~/python-flask-app
zip -r app.zip application.py requirements.txt
```

- Zip the **files**, not the parent folder — zipping the folder itself creates a nested path (`python-flask-app/application.py`) that Beanstalk cannot find at its expected root path

### Step 3 — Create the S3 bucket

**Error encountered:**
```
An error occurred (NoSuchBucket) when calling the ListObjectsV2 operation: 
The specified bucket does not exist
```

**Root cause:** The bucket was referenced in commands before it was actually created — `aws s3 ls` confirmed zero buckets existed in the account at that point.

**Fix:**
```bash
aws s3 mb s3://python-cicd1-source --region ap-southeast-1
aws s3api put-bucket-versioning \
  --bucket python-cicd1-source \
  --versioning-configuration Status=Enabled
```

**Concept: why versioning is mandatory here**

### Analogy: The Whiteboard vs. The Notebook

- A bucket **without** versioning is a whiteboard — when you erase and rewrite, there's no record that anything changed, and nobody watching gets notified
- A bucket **with** versioning is a notebook — every overwrite is a new page, timestamped, and anyone watching the notebook sees a new page appear and can react to it
- CodePipeline's S3 source trigger only reacts to "new pages," so without versioning, it's staring at a whiteboard that never seems to change

### Step 4 — Attach IAM role to EC2 (for S3 upload rights)

**Console path:**
- IAM -> Roles -> Create role -> **EC2** as trusted entity
- Attach `AmazonS3FullAccess` (or a bucket-scoped custom policy)
- EC2 Console -> instance -> **Actions -> Security -> Modify IAM role** -> attach role

**Equivalent CLI commands (run from local machine or CloudShell, not the EC2 instance itself — you need existing IAM permissions to create a role):**

```bash
# 1. Create the trust policy file (allows EC2 to assume this role)
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# 2. Create the role
aws iam create-role \
  --role-name EC2-S3-CodePipeline-Role \
  --assume-role-policy-document file://trust-policy.json

# 3. Attach S3 full access policy to the role
aws iam attach-role-policy \
  --role-name EC2-S3-CodePipeline-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# 4. Create an instance profile and add the role to it
aws iam create-instance-profile \
  --instance-profile-name EC2-S3-CodePipeline-Profile

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-CodePipeline-Profile \
  --role-name EC2-S3-CodePipeline-Role

# 5. Attach the instance profile to the running EC2 instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-09381c16204270c86 \
  --iam-instance-profile Name=EC2-S3-CodePipeline-Profile
```

**Verify the role is attached (run this ON the EC2 instance):**
```bash
aws sts get-caller-identity
```
Confirms which role/identity the instance is currently authenticating as.

**Concept: why an IAM role instead of access keys**

### Analogy: Hotel Key Card vs. Photocopied House Key

- Hardcoding AWS access keys onto an EC2 instance is like photocopying your house key and taping it under the doormat — anyone who gets to the doormat (compromises the instance) has permanent access, and the key doesn't expire
- An IAM role attached to EC2 is like a hotel key card — it's issued temporarily, scoped to specific doors (permissions), and can be revoked or rotated without you personally handing out anything

### Step 5 — Upload the zip

```bash
aws s3 cp app.zip s3://python-cicd1-source/app.zip
```

Verified with:
```bash
aws s3 ls s3://python-cicd1-source/
```

### Step 6 — Create the Elastic Beanstalk environment

**Console path:**
- Elastic Beanstalk -> Create application -> Platform: Python
- Upload the same `app.zip` as the initial version (manual, one-time bootstrap)
- Wait for environment health: **Pending -> Ok (green)**

**Equivalent CLI commands (using the Elastic Beanstalk CLI, `eb`):**

```bash
# Install the EB CLI if not already present
pip3 install awsebcli --break-system-packages

# Initialize EB config inside your app folder
cd ~/python-flask-app
eb init -p python-3.11 python-flask-app --region ap-southeast-1

# Create the environment and deploy in one step
eb create Appbeans-env

# Check status/health from the CLI
eb status
eb health

# Open the live URL directly from terminal
eb open
```

**Or using raw `aws` CLI (no `eb` tool):**
```bash
# Upload zip to the EB-managed S3 bucket and create an application version
aws elasticbeanstalk create-application-version \
  --application-name appbeans \
  --version-label v1 \
  --source-bundle S3Bucket="python-cicd1-source",S3Key="app.zip"

# Create the environment using that version
aws elasticbeanstalk create-environment \
  --application-name appbeans \
  --environment-name Appbeans-env \
  --solution-stack-name "64bit Amazon Linux 2023 v4.13.3 running Python 3.14" \
  --version-label v1

# Check environment health
aws elasticbeanstalk describe-environment-health \
  --environment-name Appbeans-env \
  --attribute-names All
```

**Concept: why Beanstalk needs its own service role AND instance profile (two separate identities)**

### Analogy: Building Manager vs. Building Tenant

- The **service role** is like the building manager — it has permission to create new floors, rearrange the lobby, and manage the building's infrastructure (EC2, load balancers, auto-scaling) on your behalf
- The **EC2 instance profile** is like the tenant's keycard — it only grants access to what the tenant (your running app code) actually needs while living inside the building (e.g., writing logs to S3)
- Confusing the two, or granting a tenant manager-level access, is how over-permissioned environments happen in the real world

### Step 7 — Create the CodePipeline

**Error encountered #1: Wrong region selected**

The pipeline creation console defaulted to `us-east-1` (N. Virginia), while the S3 bucket and Beanstalk environment lived in `ap-southeast-1` (Singapore).

**Fix:** Manually switch the region selector (top right of AWS Console) to match the bucket/environment region *before* starting pipeline creation. CodePipeline resources are region-scoped; cross-region references require extra explicit configuration this practical didn't need.

**Error encountered #2: Wrong pipeline template selected**

The new CodePipeline UI defaults to container-oriented templates (Push to ECR, Deploy to ECS Fargate) under a "Deployment" category — none of these match a simple S3-to-Beanstalk flow.

**Fix:** Select **"Build custom pipeline"** instead, which gives the classic manual Source -> Build -> Deploy stage builder.

Pipeline configuration used:
- **Source**: Amazon S3 -> bucket `python-cicd1-source` -> object key `app.zip` -> detection via CloudWatch Events
- **Build**: Skipped
- **Deploy**: AWS Elastic Beanstalk -> application `appbeans` -> environment `Appbeans-env`

**Equivalent CLI command (create pipeline from a JSON definition file):**

```bash
cat > pipeline.json << 'EOF'
{
  "pipeline": {
    "name": "krishnaPipeline",
    "roleArn": "arn:aws:iam::588738570955:role/service-role/AWSCodePipelineServiceRole-ap-southeast-1-krishnaPipeline",
    "artifactStore": {
      "type": "S3",
      "location": "codepipeline-ap-southeast-1-artifact-store"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "S3",
              "version": "1"
            },
            "outputArtifacts": [{ "name": "SourceOutput" }],
            "configuration": {
              "S3Bucket": "python-cicd1-source",
              "S3ObjectKey": "app.zip",
              "PollForSourceChanges": "false"
            }
          }
        ]
      },
      {
        "name": "Deploy",
        "actions": [
          {
            "name": "Deploy",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "ElasticBeanstalk",
              "version": "1"
            },
            "inputArtifacts": [{ "name": "SourceOutput" }],
            "configuration": {
              "ApplicationName": "appbeans",
              "EnvironmentName": "Appbeans-env"
            }
          }
        ]
      }
    ]
  }
}
EOF

aws codepipeline create-pipeline --cli-input-json file://pipeline.json --region ap-southeast-1
```

**Check pipeline state anytime from CLI:**
```bash
aws codepipeline get-pipeline-state --name krishnaPipeline --region ap-southeast-1
```

### Step 8 — First pipeline run: Deploy stage failed

**Error encountered #3:**
```
Deploy
AWS Elastic Beanstalk
1 of 1 action failed.
```

**Root cause:** The auto-generated CodePipeline service role (`AWSCodePipelineServiceRole-ap-southeast-1-krishnaPipeline`) had permissions for S3 and CloudWatch, but **no permissions for Elastic Beanstalk actions** — CodePipeline was authenticated, but not authorized to talk to Beanstalk's API.

**Fix (Console):**
1. IAM -> Roles -> search the exact role name shown in **Pipeline -> Settings -> Service role ARN**
2. Add permissions -> Attach policies
3. Note: `AWSElasticBeanstalkFullAccess` is deprecated and no longer appears in the attach-policy search
4. Attach the modern replacement instead: **`AdministratorAccess-AWSElasticBeanstalk`**
5. Retry the pipeline (**Release change**)

**Equivalent CLI fix:**

```bash
# Attach the Elastic Beanstalk policy to the pipeline's exact service role
aws iam attach-role-policy \
  --role-name AWSCodePipelineServiceRole-ap-southeast-1-krishnaPipeline \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess-AWSElasticBeanstalk

# If deploy still fails afterward, add inline PassRole permission
cat > passrole-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name AWSCodePipelineServiceRole-ap-southeast-1-krishnaPipeline \
  --policy-name PassRoleForEB \
  --policy-document file://passrole-policy.json

# Retry the pipeline execution from CLI
aws codepipeline start-pipeline-execution \
  --name krishnaPipeline \
  --region ap-southeast-1
```

**Concept: why "the pipeline succeeded" and "the deploy succeeded" are two different facts**

### Analogy: The Delivery Truck vs. The Warehouse Guard

- The pipeline's **Source stage succeeding** just means the delivery truck picked up the package (zip file) correctly
- The **Deploy stage** is the truck arriving at the warehouse and the guard checking the driver's ID badge before letting them unload
- A failed Deploy stage after a successful Source stage almost always means the "badge" (IAM permissions) wasn't recognized at the warehouse door — the package was fine, the courier just wasn't cleared for that specific building

### Step 9 — Retry after IAM fix: Success

```
krishnaPipeline    Succeeded
```

Both Source and Deploy stages turned green. Verified the live app by reloading the Elastic Beanstalk environment URL.

**Verify from CLI instead of console:**
```bash
# Check the latest pipeline execution status
aws codepipeline list-pipeline-executions \
  --pipeline-name krishnaPipeline \
  --region ap-southeast-1 \
  --max-items 1

# Check Elastic Beanstalk environment health directly
aws elasticbeanstalk describe-environments \
  --environment-names Appbeans-env \
  --query "Environments[0].[Status,Health]"

# Confirm the app responds
curl -s http://appbeans-env.eba-niqir3sn.ap-southeast-1.elasticbeanstalk.com
```

### Step 10 — Full loop test (source of truth for "is this really CI/CD")

```bash
# On EC2
cd ~/python-flask-app
nano application.py        # change the returned HTML message
zip -r app.zip application.py requirements.txt
aws s3 cp app.zip s3://python-cicd1-source/app.zip
```

- No manual "Release change" click
- Pipeline auto-triggers within moments because the S3 object key received a new version
- Deploy stage pushes the change to the live Beanstalk environment automatically

**Confirm the new object version was created (proves the trigger mechanism, not just the upload):**
```bash
aws s3api list-object-versions \
  --bucket python-cicd1-source \
  --prefix app.zip \
  --query "Versions[*].[VersionId,LastModified]"
```

**Watch the pipeline pick it up automatically:**
```bash
watch -n 5 "aws codepipeline get-pipeline-state --name krishnaPipeline --region ap-southeast-1 --query 'stageStates[*].[stageName,latestExecution.status]'"
```
(Run this from a machine with `watch` installed, or manually re-run the `aws codepipeline get-pipeline-state` command every few seconds.)

**Confirm the updated content is live:**
```bash
curl -s http://appbeans-env.eba-niqir3sn.ap-southeast-1.elasticbeanstalk.com
```

---

## Hacker Mindset / Bug Bounty Angle

This is the exact kind of infrastructure a pentester or bug bounty hunter looks at when assessing an organization's cloud CI/CD attack surface. Things worth noting from this build:

- **Over-permissioned service roles are common and dangerous.** Attaching `AdministratorAccess-AWSElasticBeanstalk` (or worse, generic `AdministratorAccess`) to a pipeline role because the narrower policy "didn't work" is exactly how privilege creep happens in real environments. A compromised CodePipeline role with admin-level EB access can create/modify environments, which often means arbitrary code execution on the underlying EC2 instances.
- **EC2 instance roles are a favorite target via SSRF.** If a web app running on this Beanstalk instance had an SSRF vulnerability, an attacker could hit the Instance Metadata Service (`http://169.254.169.254/latest/meta-data/iam/security-credentials/`) and steal the temporary credentials tied to the EC2 role — in this build, that role had `AmazonS3FullAccess`, meaning a successful SSRF chain would hand over full S3 access across the account, not just the one bucket used for deployment.
- **Public S3 buckets used for CI/CD source are a real-world finding category.** If `python-cicd1-source` were misconfigured as publicly writable, an attacker could overwrite `app.zip` directly, and the pipeline would happily deploy attacker-controlled code straight into the production Elastic Beanstalk environment — a textbook supply-chain compromise via CI/CD.
- **IAM PassRole misconfigurations are a known privilege escalation primitive.** A role with broad `iam:PassRole` (`Resource: "*"`) combined with permissions to create Beanstalk environments can sometimes be abused to pass a higher-privileged role into a new resource, effectively escalating privileges — this is a recognized AWS privilege escalation path documented in tools like `pmapper` and various cloud security research.
- **Defensive takeaway:** scope IAM policies to specific bucket ARNs and specific Beanstalk application ARNs rather than reaching for broad AWS-managed "FullAccess"/"AdministratorAccess" policies just to make an error go away — this is the fastest path from "quick fix" to "excessive attack surface."

---

## Quick Reference: Errors and Fixes Summary

| Step | Error | Root Cause | Fix |
|---|---|---|---|
| S3 check from EC2 | `NoSuchBucket` on `ListObjectsV2` | Bucket referenced before creation | `aws s3 mb` + enable versioning |
| Pipeline creation | New pipeline UI showed ECR/ECS templates | Wrong "Deployment" template category selected | Choose "Build custom pipeline" |
| Pipeline creation | Console defaulted to `us-east-1` | Region mismatch with S3 bucket/EB environment | Manually switch region before creating pipeline |
| First pipeline run | Deploy stage: `1 of 1 action failed` | CodePipeline service role missing Elastic Beanstalk permissions | Attach `AdministratorAccess-AWSElasticBeanstalk` to the exact service role from Pipeline Settings |
| (If still failing) | Deploy still fails after policy attach | Missing `iam:PassRole` for EB service/instance roles | Add inline policy allowing `iam:PassRole` |

---

## Notes

- Elastic Beanstalk's naming conventions (`application.py`, `application` variable) are platform-specific gotchas that produce silent failures rather than clear errors — always verify these first before troubleshooting IAM or networking.
- Bucket versioning is not optional for this trigger pattern — it is the mechanism, not a nice-to-have.
- Always confirm the exact service role name via the pipeline's own **Settings** tab rather than guessing from the IAM role list — auto-generated names can be inconsistent across setups.
