## AWS Command Line Interface (AWS cli)

```bash
export AWS_ACCESS_KEY_ID=[...]
export AWS_SECRET_ACCESS_KEY=[...]
```

## Définir la région

```bash
export AWS_DEFAULT_REGION=eu-west-3
```

## Création du groupe IAM

```bash
aws iam create-group --group-name kops
```

## Création des autorisations utilisateurs

```bash
aws iam attach-group-policy \
    --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess \
    --group-name kops

aws iam attach-group-policy \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
    --group-name kops

aws iam attach-group-policy \
    --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess \
    --group-name kops

aws iam attach-group-policy \
    --policy-arn arn:aws:iam::aws:policy/IAMFullAccess \
    --group-name kops
```

## Création d'un utilisateur kops

```bash
aws iam create-user \
    --user-name kops
```

## Création du groupe kops

```bash
aws iam add-user-to-group \
    --user-name kops \
    --group-name kops
```

## Création de la clé d'accès et stockage de la sortie dans le fichier "kops-creds"

```bash
aws iam create-access-key \
    --user-name kops >kops-creds

cat kops-creds
```

## Installation de jq pour analyser les infos d'identification

```bash
export AWS_ACCESS_KEY_ID=$(\
    cat kops-creds | jq -r \
    '.AccessKey.AccessKeyId')

export AWS_SECRET_ACCESS_KEY=$(
    cat kops-creds | jq -r \
    '.AccessKey.SecretAccessKey')
```

## Configuration des zones de disponibilité

```bash
aws ec2 describe-availability-zones \
    --region $AWS_DEFAULT_REGION
```

## Création de clés SSH

```bash
mkdir -p cluster

cd cluster

aws ec2 create-key-pair \
    --key-name devops23 \
    | jq -r '.KeyMaterial' \
    >devops23.pem

chmod 400 devops23.pem

ssh-keygen -y -f devops23.pem \
    >devops23.pub
```

## Stockage du nom dans un variable d'environnement

```bash
export NAME=devops23.k8s.local
```

## Création d'un bucket S3

```bash
export BUCKET_NAME=devops23-$(date +%s)

aws s3api create-bucket \
    --bucket $BUCKET_NAME \
    --create-bucket-configuration \
    LocationConstraint=$AWS_DEFAULT_REGION
```

## Stockage de l'état

```bash
export KOPS_STATE_STORE=s3://$BUCKET_NAME
```

## Installation de kops sur linux

```bash
wget -O kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64

chmod +x ./kops

sudo mv ./kops /usr/local/bin/
```

## Création du cluster

```bash
kops create cluster \
    --name $NAME \
    --master-count 3 \
    --node-count 1 \
    --node-size t2.small \
    --master-size t2.small \
    --zones $ZONES \
    --master-zones $ZONES \
    --ssh-public-key devops23.pub \
    --networking kubenet \
    --kubernetes-version v1.14.8 \
    --yes
```

## Vérification

```bash
kops get cluster

kubectl cluster-info

kops validate cluster
```

## Mise à jour du cluster

```bash
kops edit --help
```

## Modifier le cluster

```bash
kops edit cluster --name $NAME
```

## Modifier le groupe d'instances

```bash
kops edit ig --name $NAME nodes
```

## Mise à jour du cluster pour se conformer au nouvel état souhaité

```bash
kops update cluster --name $NAME --yes
```

## Explorer l'AWS ELB (Elastic Load Balancer)

```bash
aws elb describe-load-balancers
```

## Configuration kubectl

```bash
kubectl config view
```

## Créer des ressources

```bash
kubectl create \
    -f https://raw.githubusercontent.com/kubernetes/kops/master/addons/ingress-nginx/v1.6.0.yaml

kubectl --namespace kube-ingress \
    get all
```

## Récupération du DNS du nouvel équilibreur de charge

```bash
CLUSTER_DNS=$(aws elb \
    describe-load-balancers | jq -r \
    ".LoadBalancerDescriptions[] \
    | select(.DNSName \
    | contains (\"api-devops23\") \
    | not).DNSName")
```

## Déploiement des ressources

```bash
cd ~/k8s-specs

kubectl create \
    -f aws/go-demo-2.yml \
    --record --save-config

kubectl rollout status \
    deployment go-demo-2-api

curl -i "http://$CLUSTER_DNS/demo/hello"
```

## Mettre fin à un noeud de W

```bash
aws ec2 \
    describe-instances | jq -r \
    ".Reservations[].Instances[] \
    | select(.SecurityGroups[]\
    .GroupName==\"nodes.$NAME\")\
    .InstanceId"
```

## Choix d'un noeud de W aléatoire et mettre fin à ce noeud

```bash
INSTANCE_ID=$(aws ec2 \
    describe-instances | jq -r \
    ".Reservations[].Instances[] \
    | select(.SecurityGroups[]\
    .GroupName==\"nodes.$NAME\")\
    .InstanceId" | tail -n 1)

aws ec2 terminate-instances \
    --instance-ids $INSTANCE_ID
```

## Créer une configuration distribuable

```bash
cd cluster
mkdir -p config
export KUBECONFIG=$PWD/config/kubecfg.yaml

kops export kubecfg --name ${NAME}
cat $KUBECONFIG
```

## Stockage des variables d'environnement

```bash
echo "export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
export AWS_DEFAULT_REGION=$AWS_DEFAULT_REGION
export ZONES=$ZONES
export NAME=$NAME
export KOPS_STATE_STORE=$KOPS_STATE_STORE" \
    >kops
```

## Détruire le cluster

```bash
kops delete cluster \
    --name $NAME \
    --yes
```

## Suppression du compartiment S3

```bash
aws s3api delete-bucket \
    --bucket $BUCKET_NAME
```

## Créer un volume EBS

```bash
aws ec2 describe-instances
```

## Récupérer les zones de disponibilité

```bash
aws ec2 describe-instances \
    | jq -r \
    ".Reservations[].Instances[] \
    | select(.SecurityGroups[]\
    .GroupName==\"nodes.$NAME\")\
    .Placement.AvailabilityZone"
```

## Configuration des variables d'environnement

```bash
aws ec2 describe-instances \
    | jq -r \
    ".Reservations[].Instances[] \
    | select(.SecurityGroups[]\
    .GroupName==\"nodes.$NAME\")\
    .Placement.AvailabilityZone" \
    | tee zones

AZ_1=$(cat zones | head -n 1)

AZ_2=$(cat zones | tail -n 1)
```

## Créer des volumes

```bash
VOLUME_ID_1=$(aws ec2 create-volume \
    --availability-zone $AZ_1 \
    --size 10 \
    --volume-type gp2 \
    --tag-specifications "ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=$NAME}]" \
    | jq -r '.VolumeId')

VOLUME_ID_2=$(aws ec2 create-volume \
    --availability-zone $AZ_1 \
    --size 10 \
    --volume-type gp2 \
    --tag-specifications "ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=$NAME}]" \
    | jq -r '.VolumeId')

VOLUME_ID_3=$(aws ec2 create-volume \
    --availability-zone $AZ_2 \
    --size 10 \
    --volume-type gp2 \
    --tag-specifications "ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=$NAME}]" \
    | jq -r '.VolumeId')
```

## Lister le volume

```bash
aws ec2 describe-volumes \
    --volume-ids $VOLUME_ID_1
```

## Créer le volume persistant

```bash
cat pv/pv.yml \
    | sed -e \
    "s@REPLACE_ME_1@$VOLUME_ID_1@g" \
    | sed -e \
    "s@REPLACE_ME_2@$VOLUME_ID_2@g" \
    | sed -e \
    "s@REPLACE_ME_3@$VOLUME_ID_3@g" \
    | kubectl create -f - \
    --save-config --record
```

## Vérification

```bash
kubectl get pv
```

## Créer la réclamation

```bash
kubectl create -f pv/pvc.yml \
    --save-config --record
```

## Vérification

```bash
kubectl --namespace jenkins \
    get pvc
```

## Déploiement des ressources

```bash
kubectl apply \
    -f pv/jenkins-pv.yml \
    --record
```

## Vérification

```bash
kubectl --namespace jenkins \
    rollout status \
    deployment jenkins
```

## Obtenir le POD créé par Jenkins

```bash
POD_NAME=$(kubectl \
    --namespace jenkins \
    get pod \
    --selector=app=jenkins \
    -o jsonpath="{.items[*].metadata.name}")
```

## Mettre fin au processus

```bash
kubectl --namespace jenkins \
    exec -it $POD_NAME pkill java
```
