# Cert Manager on OpenShift with Let's Encrypt and AWS Route 53 for DNS01 validation

This guide will walk you through the steps to setup the Cert Manager Operator on Red Hat OpenShift, using Let's Encrypt and AWS Route53 for DNS01 validation.

To complete this guide, you will need the following:

* An OpenShift cluster, I used a Singlde Node OPenShift instance, running in my home lab.
* An  OpenShift user with cluster-admin rights.
* An AWS account with a hosted DNS zone that the cluster will reside in, `lab.naas.dev` in the case of this guide.
* Sufficient permissions to create users in AWS IAM, and associated security policies, to allow additions and deletions to the DNS zone in question.

## Setting up a DNS01 challenge handler with Route53 and IAM

Hosting DNS zones with Route53 is quite easy and cheap. I host 3 zones there and I am paying less than 2 USD per month to do so. There are other options out there like Cloud Flare, with free plans, but I am not familiar with those.

For this howto, I will be using the CloudShell to interact with AWS. All of this can be done on your local console as well, using the `aws` utility, but that is outside the scope of this document.

![Cloud Shell Link on AWS Console page](img/cloudshell-link.png)

Login on the AWS console and click on the "CloudShell" link in the bottom left on the page. Resize it to your needs.

### Retrieving the DNS Hosted Zone ID

In the CloudShell, the following command will dump info on all of the zones that are hosted in your account.

```shell
~ $ aws route53 list-hosted-zones
{
    "HostedZones": [
        {
            "Id": "/hostedzone/Z0849611A97K9P0IKM96",
            "Name": "lab.naas.dev.",
            "CallerReference": "80a2609f-95d6-4301-9952-fd1413475d1f",
            "Config": {
                "Comment": "Lab environment for naas.dev",
                "PrivateZone": false
            },
            "ResourceRecordSetCount": 17
        },
        {
            "Id": "/hostedzone/Z084274126QD3BWL2TJ1N",
            "Name": "naas.dev.",
            "CallerReference": "fc7390be-d769-4828-864a-47b94da72dd7",
            "Config": {
                "Comment": "Top-level naas.dev domain",
                "PrivateZone": false
            },
            "ResourceRecordSetCount": 7
        }
    ]
}
```

Make a note of the Id of the zone you'll be using, `lab.naas.dev` in my case, so that's `Z0849611A97K9P0IKM96`. You will need that value wwhen it comes to assigning permissions to our new user.

### Creating the IAM user

Next up, let's create the user we'll be assigning the DNS modification permissions to. In the CloudShell, issue the following command:

```shell
~ $ aws iam create-user --user-name acme-dns01
{
    "User": {
        "Path": "/",
        "UserName": "acme-dns01",
        "UserId": "GL9EMDJXEIA3NH9HQI7C4",
        "Arn": "arn:aws:iam::143092053466:user/acme-dns01",
        "CreateDate": "2025-01-19T15:08:14+00:00"
    }
}
```

The name is not super important, if you want to give it a different name, make sure you modify all subsequent references to it.

### Updating the policy file and assigning it to the user

Recall the ZoneID you retrieved for the DNS zone we're using and edit the `aws-policy.json` file:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:Get*",
                "route53:Change*",
                "route53:ListResourceRecordSets"
            ],
            "Resource": "arn:aws:route53:::hostedzone/YOURZONEID" # <-- ADD HERE
        },
        {
            "Effect": "Allow",
            "Action": "route53:ListHostedZones",
            "Resource": "*"
        }
    ]
}
```

For simplicity's sake, we'll be assinging the policy directly to the user. Issue the following command in the CloudShell to do so:

```shell
~ $ aws iam put-user-policy \
>     --user-name acme-dns01 \
>     --policy-name acme-dns01-policy \
>     --policy-document file://aws-policy.json
```

Verify that the policy has been assigned:

```shell
~ $ aws iam list-user-policies --user-name acme-dns01
{
    "PolicyNames": [
        "acme-dns01-policy"
    ]
}
```

You can also assign it to a group, and add the user to that group, but that is outside the scope of this document.

## Setting up and configuring the Certificate Manager

### Installing the operator

You can install the Certificate Manager Opartor thorugh the Operator Hub, or you can use the `operator0-sub.yaml` file I have provided. This will create the cert-manager-operator project, the `openshift-cert-manager-operator` OperatorGroup, and the subscription that will subsequently cause the operator to be installed. You can do so by issuing the following command on your computer's Terminal:

```shell
❯ oc create -f operator-sub.yaml
project.project.openshift.io/cert-manager-operator created
operatorgroup.operators.coreos.com/openshift-cert-manager-operator created
subscription.operators.coreos.com/openshift-cert-manager-operator created
```

You will need to wait a few minutes for the operator to finish installing. You can verify progress by checking the pods in the `certificate-manager-operator` project. When the operator is up and running, you should see something like this:

```shell
❯ oc get pods -n cert-manager-operator -w
NAME                                                        READY   STATUS    RESTARTS   AGE
cert-manager-operator-controller-manager-794755d6b5-85rwb   2/2     Running   0          23s
```

### Setting up the certificate issuer

To set up the certificate issuer, you will need to fetch an API Key ID and secret for the user we created earlier. You can issue the following command in the CloudShell:

```shell
~ $ aws iam create-access-key --user-name acme-dns01
{
    "AccessKey": {
        "UserName": "acme-dns01",
        "AccessKeyId": "AKIAR6V7BNF5NHDKBIVH",
        "Status": "Active",
        "SecretAccessKey": "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",
        "CreateDate": "2025-01-19T15:14:26+00:00"
    }
}
```

Edit the `cluster-issuer.yaml` file to add the ID and Secret to the Secret section, change the email address in the ClusterIssuer section, and create the resources:

```shell
❯ oc create -f cluster-issuer.yaml
secret/aws-credentials created
clusterissuer.cert-manager.io/letsencrypt-dns created
```

Give it a few minutes do its thing. You can check the status of the cluster issuer like this:

```shell
❯ oc describe clusterissuer letsencrypt-dns
Name:         letsencrypt-dns
.
.
.
Status:
  Acme:
    Last Private Key Hash:  [deleted]
    Last Registered Email:  user@example.com
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/[deleted]
  Conditions:
    Last Transition Time:  2025-01-15T14:04:19Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   2
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

## Creating the certificate

Now that the issuer is up and running, let's get a certificate for our default apps ingress. Edit the `certificate-apps.yaml` file and change `*.apps.CLUSTERNAME.EXAMPLE.COM` to whatever the names of your cluster and domain name are. Keep in mind that not all browsers will accept a wildcard certificate like `*.CLUSTERNAME.EXAMPLE.COM` for more than one subzone. Both Safari and Arc gave me an invalid certicate error when trying to use it for `*.apps.CLUSTERNAME.EXAMPLE.COM`. When you're done editing, submit the yaml file:

```shell
❯ oc create -f certificate-apps.yaml
certificate.cert-manager.io/apps-wildcard-cert created
```

Once the certifcate has been created, you should see a secret named `apps-wildcard-cert-tls` of the type `kubernetes.io/tls` appear in the `openshift-ingress` namespace:

```shell
❯ oc get secrets -n openshift-ingress
NAME                           TYPE                DATA   AGE
apps-wildcard-cert-tls         kubernetes.io/tls   2       10s   <----- our certificate!
router-metrics-certs-default   kubernetes.io/tls   2      169d
router-stats-default           Opaque              2      169d
```

Next, check the status of the secret:

```shell
❯ oc describe certificate apps-wildcard-cert -n openshift-ingress
Name:         apps-wildcard-cert
Namespace:    openshift-ingress
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2025-01-18T09:44:56Z
  Generation:          1
  Resource Version:    125668195
  UID:                 aecff206-d00d-4e53-add3-c0ffee26a9b1
Spec:
  Common Name:  *.apps.omnius.lab.naas.dev
  Dns Names:
    *.apps.omnius.lab.naas.dev
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       letsencrypt-dns
  Secret Name:  apps-wildcard-cert-tls
Status:
  Conditions:
    Last Transition Time:  2025-01-18T09:46:39Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-04-18T08:48:07Z
  Not Before:              2025-01-18T08:48:08Z
  Renewal Time:            2025-03-19T08:48:07Z
  Revision:                1
Events:                    <none>
```

You can patch the Ingress configuration to make use of it:

```shell
❯ oc patch ingresscontroller default -n openshift-ingress-operator --type=merge -p '{
  "spec": {
    "defaultCertificate": {
      "name": "apps-wildcard-cert-tls"
    }
  }
}'
```

If it looks good, you can restart the default ingress pods with the following command:

```shell
❯ oc delete pod -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
pod "router-default-df6745d55-cw92r" deleted
```

The new pods will appear shortly. When they're back up, use `curl` to verify that the connection has been secured with your new certiftcate: 

```shell
❯ curl -v https://console-openshift-console.apps.omnius.lab.naas.dev 2>&1 | grep '^* SSL' -A10
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
* ALPN: server did not agree on a protocol. Uses default.
* Server certificate:
*  subject: CN=*.apps.omnius.lab.naas.dev
*  start date: Jan 18 08:48:08 2025 GMT
*  expire date: Apr 18 08:48:07 2025 GMT
*  subjectAltName: host "console-openshift-console.apps.omnius.lab.naas.dev" matched cert's "*.apps.omnius.lab.naas.dev"
*  issuer: C=US; O=Let's Encrypt; CN=R11
*  SSL certificate verify ok.
* using HTTP/1.x
> GET / HTTP/1.1
```

The subject, start and expiration dates should all match the output of the `oc describe certifcate...` command we used earlier.

## Setting up a certicate for the API gateway

Now that you've been emboldened by the succesful replacement of the self-signed certifcate on the apps ingress, you can choose to do the same with your API gateway. It's not as scary as it sounds. When I messed it up during the making of this guide, OpenShift very conveniently reset the configuration to use a self-signed certficate. 

### Create the certificate

Edit the `certificate-apiserver.yaml` file and substitute your own values where you see `api.CLUSTERNAME.EXAMPLE.COM`. You'll notice that the namespace fopr this certificate is `openshift-config`, not `openshift-apiserver`. Apply the file:

```shell
❯ oc create -f certificate-apiserver.yaml
certificate.cert-manager.io/apiserver-cert created
```

Once the certifcate request has gone through, you should see the new secret appear in the `openshift-config` namespace:

```shell
❯ oc get secrets -n openshift-config
NAME                                      TYPE                             DATA   AGE
apiserver-cert-tls                        kubernetes.io/tls                2      2d19h
etcd-client                               kubernetes.io/tls                2      169d
etcd-metric-signer                        kubernetes.io/tls                2      169d
etcd-signer                               kubernetes.io/tls                2      169d
htpass-secret                             Opaque                           1      168d
initial-service-account-private-key       Opaque                           1      169d
pull-secret                               kubernetes.io/dockerconfigjson   1      169d
webhook-authentication-integrated-oauth   Opaque                           1      169d
```

Check the state of the certificate, as before:

```shell
❯ oc describe certificate apiserver-cert -n openshift-config
Name:         apiserver-cert
Namespace:    openshift-config
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2025-01-18T13:42:31Z
  Generation:          1
  Resource Version:    125788656
  UID:                 7acad180-d00d-4e53-add3-c0ffee26a9b1
Spec:
  Common Name:  api.omnius.lab.naas.dev
  Dns Names:
    api.omnius.lab.naas.dev
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       letsencrypt-dns
  Secret Name:  apiserver-cert-tls
Status:
  Conditions:
    Last Transition Time:  2025-01-18T13:42:31Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-04-18T12:27:44Z
  Not Before:              2025-01-18T12:27:45Z
  Renewal Time:            2025-03-19T12:27:44Z
Events:                    <none>
```

If you're happy with how that looks, it's time to make the switch!

```shell
> oc patch apiserver cluster --type=merge -p '{
  "spec": {
    "servingCerts": {
      "namedCertificates": [
        {
          "names": ["api.omnius.lab.naas.dev"],
          "servingCertificate": {
            "name": "apiserver-cert-tls"
          }
        }
      ]
    }
  }
}'
```

Restart the API server pods by deleting the old ones:

```shell
oc delete pod -n openshift-apiserver -l app=openshift-apiserver-a
```

After a brief instant, the operator will recreate the apiserver pods with the new certificates. Check that they are using the new configuration:

```shell
❯ oc get apiserver cluster -o json | jq .spec.servingCerts
{
  "namedCertificates": [
    {
      "names": [
        "api.omnius.lab.naas.dev"
      ],
      "servingCertificate": {
        "name": "apiserver-cert-tls"
      }
    }
  ]
}
```

With `curl` you can check that the API server is indeed using the new certifdicate for SSL connections:

```shell
❯ curl -v https://api.omnius.lab.naas.dev:6443 2>&1 | grep '^* SSL' -A10
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=api.omnius.lab.naas.dev
*  start date: Jan 18 12:27:45 2025 GMT
*  expire date: Apr 18 12:27:44 2025 GMT
*  subjectAltName: host "api.omnius.lab.naas.dev" matched cert's "api.omnius.lab.naas.dev"
*  issuer: C=US; O=Let's Encrypt; CN=R11
*  SSL certificate verify ok.
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://api.omnius.lab.naas.dev:6443/
```

## Reverting back to self-signed certificates

You can very easily remove the CA-signed certificates and reverting back to the selfg-signed certificates with the following commands.

For the ingress, issue the following commands:

```shell
> oc patch ingresscontroller default -n openshift-ingress-operator --type=merge -p '{
  "spec": {
    "defaultCertificate": null
  }
}'
❯ oc delete pod -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
pod "router-default-df6745d55-fw1342" deleted
```

For the API server, do the following:

```shell
> oc patch apiserver cluster --type=merge -p '{"spec":{"servingCerts":{"namedCertificates":[]}}}'
> oc delete pod -n openshift-apiserver -l app=openshift-apiserver-a
```

## Troubleshooting

Sometimes, things don't go as we planned, below is a list of things that I bumped into during the process of making this tutorial.

TBD
