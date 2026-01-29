# ✅ Checklist de déploiement Backend CI/CD

Ce document récapitule tous les éléments nécessaires pour que le workflow de déploiement fonctionne correctement.

## 🔧 Corrections apportées au workflow

### 1. ✅ Variables d'environnement corrigées
**Problème:** Les noms de variables dans le workflow ne correspondaient pas à ceux attendus par l'application Go.

**Correction:**
- `MONGODB_URI` → `MONGO_URI`
- `REDIS_ENDPOINT` → `REDIS_ADDR`
- Ajout de toutes les variables requises par l'application

### 2. ✅ Substitution des variables dans user-data
**Problème:** Le heredoc avec quotes simples `'USERDATA'` empêchait la substitution des variables.

**Correction:**
- Changement en `USERDATA` (sans quotes)
- Utilisation de `${{ env.VAR }}` pour les variables GitHub
- Utilisation de `${VAR}` pour les variables bash substituées

### 3. ✅ Dockerfile permanent créé
**Problème:** Le Dockerfile était généré à chaque build, ce qui n'est pas une bonne pratique.

**Correction:**
- Création de [Server/MuchToDo/Dockerfile](../../Server/MuchToDo/Dockerfile)
- Optimisation multi-stage avec Alpine Linux
- Ajout d'un utilisateur non-root pour la sécurité
- Ajout d'un health check

### 4. ✅ Fichier .dockerignore créé
**Correction:**
- Création de [Server/MuchToDo/.dockerignore](../../Server/MuchToDo/.dockerignore)
- Exclusion des fichiers inutiles pour optimiser le build

### 5. ✅ Version Go mise à jour
**Problème:** Go 1.21 ne supportait pas l'outil `covdata` requis pour les tests avec couverture et race detection.

**Correction:**
- Mise à jour vers Go 1.22
- Séparation des tests race et couverture

## 📋 Actions requises pour le déploiement

### Étape 1: Configurer les secrets GitHub

Aller dans **Settings** → **Secrets and variables** → **Actions** et ajouter:

#### Secrets OBLIGATOIRES ⚠️
```
AWS_ACCESS_KEY_ID         # Clé d'accès AWS
AWS_SECRET_ACCESS_KEY     # Clé secrète AWS
MONGO_URI                 # URI MongoDB (ex: mongodb+srv://user:pass@cluster.mongodb.net/)
JWT_SECRET_KEY            # Clé secrète JWT (minimum 32 caractères aléatoires)
```

#### Secrets OPTIONNELS
```
REDIS_ADDR                # Adresse Redis (ex: redis.example.com:6379)
REDIS_PASSWORD            # Mot de passe Redis
ENABLE_CACHE              # true ou false (défaut: false)
ALLOWED_ORIGINS           # Origines CORS (ex: https://app.com,https://www.app.com)
COOKIE_DOMAINS            # Domaines cookies (ex: example.com,.example.com)
SECURE_COOKIE             # true ou false (défaut: false, mettre true en production)
```

**Documentation complète:** [SECRETS.md](.github/SECRETS.md)

### Étape 2: Créer l'infrastructure AWS

Créer les ressources AWS suivantes dans la région `eu-north-1`:

1. **ECR Repository:** `starttech-backend`
2. **VPC et sous-réseaux** (2 AZs minimum)
3. **Security Groups** (ALB et EC2)
4. **IAM Role** avec permissions ECR + CloudWatch
5. **Target Group** (port 8080, health check sur `/health`)
6. **Application Load Balancer** (nom contenant "starttech")
7. **Launch Template** avec tag `Name=starttech-backend-lt`
8. **Auto Scaling Group** avec tag contenant "starttech-backend"

**Guide détaillé:** [AWS_SETUP.md](.github/AWS_SETUP.md)

### Étape 3: Vérifier la configuration

```bash
# Vérifier que l'ECR existe
aws ecr describe-repositories \
  --repository-names starttech-backend \
  --region eu-north-1

# Vérifier le Launch Template
aws ec2 describe-launch-templates \
  --filters "Name=tag:Name,Values=starttech-backend-lt" \
  --region eu-north-1

# Vérifier l'Auto Scaling Group
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(Tags[?Key=='Name'].Value, 'starttech-backend')]" \
  --region eu-north-1

# Vérifier l'ALB
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?contains(LoadBalancerName, 'starttech')]" \
  --region eu-north-1
```

### Étape 4: Tester localement (optionnel)

```bash
cd Server/MuchToDo

# Copier et configurer l'environnement
cp .env.example .env
# Éditer .env avec vos vraies valeurs

# Option 1: Exécuter directement
go run cmd/api/main.go

# Option 2: Construire et tester le Docker
docker build -t starttech-backend:local .
docker run -p 8080:8080 --env-file .env starttech-backend:local

# Tester le health check
curl http://localhost:8080/health
```

### Étape 5: Déclencher le déploiement

```bash
# Commit et push vers main
git add .
git commit -m "Configure backend CI/CD pipeline"
git push origin main

# Ou déclencher manuellement via GitHub Actions
# Aller dans Actions → Backend CI/CD Pipeline → Run workflow
```

## 🔍 Monitoring du déploiement

### Phase 1: Tests (2-3 minutes)
- ✅ Tests unitaires
- ✅ Go vet
- ✅ Go fmt
- ✅ Couverture de code

### Phase 2: Security Scan (1-2 minutes)
- ✅ Gosec
- ✅ Govulncheck

### Phase 3: Build & Push (3-5 minutes)
- ✅ Build Docker image
- ✅ Push vers ECR
- ✅ Scan Trivy

### Phase 4: Déploiement (10-15 minutes)
- ✅ Mise à jour Launch Template
- ✅ Instance Refresh
- ✅ Smoke tests
- ✅ Vérification CloudWatch

## 🐛 Dépannage

### Erreur: "Launch template not found"
```bash
# Vérifier que le tag existe
aws ec2 describe-launch-templates \
  --filters "Name=tag:Name,Values=starttech-backend-lt" \
  --region eu-north-1
```

### Erreur: "Auto Scaling Group not found"
```bash
# Vérifier les tags de l'ASG
aws autoscaling describe-auto-scaling-groups \
  --region eu-north-1 \
  --query 'AutoScalingGroups[*].[AutoScalingGroupName,Tags]'
```

### Erreur: "Health check failed"
```bash
# Vérifier les logs CloudWatch
aws logs tail /aws/ec2/starttech-backend --follow

# Se connecter à l'instance
aws ssm start-session --target <instance-id>

# Vérifier les containers
docker ps
docker logs backend
```

### Erreur: "no such tool 'covdata'"
✅ **Résolu** - Mise à jour vers Go 1.22

### Variables d'environnement non reconnues
✅ **Résolu** - Noms corrigés pour correspondre à l'application

## 📊 Indicateurs de succès

- ✅ Tests passent avec > 75% de couverture
- ✅ Aucune vulnérabilité critique
- ✅ Image Docker < 50MB
- ✅ Déploiement en < 15 minutes
- ✅ Health check répond 200 OK
- ✅ Logs apparaissent dans CloudWatch

## 📚 Fichiers de référence

- [backend-ci-cd.yml](workflows/backend-ci-cd.yml) - Workflow GitHub Actions
- [SECRETS.md](SECRETS.md) - Configuration des secrets
- [AWS_SETUP.md](AWS_SETUP.md) - Guide d'infrastructure AWS
- [Server/MuchToDo/Dockerfile](../Server/MuchToDo/Dockerfile) - Configuration Docker
- [Server/MuchToDo/.env.example](../Server/MuchToDo/.env.example) - Variables d'environnement

## ⚠️ Notes importantes

1. **Production**: Activer HTTPS et mettre `SECURE_COOKIE=true`
2. **Cache**: Activer Redis avec `ENABLE_CACHE=true` pour de meilleures performances
3. **Coûts**: Budget estimé ~$25-30/mois pour la configuration minimale
4. **Monitoring**: Configurer CloudWatch Alarms pour la production
5. **Backups**: Implémenter une stratégie de backup pour MongoDB

## 🎉 Prochaines étapes

Une fois le déploiement réussi:
1. Configurer un nom de domaine avec Route 53
2. Ajouter un certificat SSL avec ACM
3. Configurer des alarmes CloudWatch
4. Implémenter des métriques custom
5. Ajouter des tests d'intégration
6. Configurer le CI/CD pour le frontend
