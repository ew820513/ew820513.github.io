---
title: "Troubleshooting EKS pods with AWS IAM"
date: 2023-05-12T00:00:09-07:00
draft: false
---

## Introduction

Recently, I came across an issue where Kubernetes pods fails to assume AWS IAM role assigned to the pods via Service Account, all without authentication error messages. Instead, the pod picks up the default IAM role assigned to the EC2 node. I am going to document the triaging process. Although. most of which can be found [here](https://repost.aws/knowledge-center/eks-pods-iam-role-service-accounts), I hope to add some personal knowledge and the ultimate cause in this blog to help you troubleshoot similar issues. Disclaimer, it was an issue I caused myself, but it took me a while to figure out what went wrong.

IAM role for service account is a feature that allows you to associate an IAM role with a Kubernetes service account similar to EC2 instance profile to EC2 instance. Instead of having to manage credentials in your application code, pods that use the service account automatically assume the role that is assigned to the service account as credentials. This role-based access control (_RBAC_*) feature allows you to benefit from IAM's fine-grained access control and permission management, such that you can grant different pods different permissions to AWS resources isolating pods permissions.

## How EKS pods assume IAM role

Before we dive into the triaging process, let's first understand how EKS pods assume IAM role.

When you create an EKS cluster, an *iam-for-pod MutatingWebHookConfiguration* is created in the cluster. The webhook is configured to intercept pod creation requests for pods that use the aws EKS *service account annotation* to include the *IAM role* information. When a *pod* is created, the Kubelet calls the webhook to mutate the pod spec. The webhook adds *the environment variables* to the pod spec and mounts the web identity token file to the pod. The webhook adds the following environment variables to the pod spec:

```yaml
AWS_STS_REGIONAL_ENDPOINTS: regional
AWS_DEFAULT_REGION: region
AWS_ROLE_ARN: role-arn
AWS_WEB_IDENTITY_TOKEN_FILE: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

When the pod makes an AWS API call, the AWS SDK automatically searches for the environment variables and uses the web identity token file to assume the IAM role that is associated with the Kubernetes service account. AWS then authenticates the pod using the *OIDC provider* and returns the temporary credentials associated with the role to the pod. The pod can then use the temporary credentials to make AWS API calls.

## Triage process

In the previous section, I've bolded the different components that are involved in the process. We triage the issue by investigating these components.

### 0. Check the pod status, events and logs

Logs are often the first place to look for when troubleshooting. You can view the pod logs running the following commands.

```bash
# locate the pod and namespace
kubectl get pods -A
kubectl logs -n <namespace> <pod-name>
```

In my case here, the logs indicated that it can assume the EKS node role instead of the IAM role assigned to the service account. Indicated an issue with the config that is deeper embedded into the EKS cluster; however, I will still cover the triaging process for every component involved for completeness.

### 1. Check the OIDC provider

Check your OIDC provider configs. OICD provider is used to authenticate the roles inside EKS cluster with AWS IAM role. You can verify the OIDC provider configs by running the following steps.

1. Open the IAM console, and then choose Identity Providers from the navigation pane.
1. In the Provider column, identify and note the OIDC provider URL.
1. In a separate tab or window, open the Amazon EKS console, and then choose Clusters from the navigation pane.
1. Choose your cluster, and then choose the Configuration tab.
1. In the Details section, note the value of the OpenID Connect provider URL property.
1. Verify that the OIDC provider URL from the Amazon EKS console (step 5) matches the OIDC provider URL from the IAM console (step 2).

### 2. Check the IAM role assigned to the service account

Check the IAM role assigned to the service account. You can verify the IAM role assigned to the service account by running the following command.

```bash
aws iam get-role --role-name <role-name>
```

Verify that the format of your policy matches the format of the following JSON policy

```json
"Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::your-account-id:oidc-provider/oidc.eks.your-region-code.amazonaws.com/id/EXAMPLE_OIDC_IDENTIFIER"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.your-region-code.amazonaws.com/id/EXAMPLE_OIDC_IDENTIFIER:sub": "system:serviceaccount:your-namespace:your-service-account"
        }
      }
    }
  ]
}
```

### 3. Check the service account annotation

It could be an issue with your EKS setup. You can verify the IAM role assigned to the service account by running the following command.

```bash
kubectl get serviceaccount YOUR_ACCOUNT_NAME -n YOUR_NAMESPACE -o yaml
```

Verify that the format of your policy matches the format of the following JSON policy. Ensure that the role ARN is correct.

```json
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::012345678912:role/my-example-iam-role
  name: my-example-serviceaccount
  namespace: my-test-namespace

```

### 4. Check the pod spec

Check the pod spec as well. Make sure that the pod spec includes the service account name and namespace. You can verify the pod spec by running the following command.

```bash
kubectl get pod YOUR_ACCOUNT_NAME -n YOUR_NAMESPACE -o yaml
```

### 6. Check the iam-for-pod MutatingWebHookConfiguration

When you check the pod spec, you should also see multiple environment variables that are added by the iam-for-pod mutating webhook. These include, but are not limited to the following.

```yaml
AWS_ROLE_ARN=arn:aws:iam::ACCOUNT_ID:role/IAM_ROLE_NAME
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

You can verify the iam-for-pod MutatingWebHookConfiguration by running the following command.

```bash
kubectl get mutatingwebhookconfiguration iam-for-pod -o yaml
```

`mutatingwebhookconfiguration iam-for-pod` does not normally change, so you shall rarely run into any issues with it. For me, I was patching the mutatingwebhookconfiguration as I was preparing for deprecated API versions, and I accidentally copy-pasted the wrong ca-bundle data. This caused the webhook to fail to mutate the pod spec. Yes, I was the culprit, but I've also learned a lot more about the RMAC process. I hope this article will help you to triage your issue faster.