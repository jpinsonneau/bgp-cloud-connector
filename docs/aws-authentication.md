# AWS authentication

The operator needs AWS credentials to discover Route Server infrastructure and reconcile cloud-side resources (see [cloud-integration.md](cloud-integration.md)). It obtains them in one of two ways and works out which at runtime — there is no platform detection. This page describes the **recommended** setup and an **alternative** for the cases where it does not fit.

## How the operator chooses

On startup the operator asks the AWS SDK whether the pod already has usable credentials: it builds the default credential chain and forces a `Retrieve` (`ResolveCredentials` → `ambientCredentials` in `internal/platform/aws/credentials.go`). Loading the chain is not enough — it is only truly resolved by retrieving from it.

- **Nothing to retrieve →** the operator creates a `CredentialsRequest` and lets the **Cloud Credential Operator (CCO)** provide credentials. This is the **[recommended path](#recommended-cco-managed-credentials)**.
- **Credentials already present →** the operator uses them and asks the cluster for nothing. This happens when you have set up the **[alternative IRSA path](#alternative-pod-injected-irsa-credentials)** (an annotated ServiceAccount, so the pod identity webhook injects a token).

You do not choose a path directly — you perform one of the setups below, and the runtime probe lands on the matching one.

**CCO = Cloud Credential Operator**, the standard OpenShift operator (namespace `openshift-cloud-credential-operator`) that provisions cloud credentials from a `CredentialsRequest`. Its `credentialsMode` (Mint / Passthrough / Manual) determines what it can do. The operator is packaged for the CCO flow: its CSV declares `features.operators.openshift.io/token-auth-aws: "true"`, so an OperatorHub install prompts for the role ARN and wires up CCO for you.

## Which setup applies

| Your cluster | Recommended setup | Details |
|:---|:---|:---|
| **IPI, mint mode** (`credentialsMode` unset or `Mint`) | **Nothing** — CCO mints an IAM user automatically | [Recommended → Mint mode](#mint-mode-ipi) |
| **Manual / STS via `ccoctl`, ROSA, OSD, OCP Core** (OIDC configured) | Set `ROLEARN` (the OperatorHub console sets it for you); the IAM role must pre-exist | [Recommended → Manual/STS mode](#manual--sts-mode) |
| CCO unavailable/disabled, already standardized on IRSA, or quick dev/test | Annotate the operator's ServiceAccount (IRSA) | [Alternative](#alternative-pod-injected-irsa-credentials) |

---

## Recommended: CCO-managed credentials

The operator creates a `CredentialsRequest` named `bgp-cloud-connector-aws` carrying exactly the permissions it needs, and reads the `credentials` key of the secret CCO writes into the operator's namespace. That key is a shared-credentials ini file, and CCO writes one whatever mode the cluster is in — which is why this single path serves both sub-cases below. Nothing static is stored, and there is no pod-restart gotcha: the operator reconciles its own `CredentialsRequest`, so a `ROLEARN` set after startup is picked up on the next pass.

### Mint mode (IPI)

`credentialsMode` unset or `Mint`. **Nothing to set up.** CCO creates an IAM user and puts its key pair in the secret. The `BGPCloudConfiguration` reports `CloudEndpointsDiscovered=False` with reason `WaitingForCloudCredentials` for the few seconds this takes, then proceeds.

### Manual / STS mode

`credentialsMode: Manual` with an OIDC provider — what `ccoctl` installs, and what ROSA uses. CCO cannot mint anything here, so you give the operator an IAM role ARN and CCO federates a short-lived token against it.

**Step 1 — Create the IAM role** (see [Create the IAM role and policy](#create-the-iam-role-and-policy) below). The role must exist first; only you can create it, because the operator has no credentials with which to create a role for itself.

**Step 2 — Give the operator the role ARN via `ROLEARN`:**

- **Installing from OperatorHub:** the console prompts for the role ARN and sets `ROLEARN` for you (driven by the CSV's `features.operators.openshift.io/token-auth-aws` annotation).
- **Installing by hand:** put it in the Subscription:

  ```yaml
  spec:
    config:
      env:
      - name: ROLEARN
        value: arn:aws:iam::<account>:role/<role>
  ```

The operator then adds `stsIAMRoleARN` and `cloudTokenPath` to its `CredentialsRequest` (both together — CCO reads a role ARN without a token path as a request it cannot serve), and CCO writes a secret naming that role and the token the operator projects at `/var/run/secrets/openshift/serviceaccount/token`.

---

## Alternative: pod-injected IRSA credentials

Instead of CCO, you can annotate the operator's ServiceAccount with an IAM role ARN; the pod identity webhook then injects a web-identity token at pod creation and the SDK's default credential chain resolves it. Use this when:

- **CCO is unavailable or disabled** on the cluster (removed, or in a mode that will not provision the request).
- The cluster is **already standardized on IRSA / pod identity** and you prefer not to involve CCO.
- You want a **quick dev/test setup** without going through a Subscription or OperatorHub `ROLEARN` flow.

**Step 1 — Create the IAM role** (see [Create the IAM role and policy](#create-the-iam-role-and-policy) below).

**Step 2 — Annotate the operator's ServiceAccount:**

```bash
oc annotate serviceaccount openshift-bgp-cloud-connector-controller-manager \
  -n openshift-bgp-cloud-connector \
  eks.amazonaws.com/role-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:role/bgp-cloud-connector
```

The OIDC webhook automatically injects `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` environment variables into the operator pod. The AWS SDK's default credential chain picks these up — no explicit credential configuration is needed in the CR.

> **Limitation — restart required after credential changes.** The OIDC webhook only injects credentials at **pod creation time**. If the operator is already running when you complete the IRSA setup (or if you correct a misconfigured IAM role or ServiceAccount annotation), the running pod will not pick up the new credentials. Restart the operator:
>
> ```bash
> oc rollout restart deployment/openshift-bgp-cloud-connector-controller-manager -n openshift-bgp-cloud-connector
> ```
>
> The recommended CCO path does not have this limitation — it re-reads its own `CredentialsRequest` on each reconcile.

---

## Create the IAM role and policy

Both the recommended Manual/STS path and the IRSA alternative need the same IAM role, with the same trust-policy shape. Create it once.

**Create the role with a trust policy for the operator's ServiceAccount:**

```bash
# Get the cluster's OIDC provider (works on any OCP cluster: ROSA, OSD, OCP Core)
OIDC_PROVIDER=$(oc get authentication cluster -o jsonpath='{.spec.serviceAccountIssuer}' | sed 's|https://||')
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-role --role-name bgp-cloud-connector \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Federated": "arn:aws:iam::'$AWS_ACCOUNT_ID':oidc-provider/'$OIDC_PROVIDER'"},
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "'$OIDC_PROVIDER':sub": "system:serviceaccount:openshift-bgp-cloud-connector:openshift-bgp-cloud-connector-controller-manager"
        }
      }
    }]
  }'
```

> `serviceAccountIssuer` is the cluster's own OIDC endpoint, read from its config. It returns the same value ROSA's `rosa describe cluster -c <name> -o json | jq -r '.aws.sts.oidc_endpoint_url'` does, without depending on the `rosa` CLI, so this command works on ROSA, OSD, and OCP Core alike.

**Attach the required permissions policy:**

```bash
aws iam put-role-policy --role-name bgp-cloud-connector \
  --policy-name bgp-cloud-connector-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "sts:GetCallerIdentity",
          "ec2:DescribeRouteServers",
          "ec2:DescribeRouteServerEndpoints",
          "ec2:DescribeSubnets",
          "ec2:DescribeRouteServerPeers",
          "ec2:CreateRouteServerPeer",
          "ec2:DeleteRouteServerPeer",
          "ec2:CreateTags",
          "ec2:DescribeInstances",
          "ec2:ModifyNetworkInterfaceAttribute"
        ],
        "Resource": "*"
      }
    ]
  }'
```

Complete this **before** creating the `BGPCloudConfiguration` CR with `spec.aws`.

---

## Troubleshooting

The operator reports credential state through the `CloudEndpointsDiscovered` condition on `BGPCloudConfiguration`:

| Reason | Meaning | What to do |
|:---|:---|:---|
| `CloudCredentialsInvalid` | Credentials were found but `sts:GetCallerIdentity` failed. | Check the IAM role trust policy and permissions, or the secret CCO wrote. A freshly minted IAM key can fail this for a few seconds before it propagates; the reconcile requeues after 30s and it clears. |
| `WaitingForCloudCredentials` | The cluster has been asked for credentials and has not yet provided them. Not `Degraded` — the CR stays `Configuring` and requeues every 10s. | If it never clears on a Manual/STS cluster, `ROLEARN` is unset — CCO ignores a request without `stsIAMRoleARN`. |

If you used the IRSA alternative and changed credentials while the operator was running, restart it (see the [restart limitation](#alternative-pod-injected-irsa-credentials)) — the webhook-injected path only resolves at pod creation.

Inspect the live conditions:

```bash
oc get bgpcloudconfiguration cluster -o jsonpath='{.status.conditions}' | jq .
```
