# Zola DTM

Ce projet prépare un corpus de romans d’Émile Zola en vue d’étudier l’évolution de leurs thèmes avec BERTopic.

Le travail est encore en cours : la préparation des données est terminée et la recherche d’hyperparamètres est en cours. Aucune configuration BERTopic n’a encore été retenue et l’analyse thématique finale n’a pas encore été réalisée.

## État du projet

- préparation et segmentation du corpus : terminées ;
- exploration statistique des segments : réalisée ;
- lemmatisation et structuration des données : terminées ;
- calcul des embeddings : réalisé et mis en cache ;
- recherche et comparaison des hyperparamètres : en cours ;
- choix du modèle final et analyse des topics : à venir.

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
│   │   ├── 01_paquets_128.csv
│   │   └── 02_corpus_zola_lematise_128.csv
│   ├── 3_optimisation/
│   ├── 4_resultats/
│   └── donnees_annex/
│       └── data_cache/
│           └── embeddings_sentence_camembert.npy
├── requirements.txt
└── README.md
```

Le notebook `02_exploration_modele.ipynb` et le dossier `data/4_resultats/` concernent la suite du projet. Ils ne sont pas documentés ici, puisque la configuration finale et l’analyse ne sont pas encore arrêtées.

## Installation

Le projet utilise Python 3.12.

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Sous Windows :

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Les notebooks peuvent ensuite être ouverts dans VS Code ou Jupyter. Si JupyterLab n’est pas déjà installé :

```bash
python -m pip install jupyterlab
jupyter lab
```

Il faut sélectionner le noyau Python associé à l’environnement `.venv`.

## Ordre d’exécution

Les notebooks sont prévus pour être exécutés dans cet ordre :

| Ordre | Notebook | Rôle |
|---:|---|---|
| 1 | [`01_segmentation.ipynb`](01_notebooks_préparation/01_segmentation.ipynb) | Charger les romans et les découper en segments compatibles avec CamemBERT |
| 2 | [`02_stat_exploratoire.ipynb`](01_notebooks_préparation/02_stat_exploratoire.ipynb) | Examiner la longueur des segments obtenus |
| 3 | [`03_preparation_donnees.ipynb`](01_notebooks_préparation/03_preparation_donnees.ipynb) | Structurer les métadonnées et lemmatiser les textes |
| 4 | [`01_recherche_hyperparametres.ipynb`](02_notebook_analyse/01_recherche_hyperparametres.ipynb) | Comparer plusieurs configurations BERTopic et mesurer leur stabilité |

Les chemins relatifs présents dans le code supposent que le répertoire de travail correspond au dossier du notebook exécuté. Par exemple, `01_recherche_hyperparametres.ipynb` doit être lancé avec `02_notebook_analyse/` comme répertoire de travail.

## 1. Segmentation du corpus

Le notebook `01_segmentation.ipynb` charge les fichiers `.txt` placés dans `data/1_raw/corpus_zola/`. Le corpus actuel contient **31 romans**.

La segmentation s’appuie sur le tokenizer du modèle :

```text
dangvantuan/sentence-camembert-base
```

Le modèle accepte au maximum 128 tokens. Le code réserve la place nécessaire aux tokens spéciaux, vise des segments d’environ 110 tokens de contenu et vérifie qu’aucun segment ne dépasse la limite du modèle. Les lignes vides sont ignorées. Lorsqu’une ligne est trop longue pour tenir seule dans un segment, elle est exceptionnellement découpée mot par mot.

Le fichier produit est :

```text
data/2_processed/01_paquets_128.csv
```

Il contient les colonnes suivantes :

| Colonne | Contenu |
|---|---|
| `nom_fichier` | Nom du fichier source |
| `id_paquet` | Position du segment dans le roman, à partir de 0 |
| `phrases_paquet` | Texte brut du segment |
| `nb_tokens_camembert` | Nombre de tokens, tokens spéciaux compris |

L’exécution actuelle produit **61 435 segments**. Leur longueur moyenne est d’environ **105 tokens CamemBERT**, avec un maximum vérifié de **128 tokens**.

## 2. Exploration statistique

Le notebook `02_stat_exploratoire.ipynb` recharge `01_paquets_128.csv` et calcule le nombre de mots de chaque segment avec un découpage sur les espaces.

Il affiche :

- les statistiques descriptives de la longueur des segments ;
- le nombre de segments contenant moins de 127 mots ;
- le nombre de segments contenant plus de 200 mots ;
- le nombre de segments contenant plus de 250 mots.

Cette étape est uniquement descriptive et ne crée pas de nouveau fichier.

## 3. Préparation et lemmatisation

Le notebook `03_preparation_donnees.ipynb` prépare les données avant la modélisation :

- renommage des colonnes issues de la segmentation ;
- extraction de l’année à partir du nom de fichier ;
- classement chronologique des romans ;
- création du nom du roman à partir du nom de fichier ;
- tri des segments par roman et renumérotation à partir de 1.

La lemmatisation utilise le modèle spaCy `fr_core_news_lg`. Le parser et la reconnaissance d’entités nommées ne sont pas chargés, car ils ne sont pas utilisés à cette étape.

Le texte lemmatisé conserve uniquement :

- les noms (`NOUN`) ;
- les adjectifs (`ADJ`) ;
- les verbes (`VERB`).

La ponctuation, les espaces, les nombres et les lemmes de moins de deux caractères sont écartés. Le code n’applique pas de filtre séparé aux stop words et ne supprime pas spécifiquement les entités nommées.

Trois segments ne contiennent plus aucun lemme après ce traitement et sont exclus. Le corpus préparé contient donc **61 432 segments**.

Le résultat est enregistré dans :

```text
data/2_processed/02_corpus_zola_lematise_128.csv
```

| Colonne | Contenu |
|---|---|
| `roman` | Nom du roman dérivé du fichier source |
| `annee` | Année extraite du nom de fichier |
| `ordre_romans` | Rang chronologique du roman |
| `paquet_id` | Position du segment dans le roman, à partir de 1 |
| `texte` | Texte brut du segment |
| `nb_tokens_camembert` | Nombre de tokens CamemBERT |
| `phrases_lemm` | Suite de lemmes utilisée par BERTopic |

## 4. Embeddings sémantiques

Le notebook de recherche d’hyperparamètres utilise également `dangvantuan/sentence-camembert-base` pour calculer les embeddings.

Les embeddings sont calculés à partir de la colonne de texte brut `texte`. La colonne lemmatisée `phrases_lemm` sert ensuite à construire la représentation lexicale des topics dans BERTopic.

Le cache est enregistré dans :

```text
data/donnees_annex/data_cache/embeddings_sentence_camembert.npy
```

S’il existe déjà, il est chargé directement. Sinon, les embeddings sont calculés par lots de 64 documents puis sauvegardés. Le cache actuel a une dimension de **61 432 × 768**.

Le notebook vérifie que le nombre d’embeddings correspond au nombre de lignes du corpus. Si le corpus, son ordre ou le modèle d’embeddings change, le fichier de cache doit être régénéré.

Une connexion internet peut être nécessaire lors de la première exécution pour télécharger le modèle Sentence-Transformers.

## 5. Recherche d’hyperparamètres BERTopic

Le notebook `01_recherche_hyperparametres.ipynb` compare actuellement **72 configurations**, obtenues avec la grille suivante :

| Composant | Paramètre | Valeurs testées |
|---|---|---|
| UMAP | `n_neighbors` | 40, 50, 60 |
| UMAP | `n_components` | 5, 10, 15, 20 |
| HDBSCAN | `min_cluster_size` | 184, 246, 307 |
| HDBSCAN | `min_samples` | 1, 2 |

Les paramètres fixes principaux sont :

- UMAP : `min_dist=0.0`, distance cosinus, graine aléatoire `42` ;
- HDBSCAN : distance euclidienne et sélection des clusters avec la méthode `eom` ;
- `CountVectorizer` : unigrammes, `min_df=2` et `max_df=0.8` ;
- c-TF-IDF : réduction des mots fréquents et pondération BM25 ;
- BERTopic : pas de réduction forcée du nombre de topics et pas de calcul des probabilités.

### Métriques calculées

Chaque configuration est évaluée avant toute réaffectation des outliers. Le notebook calcule :

- le nombre de topics ;
- le taux d’outliers et la couverture des documents ;
- le score de silhouette dans l’espace UMAP, sur un échantillon maximal de 4 000 documents ;
- le DBCV ;
- la cohérence thématique C_v ;
- la diversité lexicale des dix premiers mots de chaque topic ;
- les tailles minimale, médiane et maximale des topics ;
- la part occupée par le topic dominant ;
- la durée d’entraînement.

Les résultats sont sauvegardés après chaque configuration dans :

```text
data/3_optimisation/resultats_grille_bertopic.csv
```

Cette sauvegarde intermédiaire permet de conserver la progression d’une recherche coûteuse.

### Classement multicritère

Avant le classement, le notebook écarte les configurations qui :

- produisent moins de 10 topics ;
- classent plus de 75 % des documents comme outliers ;
- attribuent plus de 40 % des documents assignés au topic dominant.

Les configurations restantes reçoivent un score construit à partir des rangs percentiles :

| Critère | Poids |
|---|---:|
| Cohérence C_v | 30 % |
| Silhouette | 20 % |
| DBCV | 15 % |
| Diversité lexicale | 15 % |
| Couverture | 10 % |
| Équilibre des topics | 10 % |

### Stabilité

Le notebook prévoit ensuite de réentraîner jusqu’aux dix meilleures configurations avec les graines `11`, `30`, `42`, `57` et `73`.

La stabilité est mesurée avec l’Adjusted Rand Index, d’une part sur tous les documents et d’autre part sur les documents assignés dans les deux exécutions comparées. Le notebook suit également la part de documents assignés en commun, le nombre moyen de topics et le taux moyen d’outliers.

Les fichiers de classement et de stabilité sont destinés à être enregistrés dans :

```text
data/3_optimisation/classement_modeles_bertopic.csv
data/3_optimisation/stabilite_modeles_bertopic.csv
```

## Suite du projet

La recherche d’hyperparamètres doit encore être terminée et interprétée avant de choisir une configuration. Le nombre final de topics, la gestion des outliers et les analyses par roman ou dans le temps ne sont donc pas encore fixés.

Ces éléments seront documentés après la sélection et l’entraînement du modèle final.
