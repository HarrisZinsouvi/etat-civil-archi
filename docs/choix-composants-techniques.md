# Choix des composants techniques

## 1. Objectif

Ce document complète l'architecture cible en détaillant les composants techniques proposés pour répondre aux exigences de disponibilité, de résilience, de sécurité, d'observabilité, de montée en charge et de fonctionnement offline.

Les composants présentés constituent des **choix techniques proposés** et pourront être ajustés selon les contraintes d'exploitation, les compétences disponibles, les standards de l'organisation et les exigences réglementaires.

---

## 2. Vue d'ensemble

La stack cible pourra s'appuyer sur les briques suivantes :

| Domaine | Composant proposé | Rôle principal |
|---|---|---|
| Orchestration | Kubernetes | Déploiement, réplication, scaling et self-healing |
| Ingress | NGINX Ingress ou équivalent | Entrée HTTP/HTTPS vers les workloads |
| API | API Gateway | Sécurité, routage, rate limiting et exposition des API |
| Identité | Keycloak ou IdP d'entreprise | OIDC/OAuth2, MFA, RBAC et SSO |
| Données | PostgreSQL HA | Données transactionnelles |
| Cache | Redis | Cache et données temporaires |
| Messaging | Kafka ou RabbitMQ | Événements, files et découplage asynchrone |
| Documents | S3-compatible Object Storage | Documents et fichiers volumineux |
| Metrics | Prometheus | Collecte des métriques |
| Dashboards | Grafana | Visualisation et alerting |
| Traces | OpenTelemetry | Tracing distribué |
| Logs | Loki ou OpenSearch/Elasticsearch | Centralisation et recherche des logs |
| Sécurité | SIEM | Corrélation et analyse des événements de sécurité |
| Secrets | Vault ou Secret Manager cloud | Gestion des secrets |
| Backup | Velero + sauvegarde DB | Protection des workloads et restauration |
| Archive | Object Storage immuable / WORM | Conservation réglementaire |
| CI/CD | GitHub Actions, GitLab CI ou équivalent | Build, tests et déploiements |

Le choix définitif entre plusieurs solutions devra tenir compte du contexte d'exploitation.

---

# 3. Kubernetes

## 3.1 Pourquoi Kubernetes ?

Kubernetes est proposé comme plateforme d'orchestration des services centraux.

Il répond notamment aux besoins suivants :

- réplication horizontale des services ;
- redémarrage automatique des workloads défaillants ;
- déploiements progressifs ;
- isolation des workloads ;
- scaling horizontal ;
- gestion déclarative de la configuration ;
- répartition des workloads sur plusieurs nœuds ;
- intégration avec les mécanismes de monitoring et de sécurité.

L'objectif n'est pas d'utiliser Kubernetes pour chaque composant par principe, mais de disposer d'une plateforme permettant de faire évoluer les services applicatifs de manière contrôlée.

## 3.2 Organisation proposée

Une organisation logique pourra distinguer les workloads :

```text
Kubernetes Cluster
|
+-- ingress-system
|   +-- Ingress Controller
|
+-- api
|   +-- Actes
|   +-- Référentiels
|   +-- Utilisateurs
|   +-- Synchronisation
|
+-- messaging
|   +-- Producers / Consumers
|
+-- observability
|   +-- Metrics
|   +-- Logs
|   +-- Traces
|
+-- security
|   +-- Agents / Security components
|
+-- jobs
    +-- traitements batch
    +-- exports
    +-- maintenance
```

Les workloads critiques devront être répliqués et répartis sur plusieurs nœuds.

## 3.3 Scaling

Les services stateless pourront être scalés horizontalement :

```text
                 Load Balancer
                       |
                 Ingress Controller
                       |
          +------------+------------+
          |            |            |
       API Pod       API Pod      API Pod
          |            |            |
          +------------+------------+
                       |
                PostgreSQL / Cache
```

Un HPA pourra augmenter le nombre de replicas en fonction de métriques telles que CPU, mémoire ou métriques applicatives.

Pour une charge supérieure à 10x la charge nominale, le scaling applicatif devra cependant être complété par une capacité suffisante au niveau réseau, base de données, cache et messaging.

---

# 4. Ingress et exposition des API

## 4.1 Ingress Controller

Un Ingress Controller tel que **NGINX Ingress** pourra assurer l'entrée HTTP/HTTPS dans le cluster.

Il pourra notamment gérer :

- terminaison TLS ;
- routage ;
- règles d'accès ;
- redirections ;
- limitation de certaines requêtes ;
- exposition des services.

Un autre Ingress Controller pourra être retenu si l'organisation possède déjà un standard.

## 4.2 API Gateway

Une API Gateway pourra être placée devant les services exposés.

Elle pourra assurer :

- authentification ;
- validation des tokens ;
- rate limiting ;
- quotas ;
- routage ;
- versionnement des API ;
- contrôle des consommateurs ;
- journalisation des appels ;
- protection contre certains abus.

La Gateway ne devra pas devenir un point unique de panne : elle devra être déployée en plusieurs instances.

---

# 5. Base de données : PostgreSQL

Le document de stratégie de données propose PostgreSQL pour les actes transactionnels, notamment pour bénéficier des transactions ACID, des contraintes d'intégrité et des mécanismes de réplication et de sauvegarde. fileciteturn1file0L3-L7

## 5.1 Haute disponibilité

La base pourra être organisée autour de :

```text
              DB Proxy
                 |
        +--------+--------+
        |                 |
   PostgreSQL          PostgreSQL
    Primary              Standby
        |
        +------ replication ------>
```

Un mécanisme de failover devra permettre de basculer vers un standby en cas de défaillance du primaire.

La réplication distante alimentera le site PRA, conformément à la stratégie de résilience proposée. fileciteturn1file0L118-L128

## 5.2 Pourquoi une base relationnelle ?

Les actes d'état civil nécessitent :

- intégrité référentielle ;
- transactions ;
- contraintes ;
- cohérence forte pour les opérations critiques ;
- contrôle de version ;
- audit des modifications.

Le modèle proposé est hybride : cohérence éventuelle pour les opérations offline et cohérence forte pour les opérations critiques centrales. fileciteturn1file0L26-L33

---

# 6. Redis

Redis pourra être utilisé pour les données temporaires et les traitements nécessitant une faible latence.

Exemples :

- cache de référentiels fréquemment consultés ;
- sessions techniques lorsque nécessaire ;
- rate limiting ;
- verrous distribués dans certains cas ;
- données temporaires de synchronisation.

Redis ne devra pas devenir la source de vérité des actes d'état civil.

Les données transactionnelles resteront dans PostgreSQL.

---

# 7. Messaging : Kafka ou RabbitMQ

Un broker de messages permettra de découpler les traitements synchrones des traitements asynchrones.

## 7.1 Cas d'utilisation

Exemples :

```text
              Actes Service
                    |
                    v
                Event Bus
              /     |                   v      v       v
          Audit  Notification  Reporting
```

Le broker pourra également absorber les pics provoqués par la reconnexion simultanée de nombreuses communes.

Cette approche est cohérente avec la stratégie qui prévoit queue/broker, backpressure, retry avec backoff et limitation des synchronisations concurrentes. fileciteturn1file0L169-L180

## 7.2 Kafka ou RabbitMQ ?

**Kafka** sera particulièrement intéressant si :

- le volume d'événements est important ;
- plusieurs consommateurs doivent relire les événements ;
- la rétention des événements est importante ;
- l'architecture est fortement event-driven.

**RabbitMQ** pourra être privilégié si :

- le besoin principal est la messagerie applicative ;
- les files de travail et acknowledgements sont prioritaires ;
- l'organisation recherche une plateforme de messaging plus simple à exploiter.

Le choix définitif dépendra du volume et des patterns de traitement.

---

# 8. Object Storage

Les documents volumineux associés aux actes devront être stockés séparément des données transactionnelles, dans un Object Storage. La base conservera leurs métadonnées et références. fileciteturn1file0L5-L7

Une API compatible S3 pourra être utilisée.

Exemples de données :

```text
Object Storage
|
+-- actes/
|   +-- 2026/
|       +-- COM-001/
|
+-- justificatifs/
|
+-- exports/
|
+-- archives/
```

Avantages :

- stockage adapté aux fichiers volumineux ;
- scalabilité ;
- réplication ;
- versionnement ;
- chiffrement ;
- politiques de rétention ;
- possibilité d'immutabilité/WORM pour les archives.

---

# 9. Observabilité

L'observabilité devra couvrir trois dimensions :

```text
             Observability
                  |
       +----------+----------+
       |          |          |
     Metrics     Logs      Traces
       |          |          |
   Prometheus    Loki    OpenTelemetry
       |          |          |
       +----------+----------+
                  |
               Grafana
```

## 9.1 Metrics : Prometheus

Prometheus pourra collecter :

- CPU ;
- mémoire ;
- nombre de requêtes ;
- latence ;
- taux d'erreur ;
- disponibilité ;
- métriques Kubernetes ;
- métriques PostgreSQL ;
- métriques du broker.

Des alertes pourront être déclenchées sur des seuils techniques et métier.

## 9.2 Dashboards : Grafana

Grafana pourra centraliser les dashboards :

- santé du cluster ;
- API ;
- base de données ;
- synchronisations ;
- files d'attente ;
- erreurs ;
- disponibilité ;
- capacité.

Exemple de métriques utiles au projet :

```text
sync_pending_events
sync_conflicts_total
sync_failed_total
api_request_duration
api_requests_total
api_errors_total
db_connections
broker_queue_depth
```

## 9.3 Tracing : OpenTelemetry

OpenTelemetry permettra de suivre une requête à travers plusieurs services.

Exemple :

```text
Client
  |
  v
API Gateway
  |
  v
Actes Service
  |
  +--> PostgreSQL
  |
  +--> Event Bus
         |
         +--> Audit
         +--> Notification
```

Cela permettra notamment d'identifier où se situe une latence ou une erreur dans une chaîne distribuée.

---

# 10. Centralisation des logs

Deux orientations pourront être retenues.

## Option A : Loki + Grafana

Loki sera intéressant si l'objectif principal est :

- centralisation des logs Kubernetes ;
- intégration simple avec Grafana ;
- maîtrise du coût de stockage ;
- recherche principalement opérationnelle.

Architecture :

```text
Applications
     |
     v
Log Collector
     |
     v
   Loki
     |
     v
  Grafana
```

## Option B : OpenSearch / Elasticsearch

Cette option sera intéressante lorsque les besoins de recherche, d'analyse et de corrélation des logs seront plus avancés.

Exemples :

- recherche full-text ;
- analyse détaillée ;
- dashboards avancés ;
- investigations ;
- intégration avec des outils de sécurité.

Le choix entre Loki et OpenSearch/Elasticsearch dépendra donc des besoins d'exploitation et de sécurité.

---

# 11. SIEM

Le SIEM devra être distingué de la simple centralisation des logs.

Les logs techniques répondent principalement à la question :

> « Que s'est-il passé techniquement ? »

Le SIEM répond davantage à :

> « Y a-t-il un comportement de sécurité suspect ? »

Il pourra agréger :

- authentifications ;
- échecs de connexion ;
- élévations de privilèges ;
- changements de rôles ;
- accès sensibles ;
- événements WAF ;
- événements API Gateway ;
- événements Kubernetes ;
- événements système.

Exemple :

```text
IAM ----WAF -----Gateway ---+--> SIEM --> Corrélation --> Alertes sécurité
K8s ------/
Servers --/
```

Le SIEM pourra être une solution existante de l'organisation plutôt qu'un composant imposé par l'architecture.

---

# 12. Audit métier

L'audit métier devra rester distinct des logs techniques et du SIEM.

Pour un acte d'état civil, il devra notamment permettre de retrouver :

```text
Qui ?
Quand ?
Quelle opération ?
Sur quel acte ?
Quelle ancienne version ?
Quelle nouvelle version ?
Depuis quelle commune ?
Quel résultat ?
```

Les conflits devront également être conservés, audités et soumis à une résolution métier habilitée. fileciteturn1file0L71-L84

L'audit critique devra être conservé dans un stockage approprié et protégé contre les modifications non autorisées.

---

# 13. IAM et sécurité

Une solution IAM compatible OAuth2/OIDC pourra être utilisée pour centraliser l'authentification et les autorisations.

Elle devra permettre selon les profils :

- authentification centralisée ;
- SSO ;
- MFA pour les profils sensibles ;
- RBAC ;
- gestion des rôles ;
- révocation des accès ;
- fédération avec un annuaire ou un Identity Provider existant.

Exemple :

```text
Agent commune
      |
      v
     IAM
      |
   OIDC Token
      |
      v
 API Gateway
      |
      v
Services
```

Les secrets techniques ne devront pas être stockés dans les manifests Kubernetes ou dans le dépôt Git.

---

# 14. Gestion des secrets

Un Secret Manager tel que Vault ou le service équivalent du cloud retenu pourra gérer :

- mots de passe ;
- credentials DB ;
- certificats ;
- clés API ;
- secrets de connexion aux brokers.

Kubernetes Secrets pourra être utilisé pour l'intégration avec les workloads, mais les secrets sensibles devront idéalement provenir d'un gestionnaire dédié.

---

# 15. Sauvegarde et PRA

La stratégie de données prévoit :

- sauvegardes complètes ;
- incrémentales lorsque disponibles ;
- archivage WAL/PITR ;
- chiffrement ;
- stockage indépendant ;
- copie immuable ;
- tests réguliers de restauration. fileciteturn1file0L130-L140

Pour Kubernetes, **Velero** pourra être utilisé pour sauvegarder les ressources du cluster et certains volumes compatibles.

Il ne remplacera cependant pas une stratégie de sauvegarde PostgreSQL adaptée.

Architecture simplifiée :

```text
                 PRIMARY SITE
                      |
          +-----------+-----------+
          |                       |
     PostgreSQL HA          Object Storage
          |                       |
          +-----------+-----------+
                      |
                 Backup System
                      |
             +--------+--------+
             |                 |
        Backup Store       Immutable Copy
             |
             v
          PRA Site
```

Les objectifs de conception proposés dans l'architecture sont :

- RPO ≤ 5 minutes ;
- RTO ≤ 30 minutes ;
- disponibilité cible ≥ 99,9 %.

Ces valeurs devront être validées avec le métier et l'infrastructure. fileciteturn1file0L157-L167

---

# 16. CI/CD

Une chaîne CI/CD pourra automatiser :

```text
Git
 |
 v
Build
 |
 v
Tests
 |
 v
Security Scan
 |
 v
Container Image
 |
 v
Registry
 |
 v
Deployment
 |
 v
Kubernetes
```

Les contrôles pourront inclure :

- tests unitaires ;
- tests d'intégration ;
- analyse de code ;
- scan des dépendances ;
- scan d'images ;
- validation des manifests ;
- déploiement progressif.

Une approche GitOps pourra ensuite être retenue si elle apporte une valeur opérationnelle suffisante.

---

# 17. Registry d'images

Les images applicatives devront être publiées dans un registry privé.

Le registry permettra notamment :

- contrôle des images ;
- versionnement ;
- scan de vulnérabilités ;
- gestion des accès ;
- reproductibilité des déploiements.

Les images utilisées en production devront être versionnées et idéalement immuables.

---

# 18. Synchronisation offline

La synchronisation devra utiliser une Outbox locale, un identifiant d'événement unique, l'idempotence, les ACK et les retries avec backoff. fileciteturn1file0L35-L45

Architecture :

```text
                 COMMUNE
                    |
              Application
                    |
             Local Database
                    |
                 Outbox
                    |
             Synchronisation
                    |
             API de Sync
                    |
          Validation / Auth
                    |
          Idempotency Check
                    |
              PostgreSQL
                    |
                   ACK
                    |
             Local Database
```

Une retransmission du même `eventId` ne devra pas provoquer une seconde création. fileciteturn1file0L59-L68

Le mécanisme de versionnement sera préféré à Last Write Wins pour les actes d'état civil. fileciteturn1file0L71-L84

---

# 19. Synthèse des choix

| Besoin | Choix proposé | Justification |
|---|---|---|
| Orchestration | Kubernetes | Scaling, self-healing, déploiement déclaratif |
| Entrée HTTP | Ingress Controller | Routage et TLS |
| API | API Gateway | Sécurité, quotas, rate limiting |
| IAM | OIDC/OAuth2 + IdP | Authentification et RBAC |
| Transactionnel | PostgreSQL HA | ACID et intégrité |
| Cache | Redis | Faible latence et cache |
| Async | Kafka ou RabbitMQ | Découplage et absorption des pics |
| Documents | Object Storage | Scalabilité des fichiers |
| Metrics | Prometheus | Monitoring |
| Dashboards | Grafana | Exploitation et alerting |
| Traces | OpenTelemetry | Traçabilité distribuée |
| Logs | Loki ou OpenSearch | Centralisation et recherche |
| Sécurité | SIEM | Détection et corrélation sécurité |
| Secrets | Vault / Secret Manager | Protection des credentials |
| Backup | Backup + PITR + Velero | Restauration |
| Archive | Object Storage immuable/WORM | Conservation réglementaire |
| CI/CD | GitHub Actions / GitLab CI | Automatisation |
| Registry | Registry privé | Contrôle des images |

---

# 20. Principe de sélection

Les technologies ne devront pas être considérées comme des fins en soi.

Le choix final devra respecter les principes suivants :

1. **Répondre aux exigences métier avant les préférences technologiques.**
2. **Éviter les single points of failure.**
3. **Privilégier les composants maîtrisables par l'équipe d'exploitation.**
4. **Éviter de multiplier inutilement les produits.**
5. **Privilégier les standards ouverts lorsque cela réduit le verrouillage fournisseur.**
6. **Dimensionner les composants pour les pics de charge et les reconnexions offline.**
7. **Prévoir la supervision et la restauration dès la conception.**
8. **Séparer les responsabilités : transactionnel, cache, messaging, logs, audit et sécurité.**

L'architecture cible devra ainsi rester indépendante d'un fournisseur ou d'une technologie particulière tout en disposant d'une proposition technique concrète permettant son implémentation.
