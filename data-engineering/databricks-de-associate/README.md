# Databricks Certified Data Engineer Associate — Notes de révision

> Notes personnelles de prép (exam : 4 août 2026). Rédigées avec mon COS.
> Format markdown compatible GitHub + Obsidian.
> **Organisation : 1 section = 1 domaine de l'exam guide**, avec son poids. Les points sans contenu sont laissés vides.

| # | Domaine | Poids | État des notes |
|---|---|---|---|
| 1 | Databricks Intelligence Platform | 6 % | ✅ |
| 2 | Data Ingestion and Loading | 21 % | ✅ |
| 3 | Data Transformation and Modeling | 22 % | ✅ |
| 4 | Working with Lakeflow Jobs | 16 % | ✅ |
| 5 | Implementing CI/CD | 10 % | ⬜ **vide — à écrire** |
| 6 | Troubleshooting, Monitoring, and Optimization | 10 % | ✅ |
| 7 | Governance and Security | 15 % | ✅ |

---

# 1. Databricks Intelligence Platform — 6 %

## 1.1 La pile Lakeflow — comment tout s'empile
L'ordre de construction, de bas en haut :
1. **Couche de stockage optimisée** → **Delta Lake**, Parquet, ou **Iceberg**.
2. **Gouvernance unifiée** → **Unity Catalog**, bâti *par-dessus* la couche de stockage.
3. **Lakeflow** → la solution data engineering **end-to-end** que Databricks pose au-dessus.

**Les 4 briques de la plateforme :**
| Brique | Quoi |
|---|---|
| **Storage layer** | Delta Lake / Iceberg |
| **Governance** | Unity Catalog |
| **Processing engine** | Photon (+ Spark & Structured Streaming) |
| **Orchestration** | Lakeflow Jobs = Workflows |

## 1.2 Les composants de Databricks Lakeflow
Lakeflow **unifie les activités de data engineering** en 4 composants :
- **Connectors** → **Lakeflow Connect** (l'*ingest layer*) — voir §2.5
- **Pipelines** → **Spark Declarative Pipelines** (ex-DLT)
- **Jobs** → **Lakeflow Jobs** — voir §4
- **Processing Engine** → **Spark + Structured Streaming** (accéléré par Photon)

## 1.3 Delta Lake — le protocole de stockage
- **Protocole de stockage open source** (pas un format à part).
- **Stocke les données en fichiers Parquet**, + un **transaction log**.
- Ce que le transaction log apporte : **ACID**, **time travel**, **DML** (UPDATE/DELETE/MERGE), **schema enforcement**.

## 1.4 Types de compute
| Type | Où tourne le compute | Note |
|---|---|---|
| **Interactive / All-purpose clusters** | ton compte cloud | notebooks, exploration, multi-utilisateurs |
| **Job clusters** | ton compte cloud | dédiés à un job, éphémères — **≈ 50 % moins chers** que l'all-purpose (source : Databricks Academy) |
| **Serverless** | **managé par Databricks** | démarrage quasi instantané, pas d'infra à gérer |
| **SQL Warehouse** | | compute dédié aux requêtes SQL / BI |

**Serverless — « performance optimized feature »**
- Disponible **uniquement sur le serverless**.
- Permet un **démarrage de job et une exécution plus rapides**.
- À activer pour les **workloads sensibles au temps**.
- **Photon est activé par défaut** sur le serverless.

## 1.5 Architecture d'exécution Spark
- **Driver** : porte la `SparkSession` / `SparkContext`, construit le **DAG**, planifie et distribue les tasks.
- **Executors** : exécutent les tasks et stockent les données cachées.
- **Cluster manager** : alloue les ressources (sur Databricks, couche managée au-dessus du cloud provider).

**Hiérarchie d'exécution — à connaître par cœur :**
`Action → Job → Stages → Tasks`
- une **action** déclenche un **job** ;
- le job est découpé en **stages**, séparés par les **frontières de shuffle** ;
- chaque stage est découpé en **tasks** : **1 task = 1 partition**.

## 1.6 Les APIs : RDD / DataFrame / Dataset
| API | Quoi | Note |
|---|---|---|
| **RDD** | collection distribuée immuable d'objets, **sans schéma**, lazy | API bas niveau, rarement le bon choix aujourd'hui |
| **DataFrame** | collection distribuée **organisée en colonnes nommées** | optimisée par **Catalyst** + **Tungsten** |
| **Dataset** | typage RDD + optimisation DataFrame | **Scala/Java uniquement** — en Python on n'a que le DataFrame |

## 1.7 Modèle de coût — Compute classique vs Serverless
**Classique = 3 couches de coût :**
- **Coûts directs** : DBUs (Databricks) **+** infra facturée par le cloud provider (VMs, réseau, sécurité).
- **Overhead opérationnel** (souvent oublié) : déploiement infra, automatisation, maintenance, monitoring des coûts, optimisation.
- **Complexité cachée** : gérer plusieurs relations de facturation, optimiser entre catégories de coûts, maintenir l'expertise infra cloud.

**Serverless = modèle simplifié :**
- **Facturation unifiée** : un seul prix DBU qui **inclut infra + ops** → plus de multi-fournisseurs à gérer.
- **Proposition de valeur** : simplicité + fiabilité justifient souvent un **coût unitaire plus élevé**.
- **Performance** : auto-scaling + optimisations natives → souvent meilleure perf à coût total moindre que le self-managed.
- **TCO** : en intégrant l'overhead opérationnel, le **coût total de possession est généralement plus bas** avec serverless — surtout sans équipe plateforme dédiée.

🧠 **Modèle mental :** *classique = moins cher/unité mais tu gères tout · serverless = plus cher/unité mais Databricks gère infra+ops → TCO souvent plus bas.*

## 1.8 Portabilité — Open source (Spark natif) vs Databricks-only
**Structured Streaming est natif à Apache Spark (open source, depuis Spark 2.0 / 2016)** — pas exclusif à Databricks. Databricks ajoute une couche propriétaire par-dessus = c'est là qu'est le lock-in.

| Élément | Open source (Spark natif) | Databricks-only |
|---|---|---|
| **Structured Streaming** (API, triggers, checkpoint, watermarks) | ✅ | |
| `trigger(availableNow / processingTime / continuous)` | ✅ | |
| Sources Kafka / file / socket, `foreachBatch` | ✅ | |
| **Delta Lake** core (ACID, time travel, `MERGE`, `OPTIMIZE`, `ZORDER`, `VACUUM`) | ✅ (projet Delta Lake, Linux Foundation — via `delta-spark`) | |
| **Auto Loader** (`cloudFiles`) | | ❌ propriétaire |
| **`COPY INTO`** | | ❌ commande Databricks |
| **Delta Live Tables** + `APPLY CHANGES INTO` | | ❌ propriétaire |
| **Photon**, Liquid Clustering, Predictive Optimization | | ❌ propriétaire |
| **Unity Catalog** | version OSS depuis 2024 | gouvernance complète = Databricks |

**Takeaway d'architecte (multi-cloud) :**
- Portable / sans lock-in : **Structured Streaming + Delta Lake** → réplicable sur n'importe quel cloud (EMR, Dataproc, self-managed).
- Lock-in Databricks : Auto Loader, DLT, COPY INTO, Photon → gains de productivité réels mais attachés à la plateforme.

⚠️ **Piège d'examen :** Auto Loader et DLT ne sont **pas** du Spark natif (beaucoup les croient standards). Sur l'exam Databricks on suppose l'environnement Databricks, donc ils sont « disponibles » ; en archi multi-cloud réelle, c'est une décision.

---

# 2. Data Ingestion and Loading — 21 %

## 2.1 Lire des fichiers depuis SQL — les 5 façons
| Syntaxe | Quoi |
|---|---|
| `SELECT * FROM read_files('path', ...)` | lecture de fichiers en table-valued function — **la voie moderne** |
| `COPY INTO table FROM 'path' FILEFORMAT = …` | chargement idempotent dans une table |
| `SELECT * FROM parquet.\`path\`` | lecture directe via le préfixe de format (`parquet.`, `json.`, `csv.`, `text.`, `binaryFile.`) |
| `spark.read...` | équivalent Python / DataFrame API |
| `LIST 'path'` | **lister** les fichiers d'un chemin (pas les lire) — utile pour inspecter avant d'ingérer |

## 2.2 Ingestion depuis le cloud storage — les 2 chemins
```
Batch ──────────┬─→ spark.read
                └─→ SQL CTAS

Incrémental ────┬─→ COPY INTO                     (legacy)
                └─→ Auto Loader (Python et SQL)
                      ├─ readStream / writeStream
                      └─ CREATE OR REFRESH STREAMING TABLE
```

⭐ **Recommandation Databricks (point d'examen) :**
- Utiliser les **streaming tables** pour ingérer en **SQL**, **plutôt que `COPY INTO`**.
- Une **streaming table est enregistrée dans Unity Catalog**, ce qui apporte un **support supplémentaire pour le streaming**.
- Pour créer une **streaming table à partir de fichiers dans un volume → on utilise Auto Loader**.

## 2.3 Streaming vs Batch incrémental — la règle d'or
**La latence exigée par le métier commande le choix. Le reste suit.**
Le streaming est un choix de **coût et de complexité**, pas un défaut. La plupart des « besoins temps réel » sont couverts par du **micro-batch fréquent**.

- Décision en **secondes / sous-minute** (fraude, alerting, IoT, live) → **streaming**
- Fraîcheur en **minutes → heures** (BI, reporting, ML batch) → **batch incrémental**

### Les 5 critères qui tranchent
| Critère | Batch incrémental | Streaming continu |
|---|---|---|
| Latence exigée | minutes → heures | secondes → sous-minute |
| Arrivée des données | fichiers/lots périodiques | flux continu (Kafka, Kinesis, Event Hub) |
| Coût | cluster démarre → traite → s'éteint = **moins cher** | cluster **always-on** = cher |
| Complexité ops | simple, peu d'état | checkpointing, watermarks, état, monitoring 24/7 |
| Opérations avec état | agrégations simples | fenêtres, joins de flux, dédup temporelle |

### Le point clé Databricks : même code, trigger différent
Sur Databricks c'est souvent le **même code Structured Streaming** — seul le `trigger` change :

| Trigger | Comportement | Usage |
|---|---|---|
| `trigger(availableNow=True)` | traite tout le dispo **puis s'arrête** | **batch incrémental** sur infra streaming (le meilleur des 2 mondes : incrémental + coût batch) |
| `trigger(processingTime="1 minute")` | micro-batch récurrent | quasi temps réel |
| `trigger(continuous=...)` | vrai streaming continu | rare, latence ultra-basse |

### Le réflexe d'architecte
1. **Par défaut** : batch incrémental (`Auto Loader` + `availableNow`, ou `COPY INTO`).
2. **Streaming continu SEULEMENT si** : SLA sub-minute réel **ET** source vraiment streaming **ET** coût always-on justifié.
3. Entre les deux : **micro-batch** couvre la majorité des cas.

## 2.4 Outils d'ingestion incrémentale
| Outil | Quoi | Quand |
|---|---|---|
| **Auto Loader** (`cloudFiles`) | ingestion **incrémentale de fichiers** (nouveaux seulement), suit ce qui a déjà été lu | volumes de fichiers **grands/continus** (millions), scalable ; batch OU streaming |
| **`COPY INTO`** | commande SQL **idempotente** (ignore les fichiers déjà chargés) | volumes **petits/moyens**, chargements périodiques |
| **DLT** (Delta Live Tables) | pipelines déclaratifs, streaming ou triggered | pipelines managés avec qualité/monitoring intégrés |

- **`COPY INTO` est idempotent** : réexécuter ne recharge pas les fichiers déjà traités.
- **Auto Loader** > COPY INTO quand le nombre de fichiers explose (suivi via checkpoint/RocksDB, pas de re-listing coûteux).

## 2.5 Lakeflow Connect — l'ingest layer
- **Lakeflow Connect** = le point d'entrée des données dans Databricks. Connecte une grande variété de sources :
  - **Cloud object stores** : S3, ADLS, GCS
  - **Message queues / streaming** : Kafka, Pub/Sub, Kinesis
  - **Bases traditionnelles** : SQL Server, Postgres
  - **Apps SaaS** : Salesforce, Workday
- **3 types de connecteurs Lakeflow Connect** :
  - **Manual File Uploads** : uploader rapidement des fichiers locaux dans des volumes ou tables.
  - **Standard Connectors** : ingestion depuis cloud storage, Kafka… en mode **batch, incrémental ou streaming**.
  - **Managed Connectors** : pour les **applications d'entreprise et bases de données** → ingestion incrémentale **scalable et efficace** dans le lakehouse.
  - ➡️ **Standard vs Managed — comment trancher : Annexe A.5.**
- **3 méthodes d'ingestion** (transverses aux connecteurs) : **Batch** · **Incremental batch** · **Streaming**.
- **Spark Declarative Pipelines** (évolution déclarative type DLT) = ingestion + transformation, pour bâtir des pipelines **medallion** : **bronze → silver → gold**, avec fiabilité et scalabilité.
- 🧠 **Modèle mental :** *Lakeflow Connect fait entrer la donnée → Spark Declarative Pipelines la fait progresser bronze→silver→gold.*

## 2.6 Managed Ingestion & Ingestion Gateway Pipeline
**Managed Connectors → Managed Ingestion** : le mode « clé en main » de Lakeflow Connect pour les **bases de données et applications d'entreprise**.

**L'`Ingestion Gateway Pipeline` fait 3 choses :**
1. **Se connecte à la base source**.
2. **Extrait** les **métadonnées**, les **snapshots** et les **change logs** (CDC).
3. **Les stocke dans un volume Unity Catalog**.

**Le flux complet (sources externes → tables Delta) :**
```
Sources externes ──credentials──→ Unity Catalog
        │                              │
        │  ① credentials enregistrés dans UC
        ▼
      Données ──② ──→ Ingestion Gateway / Pipeline
                              │
                              ▼
                      Staging / State management
                              │
                              ③ Managed Ingestion
                              ▼
                    Streaming Delta tables
```
🧠 *UC détient les credentials · la gateway extrait et dépose en staging · la managed ingestion matérialise en streaming Delta tables.*

## 2.7 Checkpoint — le cœur de l'incrémental
- **Checkpoint** : stocke offsets + état → **reprise après panne + exactly-once**. Indispensable au streaming.
- Le supprimer = **tout retraiter**.

## 2.8 Rescued data column — récupérer ce qui ne rentre pas dans le schéma
- **Quand** : la donnée brute **ne correspond pas au schéma** attendu (type incompatible, colonne inconnue, casse différente).
- **Ce que ça fait** : au lieu de perdre la donnée ou de faire échouer le job, les valeurs fautives sont **rangées dans une colonne dédiée** (`_rescued_data`, en JSON).
- **Disponible avec les 3 voies de lecture :**
  - `read_files()`
  - `spark.read`
  - **Auto Loader**

## 2.9 Schema evolution à l'ingestion (Auto Loader)
- `.option("cloudFiles.schemaEvolutionMode", "addNewColumns")` + `cloudFiles.schemaLocation`
- Modes : `addNewColumns` (défaut), `rescue`, `failOnNewColumns`, `none`.
- ➡️ Vue complète des 3 moments d'évolution de schéma : **§3.7**.

## 2.10 Modes de sortie Structured Streaming
| Mode | Écrit quoi | Quand |
|---|---|---|
| **Append** | uniquement les **nouvelles lignes** | défaut ; pas d'agrégation, ou agrégation avec watermark |
| **Complete** | **toute la table de résultat** à chaque trigger | agrégations sans watermark |
| **Update** | seulement les lignes **modifiées** depuis le dernier trigger | agrégations incrémentales |

## 2.11 Exemple — Ingestion incrémentale en Spark natif (sans Auto Loader)
Reproduire le pattern « traiter tous les nouveaux fichiers puis s'arrêter, sans retraiter les anciens » en **Spark open source** = 3 briques : **file source Structured Streaming + `trigger(availableNow=True)` + `checkpointLocation`**.

```python
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, TimestampType

spark = SparkSession.builder.appName("incremental-file-ingest").getOrCreate()

# 1. Schéma EXPLICITE — le file source natif n'infère pas (contrairement à Auto Loader)
schema = StructType([
    StructField("id",    StringType()),
    StructField("event", StringType()),
    StructField("ts",    TimestampType()),
])

# 2. Lecture streaming depuis un dossier — le checkpoint suit les fichiers déjà vus
df = (spark.readStream
        .schema(schema)
        .format("json")                 # ou "csv", "parquet"
        .load("/path/to/landing/"))

# (transformations éventuelles ici)

# 3. availableNow = traite tout le dispo, puis STOP
query = (df.writeStream
           .format("parquet")           # ou "delta" via le package delta-spark
           .option("checkpointLocation", "/path/to/checkpoint/")  # <- la clé de l'incrémental
           .trigger(availableNow=True)
           .outputMode("append")
           .start("/path/to/bronze/"))

query.awaitTermination()
```

**Les points clés :**
- `checkpointLocation` = le suivi incrémental (mémorise les fichiers déjà lus dans `sources/0/`). Le supprimer = tout retraiter.
- `trigger(availableNow=True)` = traite puis s'arrête (Spark 3.3+ ; avant : `trigger(once=True)`).
- `.schema(...)` **obligatoire** pour un file source natif (Auto Loader, lui, infère + fait évoluer le schéma).
- **Batch incrémental** = planifier ce job (cron / Airflow / `spark-submit`) : chaque run ingère les nouveaux fichiers et s'éteint. Pas de cluster always-on.

**Équivalent Auto Loader (Databricks) pour comparaison :**
```python
df = (spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .option("cloudFiles.schemaLocation", "/path/schema/")   # infère + évolue le schéma
        .load("/path/landing/"))

(df.writeStream
   .option("checkpointLocation", "/path/checkpoint/")
   .trigger(availableNow=True)
   .toTable("bronze"))
```

| | Spark natif (file source) | Auto Loader (Databricks) |
|---|---|---|
| Suivi incrémental | checkpoint | checkpoint + RocksDB |
| Découverte de fichiers | listing du répertoire (coûteux à très grande échelle) | file notifications / listing optimisé (scalable, millions) |
| Schéma | manuel | inférence + évolution auto |
| Portabilité | ✅ tout cloud | ❌ Databricks-only |

Delta en open source : package `delta-spark` (`--packages io.delta:delta-spark_2.12:3.2.0` + configs `DeltaSparkSessionExtension` / `DeltaCatalog`).

---

# 3. Data Transformation and Modeling — 22 %

## 3.1 Lazy evaluation
- Les **transformations sont enregistrées, pas exécutées**. Rien ne part tant qu'une **action** ne le déclenche.
- ⚠️ **Piège d'examen :** `createOrReplaceTempView()` **n'est pas une action** → ne déclenche aucun job.

## 3.2 Narrow vs Wide transformations
| | Narrow | Wide |
|---|---|---|
| Shuffle | **non** | **oui** |
| Règle | 1 partition d'entrée → au plus 1 partition de sortie | plusieurs partitions d'entrée combinées |
| Exemples | `filter()`, `map()`, `select()`, `withColumn()` | `groupBy()`, `join()`, `distinct()`, `orderBy()` |

🧠 *Le shuffle est la frontière de stage : compter les wide transformations = compter les stages.*

**Actions :** `collect()` · `count()` · `show()` · `take(n)` · `write()` · `first()` · `foreach()`
⚠️ **`collect()` rapatrie TOUT sur le driver** → OOM sur gros volume. Préférer `take(n)`, `show()`, ou écrire vers le stockage.

## 3.3 ELT avec Spark SQL / Python
- **`CREATE TABLE AS SELECT` (CTAS)** : échoue si la table existe. `CREATE OR REPLACE TABLE` : écrase.
- **Vues** : `TEMP VIEW` = portée SparkSession/notebook · `GLOBAL TEMP VIEW` = cross-session dans le cluster · vue standard = persistée dans le metastore.
- **Accès données** : `struct.champ` (point) pour un STRUCT · `colonne:champ` (deux-points) pour du JSON/semi-structuré (VARIANT). ⚠️ piège classique.
- **Dédup** : `DISTINCT`, ou `DROP DUPLICATES`, ou `MERGE` avec condition.

## 3.4 Travailler avec du JSON

### Les 3 façons de stocker du JSON
| Type | Caractéristique | Verdict |
|---|---|---|
| **STRING** | le JSON reste du texte brut | ❌ **le moins performant** |
| **STRUCT** | **impose le schéma JSON** (schema enforcement) | ✅ **plus efficace** — bon si le schéma est stable |
| **VARIANT** | **très flexible**, **performance améliorée** vs les méthodes existantes ; peut stocker **n'importe quel type de données, JSON inclus** | ⭐ **idéal pour le semi-structuré** |

### Les fonctions
| Fonction | Ce qu'elle fait |
|---|---|
| `schema_of_json()` | **retourne la structure** (le type STRUCT) d'une chaîne au format JSON |
| `from_json()` | **applique un STRUCT** à une chaîne JSON → colonne typée |
| `parse_json()` | convertit une **chaîne JSON → VARIANT** |

### Recette — convertir une chaîne JSON en colonne STRUCT (2 étapes)
1. **Obtenir le type STRUCT** de la chaîne JSON avec **`schema_of_json()`**.
2. **Appliquer ce STRUCT** à la chaîne JSON avec **`from_json()`**.

🧠 **Modèle mental :** *`schema_of_json` **découvre** le schéma → `from_json` l'**applique** → STRUCT. Si je ne veux pas figer de schéma : `parse_json` → VARIANT.*

## 3.5 Modélisation Delta — tables & time travel
- **Table MANAGED** : `DROP TABLE` supprime **métadonnées + fichiers de données**.
- **Table EXTERNAL/unmanaged** : `DROP TABLE` supprime **seulement les métadonnées** (données conservées).
- **Time travel** : `VERSION AS OF` / `TIMESTAMP AS OF` (rendu possible par le transaction log).
- **Medallion** : bronze (brut) → silver (nettoyé/conformé) → gold (agrégé/métier).

## 3.6 Upsert & CDC
- **`MERGE INTO`** : upsert transactionnel classique (hors DLT).
- **DLT CDC / upsert** : **`APPLY CHANGES INTO`** (gère les changements d'un change feed, ordonnancement par séquence).

## 3.7 Schema Evolution — laisser Delta ajouter les nouvelles colonnes à ta place
**Concept transverse :** accepter automatiquement de **nouvelles colonnes** venant de la source, sans `ALTER TABLE` manuel. Le schéma peut évoluer à **3 moments** :

| Moment | Comment l'activer | Portabilité |
|---|---|---|
| **MERGE** ⭐ | `MERGE WITH SCHEMA EVOLUTION INTO cible USING source ON …` → ajoute les colonnes nouvelles de la source ; marche avec `UPDATE SET *` / `INSERT *`. Global (alt.) : `SET spark.databricks.delta.schema.autoMerge.enabled = true` | Databricks (clause `WITH SCHEMA EVOLUTION` = DBR 15.2+) |
| **Écriture** (append / overwrite) | `.option("mergeSchema", "true")` = ajoute des colonnes · `.option("overwriteSchema", "true")` = remplace tout le schéma | Delta OSS ✅ |
| **Ingestion** (Auto Loader) | `.option("cloudFiles.schemaEvolutionMode", "addNewColumns")` + `cloudFiles.schemaLocation` (modes : `addNewColumns` défaut, `rescue`, `failOnNewColumns`, `none`) | Databricks |

🧠 **Modèle mental :** *« Schema evolution = Delta ajoute les colonnes nouvelles à ma place, à 3 moments : quand je MERGE, quand j'écris, quand j'ingère. »*
⚠️ Ça **ajoute** des colonnes ; ça ne gère pas tous les **changements de type**. À utiliser sciemment (une colonne parasite en amont se propage dans la table).

## 3.8 Qualité dans les pipelines déclaratifs
- **DLT expectations** : contraintes qualité — `EXPECT` (log seulement), `EXPECT OR DROP` (écarte la ligne), `EXPECT OR FAIL` (fait échouer le pipeline).

---

# 4. Working with Lakeflow Jobs — 16 %

## 4.1 Ce qu'est un Lakeflow Job
**Jobs = Workflows** = le **composant d'orchestration** de la plateforme.

**Les 4 dimensions d'un job :**
| Dimension | Quoi |
|---|---|
| **Control flow** | la gestion des tâches et de leur enchaînement |
| **Compute** | quel cluster exécute quoi (voir §4.4) |
| **Observability** | suivi des exécutions, logs, alertes |
| **Workflow** | la définition du pipeline de tâches lui-même |

## 4.2 Le DAG — la structure d'un workflow
**D**irected — pas de direction ambiguë · **A**cyclic — **ne contient aucun cycle** · **G**raph.

## 4.3 Orchestration multi-tâches
- Dépendance : déclarer tâche A comme **« depends on »** de B → B ne part qu'après succès de A.

**Les 4 types d'orchestration :**
| Type | Quoi |
|---|---|
| **Sequential** | les tâches s'enchaînent l'une après l'autre |
| **Parallel** | les tâches partent en même temps |
| **Conditional** | branchement selon une condition (`If/else`) |
| **For each** | itère la même tâche sur une liste d'entrées |

## 4.4 Compute par tâche
- Un job contient **une ou plusieurs tâches**, et **chaque tâche peut avoir son propre compute**.
- Les tâches d'un même job peuvent **partager le même cluster** ou **utiliser des computes différents** selon le besoin.
- Types de compute disponibles → voir **§1.4**. Pour un job : **job cluster** (le moins cher) ou **serverless** (démarrage le plus rapide, Photon par défaut).

## 4.5 Déclencheurs de job
- **Scheduled (cron)** · **Continuous** · **Manual** · **File arrival** · **Table update**.
- **`table update` trigger** : le job se lance quand des tables sont mises à jour. Jusqu'à **10 tables par trigger**. Fonctionne avec les tables **Delta managées par Unity Catalog**, **Iceberg**, **Delta externes**, **materialized views** et **streaming tables**.

## 4.6 Paramètres vs Task Values ⭐
La distinction la plus piégeuse du domaine :

| | **Paramètres** | **Task values** |
|---|---|---|
| Créés **quand** | **avant** l'exécution | **pendant** l'exécution |
| Sert à | passer une config au job/à la tâche | **partager une valeur entre tâches** pendant le run |
| Qui gagne | ⚠️ **les paramètres de JOB écrasent toujours les paramètres de TÂCHE** quand la même clé existe | — |

**API Task values :**
```python
# écrire une valeur (tâche amont)
dbutils.jobs.taskValues.set(key="...", value=...)

# lire une valeur (tâche aval)
dbutils.jobs.taskValues.get(taskKey="...", key="...")
```

**Dynamic Value References** = **paramètres intégrés** exposant les **infos d'exécution** (ex. id du run, date de début, nom de la tâche) — pas besoin de les définir soi-même.

🧠 **Modèle mental :** *param = ce que je fixe avant de lancer · task value = ce qu'une tâche calcule et transmet à la suivante.*

## 4.7 Fiabilité
- **Retries**, scheduling et **alertes** intégrés aux Jobs.

---

# 5. Implementing CI/CD — 10 %

⬜ **Section vide — aucun élément dans mes notes actuelles.**

*À couvrir (d'après l'intitulé du domaine) :* Databricks Asset Bundles, Repos / Git folders, gestion des environnements dev→staging→prod, déploiement de jobs et pipelines, tests.

---

# 6. Troubleshooting, Monitoring, and Optimization — 10 %

## 6.1 Catalyst Optimizer — les 4 phases
1. **Analyse** — résout les références (colonnes, tables) contre le catalogue.
2. **Optimisation logique** — *predicate pushdown*, *constant folding*, *projection pruning*.
3. **Planification physique** — choix des stratégies (ex. type de join : broadcast hash vs sort-merge).
4. **Génération de code** — bytecode Java via **Tungsten** (whole-stage codegen).

## 6.2 Partitionnement & défauts de config
| Config | Défaut | Ce que ça fait |
|---|---|---|
| `spark.sql.files.maxPartitionBytes` | **128 MB** | taille cible d'une partition **à la lecture de fichiers** |
| `spark.sql.shuffle.partitions` | **200** | nb de partitions **post-shuffle** uniquement ⚠️ voir nuance ↓ |
| `spark.sql.autoBroadcastJoinThreshold` | **10 MB** (10 485 760 o) | seuil auto de broadcast join ; `-1` = désactivé |
| `spark.sql.adaptive.enabled` | **true** (Spark 3.2+ ; DBR 7.3+) | active l'AQE |

⚠️ **Nuance Databricks sur `shuffle.partitions` :** 200 est le défaut **Spark OSS**. Sur Databricks, **AQE est activé par défaut depuis DBR 7.3** et **recoalesce dynamiquement** les partitions post-shuffle à chaque stage — le 200 n'est plus qu'une valeur *initiale*. On peut aussi poser `spark.sql.shuffle.partitions = auto` (*auto-optimized shuffle*) pour laisser Databricks la déterminer selon le plan et la taille des données.

**`repartition(n)` vs `coalesce(n)`**
- `repartition(n)` : **full shuffle**, donne exactement *n* partitions équilibrées (peut augmenter ou diminuer).
- `coalesce(n)` : **réduit** seulement, **sans full shuffle** (fusionne des partitions) → moins cher mais **tailles inégales** possibles.

## 6.3 Caching
- `cache()` / `persist()` matérialisent un DataFrame intermédiaire réutilisé plusieurs fois. `unpersist()` pour libérer.
- ⚠️ **Défaut selon l'API** : **DataFrame** `.cache()` → **`MEMORY_AND_DISK`** (en Spark 3.x : `MEMORY_AND_DISK_DESER`) · **RDD** `.cache()` → **`MEMORY_ONLY`**. Piège classique : on retient « MEMORY_AND_DISK » pour tout.
- Ne cacher que si **réutilisation réelle** : sinon c'est de la mémoire gaspillée.

## 6.4 Optimisations de jointure & UDF
- **Broadcast join** : `broadcast(df_petit)` force l'envoi du petit côté à tous les executors → évite le sort-merge + shuffle. Automatique sous `autoBroadcastJoinThreshold` (10 MB).
- **UDF** : ⚠️ une **UDF Python est une boîte noire pour Catalyst** (pas d'optimisation, sérialisation Python↔JVM coûteuse). **Toujours préférer les fonctions built-in** (`pyspark.sql.functions`) ; si une UDF est inévitable, préférer une **pandas UDF** (vectorisée via Arrow).

## 6.5 Maintenance des tables Delta
- **`OPTIMIZE`** : compacte les petits fichiers.
- **`ZORDER BY`** : co-localise les données corrélées → accélère les filtres (*file skipping*).
- **`VACUUM`** : supprime les fichiers de données **non référencés** par le transaction log (récupère l'espace). ⚠️ à ne pas confondre avec `OPTIMIZE`.

---

# 7. Governance and Security — 15 %

## 7.1 Unity Catalog
- **Place dans la pile** : UC est la **couche de gouvernance unifiée bâtie PAR-DESSUS la couche de stockage** (Delta/Parquet/Iceberg) — voir §1.1. L'ordre compte à l'examen.
- **Namespace à 3 niveaux** : `catalog.schema.table`.
- **`GRANT` / `REVOKE`** : gestion des permissions.
- **Metastore** : le conteneur de plus haut niveau des métadonnées UC (catalogs → schemas → tables/volumes). **1 par région**, partageable entre workspaces → détail complet en **Annexe A.2**.
- **Control plane vs data/compute plane** : où tournent réellement les données → **Annexe A.3**.

## 7.2 Ce qu'UC apporte à l'ingestion
- **Credentials des sources externes** : c'est UC qui les détient dans le flux de managed ingestion (voir §2.6).
- **Volumes UC** : où l'`Ingestion Gateway Pipeline` dépose métadonnées, snapshots et change logs.
- **Streaming tables** : enregistrées dans UC → c'est cet enregistrement qui apporte le **support supplémentaire pour le streaming** (voir §2.2).

## 7.3 Sécurité fine
- **Dynamic views** : masquage colonnes/lignes selon l'utilisateur (`current_user()`, `is_member()`).

## 7.4 Propriété des données
- **MANAGED vs EXTERNAL** : impact direct sur ce qui est supprimé au `DROP TABLE` → voir **§3.5**. Question de gouvernance autant que de modélisation.

---

# Annexe A — Les concepts de ma page « ? », éclaircis
Les 4 sujets que j'avais notés comme pas maîtrisés. Définition + **à quoi ça sert concrètement**.

## A.1 `IDENTIFIER()` — paramétrer un **nom**, pas une valeur

**Le problème.** Dans une requête paramétrée, un marqueur de paramètre (`:nom`) ne peut remplacer qu'une **valeur de donnée** — jamais un **nom d'objet** (table, schéma, colonne, fonction).
```sql
SELECT * FROM users WHERE id = :id   -- ✅ marche : :id est une VALEUR
SELECT * FROM :ma_table              -- ❌ ne marche pas : c'est un NOM
```

**La solution.** `IDENTIFIER(strLiteral | strExpr)` convertit une expression STRING constante en **nom d'objet SQL**.
```sql
DECLARE mytab = 'tab1';
CREATE TABLE IDENTIFIER(mytab)(c1 INT);   -- crée la table « tab1 »

SELECT * FROM IDENTIFIER(:nom_table);     -- nom de table passé en paramètre
```

**À quoi ça sert vraiment.** Sans `IDENTIFIER()`, paramétrer un nom de table oblige à **concaténer des chaînes** → **faille d'injection SQL**. `IDENTIFIER()` fait la substitution de manière **sûre** : c'est la façon supportée de paramétrer la *structure* d'une requête.

**Où ça marche :** `CREATE`, `ALTER`, `DROP`, `MERGE`, `UPDATE`, `DELETE`, `INSERT`, `COPY INTO`, `SHOW`, `DESCRIBE`, `USE`, appels de fonctions, références de tables/colonnes.

**Contexte data engineering — les 3 cas typiques :**
- **Notebook paramétré par widget** : le même notebook tourne sur `clients`, `commandes`, `produits`.
- **Multi-environnement** : seul le catalogue change entre dev / staging / prod → `IDENTIFIER(:env || '.silver.ventes')`.
- **Tâche `for each`** dans un job : itérer le même traitement sur une liste de tables (voir §4.3).

🧠 **Modèle mental :** *`:param` = une **valeur** · `IDENTIFIER(:param)` = un **nom**.*

> ℹ️ Si, dans mes notes, « VALUES() » visait la **clause `VALUES`**, c'est un autre sujet : elle construit une **table inline** — `SELECT * FROM VALUES (1,'a'), (2,'b') AS t(id, nom)`. Le couple à retenir pour l'examen reste bien **`IDENTIFIER()` (nom) vs marqueur de paramètre (valeur)**.

## A.2 Metastore — le conteneur racine d'Unity Catalog

**Définition.** Le **metastore Unity Catalog** est le **conteneur de plus haut niveau** des données dans UC. Il fait 3 choses :
1. **Enregistre les métadonnées** des objets sécurisables (tables, volumes, external locations, shares).
2. **Gère les permissions** d'accès à ces objets.
3. **Fournit la structure** du namespace à 3 niveaux : `catalog.schema.table`.

**Règles à retenir (souvent testées) :**
- **1 metastore par région cloud.**
- **Plusieurs workspaces d'une même région peuvent partager le même metastore** → ils voient alors **exactement les mêmes données**.
- Un utilisateur doit passer par un workspace **rattaché au metastore de sa région**.
- Le metastore stocke les **métadonnées + permissions** ; les **données** des managed tables/volumes vivent dans un stockage cloud (optionnellement défini au niveau metastore).

**À quoi ça sert.** C'est ce qui rend la gouvernance **transverse aux workspaces** : avant UC (Hive metastore), les métadonnées étaient **locales à un workspace** → impossible de gouverner de façon centralisée. Le metastore est le point où « une permission accordée vaut partout ».

**Ne pas confondre :** metastore (racine de la **gouvernance**) ≠ catalog (1er niveau du **namespace**) ≠ workspace (l'**environnement de travail**).

## A.3 Control plane vs Data plane (= compute plane)

L'architecture Databricks est coupée en deux. C'est **la** clé pour répondre aux questions « où tournent mes données ? ».

| | **Control plane** | **Data / compute plane** |
|---|---|---|
| Géré par | **Databricks** (compte Databricks) | selon le mode ↓ |
| Contient | l'**application web**, l'UI, les APIs, la config des workspaces, l'orchestration des jobs | **le calcul lui-même** — c'est là que **tes données sont traitées** |

**Les 2 variantes du compute plane :**
| Variante | Où | Conséquence |
|---|---|---|
| **Classic compute plane** | **dans TON compte cloud** (ton VPC/VNet) | isolation naturelle, mais **tu gères l'infra** — c'est le cas des all-purpose et job clusters (§1.4) |
| **Serverless compute plane** | **dans le compte Databricks**, même région que ton workspace | isolé par des frontières réseau entre workspaces ; **Databricks gère tout** |

**Où vivent les données :**
- **Workspace classique** : dans un **bucket de ton compte cloud** (workspace storage bucket) + DBFS.
- **Workspace serverless** : un **default storage** managé pour les données internes ; tes propres catalogues/tables pointent vers **tes** emplacements de stockage cloud.

🧠 **Modèle mental :** *le control plane te montre les boutons ; le compute plane fait le travail sur tes données. Le serverless, c'est juste déplacer le compute plane du côté de Databricks.*
➡️ C'est exactement ce qui explique le modèle de coût du §1.7 : en classique tu paies **DBU + infra cloud**, en serverless **un seul DBU** qui inclut l'infra.

## A.4 Structured Streaming — le traitement incrémental unifié

**Définition.** Moteur de traitement de flux d'Apache Spark (natif, depuis Spark 2.0) : un **flux est traité comme une table qui grandit sans fin**. On écrit la même logique qu'en batch (DataFrame / SQL), Spark l'exécute **par micro-batchs incrémentaux**.

**Les 4 briques :**
| Brique | Rôle |
|---|---|
| **Source** | d'où on lit : fichiers (Auto Loader), Kafka, Delta table, socket |
| **Trigger** | à quelle cadence on traite → voir §2.3 (`availableNow`, `processingTime`, `continuous`) |
| **Checkpoint** | offsets + état → **reprise après panne** et **exactly-once** → voir §2.7 |
| **Output mode** | ce qu'on écrit : append / complete / update → voir §2.10 |

**À quoi ça sert — et pourquoi c'est central à l'examen :**
- C'est le **moteur sous Auto Loader, les streaming tables et les Spark Declarative Pipelines**. Quand tu écris `CREATE OR REFRESH STREAMING TABLE`, c'est du Structured Streaming en dessous.
- Il donne l'**incrémentalité** : ne retraiter que le nouveau, sans écrire soi-même la logique « qu'ai-je déjà lu ? » — c'est le checkpoint qui la porte.
- **Point non intuitif** : Structured Streaming ≠ « temps réel obligatoire ». Avec `trigger(availableNow=True)` c'est du **batch incrémental** — même code, coût batch (§2.3).

🧠 **Modèle mental :** *Structured Streaming = « écris une requête batch, Spark la rejoue à l'infini sur ce qui est nouveau ».*

## A.5 Standard connectors vs Managed connectors (Lakeflow Connect)

**Le compromis en une phrase :** les **managed** échangent de la flexibilité contre de l'**automatisation** ; les **standard** échangent de l'automatisation contre du **contrôle et une couverture de sources plus large**.

| | **Managed connectors** | **Standard connectors** |
|---|---|---|
| Qui gère le pipeline | **Databricks** (bout en bout) | **toi** |
| Compute | **serverless** | ton compute |
| Gouvernance | **nativement Unity Catalog** | à câbler |
| Lecture/écriture | **incrémentale et efficace** par défaut (CDC) | à toi de l'implémenter |
| Sources | apps d'entreprise & bases : SQL Server, MySQL, PostgreSQL, Salesforce, Workday, HubSpot, Jira, SharePoint, Google Drive, RabbitMQ… | cloud storage, Kafka, et tout ce qui n'a pas de connecteur managé |
| Personnalisation | limitée | **large** |

**Quand choisir quoi :**
- **Managed** → la source figure dans le catalogue de connecteurs **et** tu veux du CDC sans l'écrire. C'est le défaut recommandé.
- **Standard** → la source n'a **pas** de connecteur managé, **ou** tu as besoin de logique de pipeline sur mesure.

**L'ingestion gateway (spécifique aux connecteurs de bases de données)** : tourne dans **son propre job, en tâche continue**. Elle se connecte à la base source, en extrait **métadonnées, snapshots et change logs**, et les dépose dans un **volume Unity Catalog** → c'est ce qui rend l'ingestion **incrémentale** (seulement ce qui a changé depuis le run précédent). Détail du flux : **§2.6**.

🧠 **Modèle mental :** *managed = « Databricks conduit » · standard = « je conduis, mais je vais où je veux ».*

---

# Annexe B — Logistique d'examen
- **200 USD par tentative**, **pas de retake gratuit** (FAQ certification Databricks).
- **Score de passage : non publié par Databricks** — il est fixé par analyse statistique et évolue avec les versions d'examen.
- **Poids par domaine** (source : structure d'examen fournie) : Platform 6 % · Ingestion 21 % · Transformation 22 % · Lakeflow Jobs 16 % · CI/CD 10 % · Troubleshooting/Optimization 10 % · Governance/Security 15 %.

---

# Sources
- Databricks official Exam Guide — structure et poids par domaine
- **Mes notes manuscrites de révision** (`Dbr assoc revision.pdf`, 9 p., septembre 2026) — intégrées le 2026-09-12 dans §1.1→1.4, §2.1, §2.2, §2.5, §2.6, §2.8, §3.4, §4.1→4.6, §7.1, §7.2 et Annexe A
- Databricks Academy — « Data Engineering with Databricks »
- Derar Alhussein — Practice Exams (Udemy, V4) + O'Reilly Study Guide
- Fondamentaux Spark (§1.1, §1.2, §3.1, §3.2, §6.1→6.4) : cheat sheet [databrickspracticetest.com](https://databrickspracticetest.com/blog/apache-spark-for-databricks-exam-key-concepts-cheat-sheet) (source **non officielle**), re-vérifié le 2026-09-11 contre :
  - [Databricks Certification FAQ](https://www.databricks.com/learn/certification/faq) — coût, absence de score de passage publié
  - [Adaptive query execution — Databricks docs](https://docs.databricks.com/aws/en/optimizations/aqe) — AQE par défaut DBR 7.3+, `shuffle.partitions = auto`
  - Doc Apache Spark SQL Performance Tuning — défauts `maxPartitionBytes`, `shuffle.partitions`, `autoBroadcastJoinThreshold`
- Annexe A (concepts éclaircis, 2026-09-12) — doc officielle Databricks :
  - [IDENTIFIER clause](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-names-identifier-clause) — A.1
  - [Create a Unity Catalog metastore](https://docs.databricks.com/aws/en/data-governance/unity-catalog/create-metastore) — A.2
  - [Databricks architecture overview](https://docs.databricks.com/aws/en/getting-started/overview) — A.3 (control plane / compute plane)
  - [Lakeflow Connect](https://docs.databricks.com/aws/en/ingestion/lakeflow-connect/) — A.5 (managed vs standard, ingestion gateway)
