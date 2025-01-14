---
title: "Kubernetes Admission Controller"
linkTitle: "Kubernetes Admission Controller"
weight: 10
description: >
    Configure Terrascan as a Kubernetes Admission Controller
---


## Overview
Terrascan can be integrated with K8s [admissions webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/).
Admission controllers help you control resources created on a kubernetes cluster. By using Terrascan as an admission controller, resources violating security policies can be prevented from getting created in a Kubernetes cluster. For more details and instructions, [click here to see our blog](https://www.accurics.com/blog/terrascan-blog/kubernetes-security-terrascan-validating-admission-controller/).

> **Note on SSL certificates**: You can use valid SSL certificates or create self signed certificates and have your Kubernetes cluster trust it.

## Installation Guide

To configure Terrascan as an admission controller, follow these steps:

1. Create an instance of Terrascan which meets specified requirements as detailed below. Ensure Terrascan is accessible via HTTPS from the kubernetes API server.
2. Create Terrascan config file.
3. Run Terrascan in server mode.
4. Configure a [ValidatingWebhookConfiguration](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#validatingwebhookconfiguration-v1-admissionregistration-k8s-io) resource in kubernetes cluster pointing to the Terrascan server.
6. Test your set up.

### Step 1: Create an instance of Terrascan
Create an instance of Terrascan. To scan K8s configurations, your Terrascan instance *must* meet the following requirements.

1. Make sure Terrascan is accessible via HTTPS. Ensure your cloud firewall is configured to allow this.
2. Have a valid SSL certificate for the served domain name using one of the suggested methods below:
    - Use a subdomain of your choice (e.g dev-terrascan-k8s.accurics.com) and create a valid certificate for this subdomain through your SSL certificate provider. You can use [Let's Encrypt](https://letsencrypt.org/) which is a free, simple to use certificate authority.
    - Use a reverse-proxy to serve SSL requests; for example, use Cloudflare Flexible to get a certificate by a trusted-CA to your [self-signed certificate](https://www.digitalocean.com/community/tutorials/openssl-essentials-working-with-ssl-certificates-private-keys-and-csrs).
    - Generate a self-signed certificate and configure your K8s cluster to trust it. To add a trusted CA to ca-pemstore, as demonstrated in [paraspatidar's blog post](https://medium.com/@paraspatidar/add-ssl-tls-certificate-or-pem-file-to-kubernetes-pod-s-trusted-root-ca-store-7bed5cd683d).
3. Use the Terrascan docker as demonstrated in this document, or run it from the sources.

### Step 2: Create a Terrascan config file

For instructions to create a config file, see [Usage](../../usage/config_options/)
. You can create a config file that specifies which policies to use in the scan and which violations should be rejected during admission.

- Policies below the [severity] level will be ignored.
- Policies below the [k8s-admission-control] denied-severity will be logged and displayed by Terrascan, but will not lead to a rejected admission response to the k8s API server.

#### Sample config file

A config file example: ```config.toml```

``` Bash
  [severity]
  level = "medium"
  [rules]
      skip-rules = [
          "accurics.kubernetes.IAM.107"
      ]

  [k8s-admission-control]
    denied-categories = [
        "Network Ports Security"
    ]
    denied-severity = "high"
    dashboard=true
```

You can specify the following configurations:

*  **scan-rules** - one or more rules to scan
*  **skip-rules** - one or more rules to skip while scanning
*  **severity** - the minimal level of severity of the policies to be scanned and displayed. Options are high, medium and low
*  **category** - the list of type of categories of the policies to be scanned and displayed

**k8s-admission-control** - Config options for K8s Admission Controllers and GitOps workflows:

*  **denied-severity** - Violations of this or higher severity will cause and admission rejection. Lower severity violations will be warnings. Options are high, medium. and low
*  **denied-categories** - violations from these policy categories will lead to an admission rejection. Policy violations of other categories will lead to warnings.
*  **dashboard=true** - enable the `/logs` endpoint to  log and graphically display admission requests and violations. Default is `false`

### Step 3: Run Terrascan in Server Mode
Run Terrascan docker image in your server using the following command:

 ``` Bash
  sudo docker run -p 443:9443 -v <DATA_PATH>:/data -u root -e K8S_WEBHOOK_API_KEY=<API_KEY> accurics/terrascan server --cert-path /data/cert.pem --key-path /data/key.pem -c /data/config.toml
 ```
Where,

- `<API_KEY>` is a key used for authentication between your K8s environment and  the Terrascan server. Generate your preferred key and use it here.
- `<DATA_PATH>` is a directory path in your server where both the certificate and the private key .pem files are stored.
This directory is also used to save the webhook logs. (Logs are in saves in SQLite file format)

### Step 4: Configure a ValidatingWebhookConfiguration Resource in Kubernetes Cluster
Configure a new ```ValidatingWebhookConfiguration``` in your Kubernetes environment and specify your Terrascan server endpoint.

Example:

``` Bash
  cat <<EOF | kubectl apply -f -
  apiVersion: admissionregistration.k8s.io/v1
  kind: ValidatingWebhookConfiguration
  metadata:
    name: my.validation.example.check
  webhooks:
    - name: my.validation.example.check
      rules:
        - apiGroups:
            - ""
          apiVersions:
            - v1
          operations:
            - CREATE
            - UPDATE
          resources:
            - pods
            - services
      failurePolicy: Fail
      clientConfig:
        url: https://<SERVER_ADDRESS>/v1/k8s/webhooks/<API_KEY>/scan
      sideEffects: None
      admissionReviewVersions: ["v1"]
  EOF
```

* You can modify the `rules` that trigger the webhook according to your preferences.
* Update the ```clientConfig``` URL with your terrascan server address and the API key you generated before.


### Step 5: Test your settings
Try to run a new pod / service. For example:

``` Bash
  kubectl run mynginx --image=nginx
```

Go to ```https://<SERVER_ADDRESS>/k8s/webhooks/<API_KEY>/logs``` and verify your request is logged.
