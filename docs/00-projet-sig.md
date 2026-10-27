# Projet SIG — Vision et feuille de route

[← Accueil du projet](../README.md) · [Documentation](../README.md#documentation)

**Services offerts** :
- Gestion d’événements et bénévoles*
- Gestion des stocks et des fournisseurs*
- Gestion RH des employés internes (dossiers, contrats, congés, absences)*
- Gestion financières
- Gestion de demandes internes (création de ticket, assignation de tâches, etc)*
- Gestion des inventaires étendues (matérielles de bureaux, etc.)*
- Gestion de communication interne ?
- Gestion des communications externes ? (Newsletters, etc.)
- Gestion des formations
- Gestion des incidents, risques, audits et conformité
- Gestion des horaires*
- Gestion des clients (création de compte, etc)*
- Gestion de prise de rendez-vous (réservations de salle, réservations de créneaux avec qlq un)*
- Gestion de projet (un peu à la notion, asana, trello, etc)*

*services prioritaires

**Comment** :
- Tout ça en un seul logiciel
- Chaque forme de gestion peut être débloquer ou non selon les besoins des clients et des entreprises
- Abonnements
  
**Ce qui va me différencier des autres SIG** :
- Interface ergonomique et design personnalisable
- Multi-rôles avec interface adaptée à chaque utilisateur (Admin, RH, Gestionnaire d’événements, Bénévole, Employé)
- Architecture modulaire (activer/désactiver des fonctionnalités selon l’organisation)
- Version mobile de consultation (responsive)
  
**Étapes de création de projet et documentation** :
1.  ✅ Analyse des besoins
  - 🎯 **Objectif** : comprendre qui va utiliser le SIG, pour quoi faire, et quels problèmes tu veux résoudre.
  - ✏️ **À faire** :
    - Identification des rôles utilisateurs (Admin, RH, Bénévole, Gestionnaire…)
    - Définition des cas d’utilisation principaux
  - 📄 **Livrables** :
    - Document “Analyse des besoins” : description des rôles + liste des besoins par rôle + liste des modules (v1, v2, etc.)
    - UML / schémas : diagramme de contexte + diagramme de cas d’utilisation global

2.  ✅ Conception et spécifications fonctionnelles
  - 🎯 **Objectif** : transformer les besoins en fonctionnalités claires, avec des scénarios d’utilisation.
  - ✏️ **À faire** :
    - User stories
    - Scénarios clés (ex : affectation d’un bénévole, création d’un ticket interne)
  - 📄 **Livrables** :
    - Spécifications fonctionnelles par module : fonctionnalités + règles + rôles concernés + scénarios d’utilisation.
   - UML / schémas : diagrammes de séquence UML pour les scénarios importants + diagrammes d’activité UML un workflows *+ (optionnel) Use cases détaillés par module*

3.  ✅ Conception technique
  - 🎯 **Objectif** : Décider comment tu construis le système côté technique (architecture, données, composants).
  - ✏️ **À faire** :
    - Architecture : Frontend + API REST + Base de données
    - Authentification & rôles
    - Modèle de données basé sur utilisateurs, événements, tâches, inventaire
  - 📄 **Livrables** :
    - Document d’architecture : description des choix (techno, style d’architecture) + description des services / modules back-end
    - UML / schémas : diagramme de classes UML (domaine métier) + diagramme de composants UML (haut niveau) + Diagramme de déploiement UML *+ (optionnel) 1 diagramme d’états UML*

4.  Design UX/UI
  - 🎯 **Objectif** : définir à quoi ressemble le logiciel et comment on navigue dedans.
  - ✏️ **À faire** :
    - Maquettes des modules principaux (Figma)
    - Design system minimal (boutons, tableaux, formulaires, etc)
  - 📄 **Livrables** :
    - UI : Wireframes / maquettes (Figma)
    - *(optionnel) UX : Flowcharts pour décrire des parcours*

5.  Développement backend et frontend
  - 🎯 **Objectif** : implémenter la logique métier et les API.
  - ✏️ **À faire** :
    - Backend (API) : endpoints, gestion des rôles, gestion des modules
    - Frontend (Web) : dashboards par rôle, formulaires, vues
    - Version responsive mobile
  - 📄 **Livrables** :
    - Documentation API : s’inspirer de la documentation API_OpenDock ou API_CONFORMiT

6.  Tests
  - 🎯 **Objectif** : vérifier que ça marche, que les rôles sont respectés, et que l’expérience est fluide.
  - ✏️ **À faire** :
    - Scénarios manuels sur les cas d’utilisation majeurs (pas de tests unitaires massifs)
    - Vérification des permissions selon le rôle

7.  Documentation & présentation vidéo
  - 🎯 **Objectif** : rendre le projet clairement présentable et professionnel
  - ✏️ **À faire** :
    - README : le concept du SIG + les rôles supportés. + instructions d’installation
    - Documentation utilisateur par rôle
    - Documentation API
    - Schémas UML
      - Diagramme de cas d’utilisation global
      - Diagrammes de cas d’utilisation par module
      - Diagrammes de séquence
      - Diagramme d’activité
      - Diagramme de classes
      - Diagramme de composants
      - Diagramme de déploiement
      - Diagramme d’états
    - Vidéo de présentation (un peu pub)
      - le problème initial
      - ma vision du SIG et ce qui le différencie des autres
      - une démo
      - ***Vision pour le futur : pousser la partie backend avec l’API + pousser la partie qualité avec les tests + proposés les autres services au fur et à mesure du temps***