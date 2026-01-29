# Guide des Tests

Ce document explique comment exécuter les différents types de tests dans le projet MuchToDo.

## 📋 Types de tests

### 1. Tests Unitaires
Tests rapides qui ne nécessitent pas de dépendances externes.

```bash
# Exécuter tous les tests unitaires
go test -v ./...

# Avec couverture de code
go test -coverprofile=coverage.out -covermode=atomic ./...
go tool cover -func=coverage.out

# Avec race detection (Linux/macOS uniquement)
go test -v -race ./...
```

### 2. Tests d'Intégration
Tests qui nécessitent MongoDB et Redis via Docker.

#### Sur Linux/macOS avec Docker Desktop

```bash
# Démarrer les services requis
docker-compose up -d

# Exécuter les tests d'intégration
INTEGRATION=1 go test -v -tags=integration ./...

# Arrêter les services
docker-compose down
```

#### Sur Windows
Les tests d'intégration ne peuvent pas s'exécuter localement sur Windows à cause des limitations de testcontainers. Ils s'exécutent automatiquement dans le workflow GitHub Actions.

### 3. Tests dans GitHub Actions
Le workflow CI/CD exécute automatiquement :
- ✅ Tests unitaires avec race detection
- ✅ Tests unitaires avec couverture de code
- ✅ Tests d'intégration (sur Linux avec services Docker)
- ✅ Scan de sécurité (Gosec, Govulncheck)

## 🐳 Configuration Docker pour tests locaux

Le fichier `docker-compose.yaml` fournit MongoDB et Redis pour les tests d'intégration :

```bash
# Démarrer les services
docker-compose up -d

# Vérifier que les services sont prêts
docker-compose ps

# Voir les logs
docker-compose logs -f

# Arrêter les services
docker-compose down -v  # -v pour supprimer les volumes
```

## 📊 Couverture de Code

### Voir le rapport de couverture en HTML

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### Objectifs de couverture
- **Tests unitaires** : > 75%
- **Tests d'intégration** : Tests des endpoints principaux

## 🔍 Vérifications de Code

### Go Vet
```bash
go vet ./...
```

### Go Fmt
```bash
# Vérifier le formatage
gofmt -s -l .

# Formater automatiquement
gofmt -s -w .
```

### Gosec (Scan de sécurité)
```bash
# Installer gosec
go install github.com/securego/gosec/v2/cmd/gosec@latest

# Exécuter le scan
gosec ./...
```

### Govulncheck (Vulnérabilités)
```bash
# Installer govulncheck
go install golang.org/x/vuln/cmd/govulncheck@latest

# Vérifier les vulnérabilités
govulncheck ./...
```

## 🧪 Structure des Tests

### Tests Unitaires
Fichiers : `*_test.go` (sans tag de build)
- `internal/auth/auth_test.go` - Tests du service d'authentification

### Tests d'Intégration
Fichiers : `*_test.go` avec `//go:build integration`
- `internal/handlers/handlers_test.go` - Tests des endpoints HTTP

## ✅ Checklist avant commit

```bash
# 1. Formater le code
gofmt -s -w .

# 2. Vérifier avec go vet
go vet ./...

# 3. Exécuter les tests unitaires
go test -v ./...

# 4. Vérifier la couverture
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out

# 5. (Optionnel) Tests d'intégration si Docker disponible
INTEGRATION=1 go test -v -tags=integration ./...
```

## 🚨 Dépannage

### Erreur "no such tool 'covdata'"
**Solution** : Assurez-vous d'utiliser Go 1.22 ou supérieur
```bash
go version  # Doit afficher go1.22 ou plus
```

### Erreur "rootless Docker is not supported on Windows"
**Solution** : Les tests d'intégration ne peuvent pas s'exécuter sur Windows. Utilisez le workflow GitHub Actions ou WSL2 avec Docker.

### Tests qui passent en local mais échouent dans CI/CD
**Solution** : Vérifiez que :
- La version Go est la même (1.22)
- Les dépendances sont à jour (`go mod tidy`)
- Le code est formaté (`gofmt -s -w .`)

## 📚 Ressources

- [Testing in Go](https://go.dev/doc/tutorial/add-a-test)
- [Testify Suite](https://pkg.go.dev/github.com/stretchr/testify/suite)
- [Testcontainers Go](https://golang.testcontainers.org/)
- [Go Coverage](https://go.dev/blog/cover)
