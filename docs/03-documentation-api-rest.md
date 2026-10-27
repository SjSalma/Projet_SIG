# Documentation API REST - SIG

[← Accueil du projet](../README.md) · [Documentation](../README.md#documentation) · [Conception technique](03-conception-technique.md)

> **Statut : spécification de conception (proposition V1).**  
> Les routes, schémas JSON, paramètres et codes de permission ci-dessous définissent un **contrat cible à valider** au moment de l'implémentation NestJS. Leur existence en production n'est **pas** confirmée. Ce document ne constitue pas une documentation Swagger générée à partir du code.
>
> **Référence :** `03-conception-technique.md` (architecture modulaire, Core SIG, modèle relationnel, sécurité et huit modules).

## Sommaire

1. [Objectif et portée](#1-objectif-et-portée)
2. [Conventions de l'API](#2-conventions-de-lapi)
3. [Authentification, organisations et permissions](#3-authentification-organisations-et-permissions)
4. [Core SIG - Endpoints proposés](#4-core-sig---endpoints-proposés)
5. [Modules métier](#5-modules-métier)
   1. [Module 1 - Événements et bénévoles](#module-1---événements-et-bénévoles)
   2. [Module 2 - Stocks et fournisseurs](#module-2---stocks-et-fournisseurs)
   3. [Module 3 - Ressources humaines](#module-3---ressources-humaines)
   4. [Module 4 - Demandes internes](#module-4---demandes-internes)
   5. [Module 5 - Inventaires étendus](#module-5---inventaires-étendus)
   6. [Module 6 - Horaires et réservations](#module-6---horaires-et-réservations)
   7. [Module 7 - Clients et comptes externes](#module-7---clients-et-comptes-externes)
   8. [Module 8 - Gestion de projet](#module-8---gestion-de-projet)
6. [Services transversaux](#6-services-transversaux)
7. [Exemples de requêtes et réponses](#7-exemples-de-requêtes-et-réponses)
8. [Erreurs et règles métier](#8-erreurs-et-règles-métier)
9. [Swagger / OpenAPI, tests et suivi](#9-swagger--openapi-tests-et-suivi)
10. [Décisions à valider avant implémentation](#10-décisions-à-valider-avant-implémentation)
11. [Conclusion](#11-conclusion)

---


&nbsp;
# 1. Objectif et portée

Le SIG est une **application web multi-organisation** construite autour d'un **frontend React + TypeScript**, d'un **backend NestJS (Node.js/TypeScript)** et d'une base **PostgreSQL** avec **Prisma** (choix techniques prévus). L'API REST sert de point d'entrée au frontend et pourra, ultérieurement, être exposée de manière contrôlée à des applications autorisées.

Le Core SIG centralise l'authentification, les utilisateurs, les organisations, les rôles, les permissions, les modules activables et l'audit. Huit modules métier sont prévus en V1 : événements/bénévoles, stocks/fournisseurs, RH, tickets, inventaires, horaires/réservations, comptes externes et projets.

**Portée de ce document :** noms des ressources, opérations proposées, conventions de données, autorisations et exemples représentatifs. Les schémas DTO complets, listes d'énumérations définitives et validations de chaque champ seront alignés sur les modèles Prisma et les contrôleurs NestJS au fil du développement.


&nbsp;
# 2. Conventions de l'API

## URL et versionnement

- **Préfixe cible :** `/api/v1`.
- **URL locale illustrative :** `http://localhost:3000/api/v1` (port et hôte à confirmer dans la configuration réelle).
- **Ressources métier :** `/api/v1/organizations/{organizationId}/...` pour rendre explicite le contexte organisationnel.
- **Ressources d'authentification :** `/api/v1/auth/...`.
- **Chemins :** noms pluriels en anglais, minuscules et `kebab-case`; identifiants stables (UUID proposés).
- **Dates :** ISO 8601 avec fuseau explicite, stockées en UTC; afficher les heures dans le fuseau de l'organisation ou de l'utilisateur côté interface.
- **JSON :** propriétés en `camelCase`; types et valeurs d'énumération à figer dans les DTO.
- **Suppression :** `DELETE` uniquement lorsqu'elle est autorisée; pour les comptes et les données nécessitant un historique, privilégier l'archivage ou le changement de statut.

> Le segment `{organizationId}` constitue un **choix de contrat proposé**, pas une preuve que ce routage existe. Même lorsque l'ID figure dans l'URL, le backend doit déterminer et valider l'organisation autorisée à partir de la session.

## Méthodes HTTP

| Méthode | Sens | Réponse typique |
|---|---|---|
| `GET` | Lire une ressource ou une collection | `200 OK` |
| `POST` | Créer une ressource ou effectuer une action explicite | `201 Created` ou `200 OK` |
| `PATCH` | Modifier partiellement une ressource | `200 OK` |
| `PUT` | Remplacer intégralement une ressource (si nécessaire) | `200 OK` |
| `DELETE` | Supprimer une ressource autorisée | `204 No Content` |

Une création renvoie idéalement l'en-tête `Location`. Un endpoint d'action (`approve`, `close`, `move`, etc.) est utilisé seulement lorsqu'une transition métier ne se réduit pas à une simple modification de champ.

## Formats de réponses

**Ressource unique :**
```json
{
  "data": {
    "id": "9fe8b3d9-1642-4c59-9f97-c3a1224e26a0",
    "name": "Exemple"
  }
}
```

**Collection paginée :**
```json
{
  "data": [],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 0,
    "totalPages": 0
  }
}
```

**Erreur :**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Données invalides.",
    "details": [{"field": "title", "message": "Ce champ est obligatoire."}],
    "requestId": "req-example-001"
  }
}
```

Ces enveloppes sont **proposées**. La documentation Swagger et les DTO devront respecter la même convention.

## Pagination, tri, filtres et recherche

| Paramètre | Type | Exemple | Règle proposée |
|---|---|---|---|
| `page` | entier | `1` | Débute à 1 |
| `pageSize` | entier | `20` | Défaut 20, maximum 100 |
| `sort` | texte | `createdAt:desc` | Seulement les champs autorisés |
| `search` | texte | `inventaire` | Recherche limitée aux champs métier pertinents |
| `status` | texte | `open` | Valeurs documentées par ressource |
| `from`, `to` | ISO 8601 | `2026-11-01T00:00:00Z` | Intervalles temporels |

Exemple : `GET /api/v1/organizations/{organizationId}/tickets?page=1&pageSize=20&status=open&sort=createdAt:desc`.

Les collections sont limitées **à l'organisation active**. Les critères de filtrage et les champs triables doivent être explicitement autorisés pour éviter des requêtes arbitraires.

## En-têtes, identifiants et idempotence

- `Content-Type: application/json` et `Accept: application/json` pour les échanges JSON.
- `multipart/form-data` pour les téléversements de fichiers.
- `X-Request-Id` éventuellement fourni par le client; sinon généré côté serveur.
- `Idempotency-Key` **envisagé** pour les opérations sensibles pouvant être répétées (billets, mouvements de stock, réservations). À définir avant implémentation.
- Les secrets, mots de passe, jetons et données sensibles ne figurent jamais dans les URL ni dans les journaux applicatifs.


&nbsp;
# 3. Authentification, organisations et permissions

## Sessions web (orientation proposée)

Le projet de conception prévoit la possibilité de **jetons d'accès courts et de jetons de renouvellement révocables**, transmis dans des **cookies sécurisés** pour le frontend. Le mécanisme exact sera arrêté lors de l'implémentation.

Principes retenus pour le contrat web :
1. Connexion avec identifiants; validation du statut du compte et de l'organisation.
2. Création d'une session avec cookies `HttpOnly`, `Secure` en HTTPS et politique `SameSite` adaptée au déploiement.
3. Renouvellement/révocation contrôlés côté backend.
4. Protection **CSRF** sur les requêtes modifiant des données lorsque l'authentification repose sur les cookies (jeton CSRF, vérification d'origine ou stratégie équivalente selon l'architecture).
5. En cas de frontend/backend sur des origines différentes, configurer CORS de façon restrictive avec les identifiants autorisés; ne jamais utiliser `*` avec les cookies.
6. Les appels depuis React devront inclure les cookies si l'architecture retenue l'exige (`credentials: "include"`).

Un mode `Authorization: Bearer ...` pourra être défini séparément pour d'éventuelles intégrations externes, mais **il n'est pas présenté ici comme un mécanisme actuellement disponible**.

## Règles d'autorisation

Pour chaque opération protégée, le backend doit vérifier simultanément :
1. Le compte est authentifié et actif.
2. L'utilisateur appartient à l'organisation concernée et possède le droit d'y agir.
3. Le module métier sollicité est activé pour cette organisation.
4. L'utilisateur possède la permission requise, éventuellement complétée par une règle de propriété ou de confidentialité.

Les contrôles sont effectués **côté backend**, y compris lorsqu'un bouton est masqué dans React. Les modifications de rôle, permission et activation d'un module sont auditées.

Les codes de permissions ci-dessous (par exemple `events.read` ou `tickets.manage`) sont une **proposition de nomenclature** à harmoniser avec les enregistrements `Permission` de la base.

**Comportement proposé :** `401` si la session est absente/invalide, `403` si la permission manque ou le module est désactivé, `404` pour les ressources absentes ou non visibles dans l'organisation courante (afin de ne pas divulguer leur existence).

## Module désactivé

Quand une organisation désactive un module, ses endpoints métier renvoient une erreur `403 MODULE_DISABLED`, les données restent conservées pour une réactivation éventuelle et les fonctionnalités indépendantes des autres modules restent disponibles. Les opérations transversales qui requièrent **deux modules** doivent vérifier l'activation des deux.


&nbsp;
# 4. Core SIG - endpoints proposés

Dans les tableaux, `/{org}` représente `/api/v1/organizations/{organizationId}`. Les chemins commençant par `/{org}/` utilisent ce même préfixe organisationnel. Les autres chemins indiquent explicitement `/api/v1/...`. Les noms des permissions sont **indicatifs**.

## Authentification


| Méthode | Endpoint | Description | Accès |
|---|---|---|---|
| `POST` | `/api/v1/organizations/{organizationId}/auth/client-registration` | Inscrire un client dans une organisation | Public, si inscription autorisée et module 7 actif |
| `POST` | `/api/v1/auth/email/confirm` | Confirmer le courriel et activer un compte | Jeton de confirmation valide |
| `POST` | `/api/v1/auth/email/resend` | Demander un nouveau courriel de confirmation | Public, réponse neutre et limitation des tentatives |
| `POST` | `/api/v1/auth/login` | Ouvrir une session | Public, avec limitation des tentatives |
| `POST` | `/api/v1/auth/refresh` | Renouveler une session valide | Cookie/session de renouvellement |
| `POST` | `/api/v1/auth/logout` | Révoquer la session | Connecté |
| `GET` | `/api/v1/auth/me` | Obtenir l'identité et le contexte de session | Connecté |
| `POST` | `/api/v1/auth/password-reset/request` | Demander une réinitialisation | Public, réponse neutre |
| `POST` | `/api/v1/auth/password-reset/confirm` | Finaliser la réinitialisation | Jeton de vérification à usage limité |

**Règles proposées pour l'inscription client :**
- L'inscription est toujours associée à une organisation identifiée et validée côté serveur.
- L'organisation doit autoriser l'inscription publique et disposer du module 7 actif.
- Le compte est créé dans le Core SIG avec le type `client` et le statut `pending_activation`.
- Le profil métier correspondant est créé dans le module 7.
- Le courriel doit être unique au sein de l'organisation.
- Le compte ne peut pas se connecter avant confirmation de son courriel.
- Les jetons de confirmation possèdent une durée de validité limitée et ne sont utilisables qu'une fois.
- Les comptes bénévoles et fournisseurs ne peuvent pas obtenir leurs permissions par simple inscription publique.
- L'organisation concernée doit également être identifiée lors de la connexion pour distinguer les comptes utilisant le même courriel dans différentes organisations.
- Ces endpoints sont des propositions de conception, non des fonctionnalités déjà implémentées.

## Organisations, utilisateurs, rôles et permissions

| Méthode | Endpoint | Description | Permission indicative |
|---|---|---|---|
| `GET` | `/api/v1/organizations` | Lister les organisations accessibles par l'utilisateur | `organizations.read` |
| `GET` | `/{org}` | Voir les informations de l'organisation | `organizations.read` |
| `PATCH` | `/{org}` | Modifier la configuration autorisée | `organizations.manage` |
| `GET` | `/{org}/users` | Lister les utilisateurs | `users.read` |
| `POST` | `/{org}/users` | Créer/inviter un utilisateur | `users.manage` |
| `GET` | `/{org}/users/{userId}` | Consulter un compte | `users.read` |
| `PATCH` | `/{org}/users/{userId}` | Modifier un compte | `users.manage` |
| `PATCH` | `/{org}/users/{userId}/status` | Activer, désactiver ou archiver | `users.manage` |
| `GET` | `/{org}/roles` | Lister les rôles | `roles.read` |
| `POST` | `/{org}/roles` | Créer un rôle personnalisé | `roles.manage` |
| `PATCH` | `/{org}/roles/{roleId}` | Modifier un rôle | `roles.manage` |
| `DELETE` | `/{org}/roles/{roleId}` | Supprimer un rôle si aucune règle ne l'interdit | `roles.manage` |
| `PUT` | `/{org}/roles/{roleId}/permissions` | Remplacer les permissions d'un rôle | `roles.manage` |
| `PUT` | `/{org}/users/{userId}/roles` | Remplacer les rôles d'un utilisateur | `users.manage` |
| `GET` | `/{org}/permissions` | Lister le catalogue de permissions | `roles.read` |
| `GET` | `/{org}/modules` | Lister les modules disponibles et leur état | `modules.read` |
| `PATCH` | `/{org}/modules/{moduleCode}` | Activer/désactiver un module | `modules.manage` |
| `GET` | `/{org}/audit-logs` | Consulter le journal d'audit filtré | `audit.read` |

**Règles Core SIG :** un utilisateur d'une organisation ne peut pas modifier les rôles d'une autre; une permission n'ouvre pas l'accès à un module désactivé; la désactivation d'un utilisateur révoque ses sessions; les rôles système protégés ne sont pas supprimables; les journaux d'audit sont en lecture seule et filtrés par organisation.


&nbsp;
# 5. Modules métier

## Module 1 - Événements et bénévoles

**Préfixe** : `/api/v1/organizations/{organizationId}/`.

| Méthode | Chemin relatif au préfixe organisation | Usage | Permission indicative |
|---|---|---|---|
| `GET` | `/events` | Lister les événements | `events.read` |
| `POST` | `/events` | Créer un événement | `events.manage` |
| `GET` | `/events/{eventId}` | Consulter un événement | `events.read` |
| `PATCH` | `/events/{eventId}` | Modifier un événement | `events.manage` |
| `PATCH` | `/events/{eventId}/status` | Changer son état | `events.manage` |
| `GET` | `/events/{eventId}/positions` | Lister les postes de bénévolat | `events.read` |
| `POST` | `/events/{eventId}/positions` | Créer un poste | `events.manage` |
| `PATCH` | `/events/{eventId}/positions/{positionId}` | Modifier un poste | `events.manage` |
| `GET` | `/events/{eventId}/assignments` | Lister les affectations | `volunteers.read` |
| `POST` | `/events/{eventId}/assignments` | Affecter un bénévole | `volunteers.manage` |
| `PATCH` | `/events/{eventId}/assignments/{assignmentId}` | Modifier statut/affectation | `volunteers.manage` |
| `GET` | `/events/{eventId}/ticket-types` | Lister les types de billets | `events.read` |
| `POST` | `/events/{eventId}/ticket-types` | Créer un type de billet | `events.manage` |
| `POST` | `/events/{eventId}/tickets` | Émettre/réserver un billet sous réserve de capacité | `events.manage` |
| `GET` | `/events/{eventId}/tickets` | Lister les billets autorisés | `events.read` |

**Règles métier principales :** Un bénévole doit appartenir à l'organisation (compte ou profil autorisé). Les affectations respectent les besoins des postes. Les limites de places et la billetterie sont vérifiées en transaction. Les interactions avec les horaires, actifs et comptes externes ne sont actives que si les modules correspondants sont disponibles.

## Module 2 - Stocks et fournisseurs

**Préfixe** : `/api/v1/organizations/{organizationId}/`.

| Méthode | Chemin relatif au préfixe organisation | Usage | Permission indicative |
|---|---|---|---|
| `GET` | `/stock-items` | Lister les articles | `stock.read` |
| `POST` | `/stock-items` | Créer un article | `stock.manage` |
| `GET` | `/stock-items/{itemId}` | Consulter un article | `stock.read` |
| `PATCH` | `/stock-items/{itemId}` | Modifier description, seuil et localisation | `stock.manage` |
| `GET` | `/stock-items/{itemId}/movements` | Voir l'historique des mouvements | `stock.read` |
| `POST` | `/stock-items/{itemId}/movements` | Enregistrer entrée, sortie ou ajustement | `stock.manage` |
| `GET` | `/stock-alerts` | Lister les alertes de seuil bas | `stock.read` |
| `GET` | `/stock-items/{itemId}/suppliers` | Lister les fournisseurs associés | `stock.read` |
| `POST` | `/stock-items/{itemId}/suppliers` | Associer un fournisseur externe | `stock.manage` |
| `DELETE` | `/stock-items/{itemId}/suppliers/{supplierId}` | Dissocier un fournisseur | `stock.manage` |

**Règles métier principales :** Les stocks ne doivent pas devenir négatifs sauf règle métier explicite. Chaque mouvement est historisé et appliqué **atomiquement** à la quantité disponible. Les fournisseurs renvoient vers des profils du module 7 si celui-ci est actif; aucun compte externe ne doit être dupliqué.

## Module 3 - Ressources humaines

**Préfixe** : `/api/v1/organizations/{organizationId}/`.

| Méthode | Chemin relatif au préfixe organisation | Usage | Permission indicative |
|---|---|---|---|
| `GET` | `/employees` | Lister les dossiers employés | `hr.read` |
| `POST` | `/employees` | Créer un dossier RH lié à un utilisateur | `hr.manage` |
| `GET` | `/employees/{employeeId}` | Consulter un dossier RH | `hr.read` |
| `PATCH` | `/employees/{employeeId}` | Mettre à jour un dossier RH | `hr.manage` |
| `GET` | `/employees/{employeeId}/contracts` | Consulter les contrats | `hr.read` |
| `POST` | `/employees/{employeeId}/contracts` | Ajouter un contrat et sa référence de fichier | `hr.manage` |
| `GET` | `/leave-requests` | Lister les demandes de congé autorisées | `leave.read` |
| `POST` | `/leave-requests` | Créer une demande de congé | `leave.create` |
| `GET` | `/leave-requests/{requestId}` | Consulter une demande | `leave.read` |
| `POST` | `/leave-requests/{requestId}/approve` | Approuver une demande | `leave.approve` |
| `POST` | `/leave-requests/{requestId}/reject` | Refuser une demande | `leave.approve` |
| `GET` | `/employees/{employeeId}/history` | Lire l'historique RH | `hr.audit` |

**Règles métier principales :** Les dossiers RH sont sensibles : le simple droit de consulter les utilisateurs ne donne pas accès aux contrats ou congés. Un responsable ne peut pas approuver une demande sans permission explicite; les transitions de statut sont contrôlées et historisées.

## Module 4 - Demandes internes

**Préfixe** : `/api/v1/organizations/{organizationId}/`.

| Méthode | Chemin relatif au préfixe organisation | Usage | Permission indicative |
|---|---|---|---|
| `GET` | `/tickets` | Lister les tickets visibles | `tickets.read` |
| `POST` | `/tickets` | Créer un ticket | `tickets.create` |
| `GET` | `/tickets/{ticketId}` | Consulter un ticket | `tickets.read` |
| `PATCH` | `/tickets/{ticketId}` | Modifier catégorie/priorité/champs autorisés | `tickets.manage` |
| `POST` | `/tickets/{ticketId}/assign` | Assigner un ticket | `tickets.manage` |
| `POST` | `/tickets/{ticketId}/transition` | Changer l'état via transition validée | `tickets.manage` |
| `GET` | `/tickets/{ticketId}/comments` | Consulter les commentaires | `tickets.read` |
| `POST` | `/tickets/{ticketId}/comments` | Ajouter un commentaire | `tickets.comment` |
| `GET` | `/tickets/{ticketId}/history` | Consulter l'historique du ticket | `tickets.read` |

**Règles métier principales :** Cycle cible : **Ouvert → Assigné → En cours → En attente → Résolu → Fermé**, avec transitions de retour permises uniquement si elles sont définies dans les spécifications. Le créateur et les utilisateurs autorisés voient le ticket; les tickets liés à la RH exigent une permission supplémentaire.

## Module 5 - Inventaires étendus

**Préfixe** : `/api/v1/organizations/{organizationId}/`.

| Méthode | Chemin relatif au préfixe organisation | Usage | Permission indicative |
|---|---|---|---|
| `GET` | `/assets` | Lister les actifs | `assets.read` |
| `POST` | `/assets` | Créer un actif | `assets.manage` |
| `GET` | `/assets/{assetId}` | Consulter un actif | `assets.read` |
| `PATCH` | `/assets/{assetId}` | Modifier localisation/état/attribution | `assets.manage` |
| `GET` | `/asset-categories` | Lister les catégories | `assets.read` |
| `POST` | `/asset-categories` | Créer une catégorie | `assets.manage` |
| `GET` | `/asset-locations` | Lister les emplacements | `assets.read` |
| `POST` | `/asset-locations` | Créer un emplacement | `assets.manage` |
| `GET` | `/assets/{assetId}/history` | Consulter l'historique des changements | `assets.read` |
| `POST` | `/assets/{assetId}/assign` | Attribuer un actif à un utilisateur | `assets.manage` |
| `POST` | `/assets/{assetId}/unassign` | Restituer un actif | `assets.manage` |

**Règles métier principales :** Une salle réservable est un actif de type compatible. Les références d'emplacement, de catégorie et d'utilisateur doivent appartenir à la même organisation. Les changements de propriétaire, d'état et de localisation sont audités.

## Module 6 - Horaires et réservations

**Préfixe** : `/api/v1/organizations/{organizationId}/`.

| Méthode | Chemin relatif au préfixe organisation | Usage | Permission indicative |
|---|---|---|---|
| `GET` | `/calendar-entries` | Lister les entrées de calendrier | `schedules.read` |
| `POST` | `/calendar-entries` | Créer un créneau | `schedules.manage` |
| `PATCH` | `/calendar-entries/{entryId}` | Modifier un créneau | `schedules.manage` |
| `GET` | `/appointments` | Lister les rendez-vous autorisés | `appointments.read` |
| `POST` | `/appointments` | Créer un rendez-vous | `appointments.manage` |
| `PATCH` | `/appointments/{appointmentId}` | Modifier un rendez-vous | `appointments.manage` |
| `GET` | `/appointments/{appointmentId}/participants` | Lister les participants | `appointments.read` |
| `POST` | `/appointments/{appointmentId}/participants` | Inviter un participant | `appointments.manage` |
| `GET` | `/room-reservations` | Lister les réservations | `reservations.read` |
| `POST` | `/room-reservations` | Réserver une salle | `reservations.create` |
| `GET` | `/room-reservations/{reservationId}` | Consulter une réservation | `reservations.read` |
| `PATCH` | `/room-reservations/{reservationId}` | Modifier créneau/statut | `reservations.manage` |
| `POST` | `/room-reservations/{reservationId}/cancel` | Annuler une réservation | `reservations.manage` |

**Règles métier principales :** Pour une réservation, `startAt < endAt` et l'actif doit correspondre à une salle disponible dans la même organisation. Vérifier les chevauchements au niveau serveur et préserver la cohérence sous concurrence; en cas de conflit, répondre `409 RESERVATION_CONFLICT`. Une dépendance au module Inventaires suppose qu'il soit actif.


## Module 7 - Clients et comptes externes

**Préfixe** : `/api/v1/organizations/{organizationId}/`.

| Méthode | Chemin relatif au préfixe organisation | Usage | Permission indicative |
|---|---|---|---|
| `GET` | `/external-profiles` | Lister les profils externes (filtre `type`) | `external.read` |
| `POST` | `/external-profiles` | Créer un profil externe | `external.manage` |
| `GET` | `/external-profiles/{profileId}` | Consulter un profil externe | `external.read` |
| `PATCH` | `/external-profiles/{profileId}` | Modifier un profil externe | `external.manage` |
| `GET` | `/client-tags` | Lister les étiquettes | `external.read` |
| `POST` | `/client-tags` | Créer une étiquette | `external.manage` |
| `POST` | `/external-profiles/{profileId}/tags` | Associer une étiquette | `external.manage` |
| `DELETE` | `/external-profiles/{profileId}/tags/{tagId}` | Retirer une étiquette | `external.manage` |
| `GET` | `/external-profiles/{profileId}/interactions` | Consulter les interactions | `external.read` |
| `POST` | `/external-profiles/{profileId}/interactions` | Consigner une interaction | `external.manage` |

**Règles métier principales :** Les catégories envisagées sont `client`, `volunteer` et `supplier` (valeurs à valider). Le profil externe complète un compte `User` du Core SIG lorsqu'il existe; ses droits de connexion restent administrés exclusivement par le Core SIG. La segmentation et l'historique ne doivent pas exposer les données d'une autre organisation.

## Module 8 - Gestion de projet

**Préfixe** : `/api/v1/organizations/{organizationId}/`.

| Méthode | Chemin relatif au préfixe organisation | Usage | Permission indicative |
|---|---|---|---|
| `GET` | `/project-boards` | Lister les tableaux autorisés | `projects.read` |
| `POST` | `/project-boards` | Créer un tableau | `projects.manage` |
| `GET` | `/project-boards/{boardId}` | Consulter un tableau et sa structure | `projects.read` |
| `PATCH` | `/project-boards/{boardId}` | Modifier titre/visibilité | `projects.manage` |
| `GET` | `/project-boards/{boardId}/columns` | Lister les colonnes | `projects.read` |
| `POST` | `/project-boards/{boardId}/columns` | Créer une colonne | `projects.manage` |
| `PATCH` | `/project-boards/{boardId}/columns/{columnId}` | Renommer/réordonner une colonne | `projects.manage` |
| `POST` | `/project-boards/{boardId}/columns/{columnId}/cards` | Créer une carte | `projects.manage` |
| `GET` | `/project-cards/{cardId}` | Consulter une carte | `projects.read` |
| `PATCH` | `/project-cards/{cardId}` | Modifier titre, description, échéance | `projects.manage` |
| `POST` | `/project-cards/{cardId}/move` | Déplacer une carte vers une colonne/position | `projects.manage` |
| `PUT` | `/project-cards/{cardId}/assignees` | Remplacer les personnes assignées | `projects.manage` |
| `GET` | `/project-cards/{cardId}/comments` | Lire les commentaires | `projects.read` |
| `POST` | `/project-cards/{cardId}/comments` | Ajouter un commentaire | `projects.comment` |

**Règles métier principales :** Une carte appartient à une colonne et à un tableau de la même organisation. Le déplacement vers une colonne d'un autre tableau doit être explicitement pris en charge ou refusé. Les positions des cartes doivent rester cohérentes en cas d'actions concurrentes.


&nbsp;
# 6. Services transversaux

**Préfixe** : `/api/v1/organizations/{organizationId}`.

| Méthode | Chemin | Description | Permission indicative |
|---|---|---|---|
| `GET` | `/notifications` | Lister les notifications de l'utilisateur | Authentifié |
| `PATCH` | `/notifications/{notificationId}/read` | Marquer une notification comme lue | Propriétaire |
| `POST` | `/attachments` | Téléverser un fichier autorisé | Selon ressource propriétaire |
| `GET` | `/attachments/{attachmentId}` | Lire les métadonnées | Selon ressource propriétaire |
| `GET` | `/attachments/{attachmentId}/content` | Télécharger le contenu après contrôle d'accès | Selon ressource propriétaire |
| `DELETE` | `/attachments/{attachmentId}` | Retirer une pièce jointe autorisée | Selon ressource propriétaire |

Les fichiers sont stockés hors base (stockage privé); PostgreSQL conserve les métadonnées et références. Contrôler extension/type MIME, taille, ressource propriétaire et droits d'accès à **chaque** lecture. Un identifiant de pièce jointe ne doit pas permettre un téléchargement direct sans autorisation.


&nbsp;
# 7. Exemples de requêtes et réponses

> **Important :** tous les identifiants, URL, propriétés et exemples de réponses ci-dessous sont **fictifs**. Ils illustrent le contrat API proposé et ne correspondent pas à des ressources réellement disponibles.

## Connexion

`POST /api/v1/auth/login`

```http
POST /api/v1/auth/login HTTP/1.1
Content-Type: application/json

{
  "organizationId": "org-exemple",
  "email": "admin@example.org",
  "password": "<mot-de-passe>"
}
```

**Succès proposé - `200 OK` :** le serveur définit les cookies d'authentification de manière sécurisée, sans placer de secrets dans le corps de la réponse.

```json
{
  "data": {
    "user": {
      "id": "usr-exemple",
      "email": "admin@example.org",
      "status": "active"
    },
    "organization": {
      "id": "org-exemple",
      "name": "Organisation exemple"
    }
  }
}
```

Le champ `organizationId` de la requête est une hypothèse de conception à valider avec le parcours de sélection d'organisation.

## Consulter les utilisateurs (pagination)

`GET /api/v1/organizations/org-exemple/users?page=1&pageSize=20`

**Réponse proposée - `200 OK` :**

```json
{
  "data": [
    {
      "id": "usr-001",
      "email": "membre@example.org",
      "firstName": "Samira",
      "lastName": "Exemple",
      "status": "active",
      "roleIds": ["role-staff"]
    }
  ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 1,
    "totalPages": 1
  }
}
```

## Activer un module

`PATCH /api/v1/organizations/org-exemple/modules/tickets`

```json
{
  "enabled": true
}
```

**Réponse proposée - `200 OK` :**

```json
{
  "data": {
    "code": "tickets",
    "enabled": true
  }
}
```

L'action requiert `modules.manage`, respecte les éventuelles règles d'abonnement et produit une entrée d'audit.

## Créer un ticket interne

`POST /api/v1/organizations/org-exemple/tickets`

```json
{
  "category": "it",
  "priority": "normal",
  "description": "L'imprimante du bureau ne fonctionne plus."
}
```

**Réponse proposée - `201 Created` :**

```json
{
  "data": {
    "id": "ticket-001",
    "category": "it",
    "priority": "normal",
    "description": "L'imprimante du bureau ne fonctionne plus.",
    "status": "open",
    "createdBy": "usr-001",
    "assignedTo": null,
    "createdAt": "2026-10-25T18:00:00Z"
  }
}
```

Le serveur établit `createdBy` et `organizationId` à partir du contexte autorisé, jamais à partir d'un champ que le client pourrait falsifier.

## Enregistrer un mouvement de stock

`POST /api/v1/organizations/org-exemple/stock-items/item-001/movements`

```json
{
  "movementType": "out",
  "quantity": 3,
  "reason": "Matériel utilisé pour un événement"
}
```

**Réponse proposée - `201 Created` :**

```json
{
  "data": {
    "id": "movement-001",
    "itemId": "item-001",
    "movementType": "out",
    "quantity": 3,
    "quantityAfter": 17
  }
}
```

`quantityAfter` est calculé par le serveur au cours d'une transaction; le client ne fournit pas la quantité finale.

## Réserver une salle

`POST /api/v1/organizations/org-exemple/room-reservations`

```json
{
  "assetId": "salle-001",
  "startAt": "2026-11-10T14:00:00Z",
  "endAt": "2026-11-10T15:00:00Z"
}
```

**Réponse proposée - `201 Created` :**

```json
{
  "data": {
    "id": "reservation-001",
    "assetId": "salle-001",
    "status": "confirmed",
    "startAt": "2026-11-10T14:00:00Z",
    "endAt": "2026-11-10T15:00:00Z"
  }
}
```

**Exemple de conflit - `409 Conflict` :**

```json
{
  "error": {
    "code": "RESERVATION_CONFLICT",
    "message": "Cette salle est déjà réservée sur ce créneau.",
    "requestId": "req-example-002"
  }
}
```

## Déplacer une carte Kanban

`POST /api/v1/organizations/org-exemple/project-cards/card-001/move`

```json
{
  "targetColumnId": "column-in-progress",
  "position": 2
}
```

**Réponse proposée - `200 OK` :**

```json
{
  "data": {
    "id": "card-001",
    "columnId": "column-in-progress",
    "position": 2
  }
}
```

## Appel depuis React (illustration)

L'exemple suppose une authentification par cookies. Le mécanisme de protection CSRF **reste à implémenter et à documenter** avant d'utiliser des requêtes mutantes.

```ts
const response = await fetch(
  "/api/v1/organizations/org-exemple/tickets?page=1&pageSize=20",
  {
    method: "GET",
    credentials: "include",
    headers: { Accept: "application/json" }
  }
);

if (!response.ok) {
  throw new Error(`Erreur API : ${response.status}`);
}

const result = await response.json();
console.log(result.data);
```


&nbsp;
# 8. Erreurs et règles métier

## Codes HTTP

| HTTP | Code applicatif exemple | Situation |
|---|---|---|
| `400 Bad Request` | `VALIDATION_ERROR` | Type, champ ou format invalide |
| `401 Unauthorized` | `UNAUTHENTICATED` | Session absente ou expirée |
| `403 Forbidden` | `FORBIDDEN` | Permission insuffisante |
| `403 Forbidden` | `MODULE_DISABLED` | Module non activé dans l'organisation |
| `404 Not Found` | `RESOURCE_NOT_FOUND` | Ressource inexistante ou non visible |
| `409 Conflict` | `RESERVATION_CONFLICT` | Créneau indisponible |
| `409 Conflict` | `STOCK_CONFLICT` | Quantité insuffisante ou mouvement concurrent |
| `409 Conflict` | `DUPLICATE_RESOURCE` | Unicité métier violée |
| `422 Unprocessable Content` | `BUSINESS_RULE_VIOLATION` | Requête valide techniquement, mais refusée par une règle métier |
| `429 Too Many Requests` | `RATE_LIMITED` | Trop de requêtes ou tentatives |
| `500 Internal Server Error` | `INTERNAL_ERROR` | Erreur imprévue, détails internes masqués |

Les libellés d'erreur sont stables pour les clients. Les messages affichés peuvent être localisés côté frontend. Ne pas exposer SQL, stack traces, hachages, secrets ou détails d'autres organisations.

## Invariants de sécurité et de cohérence

- **Multi-organisation :** chaque `GET`, `POST`, `PATCH`, `DELETE` filtre et valide l'organisation issue du contexte authentifié. Les IDs liés dans une requête doivent appartenir à cette organisation.
- **RBAC :** rôle, permission et module actif sont vérifiés côté serveur pour chaque endpoint; le frontend seul ne sécurise rien.
- **Transactions :** mouvements de stock, attribution de billets, réservations, changements de permissions et réordonnancement Kanban doivent être atomiques lorsque plusieurs écritures sont impliquées.
- **Journalisation :** les opérations sensibles, refus importants et changements de configuration créent des traces sans données secrètes.
- **Confidentialité RH :** contrôles spécialisés supplémentaires; minimiser les champs renvoyés aux acteurs qui n'en ont pas besoin.
- **Validation :** DTO stricts, rejet des propriétés non prévues lorsque pertinent, contrôle des identifiants, dates, statuts et associations.
- **Archivage :** utilisateurs et données avec historique ne sont pas supprimés sans stratégie de conservation.
- **Téléversements :** taille, type, propriété, contrôle d'accès au téléchargement et sécurité du stockage.
- **Concurrence :** traitements atomiques et réponses `409` explicites sur les conflits avérés.


&nbsp;
# 9. Swagger / OpenAPI, tests et suivi

## Source de vérité au développement

À mesure que les contrôleurs et DTO NestJS seront implémentés, documenter chaque route dans un schéma **OpenAPI**, par exemple avec `@nestjs/swagger`. La documentation générée devra refléter les routes réellement disponibles et les exceptions réelles. Le chemin de l'interface Swagger (par exemple `/api/docs`) restera configurable et n'est **pas encore confirmé**.

Pour chaque endpoint à implémenter, préciser dans le code : opération, description, schémas d'entrée et de sortie, autorisation, exemples pertinents et statuts HTTP. Exporter un `openapi.json` versionné et, si utile, importer ce schéma dans Postman.

## Tests minimaux par endpoint

- Cas nominal, paramètres invalides, ressource absente et absence de session.
- Permission manquante, module désactivé, compte désactivé.
- Tentative d'accès inter-organisation (ID d'une autre organisation).
- Réponse conforme au JSON attendu (contrat OpenAPI).
- Conflits métier et courses critiques (billets, stocks, salles, ordre des cartes).
- Audit des changements sensibles et absence de fuite de données.
- Navigation avec cookies/CSRF selon le mécanisme réellement implémenté.

## Tableau de suivi documentaire

| Élément | État documentaire | État d'implémentation |
|---|---|---|
| Conventions, formats et erreurs | Proposition rédigée | À vérifier |
| Authentification et sessions | Proposition rédigée | À vérifier |
| Core SIG | Contrats proposés | À vérifier |
| Modules 1 à 8 | Contrats proposés | À vérifier |
| Services transversaux | Contrats proposés | À vérifier |
| DTO et schémas complets | À détailler | À vérifier |
| Contrat OpenAPI généré | Non fourni | À vérifier |
| Collection Postman | Non fournie | À vérifier |
| Tests d'intégration et sécurité | Critères définis | À vérifier |


&nbsp;
# 10. Décisions à valider avant implémentation

1. Stratégie définitive de session web : cookies, durée de vie, refresh, CSRF, domaines et CORS.
2. Gestion de l'organisation à la connexion et standardisation du préfixe `/{organizationId}`.
3. Nomenclature définitive des permissions et politiques d'accès par ressource.
4. DTO de création/mise à jour, champs requis, valeurs des statuts et contraintes d'unicité.
5. Politique de suppression/archivage, de pagination et de traitement des conflits concurrentiels.
6. Contrats exacts des interactions entre modules activables.
7. Choix et génération du document OpenAPI ainsi que l'emplacement de Swagger.


&nbsp;
# 11. Conclusion
Ce document transforme le plan initial en **proposition de contrat API REST V1** cohérente avec la conception du SIG. Les endpoints ne doivent être qualifiés d'« implémentés » qu'après vérification du code NestJS, des DTO, des tests et de la documentation OpenAPI générée.