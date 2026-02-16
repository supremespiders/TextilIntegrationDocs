# WinTech – Extension Production Textile
## Analyse Technique et Fonctionnelle - V2

**Version du document :** 2.0
**Date :** 16 février 2026
**Basé sur :** Spécification Client V2 (révisée février 2026)

> **Guide de lecture :** Les éléments marqués en <mark>surbrillance jaune</mark> représentent les **modifications ou ajouts introduits dans la spécification V2**. Les estimations de temps sont également surlignées lorsqu'elles ont changé.

---

## Résumé Exécutif

Ce document présente l'analyse technique et fonctionnelle pour l'extension de WinTech afin de prendre en charge la production textile. Le projet comprend deux ensembles de fonctionnalités principaux :

1. **Système de Workflow par Opération** (priorité HAUTE) — Un moteur de workflow dynamique applicable à TOUTES les familles de produits (textile et technique), avec suivi des opérations, calcul des temps et traçabilité des opérateurs.

2. **Intégration Client KW** (priorité MOYENNE) — Échanges de fichiers XML spécifiques au client Kwantum pour la gestion de son stock textile et ses besoins de reporting.

**Décisions clés :**
- Le workflow s'applique à TOUTES les familles (transition fluide depuis le workflow statique actuel)
- Les concepts Étape/Sous-étape correspondent aux entités Statut existantes (compatibilité ascendante)
- Les Opérations sont la nouvelle unité de travail atomique sous chaque Statut
- La « Gamme » (variante produit) est gérée via les champs spécifiques existants + Formula API (pas de nouvelle entité)
- L'intégration KW est spécifique à ce client (pas une fonctionnalité générale)
- <mark>Les opérations textile utilisent le scannage de codes-barres (via talons) — les opérations ignorées ne bloquent pas la progression du statut</mark>
- <mark>Les opérations du département technique utilisent la confirmation par Poste — l'ordre EST imposé dans l'interface</mark>

**Technologie :** .NET Framework 4.8 (pas de mise à niveau requise)

**Effort total estimé :** ~<mark>57</mark>-<mark>68</mark> jours-personnes

---

## 1. Analyse Fonctionnelle

### 1.1 Fonctionnalité 1 : Système de Workflow par Opération

#### 1.1.1 Vue d'ensemble

Transformer WinTech d'un workflow global statique vers un **workflow dynamique par BDC**, où chaque BDC reçoit sa propre instance de workflow basée sur la famille, le produit et les valeurs des champs spécifiques.

**Périmètre :** TOUTES les familles de produits (Textile : Tenture, etc. ET Technique : Enrouleur, Duo, Vertical, Skylight, PJ)

#### 1.1.2 État Actuel vs État Cible

| Aspect | Actuel | Cible |
|--------|--------|-------|
| Définition du workflow | Chaîne statique globale (WorkflowStep) | Configuration par famille |
| Unité de suivi | Statut uniquement | Statut + Opérations |
| Calcul des temps | Aucun | Par opération via Formula API |
| Suivi des accessoires | Aucun | Par opération via Formula API |
| Affectation opérateur | Par Poste | Par Opération (avec Poste) |
| Chemins parallèles | Cas particulier codé en dur | Naturel via l'ordonnancement des opérations |

#### 1.1.3 Structure du Workflow

```
Famille
  └── FamilyWorkflow (configuration)
        └── Statuts (séquence ordonnée)
              └── Opérations (ordonnées dans le statut)
                    └── Accessoires (consommés par l'opération)

BDC Créé
  └── BdcWorkflow (instance, immuable)
        └── BdcWorkflowOperations (avec valeurs calculées)
              └── BdcWorkflowOperationAccessories
```

**Point clé :** La « Gamme » (variante produit) N'est PAS une nouvelle entité. La combinaison de valeurs de champs spécifiques existe déjà dans `CalculatedSpecificField`. La Formula API détermine quelles opérations s'appliquent et leurs paramètres.

#### 1.1.4 Exigences Fonctionnelles

**FR-WF-01 : Configuration du Workflow**
- L'administrateur configure le workflow par famille
- Chaque workflow possède des statuts ordonnés
- Chaque statut possède des opérations ordonnées
- Chaque opération a : TimeFormulaCode, OrderFormulaCode
- Chaque opération peut avoir des accessoires avec QuantityFormulaCode
- Un seul workflow actif par famille

**FR-WF-02 : Génération du Workflow**
- Déclenché lors de la création d'un BDC (Communicator ou manuel)
- Le système identifie la famille, le produit, charge le workflow actif
- Pour chaque opération : appel à la Formula API pour calculer OrderValue, TimeValue
- Pour chaque accessoire : appel à la Formula API pour calculer QuantityValue
- Création de BdcWorkflow + enregistrements enfants (instantané immuable)

**FR-WF-03 : Complétion des Opérations**
- Deux modes de complétion selon le type de département :
  - **Temps réel (Département Technique) :** L'opérateur au Poste clique sur confirmer — les opérations doivent être complétées dans l'ordre ; l'interface impose la séquence
  - **Par lot via Talon (Département Textile) :** Le superviseur scanne les codes-barres — <mark>les opérations ignorées NE bloquent PAS le statut ; voir FR-TAL-04 pour la gestion</mark>
- Marqueurs de complétion : IsCompleted, CompletedAt, CompletedById
- Quand TOUTES les opérations d'un statut sont complètes → le statut du BDC change
- BdcStatusLog créé pour la piste d'audit

**FR-WF-04 : Compatibilité Ascendante**
- Les requêtes/filtres basés sur les statuts fonctionnent sans modification
- Les rapports utilisant les statuts fonctionnent sans modification
- La confirmation par Poste fonctionne (marque les opérations comme complètes)
- Les BDC existants sans workflow continuent normalement

#### 1.1.5 Impact du Workflow sur le Système Existant

| Zone | Impact |
|------|--------|
| Création de BDC | Ajout de l'étape d'instanciation du workflow |
| Confirmation par Poste | Marque les opérations complètes dans l'ordre (Technique) |
| Affichage des statuts | Aucun changement (dérivé des opérations) |
| BdcStatusLog | Préservé (créé automatiquement au changement de statut) |
| Cas particulier de statut parallèle | Supprimé (géré naturellement par les opérations) |

---

### 1.2 Fonctionnalité 2 : Système Talon (Étiquettes Code-Barre d'Opération)

#### 1.2.1 Vue d'ensemble

Étiquettes code-barre physiques (« talons ») pour les opérateurs textile qui n'ont pas accès à un PC. Les opérateurs collectent les talons après avoir réalisé les opérations ; le superviseur les scanne pour enregistrer la complétion.

#### 1.2.2 Spécification du Talon

- **Taille :** Étiquette thermique 4,1 × 1,5 cm
- **Contenu :**
  - Numéro BDC + Client
  - Nom de l'opération + Code
  - Code-barre (<mark>encode `BdcWorkflowOperation.Id` — clé primaire DB globalement unique, identifie précisément l'instance d'opération sur le BDC exact lors du scannage</mark>)
  - Temps calculé
  - Date de génération

#### 1.2.3 Exigences Fonctionnelles

**FR-TAL-01 : Impression des Talons**
- Imprimés lors de l'impression du rapport BDC
- Le rapport contient tous les talons du BDC, <mark>ordonnés selon l'ordre d'impression en balayage (lecture de gauche à droite, de haut en bas — respectant l'ordre des statuts puis l'ordre des opérations dans chaque statut)</mark>
- L'utilisateur sélectionne l'imprimante (aucun modèle spécifique requis)
- Format grille (peut être découpé/séparé)

**FR-TAL-02 : Interface de Scannage des Talons**
- Nouveau formulaire WinTech pour le superviseur
- Scan du code-barre → Affichage des informations de l'opération
- Sélection de l'opérateur dans la liste déroulante (par nom)
- Clic sur Compléter → Marque l'opération comme terminée, affecte l'opérateur
- Liste des complétions récentes pour vérification

**FR-TAL-03 : Suivi des Performances des Opérateurs**
- Données capturées : Opérateur, Horodatage, Temps prévu
- Rapports de base : Opérations par opérateur, Temps total par opérateur
- Formule de calcul des performances : DIFFÉRÉ (phase future)

**FR-TAL-04 : <mark>Gestion des Opérations Ignorées (NOUVEAU)</mark>**
- <mark>Si le code-barre d'une opération N'est PAS scanné, mais que le code-barre de l'opération **suivante** EST scanné :</mark>
  - <mark>Le BDC avance au statut correspondant à la dernière opération scannée</mark>
  - <mark>L'opération ignorée est automatiquement marquée comme complète avec : PostId = NULL, CompletedAt = NULL, CompletedById = NULL</mark>
- <mark>Une interface dédiée **Vue Anomalies** permet aux superviseurs de :</mark>
  - <mark>Visualiser toutes les opérations non scannées (complétées automatiquement) par BDC</mark>
  - <mark>Marquer manuellement une opération ignorée comme complète (rétroactivement)</mark>
  - <mark>Affecter un opérateur à l'opération complétée manuellement</mark>

<mark>**FR-TAL-05 : Champs Personnel (NOUVEAU)**</mark>
- <mark>L'entité User est étendue avec : Matricule, FirstName, LastName, IsActive — pour prendre en charge l'identification future des opérateurs</mark>
- <mark>Formulaire de gestion du personnel (CRUD) ajouté à la configuration</mark>
- **Scannage de badge / sélection d'opérateur par Matricule dans l'interface de scannage : DIFFÉRÉ** (itération future une fois la ligne textile opérationnelle et les lecteurs de badges en place)

---

### 1.3 Fonctionnalité 3 : Intégration Client KW

#### 1.3.1 Vue d'ensemble

Le client Kwantum (KW) nécessite des échanges de fichiers XML spécifiques pour son stock textile et le suivi de ses livraisons. **Ceci est spécifique à KW, pas une fonctionnalité générale.**

#### 1.3.2 NEV/IC2S (Gestion de Stock)

**FR-KW-01 : Import NEV**
- KW envoie chaque semaine un inventaire de rouleaux textile (fichier XML NEV)
- WinTech importe via SFTP ou dossier local
- Crée les enregistrements de rouleaux avec emplacement initial « ND » (non déterminé)
- Contrôles : format XML, existence des références articles, unicité des rouleaux

**FR-KW-02 : Processus de Réception**
- Impression des étiquettes de rouleaux
- Réception physique + scannage des rouleaux
- <mark>La confrontation gère deux cas distincts :</mark>
  - <mark>**Cas 1 — Sur-réception :** Quantité scannée > quantité attendue → signalement des rouleaux en surplus, décision manuelle requise avant validation</mark>
  - <mark>**Cas 2 — Sous-réception :** Quantité scannée < quantité attendue → signalement des rouleaux manquants, possibilité de clôturer la NEV avec validation partielle</mark>
- Validation → création des enregistrements FabricRoll
- <mark>Le scannage d'un rouleau génère un fichier déclencheur `.txt` (récupéré par le Communicator pour le traitement en aval)</mark>
- Clôture de la NEV (verrouille toute modification ultérieure)

**FR-KW-03 : Retour IC2S**
- Généré après la clôture de la NEV
- <mark>Confirme les rouleaux reçus à KW, en incluant le statut de suivi de réception (complet / partiel / avec surplus)</mark>
- Envoyé via SFTP

#### 1.3.3 Fichiers de Mouvement

**FR-KW-04 : PI2S (Mouvements de Coupe)**
- Généré lors de la coupe du tissu KW
- Contient : article, palette, quantité
- Déclenché après confirmation du plan de coupe

**FR-KW-05 : SA2S (Ajustements de Stock)**
- Généré pour les surconsommations/ajustements
- Contient : article, palette, code mutation, quantité
- Mapping des types de mouvement (configurable par l'administrateur)

**FR-KW-06 : Stock Photo**
- <mark>**DIFFÉRÉ** — hors périmètre pour cette phase</mark>
- <mark>La fonctionnalité de snapshot hebdomadaire du stock sera traitée dans une version future</mark>

#### 1.3.4 Bons de Livraison

**FR-KW-07 : GOMV/VOWV**
- GOMV : Pour la famille « Rideaux Suspendus » (articles suspendus)
- <mark>VOWV : Pour **toutes les commandes textiles non suspendues** (non limité aux « Stores Bateaux » — la logique de sélection est basée sur Family.Type dans le code)</mark>
- Inclut le numéro de barre (séquentiel) et le numéro RFID
- <mark>**Numéro de Barre :** Compteur séquentiel **par entité Store** (pas global). Chaque magasin a son propre compteur. Les articles suspendus utilisent le compteur ; les articles non suspendus reçoivent la valeur = 0.</mark>

**FR-KW-08 : Export Statut BDC**
- Mappage des statuts internes vers les codes KW (1-15)
- Export XML planifié (mécanisme Communicator existant)
- <mark>Toutes les commandes KW actives sont incluses dans l'export quel que soit leur statut actuel ; le statut de chaque commande est mappé vers le code KW (1–15) selon la table de mappage</mark>

**FR-KW-09 : <mark>Suivi des Envois de Fichiers (NOUVEAU)</mark>**
- <mark>Chaque fichier généré (PI2S, SA2S, IC2S, GOMV, VOWV, Statut) doit être tracé dans une table de log dédiée</mark>
- <mark>Règle de déduplication : chaque fichier est envoyé **une seule fois** par cycle de génération ; pas de doublon d'envoi</mark>
- <mark>Champs de suivi : FileName, FileType, GeneratedAt, SentAt, Status (Pending/Sent/Failed), ErrorMessage</mark>
- <mark>Nouvelle table : `ClientFileSendLog`</mark>

---

### 1.4 Fonctionnalité 4 : Impression RFID

#### 1.4.1 Vue d'ensemble

Impression d'étiquettes d'emballage avec puces RFID pour certaines familles textiles lors de l'opération Emballage.

#### 1.4.2 Exigences Fonctionnelles

**FR-RFID-01 : Déclencheur d'Impression RFID**
- <mark>L'impression RFID s'applique aux **articles suspendus** uniquement (famille Rideaux Suspendus)</mark>
- <mark>La détermination est effectuée dans le code en vérifiant : (1) le type de `bdc.Family` est suspendu, et (2) la quantité de tissu depuis `CalculatedSpecificField` — aucun appel à la Formula API requis</mark>
- <mark>Le bouton « Imprimer RFID » sur le formulaire d'emballage est activé/désactivé selon cette vérification</mark>

**FR-RFID-02 : Génération du Numéro RFID**
- Compteur séquentiel (9 chiffres)
- <mark>Numéro de départ configuré une fois par l'administrateur (pour s'aligner sur le dernier numéro généré par Winsat — configuration de migration unique)</mark>
- Stocké dans Bdc.RfidNumber

**FR-RFID-03 : Impression RFID**
- Lors de l'opération d'emballage (Emballage)
- Bouton « Imprimer RFID » sur le formulaire d'emballage
- Envoi à l'imprimante RFID
- La réimpression nécessite une autorisation (génère un nouveau numéro)
- <mark>**Règle des 2 étiquettes :** Si le `nombre de cintres` de l'article (depuis CalculatedSpecificField) = 2, le système génère et imprime **2 étiquettes RFID** pour ce BDC</mark>

**FR-RFID-04 : Informations Client en Attente**
- ~~Modèle d'imprimante~~ **Répondu : Zebra ZT411**
- Modèle d'étiquette ZPL (mise en page visuelle + commande d'écriture RFID pour l'étiquette d'emballage)

---

### <mark>1.5 Fonctionnalité 5 : Gestion de Stock des Accessoires (NOUVEAU)</mark>

#### <mark>1.5.1 Vue d'ensemble</mark>

<mark>Les accessoires consommés par les opérations de workflow auront leur propre système de gestion de stock, reprenant le modèle de stock existant FabricRoll. Le stock est réduit automatiquement lorsqu'une opération utilisant l'accessoire est complétée, et suit le même modèle stock principal / stock temporaire / mouvement / surconsommation que le tissu.</mark>

#### <mark>1.5.2 Exigences Fonctionnelles</mark>

<mark>**FR-ACC-01 : Entrée de Stock Accessoire**</mark>
- <mark>Possibilité de saisir le stock accessoire (saisie manuelle ou import — détails à définir)</mark>
- <mark>Stock organisé en stock principal et stock temporaire (même modèle que FabricRoll)</mark>

<mark>**FR-ACC-02 : Réduction de Stock à la Complétion d'Opération**</mark>
- <mark>Lors de la complétion d'une opération de workflow, les quantités d'accessoires calculées (`BdcWorkflowOperationAccessory.QuantityValue`) sont déduites du stock</mark>
- <mark>Suit la même logique de mouvement de stock que les coupes tissu</mark>

<mark>**FR-ACC-03 : Surconsommation**</mark>
- <mark>Si la quantité réellement utilisée dépasse la quantité calculée, un enregistrement de surconsommation est créé</mark>
- <mark>Même modèle que la surconsommation tissu existante</mark>

<mark>**FR-ACC-04 : Export de Stock**</mark>
- <mark>Les mouvements de stock accessoire sont exportables (format et déclencheur à définir — à communiquer par le client)</mark>

> <mark>**Note :** Le design détaillé (interface, types de mouvements, format d'export) sera communiqué par le client. Le périmètre est confirmé ; l'estimation est basée sur la parité avec le système de stock FabricRoll existant.</mark>

---

### <mark>1.6 Fonctionnalité 6 : Facturation KW Textile (NOUVEAU)</mark>

#### <mark>1.6.1 Vue d'ensemble</mark>

<mark>Les commandes textiles KW suivent une logique de facturation différente des commandes techniques KW. La table `ImportedPrice` existante (déjà dans WinTech) contient les prix importés depuis les fichiers de prix officiels de KW. Pour le textile, ces prix importés sont la **base de facturation** — l'inverse de la facturation technique où le prix calculé par la matrice interne est la base.</mark>

#### <mark>1.6.2 Comparaison de la Logique de Facturation</mark>

| | KW Technique (existant) | <mark>KW Textile (nouveau)</mark> |
|---|---|---|
| Base de facturation | Prix matrice interne (PreInvoice) | <mark>Prix KW importé (ImportedPrice)</mark> |
| Référence / comparaison | ImportedPrice | <mark>Prix matrice interne (PreInvoice)</mark> |
| Blocage si absent | Non | <mark>Oui — facturation bloquée si aucun ImportedPrice trouvé</mark> |

#### <mark>1.6.3 Exigences Fonctionnelles</mark>

<mark>**FR-INV-01 : Résolution du Prix**</mark>
- <mark>Pour les BDC KW textile : le prix de facturation est recherché dans `ImportedPrice` par code article (ProductPriceCode)</mark>
- <mark>Si aucun `ImportedPrice` correspondant trouvé → la facturation est bloquée avec un message d'erreur explicite</mark>
- <mark>Les prix de la matrice interne (lignes PreInvoice) sont générés en colonne de référence/comparaison uniquement</mark>

<mark>**FR-INV-02 : Simulation (Inversée)**</mark>
- <mark>Simulation existante pour le technique : affiche le prix matrice Windeco et le compare au prix client importé</mark>
- <mark>Nouvelle simulation pour KW textile : affiche le prix KW importé comme base et le compare au prix Windeco calculé — présentation inversée</mark>

<mark>**FR-INV-03 : Rapport de Facture**</mark>
- <mark>Un rapport de facture KW spécifique au textile étendra ou remplacera le rapport `InvoiceKwantum` existant</mark>
- <mark>Design du modèle de rapport : **À définir — à communiquer par le client**</mark>

<mark>**FR-INV-04 : Facture XML (Communicator)**</mark>
- <mark>Un fichier de facture XML modifié est envoyé à KW via le Communicator</mark>
- <mark>Chaque ligne comprend : codes articles prix, prix unitaires, quantités</mark>
- <mark>Format XML exact / fichier exemple : **À définir — à communiquer par le client**</mark>
- <mark>Envoyé via le mécanisme SFTP du Communicator existant (nouveau type de fichier ajouté)</mark>

#### <mark>1.6.4 Notes sur l'Infrastructure Existante</mark>

- <mark>L'entité `ImportedPrice` existe déjà et est liée au BDC via `BdcId` et à `ProductPrice` via `ProductPriceId`</mark>
- <mark>La classe de rapport `InvoiceKwantum.cs` existe déjà — sera adaptée pour la variante textile</mark>
- <mark>La branche client KW est déjà présente dans `InvoiceService.cs` (ClientId 1 et 413) — le switch textile y sera ajouté selon le type de famille</mark>
- <mark>Pas de nouvelle table DB requise ; changement de logique uniquement + nouveau rapport de facture + nouveau type de fichier Communicator</mark>

#### <mark>1.6.5 Points Ouverts pour la Prochaine Réunion</mark>

- <mark>Mettre en évidence la différence entre la facture technique et la facture textile</mark>

---

## 2. Analyse Technique

### 2.1 Modifications du Schéma de Base de Données

#### 2.1.1 Nouvelles Entités

**Operation** (Données de référence)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| Code | VARCHAR(50) UNIQUE | Code de l'opération |
| Name | VARCHAR(255) | Nom affiché |
| Description | VARCHAR(1024) NULL | |
| IsActive | bool | Vrai par défaut |

**FamilyWorkflow** (Configuration)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| FamilyId | int FK | Référence à Family |
| Name | VARCHAR(255) | Nom du workflow |
| IsActive | bool | Un seul actif par famille |
| CreatedAt | DateTime | |

**FamilyWorkflowStatus** (Statut dans le Workflow)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| FamilyWorkflowId | int FK | |
| StatusId | int FK | Référence au Statut existant |
| Order | int | Ordre de séquence |

**FamilyWorkflowStatusOperation** (Configuration Opération)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| FamilyWorkflowStatusId | int FK | |
| OperationId | int FK | |
| TimeFormulaCode | VARCHAR(100) | Formule pour le calcul du temps |
| OrderFormulaCode | VARCHAR(100) | Formule pour l'ordre d'exécution |
| PostId | int FK NULL | Poste de travail assigné |

**FamilyWorkflowOperationAccessory** (Configuration Accessoire)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| FamilyWorkflowStatusOperationId | int FK | |
| AccessoryId | int FK | |
| QuantityFormulaCode | VARCHAR(100) | Formule pour la quantité |

**BdcWorkflow** (Instance par BDC)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| BdcId | int FK UNIQUE | 1:1 avec Bdc |
| FamilyWorkflowId | int FK | Workflow source |
| CreatedAt | DateTime | |

**BdcWorkflowOperation** (Instance d'Opération)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| BdcWorkflowId | int FK | |
| OperationId | int FK | |
| StatusId | int FK | Statut auquel appartient cette opération |
| OrderValue | int | Séquence calculée |
| TimeValue | decimal NULL | Temps calculé (minutes) |
| IsCompleted | bool | Faux par défaut |
| CompletedAt | DateTime NULL | |
| CompletedById | int FK NULL | Référence à User (opérateur) |
| <mark>IsAutoCompleted</mark> | <mark>bool DEFAULT false</mark> | <mark>Vrai si complété par la règle d'opération ignorée (pas de scan)</mark> |

**BdcWorkflowOperationAccessory** (Instance d'Accessoire)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| BdcWorkflowOperationId | int FK | |
| AccessoryId | int FK | |
| QuantityValue | decimal NULL | Quantité calculée |

**ClientMovementTypeMapping** (Spécifique KW)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| ClientId | int FK | |
| MovementTypeId | int FK | |
| ClientCode | VARCHAR(10) | Code KW |

**NevImport** (Spécifique KW)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| Reference | VARCHAR(50) | Référence NEV |
| ClientId | int FK | Toujours KW |
| ImportDate | DateTime | |
| IsClosed | bool | |
| FileName | VARCHAR(255) | |

**NevRoll** (Spécifique KW)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| NevImportId | int FK | |
| ArticleCode | VARCHAR(10) | |
| PaletteNumber | int | |
| Quantity | decimal | Mètres |
| IsValidated | bool | |
| FabricRollId | int FK NULL | Créé après validation |

**GlobalCounter** (Séquences)
| Colonne | Type | Description |
|---------|------|-------------|
| Id | int PK | |
| CounterName | VARCHAR(50) UNIQUE | ex. « RFID » |
| CurrentValue | int | |
| PadLength | int | Remplissage par zéros |

<mark>**StoreRodCounter** (NOUVEAU — spécifique KW, numéro de barre par magasin)</mark>
| Colonne | Type | Description |
|---------|------|-------------|
| <mark>Id</mark> | <mark>int PK</mark> | |
| <mark>StoreId</mark> | <mark>int FK UNIQUE</mark> | <mark>Référence à l'entité Store</mark> |
| <mark>CurrentValue</mark> | <mark>int</mark> | <mark>Compteur séquentiel de barre pour ce magasin</mark> |

<mark>**ClientFileSendLog** (NOUVEAU — spécifique KW, suivi des envois de fichiers)</mark>
| Colonne | Type | Description |
|---------|------|-------------|
| <mark>Id</mark> | <mark>int PK</mark> | |
| <mark>ClientId</mark> | <mark>int FK</mark> | |
| <mark>FileName</mark> | <mark>VARCHAR(255)</mark> | |
| <mark>FileType</mark> | <mark>VARCHAR(50)</mark> | <mark>ex. PI2S, SA2S, IC2S, GOMV, VOWV, Status</mark> |
| <mark>GeneratedAt</mark> | <mark>DateTime</mark> | |
| <mark>SentAt</mark> | <mark>DateTime NULL</mark> | |
| <mark>Status</mark> | <mark>VARCHAR(20)</mark> | <mark>Pending / Sent / Failed</mark> |
| <mark>ErrorMessage</mark> | <mark>VARCHAR(1024) NULL</mark> | |

#### 2.1.2 Entités Modifiées

**Bdc**
- Ajout : `RfidNumber` VARCHAR(20) NULL

**Family**
- <mark>Aucun nouveau champ nécessaire — l'éligibilité RFID est déterminée dans le code à partir du type de famille existant et de CalculatedSpecificField</mark>

<mark>**User** (MIS À JOUR — champs personnel ajoutés)</mark>
- <mark>Ajout : `Matricule` VARCHAR(50) NULL UNIQUE</mark>
- <mark>Ajout : `FirstName` VARCHAR(100) NULL</mark>
- <mark>Ajout : `LastName` VARCHAR(100) NULL</mark>
- <mark>Ajout : `IsActive` bool DEFAULT true</mark>

#### 2.1.3 Index

- `BdcWorkflow.BdcId` - UNIQUE
- `BdcWorkflowOperation(BdcWorkflowId, StatusId, OrderValue)` - Optimisation des requêtes
- `FamilyWorkflow(FamilyId, IsActive)` - Recherche du workflow actif
- `NevRoll.PaletteNumber` - Recherche lors du scannage
- <mark>`BdcWorkflowOperation.IsAutoCompleted` - Filtrage pour la vue anomalies</mark>
- <mark>`ClientFileSendLog(ClientId, FileType, Status)` - Déduplication et file d'envoi</mark>
- <mark>`User.Matricule` - Recherche opérateur par scan de badge</mark>

---

### 2.2 Couche Service

#### 2.2.1 WorkflowService (Nouveau)

**Responsabilités :**
- Charger le workflow actif pour une famille
- Instancier BdcWorkflow depuis la configuration
- Compléter les opérations
- Calculer le statut depuis la complétion des opérations
- <mark>Appliquer la règle d'ignoré : complétion automatique des opérations précédentes lors d'un scan dans le désordre</mark>

**Méthodes clés :**
```
GetActiveFamilyWorkflow(familyId) → FamilyWorkflow
InstantiateWorkflow(bdcId, familyWorkflowId) → BdcWorkflow
CompleteOperation(operationId, userId) → void
CompleteOperationByBarcode(barcodeId, userId) → CompleteOperationResult  [NOUVEAU]
GetCurrentStatusFromWorkflow(bdcWorkflowId) → Status
GetAutoCompletedOperations(bdcWorkflowId) → List<BdcWorkflowOperation>  [NOUVEAU]
```

#### 2.2.2 Ajustements Formula API

**Tâche :** S'assurer que la Formula API couvre tous les besoins de calcul pour le système de workflow.

**Exigences :**
- Calculer le temps d'opération depuis les champs spécifiques du BDC
- Calculer l'ordre d'opération (séquence)
- Calculer les quantités consommées d'accessoires
- Garantir les performances en proportion du nombre de BDC traités
- Fournir tout le contexte nécessaire : champs spécifiques, champs génériques, propriétés tissu, références accessoires

**Effort estimé :** 2-3 jours pour les ajustements et la validation

#### 2.2.3 NevService (Nouveau, spécifique KW)

**Responsabilités :**
- Analyser les fichiers XML NEV
- <mark>Gérer les deux cas de confrontation : sur-réception et sous-réception</mark>
- Générer la réponse IC2S <mark>avec statut de réception</mark>
- Upload/download SFTP
- <mark>Écrire le fichier déclencheur `.txt` lors du scannage d'un rouleau</mark>

<mark>**FileSendService** (Nouveau, spécifique KW)</mark>
- <mark>Responsabilités : Tracer tous les fichiers sortants, imposer la déduplication, journaliser les résultats d'envoi</mark>
- <mark>Méthodes clés :</mark>
  - <mark>`RegisterFile(clientId, fileName, fileType)` → ClientFileSendLog</mark>
  - <mark>`MarkSent(logId)` → void</mark>
  - <mark>`MarkFailed(logId, error)` → void</mark>
  - <mark>`HasAlreadySent(clientId, fileName, fileType)` → bool (vérification dédup)</mark>

---

### 2.3 Modifications de l'Interface WinTech

#### 2.3.1 Nouveaux Formulaires de Configuration

| Formulaire | Objectif |
|------------|---------|
| Catalogue des Opérations | CRUD pour l'entité Operation |
| Configuration Workflow Famille | Éditeur multi-niveaux : Famille → Statuts → Opérations → Accessoires |
| Mapping Types de Mouvement | Mapping des codes mouvement KW |
| Configuration Import NEV | Paramètres SFTP, planification |
| <mark>Gestion du Personnel</mark> | <mark>CRUD des enregistrements opérateurs (Matricule, FirstName, LastName, IsActive)</mark> |

#### 2.3.2 Nouveaux Formulaires de Production

| Formulaire | Objectif |
|------------|---------|
| Scannage Talon | Scanner les codes-barres, affecter les opérateurs (par nom), marquer comme terminé |
| Visualiseur Workflow BDC | Vue en lecture seule de l'instance workflow du BDC |
| Liste des Imports NEV | Consulter les imports NEV et leur statut |
| Détail Import NEV | Confrontation (sur/sous), validation, clôture |
| <mark>Vue Anomalies</mark> | <mark>Lister les opérations auto-complétées (ignorées) ; permettre la complétion manuelle + affectation opérateur</mark> |

#### 2.3.3 Formulaires Modifiés

| Formulaire | Modifications |
|------------|--------------|
| Détail BDC | Ajout : numéro RFID, bouton « Voir Workflow » |
| Formulaire Emballage | Ajout : bouton « Imprimer RFID » (<mark>activé selon vérification en code : famille suspendue + quantité tissu depuis CalculatedSpecificField</mark>) |
| Confirmation Poste | Logique : marque les opérations complètes dans l'ordre (département Technique — interface impose la séquence) |

#### 2.3.4 Nouveaux Rapports

| Rapport | Objectif |
|---------|---------|
| Impression Talons | <mark>Étiquettes code-barre d'opération (4,1 × 1,5 cm), imprimées dans l'ordre de balayage (lecture gauche-droite, haut-bas)</mark> |
| Performance Opérateurs | Opérations par opérateur, temps par opérateur |

---

### 2.4 Modifications du Communicator

#### 2.4.1 Extension de la Création de BDC

- Après la création du BDC, appeler WorkflowService.InstantiateWorkflow
- Journaliser les erreurs si l'instanciation du workflow échoue

#### 2.4.2 Traitement des Fichiers KW (spécifique KW)

| Tâche | Déclencheur | Notes |
|-------|-------------|-------|
| Import NEV | Planifié/Manuel | |
| Génération IC2S | Clôture NEV | <mark>Inclut le statut de réception dans la réponse</mark> |
| Génération PI2S | Confirmation plan de coupe | <mark>Vérification dédup via ClientFileSendLog</mark> |
| Génération SA2S | Ajustement de stock | <mark>Vérification dédup via ClientFileSendLog</mark> |
| ~~Stock Photo~~ | ~~Hebdomadaire (dimanche)~~ | <mark>**DIFFÉRÉ**</mark> |
| Génération GOMV | Finalisation livraison | <mark>Articles suspendus uniquement ; compteur de barre par Store</mark> |
| Génération VOWV | Finalisation livraison | <mark>Toutes les commandes textiles non suspendues ; compteur de barre par Store</mark> |
| Export Statut BDC | Planifié | |

<mark>**Gestion du Fichier Déclencheur (NOUVEAU)**</mark>
- <mark>Lors du scannage d'un rouleau pendant la réception NEV, un fichier déclencheur `.txt` est écrit dans un dossier surveillé</mark>
- <mark>Le Communicator récupère ce fichier et initie le traitement en aval</mark>
- <mark>Cela découple l'interface WinTech du Communicator sans appels API directs</mark>

---

## 3. Version .NET

**Décision :** Rester sur .NET Framework 4.8

**Justification :**
- Compatibilité DevExpress WinForms
- L'effort de migration ne se justifie pas pour ce périmètre
- .NET Framework 4.8 bénéficie d'un support illimité via Windows
- La Formula API est déjà sur .NET 10 (intégration HTTP, pas de conflit de runtime)

**Impact :** Aucun. Toutes les bibliothèques requises supportent .NET Framework 4.8.

---

## 4. Risques et Dépendances

### 4.1 Risques Techniques

| Risque | Probabilité | Mitigation |
|--------|-------------|------------|
| Erreurs de formule bloquant la création de BDC | Moyenne | Valider les formules lors de la configuration, messages d'erreur clairs |
| Perturbation du département technique pendant la migration | Faible | Tests approfondis, déploiement progressif |
| Intégration imprimante RFID | Faible | Tests nécessaires sur le modèle d'imprimante |
| Performance avec un grand nombre d'opérations | Moyenne | Optimiser les requêtes, utiliser le cache |
| <mark>Alignement du compteur Winsat (valeur de départ RFID)</mark> | <mark>Faible</mark> | <mark>Configuration admin unique ; documenter clairement la procédure</mark> |
| <mark>Cas particuliers de confrontation NEV (partielle)</mark> | <mark>Moyenne</mark> | <mark>Les deux cas (sur/sous réception) sont couverts ; override manuel disponible</mark> |

### 4.2 Dépendances

| Dépendance | Statut |
|------------|--------|
| Disponibilité Formula API | Disponible |
| Spécifications imprimante RFID | En attente client |
| Accès SFTP KW | À configurer |
| Admin pour configurer les workflows | Responsabilité client |
| <mark>Valeur fin du compteur RFID Winsat</mark> | <mark>À fournir par le client (unique)</mark> |

---

## 5. Questions pour le Client

| # | Question | Impact | Statut |
|---|----------|--------|--------|
| 1 | Modèle d'imprimante RFID ? | Implémentation RFID | <mark>**Répondu : Zebra ZT411**</mark> |
| 1b | Modèle d'étiquette ZPL pour l'étiquette d'emballage RFID (mise en page + commande d'écriture RFID) ? | Implémentation RFID | En attente |
| 2 | Nouveaux champs nécessaires sur les tables Client/Tissu/Produit ? | Schéma de base de données | En attente |
| 3 | Format XML de la facture KW Textile (fichier exemple) ? | Fonctionnalité 6 facturation | En attente |
| 4 | Modèle/template du rapport de facture KW Textile ? | Rapport facture Fonct. 6 | En attente |

*Note : Les configurations de formules, listes de statuts et listes d'accessoires seront configurées par l'administrateur client pendant le développement.*

---

## 6. Feuille de Route d'Implémentation

### 6.1 Découpage par Phase

| Phase | Description | Jours est. |
|-------|-------------|------------|
| **Phase 1 : Socle** | Schéma DB, modèles d'entités, migrations (<mark>+champs User, +ClientFileSendLog, +StoreRodCounter, +IsAutoCompleted</mark>) | <mark>3</mark> |
| **Phase 2 : Workflow Core** | WorkflowService, instanciation du workflow, logique de complétion, <mark>règle d'ignoré & auto-complétion</mark> | <mark>6</mark> |
| **Phase 3 : Intégration Formula API** | Ajustements Formula API, calcul des opérations (temps, ordre, accessoires) | <mark>3</mark> |
| **Phase 4 : Interface de Configuration** | Catalogue des opérations, config workflow famille, <mark>formulaire gestion personnel</mark> | <mark>5</mark> |
| **Phase 5 : Système Talon** | Rapport talon (<mark>ordre de balayage</mark>), interface scannage, <mark>formulaire Vue Anomalies</mark> | <mark>5</mark> |
| **Phase 6 : Confirmation Poste** | Modifier le flux de confirmation pour les opérations (dept. Technique, ordre imposé) | <mark>4</mark> |
| **Phase 7 : KW NEV/IC2S** | Import NEV, confrontation (<mark>cas sur + sous réception</mark>), export IC2S (<mark>avec statut</mark>), <mark>mécanisme fichier déclencheur</mark> | <mark>7</mark> |
| **Phase 8 : Fichiers KW** | PI2S, SA2S, GOMV (<mark>compteur barre par Store</mark>), VOWV (<mark>toutes non suspendues</mark>), export statut, <mark>FileSendService + dédup</mark> | <mark>4</mark> |
| **Phase 9 : RFID** | Impression RFID (<mark>vérification en code, règle 2 étiquettes, config départ Winsat</mark>) | <mark>3</mark> |
| **Phase 10 : Tests** | Tests d'intégration, support UAT | <mark>5</mark> |
| **Phase 11 : Rapports** | Performance opérateurs, synthèse opérations | <mark>4</mark> |
| **Phase 12 : Déploiement** | Migration, déploiement, hypercare | <mark>2</mark> |
| <mark>**Phase 13 : Gestion Stock Accessoires**</mark> | <mark>Entrée stock, réduction à la complétion, surconsommation, mouvements, export (détails à définir)</mark> | <mark>3</mark> |
| <mark>**Phase 14 : Facturation KW Textile**</mark> | <mark>Switch logique facturation (textile vs technique), blocage, simulation inversée, rapport facture (dès réception template), facture XML via Communicator (dès réception format)</mark> | <mark>3</mark> |

**Total : ~<mark>57</mark> jours-personnes**

### 6.2 Ordre de Priorité

1. **Phases 1-3 :** Socle + Workflow Core + Formula API (CHEMIN CRITIQUE)
2. **Phases 4-6 :** Interface Config + Talon + Confirmation Poste (permet l'utilisation en production)
3. **Phases 7-8 :** Intégration KW (spécifique client, peut être parallèle)
4. **Phase 9 :** RFID (dépend des specs client)
5. **Phases 10-12 :** Tests, Rapports, Déploiement

### 6.3 Estimation du Calendrier

**Avec 1 développeur :**
- ~<mark>57</mark> jours ouvrés

**Avec contingence (20%) :** ~<mark>68</mark> jours

---

## 7. Synthèse

### 7.1 Ce que Nous Construisons

| Composant | Périmètre | Effort |
|-----------|-----------|--------|
| Système Workflow/Opérations | Toutes les familles, changement d'architecture majeur | LARGE (<mark>15 jours</mark>) |
| Système Talon (<mark>+ Vue Anomalies</mark>) | Département textile, impression + scannage + gestion anomalies | MOYEN (<mark>13 jours</mark>) |
| Intégration KW (<mark>+ dédup fichiers + fichier déclencheur</mark>) | Un client, fichiers XML | MOYEN (<mark>11 jours</mark>) |
| Impression RFID (<mark>vérification en code, règle 2 étiquettes, Zebra ZT411</mark>) | Articles textiles suspendus | PETIT (<mark>3 jours</mark>) |
| <mark>Gestion Stock Accessoires</mark> | <mark>Toutes les familles utilisant des accessoires</mark> | <mark>PETIT (3 jours — détails à définir)</mark> |
| <mark>Facturation KW Textile</mark> | <mark>Client KW, commandes textiles</mark> | <mark>PETIT (3 jours — détails à définir)</mark> |
| Support (config, rapports, tests) | Divers | MOYEN (<mark>9 jours</mark>) |

### 7.2 Ce que Nous ne Construisons PAS (Différé)

- Intégration SWIBTIME
- Aléas (incidents de production)
- Opérations non liées à un BDC
- Formule de calcul de performance
- ~~Gestion de stock accessoires~~ <mark>MAINTENANT EN PÉRIMÈTRE — voir Fonctionnalité 5</mark>
- <mark>Stock Photo (snapshot hebdomadaire) — reporté à une phase future</mark>

### 7.3 Critères de Succès

1. La production textile peut fonctionner avec le workflow basé sur les talons
2. La production technique continue de fonctionner (transition fluide)
3. KW reçoit les fichiers XML requis
4. Le suivi des statuts BDC est préservé et précis
5. Les données de performance des opérateurs sont capturées pour les rapports
6. <mark>Les opérations ignorées sont visibles et gérables via la Vue Anomalies</mark>
7. <mark>Les fichiers sont envoyés à KW sans doublons</mark>

---

## Annexe A : Diagramme Entité-Relation

```
Famille ─────────────────────────────────────────┐
  │                                               │
  └── FamilyWorkflow (1:M)                        │
        │                                         │
        └── FamilyWorkflowStatus (1:M)            │
              │                                   │
              ├── Status (M:1) ←──────────────────┤
              │                                   │
              └── FamilyWorkflowStatusOperation   │
                    │                             │
                    ├── Operation (M:1)           │
                    │                             │
                    └── FamilyWorkflowOperationAccessory
                          │
                          └── Accessory (M:1)

Bdc ──────────────────────────────────────────────────────┐
  │                                                        │
  └── BdcWorkflow (1:1)                                    │
        │                                                  │
        └── BdcWorkflowOperation (1:M)                     │
              │                                            │
              ├── Operation (M:1)                          │
              ├── Status (M:1) ←───────────────────────────┤
              ├── User [CompletedBy] (M:1)                 │
              │                                            │
              └── BdcWorkflowOperationAccessory (1:M)      │
                    │                                      │
                    └── Accessory (M:1)                    │

[Spécifique KW]
Client (KW) ── NevImport (1:M) ── NevRoll (1:M) ── FabricRoll (1:1)
            ── ClientMovementTypeMapping (1:M)
            ── ClientFileSendLog (1:M)  [NOUVEAU]

Store ── StoreRodCounter (1:1)  [NOUVEAU]
```

---

## Annexe B : Synthèse des Modifications depuis la Spécification V2

Les éléments suivants sont nouveaux ou mis à jour par rapport à la spécification V1 :

| # | Modification | Section |
|---|-------------|---------|
| 1 | Le code-barre encode BdcWorkflowOperation.Id (clé primaire DB globalement unique) | FR-TAL (contenu talon) |
| 2 | L'ordre d'impression des talons suit la logique de balayage / lecture ligne par ligne | FR-TAL-01 |
| 3 | Les opérations ignorées NE bloquent PAS le statut dans le workflow textile | FR-TAL-04 |
| 4 | Opérations auto-complétées ignorées : PostId=NULL, CompletedAt=NULL | FR-TAL-04 |
| 5 | Vue Anomalies : EN PÉRIMÈTRE — visualiser/compléter manuellement les opérations ignorées + affecter opérateur | FR-TAL-04 |
| 6 | Plusieurs opérateurs par Poste ; identifiés par Matricule | FR-TAL-02, FR-TAL-05 |
| 7 | Personnel : entité User étendue avec Matricule, FirstName, LastName, IsActive | FR-TAL-05 |
| 8 | Le scannage d'un rouleau génère un fichier déclencheur .txt pour le Communicator | FR-KW-02 |
| 9 | La confrontation gère 2 cas : sur-réception ET sous-réception | FR-KW-02 |
| 10 | La réponse IC2S inclut le statut de suivi de réception | FR-KW-03 |
| 11 | Règle de déduplication : chaque fichier envoyé une seule fois par cycle | FR-KW-09 |
| 12 | Nouvelle table : ClientFileSendLog (suivi structuré des envois de fichiers) | FR-KW-09, schéma DB |
| 13 | Stock Photo : DIFFÉRÉ à une phase future | FR-KW-06 |
| 14 | Déclencheur RFID : vérification en code (famille suspendue + quantité tissu CalculatedSpecificField) — pas de Formula API | FR-RFID-01 |
| 15 | Compteur RFID : configurer le numéro de départ pour s'aligner avec le compteur Winsat | FR-RFID-02 |
| 16 | 2 étiquettes RFID imprimées si le nombre de cintres de l'article = 2 | FR-RFID-03 |
| 17 | VOWV : s'applique à TOUTES les commandes textiles non suspendues (pas seulement Stores Bateaux) | FR-KW-07 |
| 18 | Numéro de barre : compteur séquentiel par entité Store (pas global) | FR-KW-07 |
| 19 | <mark>Fonctionnalité 5 ajoutée : Gestion Stock Accessoires — stock principal/temporaire, réduction à la complétion, surconsommation, mouvements, export</mark> | FR-ACC-01 à 04 |
| 20 | <mark>Fonctionnalité 6 ajoutée : Facturation KW Textile — base de facturation = ImportedPrice, simulation inversée vs technique, blocage si prix absent</mark> | FR-INV-01 à 04 |

---

**FIN DU DOCUMENT**
