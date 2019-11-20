## Reattaching to a Kubernetes PV

This readme shows how to retain a Kubernetes Persistent Volume (PV)
after a PVC or Helm chart is deleted.  It then shows how to re-attach
to the PV when a Helm chart is reinstalled.

### Set the PV to Retain

This is an example of how to set an existing PV to be retained if the PVC or Helm 
chart is deleted.

```
kubectl patch pv tomcat -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}' -n tomcat
```

### Delete the Helm Chart

You can now delete the Helm chart associated with the PV and the PV will not be deleted.

```helm delete --purge tomcat```


### Make the PV available to be reclaimed

You must make the PV available by running the following patch command.  The
PV will then be allowed to attached to a new pod.  

```
kubectl patch pv pvc-e6a32e2b-c69a-44c2-9993-abbea1d8620a -p '{"spec":{"claimRef": null}}'
```

### Set the PVC to use the volumeName

In order to re-deploy a Helm chart an re-attached to the preexisting PV you have to 
mention the name of the PV in the PVC with the `volumeName` field.

```
{{- if .Values.persistence.enabled -}}
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: {{ template "tomcat.fullname" . }}
  labels:
    app: {{ template "tomcat.fullname" . }}
    chart: {{ template "tomcat.chart" . }}
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
  annotations:
    volume.alpha.kubernetes.io/storage-class: {{ ternary "default" (trimPrefix "storageClassName: " (include "tomcat.storageClass" .)) (empty (include "tomcat.storageClass" .)) }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode | quote }}
  resources:
    requests:
      storage: {{ .Values.persistence.size | quote }}
  volumeName: "pvc-e6a32e2b-c69a-44c2-9993-abbea1d8620a"
  {{ include "tomcat.storageClass" . }}
{{- end -}}
```

### Reinstall the Helm chart

You can now reinstall the Helm chart and it will reuse the original PV.

```
helm install --name tomcat --namespace tomcat . -f values.yaml
```

Doing a get on the pvc will show the attachment to the original PV.

```
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
tomcat   Bound    pvc-e6a32e2b-c69a-44c2-9993-abbea1d8620a   8Gi        RWO            longhorn       36m
```