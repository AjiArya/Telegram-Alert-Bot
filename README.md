# Telegram-Alert-Bot
Send Alert from OpenShift Monitoring Stack to Telegram

This is only a guide on how to deploy [alertmanager-bot](https://github.com/metalmatze/alertmanager-bot) on your OKD 4.5

## Do below steps on Telegram
1. Search @userinfobot
  * Note your userid

2. Search @BotFather
  * Create New Bot
  * Note the API Key

## Do below steps on OKD
1. Create telebot.yaml
```yaml
apiVersion: v1
kind: List
items:
- apiVersion: v1
  data:
    admin: YOUR_USER_ID
    token: YOUR_API_KEY
  kind: Secret
  metadata:
    labels:
      app.kubernetes.io/name: alertmanager-bot
    name: alertmanager-bot
    namespace: telegram-bot
  type: Opaque
- apiVersion: v1
  kind: Service
  metadata:
    labels:
      app.kubernetes.io/name: alertmanager-bot
    name: alertmanager-bot
    namespace: telegram-bot
  spec:
    ports:
    - name: http
      port: 8080
      targetPort: 8080
    selector:
      app.kubernetes.io/name: alertmanager-bot
- apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    labels:
      app.kubernetes.io/name: alertmanager-bot
    name: alertmanager-bot
    namespace: telegram-bot
  spec:
    podManagementPolicy: OrderedReady
    replicas: 1
    selector:
      matchLabels:
        app.kubernetes.io/name: alertmanager-bot
    serviceName: alertmanager-bot
    template:
      metadata:
        labels:
          app.kubernetes.io/name: alertmanager-bot
        name: alertmanager-bot
        namespace: telegram-bot
      spec:
        containers:
        - args:
          - --alertmanager.url=https://alertmanager-main-openshift-monitoring.apps.openshift.podX.io
          - --log.level=info
          - --store=bolt
          - --bolt.path=/data/bot.db
          env:
          - name: TELEGRAM_ADMIN
            valueFrom:
              secretKeyRef:
                key: admin
                name: alertmanager-bot
          - name: TELEGRAM_TOKEN
            valueFrom:
              secretKeyRef:
                key: token
                name: alertmanager-bot
          image: metalmatze/alertmanager-bot:0.4.2
          imagePullPolicy: IfNotPresent
          name: alertmanager-bot
          ports:
          - containerPort: 8080
            name: http
          resources:
            limits:
              cpu: 100m
              memory: 128Mi
            requests:
              cpu: 25m
              memory: 64Mi
          volumeMounts:
          - mountPath: /data
            name: data
        restartPolicy: Always
        volumes:
        - name: data
          persistentVolumeClaim:
            claimName: alertmanager-bot
- apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: alertmanager-bot
  spec:
    storageClassName: managed-nfs-storage
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

```bash
oc apply -f telebot.yaml

oc expose svc alertmanager-bot
```

2. Edit alertmanager configuration
```bash
oc extract secret/alertmanager-main --to /tmp/ -n openshift-monitoring --confirm

vim /tmp/alertmanager.yaml
```

```yaml
---output omitted---
"receivers":
- name: 'alertmanager-bot'
  webhook_configs:
  - send_resolved: true
    url: 'http://alertmanager-bot-telegram-bot.apps.openshift.podX.io'
---output omitted---
"route":
  - match:
      severity: warning
    receiver: alertmanager-bot
```

```bash
oc apply -f /tmp/alertmanager.yaml
```

## Do below steps on Telegram

1. Search your bot
2. Try to send some commands
