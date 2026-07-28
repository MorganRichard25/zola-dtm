# Zola DTM

Analyse thématique dynamique d'un corpus de romans d'Émile Zola avec BERTopic.

## Présentation

Ce projet construit un pipeline complet de modélisation thématique :

1. chargement des romans au format texte ;
2. segmentation du corpus en paquets ;
3. exploration statistique ;
4. préparation et lemmatisation des textes ;
5. recherche des meilleurs hyperparamètres BERTopic ;
6. entraînement et exploration du modèle final ;
7. analyse de la distribution des topics dans le temps et par roman.

Le corpus de travail actuel comprend **31 romans**, découpés en **5 327 paquets**. Le modèle final retenu produit **33 topics**.

## Structure du projet

```text
zola-dtm/
├── 01_notebooks_préparation/
│   ├── 01_segmentation.ipynb
│   ├── 02_stat_exploratoire.ipynb
│   └── 03_preparation_donnees.ipynb
├── 02_notebook_analyse/
│   ├── 01_recherche_hyperparametres.ipynb
│   └── 02_exploration_modele.ipynb
├── data/
│   ├── 1_raw/
│   │   └── corpus_zola/
│   ├── 2_processed/
│   ├── 3_optimisation/
│   ├── 4_resultats/
│   └── donnees_annex/
├── requirements.txt
└── README.md
```

## Installation

Le projet a été développé avec Python 3.12.

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Sous Windows, l'environnement virtuel peut être activé avec :

```powershell
.\.venv\Scripts\Activate.ps1
```

Les notebooks peuvent ensuite être ouverts dans VS Code ou dans une interface Jupyter. Si JupyterLab n'est pas déjà installé :

```bash
python -m pip install jupyterlab
jupyter lab
```

Sélectionner le noyau Python associé à `.venv`.

## Ordre d'exécution

Les notebooks doivent être exécutés dans l'ordre suivant :

| Ordre | Notebook | Rôle principal |
|---:|---|---|
| 1 | [`01_segmentation.ipynb`](01_notebooks_préparation/01_segmentation.ipynb) | Charger les fichiers texte et les segmenter en paquets de 40 lignes non vides |
| 2 | [`02_stat_exploratoire.ipynb`](01_notebooks_préparation/02_stat_exploratoire.ipynb) | Examiner la distribution du nombre de mots par paquet |
| 3 | [`03_preparation_donnees.ipynb`](01_notebooks_préparation/03_preparation_donnees.ipynb) | Structurer les métadonnées, nettoyer et lemmatiser les textes |
| 4 | [`01_recherche_hyperparametres.ipynb`](02_notebook_analyse/01_recherche_hyperparametres.ipynb) | Comparer les configurations BERTopic et sélectionner les paramètres |
| 5 | [`02_exploration_modele.ipynb`](02_notebook_analyse/02_exploration_modele.ipynb) | Entraîner le modèle final et analyser les topics |

Les chemins relatifs utilisés dans les notebooks supposent que le répertoire de travail du noyau correspond au dossier contenant le notebook. Avant l'exécution, vérifier ce point avec `Path.cwd()` et, si nécessaire, configurer le répertoire de travail dans Jupyter ou VS Code.

## Préparation du corpus

### 1. Segmentation

`01_segmentation.ipynb` lit les fichiers `.txt` placés dans `data/1_raw/corpus_zola/`. Les lignes vides sont supprimées, puis chaque roman est regroupé en paquets de 40 lignes non vides.

Le résultat est enregistré dans :

```text
data/2_processed/01_paquets_phrases.csv
```

### 2. Exploration statistique

`02_stat_exploratoire.ipynb` décrit la longueur des paquets. Dans le corpus actuel :

- 5 327 paquets sont analysés ;
- la moyenne est de 785 mots par paquet ;
- la médiane est de 753 mots ;
- les paquets très courts ou très longs sont rares et sont conservés.

Ce notebook est descriptif et ne produit pas de nouveau fichier.

### 3. Structuration et lemmatisation

`03_preparation_donnees.ipynb` :

- extrait l'année et le titre du roman à partir du nom de fichier ;
- ordonne les romans chronologiquement ;
- attribue un identifiant à chaque paquet ;
- calcule le nombre de mots ;
- lemmatise les textes avec `fr_core_news_lg` ;
- retire les stop words, la ponctuation, les nombres et les entités nommées ;
- conserve les noms et les adjectifs de plus de deux caractères.

Il produit successivement :

```text
data/2_processed/02_corpus_zola.csv
data/2_processed/03_corpus_lematise.csv
```

## Embeddings sémantiques

Les deux notebooks d'analyse utilisent le modèle Sentence-Transformers :

```text
dangvantuan/sentence-camembert-base
```

Les embeddings sont calculés sur la colonne `texte`, tandis que BERTopic construit ses représentations lexicales à partir de la colonne lemmatisée `phrases_lemm`.

Le cache partagé est enregistré dans :

```text
data/donnees_annex/embeddings_sentence_camembert.npy
```

Lorsqu'il existe, les deux notebooks chargent directement ce fichier. Dans le cas contraire, le premier notebook exécuté génère les embeddings puis les sauvegarde. Le cache actuel contient une matrice de dimension **5 327 × 768**.

Si le corpus, l'ordre de ses lignes ou le modèle d'embeddings change, supprimer ce fichier avant de relancer l'analyse afin d'éviter d'utiliser des représentations devenues obsolètes.

Une connexion internet peut être nécessaire lors de la première exécution pour télécharger le modèle Sentence-Transformers s'il n'est pas déjà présent dans le cache local.

## Optimisation de BERTopic

`01_recherche_hyperparametres.ipynb` teste 256 configurations issues de la grille suivante :

| Composant | Paramètre | Valeurs testées |
|---|---|---|
| UMAP | `n_neighbors` | 10, 25, 50, 60 |
| UMAP | `n_components` | 5, 10, 15, 20 |
| HDBSCAN | `min_cluster_size` | 20, 40, 80, 120 |
| HDBSCAN | `min_samples` | 1, 5, 10, 15 |

Les configurations sont évaluées avec plusieurs critères :

- cohérence thématique C_v ;
- score de silhouette dans l'espace UMAP ;
- DBCV ;
- diversité lexicale ;
- couverture des documents ;
- proportion du topic dominant ;
- stabilité entre plusieurs graines aléatoires, mesurée avec l'Adjusted Rand Index.

Les dix meilleures configurations sont réentraînées avec les graines `1`, `11`, `35`, `42`, `73` et `89`.

### Configuration retenue

```json
{
  "n_neighbors": 10,
  "n_components": 20,
  "min_cluster_size": 20,
  "min_samples": 5
}
```

Le modèle à 33 topics est sélectionné volontairement pour sa granularité interprétative. Il correspond à la deuxième configuration du classement final : la première produit seulement 8 topics, ce qui a été jugé trop général pour l'analyse du corpus.

Les paramètres retenus sont enregistrés dans :

```text
data/donnees_annex/meilleurs_parametres_bertopic.json
```

La recherche complète est coûteuse : elle entraîne 256 modèles pour la grille, puis 60 modèles supplémentaires pour l'analyse de stabilité. Les résultats de la grille sont sauvegardés après chaque configuration.

## Exploration du modèle final

`02_exploration_modele.ipynb` :

- recharge les paramètres et les embeddings ;
- entraîne le modèle BERTopic final ;
- compare plusieurs seuils de réaffectation des outliers ;
- retient un seuil final de `0.30` ;
- affine la représentation lexicale avec une liste de stop words propre au corpus ;
- calcule la cohérence et la diversité finales ;
- construit une hiérarchie des topics ;
- étudie leur évolution temporelle ;
- calcule leur fréquence et leur proportion dans chaque roman.

### Résultats de l'exécution actuelle

| Indicateur | Valeur |
|---|---:|
| Documents analysés | 5 327 |
| Romans | 31 |
| Topics finaux | 33 |
| Taux brut d'outliers | 55,6 % |
| Taux d'outliers après réaffectation | 0,0 % |
| Cohérence C_v finale | 0,471 |
| Diversité lexicale finale | 0,915 |

Ces valeurs correspondent à l'exécution actuellement enregistrée. Elles peuvent évoluer si le corpus, les dépendances, les paramètres ou les règles de prétraitement sont modifiés.

## Fichiers produits

### Données préparées

| Fichier | Contenu |
|---|---|
| `data/2_processed/01_paquets_phrases.csv` | Paquets issus de la segmentation des romans |
| `data/2_processed/02_corpus_zola.csv` | Corpus structuré avec les métadonnées et le texte brut |
| `data/2_processed/03_corpus_lematise.csv` | Corpus enrichi avec la colonne `phrases_lemm` |

### Optimisation

| Fichier | Contenu |
|---|---|
| `data/3_optimisation/resultats_grille_bertopic.csv` | Résultats bruts des 256 configurations |
| `data/3_optimisation/classement_modeles_bertopic.csv` | Configurations filtrées et classées par score multicritère |
| `data/3_optimisation/stabilite_modeles_bertopic.csv` | Stabilité des dix meilleures configurations |

### Résultats finaux

| Fichier | Contenu |
|---|---|
| `data/4_resultats/documents_avec_topics.csv` | Topic attribué à chaque paquet du corpus |
| `data/4_resultats/informations_topics.csv` | Taille, nom, représentation et documents représentatifs de chaque topic |
| `data/4_resultats/topics_par_roman_comptages.csv` | Nombre de paquets par topic et par roman |
| `data/4_resultats/topics_par_roman_proportions.csv` | Proportion des topics dans chaque roman |
| `data/4_resultats/bertopic_topics_per_class.csv` | Représentation BERTopic des topics pour chaque roman |

## Reproductibilité

- La graine principale utilisée par UMAP est `42`.
- Les embeddings sont mis en cache pour éviter leur recalcul.
- Le nombre de lignes du cache est vérifié avant la modélisation.
- Les fichiers de résultats sont enregistrés en UTF-8.
- Toute modification du corpus ou du modèle Sentence-Transformers nécessite la régénération du cache d'embeddings.
