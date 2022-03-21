# Mongodb install 
```
helm repo add bitnami https://charts.bitnami.com/bitnami
kubectl create namespace mongodb
helm install mongo bitnami/mongodb --namespace mongodb \
    --set architecture="replicaset"
```

mongodb-hooks.yaml:
```
apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mongo-hooks
actions:
  backupPrehook:
    phases:
    - func: KubeExecAll
      name: lockMongo
      objects:
        mongoDbSecret:
          kind: Secret
          name: '{{ .StatefulSet.Name }}'
          namespace: "{{ .StatefulSet.Namespace }}"        
      args:
        namespace: "{{ .StatefulSet.Namespace }}"
        pods: "{{ range .StatefulSet.Pods }} {{.}}{{end}}"
        containers: "mongodb"
        command:
        - bash
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          export MONGODB_ROOT_PASSWORD='{{ index .Phases.lockMongo.Secrets.mongoDbSecret.Data "mongodb-root-password" | toString }}'
          mongo --authenticationDatabase admin -u root -p "$MONGODB_ROOT_PASSWORD" --eval="db.fsyncLock()"
  backupPosthook:
    phases:
    - func: KubeExecAll
      name: unlockMongo
      objects:
        mongoDbSecret:
          kind: Secret
          name: '{{ .StatefulSet.Name }}'
          namespace: "{{ .StatefulSet.Namespace }}" 
      args:
        namespace: "{{ .StatefulSet.Namespace }}"
        pods: "{{ range .StatefulSet.Pods }} {{.}}{{end}}"
        containers: "mongodb"
        command:
        - bash
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          export MONGODB_ROOT_PASSWORD='{{ index .Phases.unlockMongo.Secrets.mongoDbSecret.Data "mongodb-root-password" | toString }}'
          mongo --authenticationDatabase admin -u root -p "$MONGODB_ROOT_PASSWORD" --eval="db.fsyncUnlock()"
    ```