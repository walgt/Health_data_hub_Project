# 🏥 Projet — Health Data Hub

Analyse de qualité des données du projet de cohorte `PROJET_COHORTE_2026`.  
Ce notebook identifie et documente une anomalie critique d'attribution de `Patient_ID` entre la zone **SOURCE** et la zone **DELIVERY**.

---

## 📁 Structure du projet

```
PROJET_COHORTE_2026/
├── SITE_A/                         # Zone SOURCE
│   └── <dir_patient>/
│       └── <dir_visit>/
│           └── <FILE_ID>.json
│
PROJET_COHORTE_2026_delivery/
├── SITE_A/                         # Zone DELIVERY
│   └── <dir_patient>/
│       └── <dir_visit>/
│           └── <FILE_ID>.json
│
HDH_TEST.ipynb                      # Notebook d'analyse principal
README.md
```

### Format d'un fichier JSON

Chaque fichier JSON contient les champs suivants :

| Champ        | Description                          |
|-------------|--------------------------------------|
| `Patient_ID` | Identifiant unique du patient        |
| `VISIT_ID`   | Identifiant de la visite             |
| `FILE_ID`    | Identifiant unique du fichier        |
| `Category`   | Catégorie du document (`TYPE_A`, `TYPE_B`, `TYPE_C`) |

---

## ⚙️ Prérequis

- Python 3.8+
- [pandas](https://pandas.pydata.org/)

```bash
pip install pandas
```

---

## 🚀 Lancement

1. Cloner le dépôt et configurer les chemins dans le notebook :

```python
SOURCE_PATH   = r"C:\...\PROJET_COHORTE_2026\SITE_A"
DELIVERY_PATH = r"C:\...\PROJET_COHORTE_2026_delivery\SITE_A"
```

2. Exécuter le notebook cellule par cellule :

```bash
jupyter notebook HDH_TEST.ipynb
```

---

## 🔍 Démarche d'analyse

L'investigation suit une logique hypothético-déductive en 4 étapes.

### 1. Chargement des données

Parcours récursif de tous les fichiers `.json` via `Path.rglob()`.  
Chaque fichier est chargé dans un DataFrame pandas enrichi du chemin relatif (`dir_patient`, `dir_visit`).

```
Source   : 6 800 fichiers chargés
Delivery : 6 800 fichiers chargés
```

### 2. Profiling

| Métrique          | SOURCE | DELIVERY | Écart   |
|-------------------|--------|----------|---------|
| Fichiers total    | 6 800  | 6 800    | 0       |
| Patients uniques  | 101    | 81       | **-20** |
| Visites uniques   | 608    | 588      | **-20** |
| FILE_IDs uniques  | 6 800  | 6 800    | 0       |
| Doublons FILE_ID  | 0      | 0        | 0       |

> ⚠️ 20 patients disparaissent entre la source et la delivery, alors que le nombre de fichiers est identique.

### 3. Hypothèse 1 — Fichiers manquants ?

Comparaison des ensembles de `FILE_ID` entre les deux zones.

```
FILE_IDs présents en source mais absents en delivery : 0
FILE_IDs présents en delivery mais absents en source : 0
```

**→ Rejetée.** Aucun fichier n'est manquant. Le problème est une mauvaise classification.

### 4. Hypothèse 2 — Mauvaise attribution des Patient_ID ?

Identification des patients absents de la delivery et localisation de leurs fichiers.

```
Patients absents de la delivery              : 20
Fichiers source appartenant à ces patients   : 1 228
Fichiers retrouvés dans la delivery          : 1 228  (100 %)
Patient "gouffre" dans la delivery           : 1 seul (28342286687094591113440599999)
```

**→ Confirmée.** Les 1 228 fichiers ont tous été redirigés vers un unique patient.

---

## 🐛 Cause racine

La zone SOURCE contient **20 fichiers « intrus »** — des fichiers portant le `Patient_ID` du patient SINK (`28342…`) mais physiquement placés dans le dossier d'un autre patient.

Le script de transformation déduit le `Patient_ID` de l'ensemble du dossier à partir du **premier fichier rencontré**. Lorsque ce premier fichier est un intrus, tous les fichiers du dossier — toutes visites confondues — sont attribués au patient SINK.

```
Dossiers source contenant plusieurs Patient_IDs : 20
Fichiers intrus par dossier affecté             :  1
```

---

## 💥 Impact métier

| Impact                      | Détail                                                                 |
|----------------------------|------------------------------------------------------------------------|
| **20 patients fantômes**    | Absents de la delivery, introuvables pour les chercheurs               |
| **109 visites inaccessibles** | Non retrouvables sous le bon identifiant patient                     |
| **1 228 fichiers mal classés** | Regroupés sous un unique patient anormalement volumineux            |
| **Analyses biaisées**       | Toute requête filtrée par `Patient_ID` retourne des résultats faux    |

> ✅ Dans la delivery, la structure des dossiers est cohérente avec les JSON (`dir_patient == Patient_ID` pour 6 800/6 800 fichiers). Le problème réside dans les JSON eux-mêmes, dont le `Patient_ID` a été écrasé.

---

## ✅ Recommandations

### Court terme — Corriger la delivery

- Reclasser les 1 228 fichiers dans le bon dossier patient.
- Reconstruction possible à 100 % via la correspondance `FILE_ID` ↔ source.
- Vérifier après correction que `dir_patient == Patient_ID` pour tous les fichiers.

### Correction du script de transformation

```python
# ❌ Mauvais : déduire le Patient_ID à partir d'un seul fichier
patient_id = first_file["Patient_ID"]

# ✅ Correct : vérifier la cohérence de tous les fichiers du dossier
patient_ids = {f["Patient_ID"] for f in dossier_files}
if len(patient_ids) > 1:
    raise ValueError(f"Incohérence Patient_ID dans le dossier : {patient_ids}")
patient_id = patient_ids.pop()
```

### Contrôle qualité de la source

- Identifier et corriger les 20 fichiers intrus (mauvais dossier d'origine).
- Mettre en place une validation à l'ingestion : `Patient_ID` dans le JSON == nom du dossier parent.

---

## 📊 Résumé chiffré

| Observation                          | Valeur      |
|--------------------------------------|-------------|
| Fichiers total (source = delivery)   | 6 800       |
| Patients source                      | 101         |
| Patients delivery                    | 81          |
| Patients manquants                   | **20**      |
| Fichiers mal attribués               | **1 228**   |
| Patient « gouffre » dans delivery    | 1 seul      |
| Fichiers intrus dans la source       | **20**      |
| FILE_IDs perdus                      | **0**       |

---

## 📄 Licence

Usage interne — Health Data Hub, 2026.
