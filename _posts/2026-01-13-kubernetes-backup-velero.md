---
layout: post
collection: kubernetes
permalink: /kubernetes/backup-velero
title:  "Backup Velero"
author: Paulo Rogério
date:   2026-01-13 21:08:00 -0300
categories: [kubernetes]
published: true
---

# 🚀  Backup / Restore Cluster Kubernetes Usando Velero

- [Preparando Cluster](#preparando-cluster)
- [Deployment Exemplo](#deployment-exemplo)
- [Configurando Minio](#configurando-minio)
- [Configurando Longhorn](#configurando-longhorn)
- [Configurando CSI Kubernetes](#configurando-csi-kubernetes)
- [Instalar Velero Client](#instalar-velero-client)
- [Deploy Velero](#deploy-velero)
- [Criando Backup](#criando-backup)
- [Problemas com Backup](#problemas-com-backup)
- [Restaurando Backup](#restaurando-backup)
- [Demo](#demo)

## Preparando Cluster

Para esse laboratório vamos assumir que o cluster foi recém criado e que os deployments abaixos sejam pre-requisitos para iniciarmos os trabalhos:

- Minio
- Longhorn

![k9s-deploy](../../../../images/kubernetes/backup/velero/backup-velero-01.png)

## Deployment Exemplo

Criando um deployment para usar o **longhorn** como storageClass

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-paulo
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-data
  namespace: nginx-paulo
  labels:
    app: nginx-data
spec:
  storageClassName: longhorn 
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-paulo
  namespace: nginx-paulo
  labels:
    app: nginx-paulo
spec:
  selector:
    matchLabels:
      app: nginx-paulo
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx-paulo
    spec:
      containers:
      - name: nginx-paulo
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: nginx-paulo
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nginx-data
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-paulo
  namespace: nginx-paulo
spec:
  ports:
    - name: nginx
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: nginx-paulo
  type: ClusterIP
EOF
```

Aplicação **nginx-paulo** deployada...


![nginx-deploy](../../../../images/kubernetes/backup/velero/backup-velero-02.png)

Após o deployment ficar com status **Running**, conecte-s ao host e crie **100 arquivo txt** para simular um **backup / restore**. Se tiver usando **k9s**, basta selecionar o Pod e pressionar **s** que será mostrado o shell do container.

Se preferir fazer isso manualmente execute o comando abaixo:

```bash
kubectl get pods -n nginx-paulo 
```
Escolha qualquer um dos Pods para se conectar ...

```bash
NAME                           READY   STATUS    RESTARTS      AGE
nginx-paulo-76b9dcd75c-lt96b   1/1     Running   1 (24m ago)   2d15h
nginx-paulo-76b9dcd75c-z7wtt   1/1     Running   1 (24m ago)   2d15h
```

```bash
kubectl exec -it -n nginx-paulo nginx-paulo-76b9dcd75c-lt96b -- bash
```

```bash
root@nginx-paulo-76b9dcd75c-lt96b:/# for i in {1..100}; do touch /data/$i.txt; done
```

![pod-list-files](../../../../images/kubernetes/backup/velero/backup-velero-03.png)

Conectar no serviço do **longhorn** e checar se o volume está funcional...

![port-forward](../../../../images/kubernetes/backup/velero/backup-velero-04.png)

![longhorn-list-volumes](../../../../images/kubernetes/backup/velero/backup-velero-05.png)

## Configurando Minio

Um passo importante antes de iniciarmos o backup com **velero**, é garantir que o **Minio** esteja apto para receber os dados do **longhorn**.

**OBS.: Se não definir a env MINIO_REGION, o longhorn não consegue conectar-se.**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env
  namespace: namespace-xxx
data:
  MINIO_ROOT_USER: "meu-usuario"
  MINIO_ROOT_PASSWORD: "minha-senha"
  MINIO_DOMAIN: "s3.dominio.com"
  MINIO_REGION: "sa-east-1"
```

## Configurando Longhorn

Com Minio configurado, é necessário configurar o **longhorn** para usar o **minio** como repositório de bakcup. Primeiramente vamos configurar um **bucket**.

![config-bucket-s3](../../../../images/kubernetes/backup/velero/backup-velero-06.png)

Depois de criado o bucket, deve-se criar uma **secret**.

![config-secret-s3](../../../../images/kubernetes/backup/velero/backup-velero-07.png)

Essa secret gerada pelo **Minio** deve ser conhecida dentro do namespace **longhorn-system**, para isso crie uma **secret** generic contendo as 3 variáveis solicitadas:

- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_ENDPOINTS

```yaml
kubectl create secret generic minio-secret \
  --namespace=longhorn-system \
  --type=Opaque \
  --from-literal=AWS_ACCESS_KEY_ID=atIjK6cdGh5nXdy3uFUW \
  --from-literal=AWS_SECRET_ACCESS_KEY=ghLRO4qGCOv2kYeyESmR3AiocUUFVTA4PmFVQYyS \
  --from-literal=AWS_ENDPOINTS=http://minio-backup.minio-backup.svc.cluster.local:9000 \
  --dry-run=client -o yaml
```


```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: YXRJaks2Y2RHaDVuWGR5M3VGVVc=
  AWS_ENDPOINTS: aHR0cDovL21pbmlvLXZhdWx0Lm1pbmlvLXZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsOjkwMDA=
  AWS_SECRET_ACCESS_KEY: Z2hMUk80cUdDT3Yya1lleUVTbVIzQWlvY1VVRlZUQTRQbUZWUVl5Uw==
kind: Secret
metadata:
  creationTimestamp: null
  name: minio-secret
  namespace: longhorn-system
type: Opaque
EOF
```

Configure um backup target...

![config-longhorn](../../../../images/kubernetes/backup/velero/backup-velero-08.png)

## Configurando CSI Kubernetes

Para qualquer ferramenta que irá realizar backup do cluster, é necessário implementar uma funcionalidade do cluster **kubernetes** para que tal ação seja realizada. Sem esse passo configurado, a ferramenta de backup não consegue interagir com sotrageClass que está servindo persistencia de dados para os Pods.

Para conhecer mais do produto, sugiro a leitura do Readme do repositório oficial [Github CSI Snapshotter](https://github.com/kubernetes-csi/external-snapshotter).


Instale a ultima versão da feacture **CSI Kubernetes**

```bash
export LAST_VERSION=$(curl -sSL https://api.github.com/repos/kubernetes-csi/external-snapshotter/releases/latest | jq -r .tag_name)

kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/$LAST_VERSION/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/$LAST_VERSION/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/$LAST_VERSION/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/$LAST_VERSION/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/$LAST_VERSION/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

**Se preferir pode-se baixar os manifestos yaml em um diretório e aplica-los**


Após instalado deve-se criar um objeto no kubernetes para **Criar Snapshot** usando o driver de conexão do longhorn. Até o momento dessa documentação o **storageClass** usado para persistencia de dados é o **Longhorn**, e essa implementação e para suportar **Longhorn**.


## Instalar Velero Client

A documetnação do velero é bem extensa e completa, para um entendimento melhor da ferramenta e caso necessite customizar alguma opção, deve-se consultar o material oficial [Velero](https://velero.io/docs/main/).

O velero faz uso de plugins para suportar recursos externos **(ex: S3)**. Sempre que instalar e configurar um novo **Backup Velero**, procure usar a última versão dos plugins.

[Plugins Velero - S3](https://github.com/vmware-tanzu/velero-plugin-for-aws)

Após instalar em sua estação de trabalho, no meu caso como uso **linux**, faça a instalação para linux ***GitHub release***.

```bash
which velero
/usr/local/bin/velero
```

```bash
velero version
Client:
	Version: v1.16.0
	Git commit: 8f31599fe4af5453dee032beaf8a16bd75de91a5
```

## Deploy Velero

Crie uma arquivo contendo informações usadas pelo deployment do velero. Altere os valores de acordo com sua realidade.

O **minio** por padrão entrega **2 enpoints**, sendo um para manipular via **Web** , outro endpoint para usar o protocolo **S3**. Para esse lab estamos assumindo que os endpoints de acesso ao s3 são:


**Endpoints Web:**  minio-vault.prgs-corp.xyz 

**Endpoints S3:**  s3-minio-vault.prgs-corp.xyz


```bash
cat > values.yaml <<EOF 
credentials:
  secretContents:
    cloud: |
      [default]
      aws_access_key_id = atIjK6cdGh5nXdy3uFUW
      aws_secret_access_key = ghLRO4qGCOv2kYeyESmR3AiocUUFVTA4PmFVQYyS

configuration:
  backupStorageLocation:
    - name: default
      provider: aws
      bucket: velero
      prefix: migrator
      config:
        region: sa-east-1
        s3ForcePathStyle: true
        s3Url: https://s3-minio-vault.prgs-corp.xyz
        insecureSkipTLSVerify: true
  features: EnableCSI

image:
  repository: velero/velero
  tag: v1.16.1
  pullPolicy: IfNotPresent

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.12.1
    volumeMounts:
      - mountPath: /target
        name: plugins

deployNodeAgent: true
snapshotsEnabled: false     
EOF
```

Atualize o repositório...

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update
```

Instale a ultima versão...

```bash
helm search repo vmware-tanzu  --versions
```

```bash
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  -f ./values.yaml \
  --version 10.0.7
```

![velero-deployed](../../../../images/kubernetes/backup/velero/backup-velero-09.png)

Após configure os StorageClass para ser invocado via velero...

```bash
cat <<EOF | kubectl apply -f -
  kind: VolumeSnapshotClass
  apiVersion: snapshot.storage.k8s.io/v1
  metadata:
    name: longhorn-backup-vsc
    labels:
      velero.io/csi-volumesnapshot-class: "true"  
  driver: driver.longhorn.io
  deletionPolicy: Delete
  parameters:
    type: bak
EOF
```

```bash
cat <<EOF | kubectl apply -f -
  kind: VolumeSnapshotClass
  apiVersion: snapshot.storage.k8s.io/v1
  metadata:
    name: longhorn-snapshot-vsc
    labels:
      velero.io/csi-volumesnapshot-class: "true"
  driver: driver.longhorn.io
  deletionPolicy: Delete
  parameters:
    type: snap
EOF
```

```bash
cat <<EOF | kubectl apply -f -
  apiVersion: velero.io/v1
  kind: VolumeSnapshotLocation
  metadata:
    name: longhorn-location
    namespace: velero
    annotations:
      velero.io/is-default-volume-snapshot-location: "true"
  spec:
    provider: csi
EOF
```


```bash
kubectl get volumesnapshotclasses
NAME                    DRIVER               DELETIONPOLICY   AGE
longhorn-backup-vsc     driver.longhorn.io   Delete           50s
longhorn-snapshot-vsc   driver.longhorn.io   Delete           25m
```

## Criando Backup

Com a ferramenta deployada, vamos iniciar um backup manualmente.

Listando Jobs 

```bash
velero backup get
```

Criando Backup

```bash
velero backup create nginx --include-namespaces nginx-paulo --wait

Backup request "nginx" submitted successfully.
Waiting for backup to complete. You may safely press ctrl-c to stop waiting - your backup will continue in the background.
......
Backup completed with status: Completed. You may check for more information using the commands `velero backup describe nginx` and `velero backup logs nginx`.
```

```bash
velero backup describe nginx
```

![backup-velero-describe-sucess](../../../../images/kubernetes/backup/velero/backup-velero-10.png)

Conecte-se no **Minio** e valide a existencia dos backups...

![s3-backup-files](../../../../images/kubernetes/backup/velero/backup-velero-11.png)

## Problemas com Backup

Maiores detalhes do erro pode ser obtido nos logs.

```bash
velero backup describe nginx
velero backup logs nginx
```

Analisar os logs será chave para desvendar o problema. Esse erro foi proposital, e ele foi ocasionado porque o bucket **velero** não existia.

![bucket-velero](../../../../images/kubernetes/backup/velero/backup-velero-12.png)

Listar os Backups... 

```bash
velero backup get
NAME    STATUS   ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
nginx   Failed   0        0          2025-06-23 10:39:02 -0300 -03   29d       default            <none>
```

Remover Backup...

```bash
velero backup delete nginx --confirm

Request to delete backup "nginx" submitted successfully.
The backup will be fully deleted after all associated data (disk snapshots, backup files, restores) are removed.
```

Conectar no **minio** e criar o bucket manualmente.

![bucket-velero](../../../../images/kubernetes/backup/velero/backup-velero-12.png)

## Restaurando Backup

Para validar o processo de restore, vamos remover tudo referente ao deploy **nginx-paulo** 

![delete-deployment](../../../../images/kubernetes/backup/velero/backup-velero-13.png)

Garanta que o **pvc** será removido também...

![delete-pvc](../../../../images/kubernetes/backup/velero/backup-velero-14.png)

Ao conectar no Longhor, poderá visualizar que não existe nenhum pvc em uso.

![delete-pvc](../../../../images/kubernetes/backup/velero/backup-velero-15.png)


Agora que não tenho nada relacionado ao produto deployado, minha única esperaça é o **backup**. O processo de restore, deverá restaurar o deployment de forma íntegra e garantir que os ***100 arquivos .txt*** estejam presente no volume.

```bash
velero restore create nginx-paulo --from-backup nginx --wait
```

![restore-velero](../../../../images/kubernetes/backup/velero/backup-velero-16.png)

![restore-velero-describe](../../../../images/kubernetes/backup/velero/backup-velero-17.png)

Após o restore, podemos ver que o **pvc** foi criado com sucesso no **Longhorn**...

![longhorn-restore-pvc](../../../../images/kubernetes/backup/velero/backup-velero-18.png)

Checando os Pods...

![pods-running-after-restore](../../../../images/kubernetes/backup/velero/backup-velero-19.png)


```bash
kubectl get pods -n nginx-paulo 
```
Escolha qualquer um dos Pods para se conectar ...

```bash
NAME                           READY   STATUS    RESTARTS      AGE
nginx-paulo-76b9dcd75c-lt96b   1/1     Running   0          11m
nginx-paulo-76b9dcd75c-z7wtt   1/1     Running   0          11m
```

```bash
kubectl exec -it -n nginx-paulo nginx-paulo-76b9dcd75c-lt96b -- bash
```

## Não trouxe os arquivos? Entao o Backup não está valido...

```bash
root@nginx-paulo-76b9dcd75c-lt96b:/# ls /data/
1.txt	 13.txt  18.txt  22.txt  27.txt  31.txt  36.txt  40.txt  45.txt  5.txt	 54.txt  59.txt  63.txt  68.txt  72.txt  77.txt  81.txt  86.txt  90.txt  95.txt
10.txt	 14.txt  19.txt  23.txt  28.txt  32.txt  37.txt  41.txt  46.txt  50.txt  55.txt  6.txt	 64.txt  69.txt  73.txt  78.txt  82.txt  87.txt  91.txt  96.txt
100.txt  15.txt  2.txt	 24.txt  29.txt  33.txt  38.txt  42.txt  47.txt  51.txt  56.txt  60.txt  65.txt  7.txt	 74.txt  79.txt  83.txt  88.txt  92.txt  97.txt
11.txt	 16.txt  20.txt  25.txt  3.txt	 34.txt  39.txt  43.txt  48.txt  52.txt  57.txt  61.txt  66.txt  70.txt  75.txt  8.txt	 84.txt  89.txt  93.txt  98.txt
12.txt	 17.txt  21.txt  26.txt  30.txt  35.txt  4.txt	 44.txt  49.txt  53.txt  58.txt  62.txt  67.txt  71.txt  76.txt  80.txt  85.txt  9.txt	 94.txt  99.txt
```

## Demo

[Demo](https://gitlab.com/meetups-prgs/backup/velero)

