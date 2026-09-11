# Databricks Certified Data Engineer Associate — Notes de révision

> Notes personnelles de prép (exam : 4 août 2026). Rédigées avec mon COS.
> Format markdown compatible GitHub + Obsidian. À pousser dans le repo de révision.

---

## 1. Ingestion : Streaming vs Batch Incremental

### La règle d'or
**La latence exigée par le métier commande le choix. Le reste suit.**
Le streaming est un choix de **coût et de complexité**, pas un défaut. La plupart des « besoins temps réel » sont couverts par du **micro-batch fréquent**.

- Décision en **secondes / sous-minute** (fraude, alerting, IoT, live) → **streaming**
- Fraîcheur en **minutes → heures** (BI, reporting, ML batch) → **batch incrémental**

### Les 5 raisons qui tranchent
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

---

## 2. Outils d'ingestion incrémentale

| Outil | Quoi | Quand |
|---|---|---|
| **Auto Loader** (`cloudFiles`) | ingestion **incrémentale de fichiers** (nouveaux seulement), suit ce qui a déjà été lu | volumes de fichiers **grands/continus** (millions), scalable ; batch OU streaming |
| **`COPY INTO`** | commande SQL **idempotente** (ignore les fichiers déjà chargés) | volumes **petits/moyens**, chargements périodiques |
| **DLT** (Delta Live Tables) | pipelines déclaratifs, streaming ou triggered | pipelines managés avec qualité/monitoring intégrés |

- **`COPY INTO` est idempotent** : réexécuter ne recharge pas les fichiers déjà traités.
- **Auto Loader** > COPY INTO quand le nombre de fichiers explose (suivi via checkpoint/RocksDB, pas de re-listing coûteux).

### Lakeflow Connect + Spark Declarative Pipelines (le flux end-to-end)
- **Lakeflow Connect** = le point d'entrée des données dans Databricks. Connecte une grande variété de sources :
  - **Cloud object stores** : S3, ADLS, GCS
  - **Message queues / streaming** : Kafka, Pub/Sub, Kinesis
  - **Bases traditionnelles** : SQL Server, Postgres
  - **Apps SaaS** : Salesforce, Workday
- **3 types de connecteurs Lakeflow Connect** :
  - **Manual File Uploads** : uploader rapidement des fichiers locaux dans des volumes ou tables.
  - **Standard Connectors** : ingestion depuis cloud storage, Kafka… en mode **batch, incrémental ou streaming**.
  - **Managed Connectors** : pour les **applications d'entreprise et bases de données** → ingestion incrémentale **scalable et efficace** dans le lakehouse.
- **Spark Declarative Pipelines** (évolution déclarative type DLT) = ingestion + transformation, pour bâtir des pipelines **medallion** : **bronze → silver → gold**, avec fiabilité et scalabilité.
- 🧠 **Modèle mental :** *Lakeflow Connect fait entrer la donnée → Spark Declarative Pipelines la fait progresser bronze→silver→gold.*

---

## 3. Concepts clés par domaine (auto-diagnostic)

### Delta Lake / Lakehouse
- **`VACUUM`** : supprime les fichiers de données non référencés par le transaction log (récupère l'espace). ⚠️ pas `OPTIMIZE`.
- **`OPTIMIZE`** : compacte les petits fichiers. **`ZORDER BY`** : co-localise les données pour accélérer les filtres.
- **Table MANAGED** : `DROP TABLE` supprime **métadonnées + fichiers de données**.
- **Table EXTERNAL/unmanaged** : `DROP TABLE` supprime **seulement les métadonnées** (données conservées).
- **Time travel** : `VERSION AS OF` / `TIMESTAMP AS OF` (rendu possible par le transaction log).

### ELT avec Spark SQL / Python
- **`CREATE TABLE AS SELECT` (CTAS)** : échoue si la table existe. `CREATE OR REPLACE TABLE` : écrase.
- **Vues** : `TEMP VIEW` = portée SparkSession/notebook · `GLOBAL TEMP VIEW` = cross-session dans le cluster · vue standard = persistée dans le metastore.
- **Accès données** : `struct.champ` (point) pour un STRUCT · `colonne:champ` (deux-points) pour du JSON/semi-structuré (VARIANT). ⚠️ piège classique.
- **Dédup** : `DISTINCT`, ou `DROP DUPLICATES`, ou `MERGE` avec condition.

### Incremental Data Processing
- **Auto Loader vs COPY INTO** : voir §2.
- **Checkpoint** : stocke offsets + état → **reprise après panne + exactly-once**. Indispensable au streaming.
- **DLT CDC / upsert** : **`APPLY CHANGES INTO`** (gère les changements d'un change feed, ordonnancement par séquence).
- **`MERGE INTO`** : upsert transactionnel classique (hors DLT). Voir *Schema Evolution* ci-dessous pour l'auto-évolution des colonnes.

### Schema Evolution — laisser Delta ajouter les nouvelles colonnes à ta place
**Concept transverse :** accepter automatiquement de **nouvelles colonnes** venant de la source, sans `ALTER TABLE` manuel. Le schéma peut évoluer à **3 moments** :

| Moment | Comment l'activer | Portabilité |
|---|---|---|
| **MERGE** ⭐ | `MERGE WITH SCHEMA EVOLUTION INTO cible USING source ON …` → ajoute les colonnes nouvelles de la source ; marche avec `UPDATE SET *` / `INSERT *`. Global (alt.) : `SET spark.databricks.delta.schema.autoMerge.enabled = true` | Databricks (clause `WITH SCHEMA EVOLUTION` = DBR 15.2+) |
| **Écriture** (append / overwrite) | `.option("mergeSchema", "true")` = ajoute des colonnes · `.option("overwriteSchema", "true")` = remplace tout le schéma | Delta OSS ✅ |
| **Ingestion** (Auto Loader) | `.option("cloudFiles.schemaEvolutionMode", "addNewColumns")` + `cloudFiles.schemaLocation` (modes : `addNewColumns` défaut, `rescue`, `failOnNewColumns`, `none`) | Databricks |

🧠 **Modèle mental :** *« Schema evolution = Delta ajoute les colonnes nouvelles à ma place, à 3 moments : quand je MERGE, quand j'écris, quand j'ingère. »*
⚠️ Ça **ajoute** des colonnes ; ça ne gère pas tous les **changements de type**. À utiliser sciemment (une colonne parasite en amont se propage dans la table).

### Production Pipelines
- **Jobs / Workflows** : orchestration multi-tâches. Dépendance : déclarer tâche A comme **« depends on »** de B → B ne part qu'après succès de A.
- **Compute par tâche** : un job contient **une ou plusieurs tâches**, et **chaque tâche peut avoir son propre compute**. Les tâches d'un même job peuvent **partager le même cluster** ou **utiliser des computes différents** selon le besoin.
- **Déclencheurs de job (Lakeflow Jobs)** : scheduled (cron), file arrival, continuous, et **table update** — le job se lance quand des tables sont mises à jour.
  - **`table update` trigger** : jusqu'à **10 tables par trigger**. Fonctionne avec les tables **Delta managées par Unity Catalog**, **Iceberg**, **Delta externes**, **materialized views** et **streaming tables**.
- **DLT expectations** : contraintes qualité (`EXPECT`, `EXPECT OR DROP`, `EXPECT OR FAIL`).
- Retries, scheduling, alertes intégrés aux Jobs.

### Data Governance / Unity Catalog
- **Namespace à 3 niveaux** : `catalog.schema.table`.
- **`GRANT` / `REVOKE`** : permissions.
- **Dynamic views** : masquage colonnes/lignes selon l'utilisateur (`current_user()`, `is_member()`).

### Coût — Classique vs Serverless
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

---

## 4. Portabilité — Open source (Spark natif) vs Databricks-only

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

## 5. Exemple — Ingestion incrémentale en Spark natif (sans Auto Loader)

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

**Les 3 points clés :**
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

## Sources
- Databricks official Exam Guide (V4, mai 2026) — confirmer les poids par domaine
- Databricks Academy — « Data Engineering with Databricks »
- Derar Alhussein — Practice Exams (Udemy, V4) + O'Reilly Study Guide
