---
title: Store Nginx Access Logs in Amazon CloudWatch with Logging operator
linktitle: Amazon CloudWatch with Fluentd
weight: 100
aliases:
    - /docs/one-eye/logging-operator/quickstarts/cloudwatch-nginx/
---

<p align="center"><img src="../../img/nlw.png" alt="Logos" width="340"></p>

This guide describes how to collect application and container logs in Kubernetes using the Logging operator, and how to send them to CloudWatch.

{{< include-headless "quickstart-figure-intro.md" >}}

<p align="center"><img src="../../img/nginx-cloudwatch.png" alt="Architecture" width="900"></p>

## Deploy the Logging operator and a demo Application

Install the Logging operator and a demo application using [Helm](#helm).

### Deploy the Logging operator with Helm {#helm}

{{< include-headless "deploy-helm-intro.md" >}}

1. {{< include-headless "helm-install-logging-operator.md" >}}

1. Create AWS `secret`

    > If you have your `$AWS_ACCESS_KEY_ID` and `$AWS_SECRET_ACCESS_KEY` set you can use the following snippet.

    ```bash
        kubectl -n logging create secret generic logging-cloudwatch --from-literal "awsAccessKeyId=$AWS_ACCESS_KEY_ID" --from-literal "awsSecretAccessKey=$AWS_SECRET_ACCESS_KEY"
    ```

    Or set up the secret manually.

    ```bash
        kubectl -n logging apply -f - <<"EOF"
        apiVersion: v1
        kind: Secret
        metadata:
          name: logging-cloudwatch
        type: Opaque
        data:
          awsAccessKeyId: <base64encoded>
          awsSecretAccessKey: <base64encoded>
        EOF
    ```

1. Create the `logging` resource.

     ```bash
     kubectl -n logging apply -f - <<"EOF"
     apiVersion: logging.banzaicloud.io/v1beta1
     kind: Logging
     metadata:
       name: default-logging-simple
     spec:
       fluentd: {}
       fluentbit: {}
       controlNamespace: logging
     EOF
     ```

     > Note: You can use the `ClusterOutput` and `ClusterFlow` resources only in the `controlNamespace`.

1. Create an CloudWatch `output` definition.

     ```bash
    kubectl -n logging apply -f - <<"EOF"
    apiVersion: logging.banzaicloud.io/v1beta1
    kind: Output
    metadata:
      name: cloudwatch-output
      namespace: logging
    spec:
      cloudwatch:
        aws_key_id:
          valueFrom:
            secretKeyRef:
              name: logging-cloudwatch
              key: awsAccessKeyId
        aws_sec_key:
          valueFrom:
            secretKeyRef:
              name: logging-cloudwatch
              key: awsSecretAccessKey
        log_group_name: operator-log-group
        log_stream_name: operator-log-stream
        region: us-east-1
        auto_create_stream: true
        buffer:
          timekey: 30s
          timekey_wait: 30s
          timekey_use_utc: true
    EOF
     ```

     > Note: In production environment, use a longer `timekey` interval to avoid generating too many objects.

1. Create a `flow` resource.

     ```bash
     kubectl -n logging apply -f - <<"EOF"
     apiVersion: logging.banzaicloud.io/v1beta1
     kind: Flow
     metadata:
       name: cloudwatch-flow
     spec:
       filters:
         - tag_normaliser: {}
         - parser:
             remove_key_name_field: true
             reserve_data: true
             parse:
               type: nginx
       match:
         - select:
             labels:
               app.kubernetes.io/name: log-generator
       localOutputRefs:
         - cloudwatch-output
     EOF
     ```

1. Install log-generator to produce logs with the label `app.kubernetes.io/name: log-generator`

     ```bash
     helm upgrade --install --wait --create-namespace --namespace logging log-generator oci://ghcr.io/kube-logging/helm-charts/log-generator
     ```

1. [Validate your deployment](#validate).

## Validate the deployment {#validate}

<p align="center"><img src="../../img/cw.png" alt="Cloudwatch dashboard"></p>

{{< include-headless "note-troubleshooting.md" >}}
