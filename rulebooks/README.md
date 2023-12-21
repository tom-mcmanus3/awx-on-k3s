<!-- omit in toc -->
# [Experimental] Integrate AWX with EDA Server

The guide to deploy and use EDA Server with AWX on K3s.

In this guide, [EDA Server Operator](https://github.com/ansible/eda-server-operator) is used to deploy EDA Server.

**Note that [EDA Server Operator](https://github.com/ansible/eda-server-operator) is not a fully supported installation method for EDA Server since it's not listed in [the deployment guide](https://github.com/ansible/eda-server/blob/main/docs/deployment.md).**

- [Ansible Blog | Ansible.com | Event-Driven Ansible](https://www.ansible.com/blog)
- [Welcome to Ansible Rulebook documentation — Ansible Rulebook Documentation](https://ansible.readthedocs.io/projects/rulebook/en/latest/)
- [ansible/eda-server](https://github.com/ansible/eda-server)
- [ansible/eda-server-operator](https://github.com/ansible/eda-server-operator)

<!-- omit in toc -->
## Table of Contents

- [Prerequisites](#prerequisites)
- [Deployment Instruction](#deployment-instruction)
  - [Install EDA Server Operator](#install-eda-server-operator)
  - [Prepare required files to deploy EDA Server](#prepare-required-files-to-deploy-eda-server)
  - [Deploy EDA Server](#deploy-eda-server)
- [Demo: Use EDA Server](#demo-use-eda-server)
  - [Configure EDA Server](#configure-eda-server)
    - [Issue new token for AWX and add it on EDA Server](#issue-new-token-for-awx-and-add-it-on-eda-server)
    - [Add Decision Environment on EDA Server](#add-decision-environment-on-eda-server)
    - [Add Project on EDA Server](#add-project-on-eda-server)
    - [Activate Rulebook](#activate-rulebook)
    - [Deploy Ingress resource for the webhook](#deploy-ingress-resource-for-the-webhook)
  - [Trigger Rule using Webhook](#trigger-rule-using-webhook)
  - [Appendix: Use MQTT as a source](#appendix-use-mqtt-as-a-source)

## Prerequisites

EDA Server is designed to use with AWX, so we have to have working AWX instance. Refer to [the main guide on this repository](../README.md) to deploy AWX on K3s.

## Deployment Instruction

### Install EDA Server Operator

Clone this repository and change directory.

```bash
cd ~
git clone https://github.com/kurokobo/awx-on-k3s.git
cd awx-on-k3s
```

Then invoke `kubectl apply -k rulebooks/operator` to deploy EDA Server Operator.

```bash
kubectl apply -k rulebooks/operator
```

The EDA Server Operator will be deployed to the namespace `eda`.

```bash
$ kubectl -n eda get all
NAME                                                          READY   STATUS    RESTARTS   AGE
pod/eda-server-operator-controller-manager-7bf7578d44-7r87w   2/2     Running   0          12s

NAME                                                             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/eda-server-operator-controller-manager-metrics-service   ClusterIP   10.43.3.124   <none>        8443/TCP   12s

NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/eda-server-operator-controller-manager   1/1     1            1           12s

NAME                                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/eda-server-operator-controller-manager-7bf7578d44   1         1         1       12s
```

### Prepare required files to deploy EDA Server

Generate a Self-Signed certificate for the Web UI and API for EDA Server. Note that IP address can't be specified.

```bash
EDA_HOST="eda.example.com"
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out ./rulebooks/server/tls.crt -keyout ./rulebooks/server/tls.key -subj "/CN=${EDA_HOST}/O=${EDA_HOST}" -addext "subjectAltName = DNS:${EDA_HOST}"
```

Modify `hostname` and `automation_server_url` in `rulebooks/server/eda.yaml`. Note `hostname` is the hostname for your EDA Server instance, and `automation_server_url` is the URL for your AWX instance that accessible from EDA Server.

```yaml
...
spec:
  ...
  ingress_type: ingress
  ingress_tls_secret: eda-secret-tls
  hostname: eda.example.com     👈👈👈

  automation_server_url: https://awx.example.com/     👈👈👈
  automation_server_ssl_verify: no
...
```

Modify two `password`s in `rulebooks/server/kustomization.yaml`.

```yaml
...
  - name: eda-database-configuration
    type: Opaque
    literals:
      - host=eda-postgres-13
      - port=5432
      - database=eda
      - username=eda
      - password=Ansible123!     👈👈👈
      - type=managed

  - name: eda-admin-password
    type: Opaque
    literals:
      - password=Ansible123!     👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `base/pv.yaml`. This directory will be used to store your database.

```bash
sudo mkdir -p /data/eda/postgres-13/data
sudo chmod 755 /data/eda/postgres-13/data
sudo chown 26:0 /data/eda/postgres-13/data
```

### Deploy EDA Server

Deploy EDA Server, this takes few minutes to complete.

```bash
kubectl apply -k rulebooks/server
```

To monitor the progress of the deployment, check the logs of `deployment/eda-server-operator-controller-manager`:

```bash
kubectl -n eda logs -f deployment/eda-server-operator-controller-manager
```

When the deployment completes successfully, the logs end with:

```txt
$ kubectl -n eda logs -f deployment/eda-server-operator-controller-manager
...
----- Ansible Task Status Event StdOut (eda.ansible.com/v1alpha1, Kind=EDA, eda/eda) -----
PLAY RECAP *********************************************************************
localhost                  : ok=53   changed=0    unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
```

Required objects has been deployed next to AWX Operator in `awx` namespace.

```bash
$ kubectl -n eda get eda,all,ingress,configmap,secret
NAME                      AGE
eda.eda.ansible.com/eda   4m2s

NAME                                                          READY   STATUS    RESTARTS   AGE
pod/eda-server-operator-controller-manager-6ff679b85d-djcrt   2/2     Running   0          5m6s
pod/eda-redis-7d78cdf7d5-f4nsz                                1/1     Running   0          3m47s
pod/eda-postgres-13-0                                         1/1     Running   0          3m38s
pod/eda-ui-69569559b8-k4rdq                                   1/1     Running   0          2m50s
pod/eda-default-worker-5cd8664bcd-zv2v8                       1/1     Running   0          2m46s
pod/eda-default-worker-5cd8664bcd-7bvc6                       1/1     Running   0          2m46s
pod/eda-activation-worker-65bbc877fd-g6nj4                    1/1     Running   0          2m43s
pod/eda-activation-worker-65bbc877fd-6c7lm                    1/1     Running   0          2m43s
pod/eda-activation-worker-65bbc877fd-wbp92                    1/1     Running   0          2m43s
pod/eda-scheduler-bbf6554f4-v6x5s                             1/1     Running   0          2m40s
pod/eda-api-798787c5bf-pp82t                                  2/2     Running   0          2m53s

NAME                                                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/eda-server-operator-controller-manager-metrics-service   ClusterIP   10.43.22.48     <none>        8443/TCP   5m6s
service/eda-redis-svc                                            ClusterIP   10.43.2.121     <none>        6379/TCP   3m49s
service/eda-postgres-13                                          ClusterIP   None            <none>        5432/TCP   3m40s
service/eda-api                                                  ClusterIP   10.43.57.93     <none>        8000/TCP   2m55s
service/eda-daphne                                               ClusterIP   10.43.249.197   <none>        8001/TCP   2m55s
service/eda-ui                                                   ClusterIP   10.43.250.22    <none>        80/TCP     2m51s
service/eda-default-worker                                       ClusterIP   10.43.66.37     <none>        8080/TCP   2m47s
service/eda-activation-worker                                    ClusterIP   10.43.221.86    <none>        8080/TCP   2m45s

NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/eda-server-operator-controller-manager   1/1     1            1           5m6s
deployment.apps/eda-redis                                1/1     1            1           3m47s
deployment.apps/eda-ui                                   1/1     1            1           2m50s
deployment.apps/eda-default-worker                       2/2     2            2           2m46s
deployment.apps/eda-activation-worker                    3/3     3            3           2m43s
deployment.apps/eda-scheduler                            1/1     1            1           2m40s
deployment.apps/eda-api                                  1/1     1            1           2m53s

NAME                                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/eda-server-operator-controller-manager-6ff679b85d   1         1         1       5m6s
replicaset.apps/eda-redis-7d78cdf7d5                                1         1         1       3m47s
replicaset.apps/eda-ui-69569559b8                                   1         1         1       2m50s
replicaset.apps/eda-default-worker-5cd8664bcd                       2         2         2       2m46s
replicaset.apps/eda-activation-worker-65bbc877fd                    3         3         3       2m43s
replicaset.apps/eda-scheduler-bbf6554f4                             1         1         1       2m40s
replicaset.apps/eda-api-798787c5bf                                  1         1         1       2m53s

NAME                               READY   AGE
statefulset.apps/eda-postgres-13   1/1     3m38s

NAME                                    CLASS     HOSTS             ADDRESS         PORTS     AGE
ingress.networking.k8s.io/eda-ingress   traefik   eda.example.com   192.168.0.219   80, 443   2m49s

NAME                               DATA   AGE
configmap/kube-root-ca.crt         1      5m7s
configmap/eda-eda-env-properties   9      2m56s
configmap/eda-server-operator      0      5m5s

NAME                                     TYPE                DATA   AGE
secret/redhat-operators-pull-secret      Opaque              1      5m6s
secret/eda-admin-password                Opaque              1      4m2s
secret/eda-database-configuration        Opaque              6      4m2s
secret/eda-secret-tls                    kubernetes.io/tls   2      4m2s
secret/eda-db-fields-encryption-secret   Opaque              1      3m4s
```

Now your EDA Server is available at `https://eda.example.com/` or the hostname you specified.

## Demo: Use EDA Server

Here is a demo of configuring a webhook on the EDA Server side, and triggering a Job Template on AWX by posting payload that contains a specific `message` to the webhook.

In this demo, following example Rulebook is used. Review the Rulebook.

- **Webhook as a source**: [demo_webhook.yaml](demo_webhook.yaml)
  - The webhook that listening on `0.0.0.0:5000` is defined as a source of the Ruleset.
  - This Ruleset has a rule that if the payload contains `message` field with the body `Hello EDA`, trigger `Demo Job Template` in `Default` organization on AWX.

In addition to the webhook demo, a quick demo to use MQTT as a source is also provided.

- **MQTT as a source**: [demo_mqtt.yaml](demo_mqtt.yaml)
  - As a source of the Ruleset, subscribing MQTT topic on the MQTT broker is defined. Actual connection information for MQTT can be defined by Rulebook Variables.
  - This Ruleset has a rule that if the received data contains `message` field with the body `Hello EDA`, trigger `Demo Job Template` in `Default` organization on AWX.

### Configure EDA Server

In order to the webhook to be ready to receive messages, the following tasks need to be done.

- Issue new token for AWX and add it on EDA Server
- Add Decision Environment on EDA Server
- Add Project on EDA Server
- Activate Rulebook
- Deploy Ingress resource for the webhook

#### Issue new token for AWX and add it on EDA Server

EDA Server uses a token to access AWX. This token has to be issued by AWX and registered on EDA Server.

To issue new token by AWX, in the Web UI for AWX, open `User Details` page (accessible by user icon at the upper right corner), follow to the `Tokens` tab, and then click `Add` button. Specify `Write` as `Scope` and click `Save`, then keep the issued token in the safe place.

Alternatively we can issue new token by CLI as follows:

```bash
$ kubectl -n awx exec deployment/awx-task -- awx-manage create_oauth2_token --user=admin
4sIZrWXi**************8xChmahb
```

To register the token on EDA Server, in the Web UI for EDA Server, open `User details` page (accessible by user icon at the upper right corner), follow to the `Controller Tokens` tab, and then click `Create controller token` button.

Fill the form as follows, then click `Create controller token` button on the bottom of the page:

| Key | Value |
| - | - |
| Name | `awx.example.com` |
| Token | `<YOUR_TOKEN>` |

#### Add Decision Environment on EDA Server

Decision Environment (DE) is an environment for running Ansible Rulebook (`ansible-rulebook`) by the EDA Server, like Execution Environment (EE) for running Ansible Runner (`ansible-runner`) by the AWX.

There is no default DE on EDA Server, so we have to register new one.

Open `Decision Environments` under `Resources` on Web UI for EDA Server, then click `Create decision environment` button.

Fill the form as follows, then click `Create decision environment` button on the bottom of the page:

| Key | Value |
| - | - |
| Name | `Minimal DE` |
| Image | `quay.io/ansible/ansible-rulebook:latest` |

#### Add Project on EDA Server

To run Ansible Rulebook by EDA Server, the repository on SCM that contains Rulebooks have to be registered as Project on EDA Server.

This repository contains some example Rulebooks under [rulebooks](./) directory, so we can register this repository as Project.

Open `Projects` under `Resources` on Web UI for EDA Server, then click `Create project` button.

Fill the form as follows, then click `Create project` button on the bottom of the page:

| Key | Value |
| - | - |
| Name | `Demo Project` |
| SCM URL | `https://github.com/kurokobo/awx-on-k3s.git` |

Refresh the page and wait for the `Status` for the project to be `Completed`.

#### Activate Rulebook

To run Ansible Rulebook by EDA Server, activate the Rulebook.

Open `Rulebook Activations` under `Views` on Web UI for EDA Server, then click `Create rulebook activation` button.

Fill the form as follows, then click `Create rulebook activation` button on the bottom of the page:

| Key | Value |
| - | - |
| Name | `Trigger Demo Job Template by Webhook` |
| Project | `Demo Project` |
| Rulebook | `demo_webhook.yaml` |
| Decision environment | `Minimal DE` |

Refresh the page and wait for the `Activation status` for the Rulebook to be `Running`.

Ensure you have Activation ID which can be found at the end of the URL, e.g. `http://eda.example.com/eda/rulebook-activations/details/1`.

The new Job is created on `eda` namespace.

```bash
$ ACTIVATION_ID=1
$ kubectl -n eda get job -l activation-id=${ACTIVATION_ID}
NAME                 COMPLETIONS   DURATION   AGE
activation-job-1-1   0/1           7m3s       7m3s
```

By this Job, new Pod that `ansible-rulebook` running on is also created.

```bash
$ JOB_NAME=$(kubectl -n eda get job -l activation-id=${ACTIVATION_ID} -o jsonpath='{.items[*].metadata.name}')
$ kubectl -n eda get pod -l job-name=${JOB_NAME}
NAME                       READY   STATUS    RESTARTS   AGE
activation-job-1-1-ctz24   1/1     Running   0          7m16s
```

The new Service is also created by EDA Server. This service provides the endpoint for the webhook.

```bash
$ kubectl -n eda get service -l job-name=${JOB_NAME}
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
activation-job-1-1-5000   ClusterIP   10.43.82.223   <none>        5000/TCP   7m40s
```

#### Deploy Ingress resource for the webhook

To make the webhook externally accessible, we have to expose the Service that created by EDA Server.

To achieve this, in this example, we create new Ingress.

Modify `hosts`, `host`, and `name` under `service` in `rulebooks/webhook/ingress.yaml`. Here, the same hostname as the EDA Server are specified so that the endpoint for webhook can be accessed under the same URL as the EDA Server. Note that the `name` of the `service` has to be the name of the Service that created by EDA Server, as reviewed above.

```yaml
...
spec:
  tls:
    - hosts:
        - eda.example.com     👈👈👈
      secretName: eda-secret-tls
  rules:
    - host: eda.example.com     👈👈👈
      http:
        paths:
          - path: /webhooks/demo
            pathType: ImplementationSpecific
            backend:
              service:
                name: activation-job-1-1-5000     👈👈👈
                port:
                  number: 5000
```

Modify `replicas` for `worker` in the same file. The number of rulebooks that can be activated simultaneously is equal to the number of this value.

```yaml
  ...
  worker:
    replicas: 2    👈👈👈
    resource_requirements:
      requests: {}
  ...
```

By applying this file, your webhook can be accessed on the URL `https://eda.example.com/webhooks/demo`.

```bash
$ kubectl apply -f rulebooks/webhook/ingress.yaml
...

$ kubectl -n eda get ingress
NAME                  CLASS     HOSTS             ADDRESS         PORTS     AGE
eda-ingress           traefik   eda.example.com   192.168.0.219   80, 443   4h45m
eda-ingress-webhook   traefik   eda.example.com   192.168.0.219   80, 443   1s     👈👈👈
```

### Trigger Rule using Webhook

In [the Rulebook (`demo_webhook.yaml`) that used in this demo](demo_webhook.yaml), the Job Template (`Demo Job Template`) is invoked on AWX on the condition that `message` in the payload that posted to the webhook is `Hello EDA`.

Post that payload to the webhook, and review the Job Template is triggered.

```bash
$ curl -k \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello EDA"}' \
  https://eda.example.com/webhooks/demo
```

Review `Rule Audit` page under `Views` on the Web UI for EDA Server, and `Jobs` page under `Views` on the Web UI for AWX.

### Appendix: Use MQTT as a source

For a more authentic example, here is an example of using MQTT as a source.

To use MQTT, a MQTT broker is required. If you don't have one, you can use `kubectl apply -k rulebooks/mqtt/broker` to deploy a minimal MQTT broker on K3s that listens on `31883` or use a public broker such as [test.mosquitto.org](https://test.mosquitto.org/). Also, this demonstration is prepared assuming that neither authentication nor encryption is used.

Define the Decision Environment with the following information, just as you configured for the webhook.

| Key | Value |
| - | - |
| Name | `Minimal DE with MQTT` |
| Image | `docker.io/kurokobo/ansible-rulebook:v1.0.1-mqtt` |

Note that the image specified above is based on `quay.io/ansible/ansible-rulebook:v1.0.1` and includes [PR #113](https://github.com/ansible/event-driven-ansible/pull/113) with small patch to make `ansible.eda.mqtt` workable. The Dockerfile for this image is available under [mqtt/de directory](./mqtt/de).

Then define Rulebook Activation as follows. Note that you should modify actual values for `Variables` to suit your environment:

| Key | Value |
| - | - |
| Name | `Trigger Demo Job Template by MQTT` |
| Project | `Demo Project` |
| Rulebook | `demo_mqtt.yaml` |
| Decision environment | `Minimal DE with MQTT` |
| Variables | `mqtt_host: mqtt.example.com`<br>`mqtt_port: 31883`<br>`mqtt_topic: demo` |

Activate the Rulebook, and publish specific message that matches the condition in the Rule to the topic you've defined.

```bash
docker run -it --rm efrecon/mqtt-client pub \
        -h mqtt.example.com \
        -p 31883 \
        -t demo \
        -m '{"message": "Hello EDA"}'
```

Ensure your Job Template on AWX has been triggered.
