# Stratégie de données et résilience

## 1. Modèle de données

Les actes d'état civil devront être considérés comme des données transactionnelles structurées. Une base relationnelle de type PostgreSQL est proposée afin de bénéficier de transactions ACID, de contraintes d'intégrité et de mécanismes robustes de réplication et de sauvegarde.

Les documents volumineux associés aux actes devront être stockés dans un Object Storage. La base devra conserver leurs métadonnées et références.

### Entité conceptuelle

```text
CivilRecord
------------
id
record_type
status
version
commune_id
created_at
updated_at
created_by
integrity_hash
document_reference
```

## 2. Cohérence

Le modèle devra être hybride :

- **offline : cohérence éventuelle** entre la commune et le SI national ;
- **central : cohérence forte** pour les opérations critiques de validation et de modification.

Le système national devra constituer la source d'autorité pour les actes définitivement validés.

## 3. Synchronisation

La synchronisation devra reposer sur :

- Outbox locale ;
- identifiant d'événement unique ;
- idempotence ;
- accusé de réception ;
- retry avec backoff ;
- gestion explicite des erreurs ;
- dead-letter queue pour les événements bloqués durablement.

### Exemple d'événement

```json
{
  "eventId": "evt-2026-000001234",
  "aggregateId": "acte-2026-00005678",
  "operation": "CREATE",
  "version": 1,
  "communeId": "COM-001"
}
```

## 4. Idempotence

Le serveur devra conserver les identifiants d'événements traités ou disposer d'un mécanisme équivalent permettant leur déduplication.

Une retransmission du même `eventId` ne devra pas provoquer une seconde création.

```text
evt-123 -> traitement
evt-123 -> déjà traité
evt-123 -> déjà traité
```

## 5. Gestion des conflits

`Last Write Wins` ne devra pas être retenu pour les actes d'état civil.

Le contrôle optimiste par version devra être utilisé :

```text
Version centrale = 3

Client A -> modification version 4 -> OK
Client B -> modification basée sur version 3 -> CONFLICT
```

Le conflit devra être conservé, audité et soumis à une résolution métier habilitée.

## 6. Intégrité

Chaque acte ou événement critique pourra être associé à une empreinte cryptographique.

```text
Donnée canonique
      |
    SHA-256
      |
 integrity_hash
```

Pour les cas nécessitant une preuve forte, une signature électronique/numérique pourra être ajoutée.

## 7. Cycle de vie

```text
DRAFT
  |
PENDING_SYNC
  |
RECEIVED
  |
VALIDATED
  |
CONFIRMED
  |
ARCHIVED
```

Une correction d'un acte confirmé devra passer par un workflow contrôlé et audité.

## 8. Haute disponibilité

Le PostgreSQL central devra être déployé avec redondance :

- nœud primaire ;
- nœud(s) standby ;
- réplication adaptée à la topologie ;
- mécanisme de failover ;
- proxy ou couche de routage DB.

Une réplication distante asynchrone devra alimenter le site PRA.

## 9. Sauvegardes

La stratégie devra comprendre :

- sauvegardes complètes ;
- sauvegardes incrémentales lorsque disponibles ;
- archivage WAL / PITR ;
- chiffrement ;
- stockage indépendant ;
- copie immuable ;
- tests réguliers de restauration.

## 10. Archivage légal

Les données soumises à conservation réglementaire devront être transférées vers un stockage d'archives approprié.

L'archivage devra prendre en compte :

- durée de conservation ;
- rétention ;
- droits d'accès ;
- intégrité ;
- traçabilité ;
- mécanismes d'immutabilité/WORM lorsque requis.

Les durées exactes devront être définies avec les parties prenantes réglementaires.

## 11. RPO / RTO

Objectifs de conception proposés :

| Indicateur | Cible |
|---|---:|
| RPO | ≤ 5 minutes |
| RTO | ≤ 30 minutes |
| Disponibilité cible | ≥ 99,9 % |

Ces valeurs devront être validées par le métier et les responsables infrastructure.

## 12. Reconnexion massive

Après une coupure prolongée, plusieurs communes pourront se reconnecter simultanément.

Pour éviter un pic brutal :

- backpressure ;
- rate limiting ;
- queue/broker ;
- retry avec backoff ;
- traitement par lots si approprié ;
- limitation du nombre de synchronisations concurrentes.

## 13. Commune offline

Une opération validement enregistrée localement devra rester conservée jusqu'à confirmation de sa synchronisation centrale.

```text
LOCAL
  |
  v
PENDING
  |
  v
SENDING
  |
  v
CENTRAL VALIDATION
  |
  +--> CONFIRMED
  |
  +--> CONFLICT
  |
  +--> RETRY
```

## 14. Principe de non-perte

Le système devra éviter qu'une panne réseau, une retransmission ou une panne applicative puisse supprimer silencieusement un acte.

La combinaison suivante devra être mise en œuvre :

**transaction locale + Outbox + identifiant unique + idempotence + ACK + retry + audit + sauvegarde.**
