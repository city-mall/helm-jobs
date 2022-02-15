# jobs-deployment-poc

A POC for deployment of cron jobs. Currently, jobs are written in one folder, See `packages/jobs`, deployed as a part of one ECR
image to production and are run by calling `npm start`. This would import all such jobs and run them via 
JS cron syntax or by calling `setTimeout()` at regular intervals.

The current process has multiple flaws:
1. JavaScript runs on single thread and as the number of such jobs increase, they start colliding with each other. 
2. If one job breaks, it could break other jobs as well because all are running on the same process
3. Since all of these jobs run on the same pod, all of them have common memory and cpu available
4. It could happen that a lot of times, none of the jobs are running but we are still paying for those resources

## How to run the repo
1. Get AWS credentials and add those to your local machine
2. Allow AWS credentials the permission to decrypt `deployments/environment-vars/secrets.cmdev.yaml` (using AWS KMS). Note that your AWS user would need read permissions for `app-services-helm-secrets`
3. Node version 12+ should be supported
4. Run `npm install`
5. Run `npm start` to start the jobs

## Current Deployment
1. Current deployment is done via github actions
2. A docker build for the codebase is created using `deployments/dockerfiles` and is pushed to ECR
3. Github workflows are triggered which use a custom github action - `https://github.com/city-mall/helm` which is a fork of `https://github.com/deliverybot/helm` to support running `helm secrets`

## What do we plan to achieve
1. We need to do all such deployments using kubernetes cronjobs
2. We need to make sure that we make least changes in codebase / dockerfiles
3. We need to achieve multiple kubernetes cronjobs - with one Dockerfile, one ECR image and one github deployment
4. It should be possible to achieve the same using Jenkins deployment
5. It should be possible to run all the jobs as a single node process
6. We should setup monitoring for all the jobs using healthchecks.io or a similar service
7. It should be very simple for a developer to add a new job, should typically mean writing his code, exporting one function via a file and making some changes in a json config file



## From Notes.txt
1. Get the application URL by running these commands:
{{- if .Values.ingressEnabled }}
{{- range $host := .Values.hosts }}
  {{- range .paths }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
  {{- end }}
{{- end }}
{{- else if contains "NodePort" .Values.serviceType }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "crm-charts.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.serviceType }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "crm-charts.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "crm-charts.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.serviceType }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "crm-charts.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
