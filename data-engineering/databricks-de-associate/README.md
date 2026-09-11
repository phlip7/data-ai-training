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

## 1.1 Architecture d'exécution Spark
- **Driver** : porte la `SparkSession` / `SparkContext`, construit le **DAG**, planifie et distribue les tasks.
- **Executors** : exécutent les tasks et stockent les données cachées.
- **Cluster manager** : alloue les ressources (sur Databricks, couche managée au-dessus du cloud provider).

**Hiérarchie d'exécution — à connaître par cœur :**
`Action → Job → Stages → Tasks`
- une **action** déclenche un **job** ;
- le job est découpé en **stages**, séparés par les **frontières de shuffle** ;
- chaque stage est découpé en **tasks** : **1 task = 1 partition**.

## 1.2 Les APIs : RDD / DataFrame / Dataset
| API | Quoi | Note |
|---|---|---|
| **RDD** | collection distribuée immuable d'objets, **sans schéma**, lazy | API bas niveau, rarement le bon choix aujourd'hui |
| **DataFrame** | collection distribuée **organisée en colonnes nommées** | optimisée par **Catalyst** + **Tungsten** |
| **Dataset** | typage RDD + optimisation DataFrame | **Scala/Java uniquement** — en Python on n'a que le DataFrame |

## 1.3 Modèle de coût — Compute classique vs Serverless
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

## 1.4 Portabilité — Open source (Spark natif) vs Databricks-only
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

## 2.1 Streaming vs Batch incrémental — la règle d'or
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

## 2.2 Outils d'ingestion incrémentale
| Outil | Quoi | Quand |
|---|---|---|
| **Auto Loader** (`cloudFiles`) | ingestion **incrémentale de fichiers** (nouveaux seulement), suit ce qui a déjà été lu | volumes de fichiers **grands/continus** (millions), scalable ; batch OU streaming |
| **`COPY INTO`** | commande SQL **idempotente** (ignore les fichiers déjà chargés) | volumes **petits/moyens**, chargements périodiques |
| **DLT** (Delta Live Tables) | pipelines déclaratifs, streaming ou triggered | pipelines managés avec qualité/monitoring intégrés |

- **`COPY INTO` est idempotent** : réexécuter ne recharge pas les fichiers déjà traités.
- **Auto Loader** > COPY INTO quand le nombre de fichiers explose (suivi via checkpoint/RocksDB, pas de re-listing coûteux).

## 2.3 Lakeflow Connect + Spark Declarative Pipelines (le flux end-to-end)
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

## 2.4 Checkpoint — le cœur de l'incrémental
- **Checkpoint** : stocke offsets + état → **reprise après panne + exactly-once**. Indispensable au streaming.
- Le supprimer = **tout retraiter**.

## 2.5 Schema evolution à l'ingestion (Auto Loader)
- `.option("cloudFiles.schemaEvolutionMode", "addNewColumns")` + `cloudFiles.schemaLocation`
- Modes : `addNewColumns` (défaut), `rescue`, `failOnNewColumns`, `none`.
- ➡️ Vue complète des 3 moments d'évolution de schéma : **§3.5**.

## 2.6 Modes de sortie Structured Streaming
| Mode | Écrit quoi | Quand |
|---|---|---|
| **Append** | uniquement les **nouvelles lignes** | défaut ; pas d'agrégation, ou agrégation avec watermark |
| **Complete** | **toute la table de résultat** à chaque trigger | agrégations sans watermark |
| **Update** | seulement les lignes **modifiées** depuis le dernier trigger | agrégations incrémentales |

## 2.7 Exemple — Ingestion incrémentale en Spark natif (sans Auto Loader)
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

## 3.4 Modélisation Delta — tables & time travel
- **Table MANAGED** : `DROP TABLE` supprime **métadonnées + fichiers de données**.
- **Table EXTERNAL/unmanaged** : `DROP TABLE` supprime **seulement les métadonnées** (données conservées).
- **Time travel** : `VERSION AS OF` / `TIMESTAMP AS OF` (rendu possible par le transaction log).
- **Medallion** : bronze (brut) → silver (nettoyé/conformé) → gold (agrégé/métier).

## 3.5 Upsert & CDC
- **`MERGE INTO`** : upsert transactionnel classique (hors DLT).
- **DLT CDC / upsert** : **`APPLY CHANGES INTO`** (gère les changements d'un change feed, ordonnancement par séquence).

## 3.6 Schema Evolution — laisser Delta ajouter les nouvelles colonnes à ta place
**Concept transverse :** accepter automatiquement de **nouvelles colonnes** venant de la source, sans `ALTER TABLE` manuel. Le schéma peut évoluer à **3 moments** :

| Moment | Comment l'activer | Portabilité |
|---|---|---|
| **MERGE** ⭐ | `MERGE WITH SCHEMA EVOLUTION INTO cible USING source ON …` → ajoute les colonnes nouvelles de la source ; marche avec `UPDATE SET *` / `INSERT *`. Global (alt.) : `SET spark.databricks.delta.schema.autoMerge.enabled = true` | Databricks (clause `WITH SCHEMA EVOLUTION` = DBR 15.2+) |
| **Écriture** (append / overwrite) | `.option("mergeSchema", "true")` = ajoute des colonnes · `.option("overwriteSchema", "true")` = remplace tout le schéma | Delta OSS ✅ |
| **Ingestion** (Auto Loader) | `.option("cloudFiles.schemaEvolutionMode", "addNewColumns")` + `cloudFiles.schemaLocation` (modes : `addNewColumns` défaut, `rescue`, `failOnNewColumns`, `none`) | Databricks |

🧠 **Modèle mental :** *« Schema evolution = Delta ajoute les colonnes nouvelles à ma place, à 3 moments : quand je MERGE, quand j'écris, quand j'ingère. »*
⚠️ Ça **ajoute** des colonnes ; ça ne gère pas tous les **changements de type**. À utiliser sciemment (une colonne parasite en amont se propage dans la table).

## 3.7 Qualité dans les pipelines déclaratifs
- **DLT expectations** : contraintes qualité — `EXPECT` (log seulement), `EXPECT OR DROP` (écarte la ligne), `EXPECT OR FAIL` (fait échouer le pipeline).

---

# 4. Working with Lakeflow Jobs — 16 %

## 4.1 Orchestration multi-tâches
- **Jobs / Workflows** : orchestration multi-tâches. Dépendance : déclarer tâche A comme **« depends on »** de B → B ne part qu'après succès de A.

## 4.2 Compute par tâche
- Un job contient **une ou plusieurs tâches**, et **chaque tâche peut avoir son propre compute**.
- Les tâches d'un même job peuvent **partager le même cluster** ou **utiliser des computes différents** selon le besoin.

## 4.3 Déclencheurs de job
- **Scheduled (cron)** · **File arrival** · **Continuous** · **Table update**.
- **`table update` trigger** : le job se lance quand des tables sont mises à jour. Jusqu'à **10 tables par trigger**. Fonctionne avec les tables **Delta managées par Unity Catalog**, **Iceberg**, **Delta externes**, **materialized views** et **streaming tables**.

## 4.4 Fiabilité
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
- **Namespace à 3 niveaux** : `catalog.schema.table`.
- **`GRANT` / `REVOKE`** : gestion des permissions.

## 7.2 Sécurité fine
- **Dynamic views** : masquage colonnes/lignes selon l'utilisateur (`current_user()`, `is_member()`).

## 7.3 Propriété des données
- **MANAGED vs EXTERNAL** : impact direct sur ce qui est supprimé au `DROP TABLE` → voir **§3.4**. Question de gouvernance autant que de modélisation.

---

# Annexe — Logistique d'examen
- **200 USD par tentative**, **pas de retake gratuit** (FAQ certification Databricks).
- **Score de passage : non publié par Databricks** — il est fixé par analyse statistique et évolue avec les versions d'examen.
- **Poids par domaine** (source : structure d'examen fournie) : Platform 6 % · Ingestion 21 % · Transformation 22 % · Lakeflow Jobs 16 % · CI/CD 10 % · Troubleshooting/Optimization 10 % · Governance/Security 15 %.

---

# Sources
- Databricks official Exam Guide — structure et poids par domaine
- Databricks Academy — « Data Engineering with Databricks »
- Derar Alhussein — Practice Exams (Udemy, V4) + O'Reilly Study Guide
- Fondamentaux Spark (§1.1, §1.2, §3.1, §3.2, §6.1→6.4) : cheat sheet [databrickspracticetest.com](https://databrickspracticetest.com/blog/apache-spark-for-databricks-exam-key-concepts-cheat-sheet) (source **non officielle**), re-vérifié le 2026-09-11 contre :
  - [Databricks Certification FAQ](https://www.databricks.com/learn/certification/faq) — coût, absence de score de passage publié
  - [Adaptive query execution — Databricks docs](https://docs.databricks.com/aws/en/optimizations/aqe) — AQE par défaut DBR 7.3+, `shuffle.partitions = auto`
  - Doc Apache Spark SQL Performance Tuning — défauts `maxPartitionBytes`, `shuffle.partitions`, `autoBroadcastJoinThreshold`
