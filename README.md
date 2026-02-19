# archiCICD
🚀 CI/CD Pipeline – Spring Petclinic REST (Enterprise DevOps)

📌 Projet

Pipeline CI/CD complet basé sur le projet officiel :

🔗 https://github.com/spring-petclinic/spring-petclinic-rest

Ce pipeline implémente une chaîne DevOps complète incluant :

✅ Intégration Continue (CI)

✅ Analyse qualité (SonarQube)

✅ Scan sécurité (Trivy)

✅ Containerisation Docker

✅ Déploiement Staging

✅ Tests E2E REST

✅ API Chaining

🎯 Objectif du Pipeline

L’objectif est de garantir que chaque modification du code :

Compile correctement

Passe les tests unitaires

Respecte les standards qualité

Ne contient pas de vulnérabilités critiques

Se déploie correctement

Exécute un scénario métier complet sans erreur

👉 Si une étape échoue → le pipeline est stoppé automatiquement.

🏗️ Architecture Générale
Developer Push
      ↓
    Jenkins
      ↓
 ┌──────────────────────┐
 │        CI            │
 │  - Checkout          │
 │  - Build Maven       │
 │  - Unit Tests        │
 │  - SonarQube         │
 └──────────────────────┘
      ↓
 ┌──────────────────────┐
 │     SECURITY         │
 │  - Docker Build      │
 │  - Trivy Scan        │
 └──────────────────────┘
      ↓
 ┌──────────────────────┐
 │      STAGING         │
 │  - Deploy Container  │
 │  - Health Check      │
 │  - E2E CRUD Tests    │
 │  - API Chaining      │
 └──────────────────────┘
      ↓
   Cleanup automatique


✅ Tests automatiques de non-régression



🧪 Tests E2E – Full CRUD Enterprise
🎯 But

Valider un scénario métier complet via API REST.

Ce test est un test de non-régression fonctionnelle automatisé.

🔗 API Chaining – Principe

Les appels API sont dépendants les uns des autres.

Étapes exécutées :
1️⃣ CREATE

POST /api/owners
→ récupérer l’ID

2️⃣ READ

GET /api/owners/{id}
→ vérifier les données

3️⃣ UPDATE

PUT /api/owners/{id}
→ modifier les données

4️⃣ DELETE

DELETE /api/owners/{id}

Si une étape échoue → le pipeline échoue.

🧠 Pourquoi c’est important ?

Si un développeur :

Change le mapping JSON

Modifie les status HTTP

Casse un endpoint

Change la sécurité

Introduit une régression

👉 Le pipeline le détecte immédiatement.
