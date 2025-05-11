## NOM : DAWOUSSOU

PRENOMS : KODZO SEWONU BRILLANT

# Projet AWS CloudFormation - Déploiement d'une Infrastructure Complète

## 1- Description

Ce projet déploie automatiquement une infrastructure AWS à l’aide d’un template CloudFormation.

Il comprend :

Ce template CloudFormation déploie automatiquement une infrastructure complète :

* Bucket S3 pour dépôt de fichiers
* Table DynamoDB pour stocker les métadonnées des fichiers
* Fonction Lambda déclenchée à chaque upload dans S3
* Instance EC2 avec :

  - Environnement Python isolé (**venv**)
  - Script **UserData** (**LVM, cron, virtualenv**)
  - Script Python scannant **DynamoDB**
  - Deux disques EBS : **xvda** et **xvdm**
  - **xvdm** comportant deux volumes logiques LVM :
    - **/tmp** (depuis lv_tmp)
    - **/var/log/dynamodb** (depuis lv_log_dynamodb)
* Groupe de sécurité EC2 (**SSH, HTTPS, ICMP, HTTP**)
* Montages persistants via `/etc/fstab`:
* Les **permissions Lambda** (invoke par S3)

> **Ressource**

| Ressource       | Automatisé ? |
| --------------- | ------------- |
| EC2 Instance    | OUI           |
| S3 Bucket       | OUI           |
| DynamoDB Table  | OUI           |
| Lambda Function | OUI           |
| Cron Setup      | OUI           |
| LVM + Mount     | OUI           |
| Script Python   | OUI           |

> **Chemins Utiles**

| Usage                        | Chemin                                     |
| :--------------------------- | ------------------------------------------ |
| Script Python                | `/home/ubuntu/scripts/scan_dynamodb.py`  |
| Logs du scan DynamoDB        | `/var/log/dynamodb/dynamodb_scan.log`    |
| Volume temporaire (`/tmp`) | Monté sur `/dev/datavg/lv_tmp`          |
| Volume logs DynamoDB         | Monté sur `/dev/datavg/lv_log_dynamodb` |

> **Pré-requis IAM : Votre utilisateur doit disposer des permissions suivantes :**

> cloudformation:
> ec2
> iam
> lambda
> dynamodb
> s3

## 2- Structure du projet

```
.
├── templates/
│   └── devoiraws-template.yaml     # Template principal CloudFormation
├── sshkeys/
│   └── iabdkey.pem                 # Clé privée SSH EC2 (à protéger)
└── README.md                       # Documentation
```

## 3- Actions automatiques UserData (Instance EC2)

Lors du lancement de l’instance EC2, les actions suivantes sont effectuées automatiquement :

1. Installation des paquets `python3`, `pip`, `lvm2`
2. Création d’un virtualenv dans `/opt/scanenv` avec `boto3`
3. Création de volumes logiques (LVM) sur `/dev/xvdm`
   * `/backups`
   * `/var/log/dynamodb`
4. Ajout des volumes à `/etc/fstab`
5. Écriture du script `/home/ubuntu/scripts/scan_dynamodb.py`
6. Logging dans `/var/log/dynamodb/dynamodb_scan.log`
7. Tâche cron exécutée chaque heure pour scanner la table DynamoDB

## 4- Script Python `scan_dynamodb.py`

Chemin : `/home/ubuntu/scripts/scan_dynamodb.py`

Log : `/var/log/dynamodb/dynamodb_scan.log`

Que fait le script ? :

* Scanne la table DynamoDB `FileMetadata-<EnvName>`
* Écrit chaque `FileName` et `BucketName` dans le fichier de log

## 5- Commandes utiles

* **Redéployer sans recréer les ressources inchangées :**

```bash
aws cloudformation deploy --no-fail-on-empty-changeset ...
```

* **Supprimer le stack et toutes les ressources :**

```bash
aws cloudformation delete-stack --stack-name devoir-iabd-brillantdawoussou-stack

# NB: Le nom de la stack est variable
```

* **Supprimer une instance EC2 spécifique :**

```bash
aws ec2 terminate-instances --instance-ids <instance-id>
```

* **Lister les ressources du stack :**

```bash
aws cloudformation describe-stack-resources --stack-name devoir-iabd-brillantdawoussou-stack
```

* **Vérifier le log utilisateur :**

```bash
sudo cat /var/log/userdata.log
```

* **Vérifier les volumes LVM montés :**

```bash
df -h | grep -E '/tmp|/var/log/dynamodb'
```

* **Exécuter manuellement le script de scan DynamoDB :**

```bash
sudo /opt/scanenv/bin/python /home/ubuntu/scripts/scan_dynamodb.py
```

* **Vérifier le fichier de log d'exécution :**

```bash
cat /var/log/dynamodb/dynamodb_scan.log
```

## 6- Remarques de sécurité

* Les clés AWS (`aws_access_key_id`, `aws_secret_access_key`) **ne doivent jamais être codées en dur** (le template le fait uniquement à des fins de test).
* Il faudra utilisé un **IAM Role attaché à l'EC2** avec les permissions `DynamoDBReadOnlyAccess`.
* Et Activez les **logs CloudWatch** et **alertes de sécurité** pour production.

## 7- Déploiement du Stack

1. **Validation du template :**

```bash
aws cloudformation validate-template \
  --template-body file://templates/devoiraws-template.yaml
```

2. **Déploiement du stack :**

Linux
```bash
aws cloudformation deploy \
  --template-file templates/devoiraws-template.yaml \
  --stack-name devoir-iabd-brillantdawoussou-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --no-fail-on-empty-changeset \
  --parameter-overrides EnvName=brillantdawoussou
```

Windows PowerShell

```bash
aws cloudformation deploy `
--template-file ".\templates\devoiraws-template-20250510.yaml" `
--stack-name "devoir-iabd-brillantdawoussou-stack" `
--capabilities CAPABILITY_NAMED_IAM `
--no-fail-on-empty-changeset `
--parameter-overrides EnvName=brillantdawoussou
```

NB: Lancer deux fois le déploiement. La deuxieme ouvrer le template et decommenter la section NotificationConfiguration avant de lancer le déploiement de la stack

## 8- Test du déclencheur S3 → Lambda

Pour tester l’insertion automatique dans DynamoDB via Lambda :

```bash
aws s3 cp FileToUpload s3://s3-bucket-file-metadata-brillantdawoussou-bucket
```

> Ici FileToUpload represente le fichier à uploader dans le bucket. Ce fichier va déclencher l’appel de la fonction Lambda et insérer un enregistrement dans DynamoDB.

## 9- Teste du déploiement effectué

    **1. Récupérer l’adresse IP publique de l’instance EC2**

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=brillantdawoussou-EC2Instance" \
  --query "Reservations[*].Instances[*].PublicIpAddress" \
  --output text
```

 **2. Connexion SSH à l'instance EC2**

    Windows/Linux/MacOS :

```bash
ssh -i sshkeys/iabdkey.pem ubuntu@<IP_PUBLIC>
```

    Windows PowerShell :

    **3. Alternative**

    a. (PowerShell) :

```powershell
$ip = aws ec2 describe-instances `
  --filters "Name=tag:Name,Values=brillantdawoussou-EC2Instance" `
  --query "Reservations[*].Instances[*].PublicIpAddress" `
  --output text

ssh -i .\sshkeys\iabdkey.pem ubuntu@$ip
```

    b. Linux / macOS :

```powershell
$ip = $(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=brillantdawoussou-EC2Instance" \
  --query "Reservations[*].Instances[*].PublicIpAddress" \
  --output text)
ssh -i ./sshkeys/iabdkey.pem ubuntu@$ip
```

Une fois connecter lancer la commande utile  : "**Vérifier le fichier de log d'exécution**" ecrit plut haut

## Auteur

**DAWOUSSOU KODZO SEWONU BRILLANT**

CloudFormation - Infrastructure AWS
