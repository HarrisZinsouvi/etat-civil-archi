# Architecture Decision Records

## ADR-001 — Mode Offline-first

### Décision
Les communes doivent pouvoir enregistrer des actes sans connectivité permanente.

### Raisons
Les coupures peuvent durer plusieurs heures.

### Conséquences
- Base locale
- Outbox
- Synchronisation différée
- Cohérence éventuelle pendant le mode offline

---

## ADR-002 — Idempotence de la synchronisation

### Décision
Chaque événement possède un identifiant unique.

### Raisons
Une opération peut être retransmise plusieurs fois après une coupure réseau.

### Conséquences
Le serveur doit accepter les retransmissions sans créer de doublons.

---

## ADR-003 — Gestion explicite des conflits

### Décision
Ne pas utiliser Last Write Wins.

### Raisons
Les actes d'état civil sont juridiquement sensibles.

### Conséquences
Versionnement optimiste + résolution manuelle.

---

## ADR-004 — PostgreSQL pour les données transactionnelles

### Décision
Utiliser une base relationnelle ACID pour les actes.

### Raisons
Les données nécessitent intégrité référentielle, transactions et cohérence forte pour les opérations critiques.

---

## ADR-005 — Event-driven pour les traitements secondaires

### Décision
Utiliser un broker pour découpler les traitements non critiques.

### Raisons
Absorber les pics et protéger le parcours transactionnel principal.

---

## ADR-006 — Site secondaire pour le PRA

### Décision
Mettre en place une réplication vers un site secondaire.

### Raisons
Réduire l'impact d'un sinistre majeur du site primaire.

### Objectifs
RPO ≤ 5 min
RTO ≤ 30 min