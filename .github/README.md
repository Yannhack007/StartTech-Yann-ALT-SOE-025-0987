# GitHub Actions & Infrastructure Documentation

Ce dossier contient tous les workflows CI/CD et la documentation d'infrastructure pour le projet MuchToDo.

## 📁 Structure

```
.github/
├── workflows/
│   └── backend-ci-cd.yml       # Pipeline CI/CD du backend
├── AWS_SETUP.md                # Guide de configuration AWS
├── DEPLOYMENT_CHECKLIST.md     # Checklist complète de déploiement
├── SECRETS.md                  # Documentation des secrets GitHub
└── README.md                   # Ce fichier
```

## 🚀 Quick Start

### Pour déployer le backend:

1. **Configurer les secrets GitHub** (voir [SECRETS.md](SECRETS.md))
   - Secrets AWS obligatoires
   - URI MongoDB
   - JWT Secret Key

2. **Créer l'infrastructure AWS** (voir [AWS_SETUP.md](AWS_SETUP.md))
   - ECR Repository
   - VPC et réseau
   - Launch Template
   - Auto Scaling Group
   - Application Load Balancer

3. **Vérifier la configuration** (voir [DEPLOYMENT_CHECKLIST.md](DEPLOYMENT_CHECKLIST.md))
   - Tous les secrets sont configurés
   - Infrastructure AWS est en place
   - Tests locaux passent

4. **Déclencher le déploiement**
   ```bash
   git push origin main
   ```

## 📋 Workflows disponibles

### Backend CI/CD Pipeline
**Fichier:** `workflows/backend-ci-cd.yml`

**Déclencheurs:**
- Push sur `main` avec changements dans `Server/MuchToDo/**`
- Pull Request vers `main`
- Déclenchement manuel

**Jobs:**
1. **test** - Tests unitaires, vérifications de code
2. **security-scan** - Scan de sécurité avec Gosec et Govulncheck
3. **build-and-push** - Build et push de l'image Docker vers ECR
4. **deploy** - Déploiement sur EC2 Auto Scaling via instance refresh

**Durée estimée:** 15-20 minutes

## 🔐 Secrets requis

Voir [SECRETS.md](SECRETS.md) pour la liste complète.

### Obligatoires
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `MONGO_URI`
- `JWT_SECRET_KEY`

### Optionnels
- `REDIS_ADDR`, `REDIS_PASSWORD`, `ENABLE_CACHE`
- `ALLOWED_ORIGINS`, `COOKIE_DOMAINS`, `SECURE_COOKIE`

## 🏗️ Infrastructure AWS

Voir [AWS_SETUP.md](AWS_SETUP.md) pour les commandes de création.

### Ressources requises
- ECR Repository: `starttech-backend`
- Launch Template avec tag: `Name=starttech-backend-lt`
- Auto Scaling Group avec tag contenant: `starttech-backend`
- ALB avec nom contenant: `starttech`
- Security Groups, IAM Roles, VPC, etc.

### Région
**eu-north-1** (Stockholm)

## 📊 Monitoring

### CloudWatch Logs
Les logs de l'application sont envoyés vers:
```
/aws/ec2/starttech-backend
```

### Métriques
- Health checks via ALB
- Métriques Auto Scaling
- Logs des containers Docker

### Accès aux logs
```bash
aws logs tail /aws/ec2/starttech-backend --follow --region eu-north-1
```

## 🐛 Dépannage

### Le workflow échoue aux tests
```bash
# Vérifier localement
cd Server/MuchToDo
go test -v ./...
go vet ./...
gofmt -s -l .
```

### Le build Docker échoue
```bash
# Tester le build localement
cd Server/MuchToDo
docker build -t starttech-backend:test .
```

### Le déploiement échoue
1. Vérifier que tous les secrets sont configurés
2. Vérifier que l'infrastructure AWS existe
3. Consulter les logs dans GitHub Actions
4. Vérifier les logs CloudWatch

### L'application ne démarre pas
```bash
# Vérifier les logs du container
aws ssm start-session --target <instance-id>
docker logs backend
```

## 🔄 Workflow de développement

### Pour une nouvelle fonctionnalité
```bash
# Créer une branche
git checkout -b feature/my-feature

# Faire vos modifications
# ...

# Commit et push
git add .
git commit -m "Add my feature"
git push origin feature/my-feature

# Créer une Pull Request sur GitHub
# Le workflow test s'exécutera automatiquement
```

### Pour un hotfix en production
```bash
# Créer une branche hotfix
git checkout -b hotfix/fix-issue

# Appliquer le fix
# ...

# Merge vers main
git checkout main
git merge hotfix/fix-issue
git push origin main

# Le déploiement se lance automatiquement
```

## 📚 Ressources

- [GitHub Actions Documentation](https://docs.github.com/actions)
- [AWS ECR Documentation](https://docs.aws.amazon.com/ecr/)
- [AWS Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)

## 💡 Bonnes pratiques

1. **Toujours tester localement** avant de push
2. **Utiliser des Pull Requests** pour les changements importants
3. **Monitorer les logs** après chaque déploiement
4. **Garder les secrets à jour** et sécurisés
5. **Documenter les changements** dans les commits
6. **Vérifier les coûts AWS** régulièrement

## 📞 Support

Pour toute question ou problème:
1. Consulter [DEPLOYMENT_CHECKLIST.md](DEPLOYMENT_CHECKLIST.md)
2. Vérifier les logs GitHub Actions
3. Consulter les logs CloudWatch
4. Ouvrir une issue sur GitHub

## 🔄 Mises à jour

Ce fichier et les workflows sont maintenus activement. Consultez l'historique Git pour les changements récents:
```bash
git log --follow .github/
```
