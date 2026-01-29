# GitHub Secrets Configuration

Ce document liste tous les secrets GitHub qui doivent être configurés pour le workflow CI/CD.

## 📋 Secrets Requis

### AWS Credentials (OBLIGATOIRES)
```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```
**Description:** Identifiants AWS pour accéder à ECR, EC2, et autres services AWS.

### Base de données (OBLIGATOIRE)
```
MONGO_URI
```
**Description:** URI de connexion MongoDB  
**Exemple:** `mongodb+srv://username:password@cluster.mongodb.net/`

### Authentication (OBLIGATOIRE)
```
JWT_SECRET_KEY
```
**Description:** Clé secrète pour signer les tokens JWT  
**Exemple:** `your-super-secret-key-that-is-long-and-random-at-least-32-chars`

### Redis/Cache (OPTIONNEL)
```
ENABLE_CACHE
```
**Description:** Active ou désactive le cache Redis  
**Valeurs:** `true` ou `false`  
**Par défaut:** `false`

```
REDIS_ADDR
```
**Description:** Adresse du serveur Redis (si cache activé)  
**Exemple:** `my-redis-cluster.cache.amazonaws.com:6379`

```
REDIS_PASSWORD
```
**Description:** Mot de passe Redis (si nécessaire)  
**Par défaut:** vide

### CORS et Cookies (OPTIONNEL)
```
ALLOWED_ORIGINS
```
**Description:** Origines autorisées pour CORS (séparées par des virgules)  
**Exemple:** `https://app.example.com,https://www.example.com`  
**Par défaut:** `http://localhost:5173`

```
COOKIE_DOMAINS
```
**Description:** Domaines autorisés pour les cookies (séparés par des virgules)  
**Exemple:** `example.com,.example.com`  
**Par défaut:** `localhost`

```
SECURE_COOKIE
```
**Description:** Active les cookies sécurisés (HTTPS uniquement)  
**Valeurs:** `true` ou `false`  
**Par défaut:** `false`

## 🔧 Comment configurer les secrets

1. Allez dans votre repository GitHub
2. Cliquez sur **Settings** → **Secrets and variables** → **Actions**
3. Cliquez sur **New repository secret**
4. Ajoutez chaque secret avec son nom exact et sa valeur

## ⚠️ Important

- Ne commitez JAMAIS les valeurs réelles des secrets dans le code
- Les secrets marqués comme OBLIGATOIRES doivent être configurés pour que le déploiement fonctionne
- Les valeurs par défaut sont utilisées si les secrets optionnels ne sont pas configurés
- Pour la production, assurez-vous que:
  - `SECURE_COOKIE=true` (si HTTPS)
  - `ENABLE_CACHE=true` (pour de meilleures performances)
  - `JWT_SECRET_KEY` est une valeur aléatoire et complexe

## 📝 Infrastructure AWS requise

Avant le déploiement, assurez-vous que les ressources AWS suivantes existent:

- ✅ ECR Repository: `starttech-backend`
- ✅ Launch Template avec tag: `Name=starttech-backend-lt`
- ✅ Auto Scaling Group avec tag contenant: `starttech-backend`
- ✅ Application Load Balancer avec nom contenant: `starttech`
- ✅ IAM Role pour EC2 avec permissions:
  - ECR pull images
  - CloudWatch logs
  - Auto Scaling
- ✅ Security Groups configurés pour le trafic HTTP/HTTPS

## 🧪 Test local

Pour tester localement avant le déploiement:

```bash
cd Server/MuchToDo

# Copier le fichier d'exemple
cp .env.example .env

# Éditer .env avec vos valeurs
# Puis lancer l'application
go run cmd/api/main.go
```
