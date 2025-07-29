---
title:  "Using IRSA/EKS Pod Identity for Old AWS SDKs: A Sidecar Solution"
last_modified_at:   2025-07-29 11:07:26 -0700
categories: 
 - dev
tags:
 - security
 - kubernetes
 - irsa
 - aws
---

If you've been working with Kubernetes on AWS for a while, you've probably run into this scenario: you're trying to use modern AWS authentication mechanisms like IRSA (IAM Roles for Service Accounts) or EKS Pod Identity, but your application is stuck with an ancient AWS SDK that doesn't support these features. I've outlined a solution here allows you to leverage the latest authentication mechanisms with ANY verison of AWS SDK.

## The Problem: Legacy SDKs Don't Play Nice with Modern Auth

I've hit this wall more times than I care to count. The most recent example? Harbor, the popular container registry. Harbor leverages the [distribution project](github.com/distribution/distribution) version 2.8.3, which uses [AWS SDK Go version v1.15.11](https://github.com/aws/aws-sdk-go/tree/v1.15.11) from way back in 2018. This old SDK has some serious limitations:

- No support for IRSA or EKS Pod Identity
- Can't override the `AWS_EC2_METADATA_SERVICE_ENDPOINT` environment variable for proxying
- Forces you into workarounds that nobody really wants to deal with

In the past, I've had to fall back to kube2iam for authentication without hardcoding credentials. While kube2iam works, it's far from ideal. You end up having to associate every IAM role with the node role you're running on, which creates a management nightmare and doesn't follow the principle of least privilege.

## The Solution: Sidecar Container Magic

After banging my head against this problem repeatedly, There's a clean workaround using a sidecar container. The key insight is leveraging native sidecar container support with an init container that has a restart policy of 'always'.

Here's how it works:

1. **Create a shared volume** where your main application container can read the `~/.aws/credentials` file
2. **Deploy a sidecar container** that has access to modern AWS authentication
3. **Let the sidecar handle credential management** while your legacy SDK reads from the traditional credentials file
## Architecture

![Credentials Sidecar Architecture]({{ site.baseurl }}/assets/images/credentials-sidecar.png)

## Implementation Details

The sidecar approach works by creating a volume mount that both containers can access. In Harbor's case, since the container runs with the `harbor` user, the credentials need to be accessible at `/home/harbor/.aws/credentials`. For other applications, this might be `/root/.aws/credentials` or wherever your application expects to find them.

Here's the actual Kubernetes deployment configuration example (with irrelevant parts ommited):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: harbor-registry
  namespace: harbor
spec:
  template:
    spec:
      containers:
      - image: goharbor/registry-photon:v2.12.1
        volumeMounts:
        - mountPath: /home/harbor/.aws
          name: aws-credentials
      - image: goharbor/harbor-registryctl:v2.12.1
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: /home/harbor/.aws
          name: aws-credentials
      initContainers:
      - command:
        - /bin/sh
        - -c
        - |
          set -e

          # Function to get credentials
          get_credentials() {
            echo "Getting AWS credentials..."
            CREDS=$(aws sts assume-role --role-arn $AWS_ROLE_ARN --role-session-name harbor-session --duration-seconds 3600 --output text)
            ACCESS_KEY=$(echo "$CREDS" | grep CREDENTIALS | cut -f2)
            SECRET_KEY=$(echo "$CREDS" | grep CREDENTIALS | cut -f4)
            SESSION_TOKEN=$(echo "$CREDS" | grep CREDENTIALS | cut -f5)

            cat > /home/harbor/.aws/credentials << EOF
          [default]
          aws_access_key_id = $ACCESS_KEY
          aws_secret_access_key = $SECRET_KEY
          aws_session_token = $SESSION_TOKEN
          EOF

            echo "Credentials updated at $(date)"
          }

          get_credentials

          while true; do
            sleep 1800  # 30 minutes
            get_credentials
          done &

          # Keep container running
          tail -f /dev/null
        image: public.ecr.aws/aws-cli/aws-cli:latest
        imagePullPolicy: Always
        name: aws-credentials-refresh
        restartPolicy: Always
        securityContext:
          runAsUser: 0
        volumeMounts:
        - mountPath: /home/harbor/.aws
          name: aws-credentials
      serviceAccount: serviceaccountname
      volumes:
      - emptyDir: {}
        name: aws-credentials
```

### Key Components

**Init Container with restartPolicy: Always**: This is the magic that makes it work. The init container runs continuously, refreshing credentials every 30 minutes to ensure they don't expire.

**Shared emptyDir Volume**: Both the main application containers and the credential sidecar mount the same volume at `/home/harbor/.aws`, allowing credential sharing.

**AWS STS Assume Role**: The sidecar uses `aws sts assume-role` to generate temporary credentials, which requires your IAM role to have self-assumption capabilities.

**Automatic Refresh**: The background process ensures credentials are refreshed before they expire, preventing authentication failures. The Credentials are valid for 1 hour, but we refresh them every 30 minutes

## IAM Role Configuration

For this pattern to work, you need to configure your IAM role properly. The role needs both IRSA/Pod Identity settings and self-assumption capabilities. Your trust policy needs to handle two scenarios:

1. **IRSA/Pod Identity assumption**: Allow the Kubernetes service account to assume the role
2. **Self-assumption**: Allow the role to assume itself for credential refresh

Here's what your IAM role trust policy should look like for **IRSA**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR-ACCOUNT-ID:oidc-provider/oidc.eks.YOUR-REGION.amazonaws.com/id/YOUR-CLUSTER-OIDC-ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.YOUR-REGION.amazonaws.com/id/YOUR-CLUSTER-OIDC-ID:sub": "system:serviceaccount:namespace:serviceaccountname",
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR-ACCOUNT-ID:role/YOUR-ROLE-NAME"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Or for **EKS Pod Identity**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR-ACCOUNT-ID:role/YOUR-ROLE-NAME"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

And the permission policy needs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ],
      "Resource": "arn:aws:iam::YOUR-ACCOUNT-ID:role/YOUR-ROLE-NAME"
    }
  ]
}
```

This self-assumption pattern allows the `aws sts assume-role` command in the sidecar to work properly, generating the temporary credentials that get written to the shared credentials file.

## Adapting the Pattern

This pattern works well beyond Harbor. The same approach can be applied to any application stuck with an old AWS SDK:

This solution addresses several pain points:

- **Security**: You get the security benefits of IRSA/Pod Identity without modifying legacy code
- **Maintainability**: No need to fork old projects or maintain custom patches
- **Scalability**: Works across multiple applications with the same authentication challenges
- **Future-proofing**: When you eventually upgrade your SDKs, you can simply remove the sidecar

## Real-World Impact

Since implementing this approach, I've been able to modernize authentication for several legacy applications without touching their core code. It's particularly valuable in enterprise environments where you're dealing with a mix of old and new applications, all needing to follow the same security standards.

The sidecar pattern has proven reliable in production, handling credential rotation seamlessly and providing the security posture that modern AWS deployments require.

## Final Thoughts

Look, I get it - dealing with legacy code can be incredibly frustrating.

This sidecar solution has honestly been a lifesaver. I can just drop in this pattern and move on with my life.
If you're pulling your hair out trying to modernize authentication for legacy apps, give this approach a shot.  