# ETL Apache Hop – PoC (Docker)

> Mini‑chaîne ETL packagée en **Docker Compose** :  
> **Apache Hop Web** (UI), **MariaDB** (cible), **Adminer** (console SQL).  
> Objectif : charger un CSV d’agents dans une base, comprendre le principe d’une *pipeline* ETL et disposer d’un socle reproductible.

---

## 1) Architecture & principes

```
┌────────────┐        Hop Web UI         ┌─────────────┐
│ agents.csv │ ────────────────────────► │   Pipeline  │
└────────────┘                           │  CSV → DB   │
                                         └─────┬───────┘
                                               │ JDBC
                                               ▼
                                         ┌─────────────┐
                                         │   MariaDB   │
                                         └─────┬───────┘
                                               │
                                 Web console   ▼
                                         ┌─────────────┐
                                         │  Adminer    │
                                         └─────────────┘
```

- **Hop Web** sert à modéliser et exécuter les pipelines (chargements, nettoyages…).
- **MariaDB** héberge les tables cibles.
- **Adminer** permet de vérifier rapidement les données côté base.

---

## 2) Pré‑requis

- Docker ≥ 20.x et Docker Compose plugin
- Ports disponibles localement : `8086` (Hop Web), `8081` (Adminer), `3307` (exposition MariaDB)

---

## 3) Lancer l’environnement

```bash
# 1) Cloner le dépôt
git clone https://github.com/avisse/ETL-apach-Hop.git
cd ETL-apach-Hop

# 2) (Optionnel) éditer .env si besoin (ports / credentials)
#    sinon laisser les valeurs par défaut

# 3) Démarrer
docker compose up -d

# 4) Vérifier les conteneurs
docker compose ps
```

**Accès UI**  

- Hop Web : <http://localhost:8086/ui>
- Adminer : <http://localhost:8081>

> ⚠️ Dans **Adminer**, le champ *Serveur* est `mariadb` (nom du service Docker) et non `localhost`.
>
> **Identifiants par défaut** (cf. `.env` et `init.sql`) :  
> Utilisateur : `etl_user` – Mot de passe : `etl_pass` – Base : `etl_demo`

---

## 4) Données & schéma

### 4.1 CSV exemple

`data/agents.csv` (monté dans Hop sous `/files/data/agents.csv`) :

```
Matricule,Nom,Prenom,Service,Email,Salaire,DateEmbauche,Actif
A011,dubois,lEna,Finances,LENA.DUBOIS@MAIRIE.FR,2450,2021/02/15,oui
A012,Martin,Bob,RH,bob.martin@mairie.fr,2280,,1
A013,Durand,Alice,Informatique,alice.durand@mairie.fr,NaN,2020-11-03,0
...
```

### 4.2 Base MariaDB

`init.sql` est automatiquement exécuté au démarrage pour créer la base `etl_demo`, l’utilisateur `etl_user` et deux tables :

```sql
CREATE TABLE IF NOT EXISTS agents (
  Matricule     VARCHAR(20) PRIMARY KEY,
  Nom           VARCHAR(100),
  Prenom        VARCHAR(100),
  Service       VARCHAR(100),
  Email         VARCHAR(255),
  Salaire       DECIMAL(10,2),
  DateEmbauche  DATETIME NULL,
  Actif         TINYINT(1)
);

CREATE TABLE IF NOT EXISTS agents_staging LIKE agents;
```

---

## 5) Pipelines dans Hop Web (PoC)

Le PoC contient une *pipeline* ultra simple **CSV → MariaDB**.  
Si vous repartez de zéro, voici la démarche (5 minutes) :

### 5.1 Créer la connexion MariaDB

- **File → New → Metadata → Relational Database Connection**
- `Connection type` : **MariaDB**
- `Server host name` : **host.docker.internal**
- `Port` : **3307**  
  (On expose le port 3306 du conteneur sur 3307 côté hôte.)
- `Database name` : **etl_demo**
- `Username` : **etl_user**
- `Password` : **etl_pass**
- Test **OK**

> Alternative depuis Hop (en conteneur) : `mariadb:3306` (mais depuis le navigateur hôte, `host.docker.internal:3307` fonctionne partout).

### 5.2 Pipeline 1 : *CSV → agents*

- **New → Pipeline**
- **Input** : *CSV file input*  
  `Filename` : `/files/data/agents.csv` – `Delimiter` : `,` – `Enclosure` : `"` – cochez **Header row present** – **Get fields**
- **Output** : *Table output*  
  `Connection` : *mariadb_etl* – `Target table` : `agents` – mappez les champs si besoin
- Reliez les deux éléments, **Exécutez**, vérifiez les compteurs.

### 5.3 (Optionnel) Pipeline 2 : *staging → agents (upsert)*

- *Table input* : `SELECT * FROM agents_staging`
- *Select Values / Trim* : nettoyer `Nom`, `Prenom`, `Email`
- *Value Mapper* : `Actif` → {`oui`→1, `non`→0} (ou équivalent)
- *Insert/Update* : **Keys** = `Matricule` ; **Update** des autres colonnes

> Cette logique permet des rechargements idempotents (upsert) et sépare l’atterrissage brut de la consolidation.

### 5.4 (Optionnel) Workflow

- **New → Workflow** : enchaînez *CSV→staging*, puis *staging→agents* et ajoutez en *Post‑step* un `TRUNCATE TABLE agents_staging;`.

---

## 6) Vérifier côté base (Adminer)

- System : **MySQL**
- Server : **mariadb**
- Username : **etl_user**
- Password : **etl_pass**
- Database : **etl_demo**

Exemples rapides :
```sql
SELECT COUNT(*) AS total, SUM(Actif) AS actifs FROM agents;
SELECT Service, COUNT(*) AS nb FROM agents GROUP BY Service;
```

---

## 7) Arborescence du dépôt

```
.
├── .env                 # Variables (ports & credentials)
├── data/
│   └── agents.csv       # Jeu de données d'exemple
├── docker-compose.yml   # Stack Hop + MariaDB + Adminer
└── init.sql             # Création base / utilisateur / tables
```

> Les pipelines Hop peuvent être exportés depuis l’UI (File → Save as) puis versionnés dans un dossier `hop-project/` ou `pipelines/`. Le PoC se veut minimaliste et reproductible ; le focus est sur l’environnement et la logique de chargement.

---

## 8) Paramètres (.env)

```
# Ports hôte
HOP_WEB_PORT=8086
ADMINER_PORT=8081
MARIADB_PORT=3307

# MariaDB
MARIADB_DATABASE=etl_demo
MARIADB_USER=etl_user
MARIADB_PASSWORD=etl_pass
MARIADB_ROOT_PASSWORD=root
```

Modifiez `.env` puis relancez : `docker compose up -d`.

---

## 9) Dépannage

- **Driver MariaDB introuvable dans Hop** : vérifier que la variable `HOP_SHARED_JDBC_FOLDERS` inclut `/usr/local/hop/lib/jdbc` et que `mariadb-java-client.jar` est monté (voir compose). Redémarrer la stack après modification.
- **Adminer “Name does not resolve”** : utilisez `mariadb` (nom du service Docker) comme *Serveur*, pas `localhost`.
- **Impossible de se connecter à la DB depuis Hop** : depuis le navigateur (hôte), utilisez `host.docker.internal:3307`.

---

## 10) Nettoyer

```bash
docker compose down -v
```

---

## 11) Licence

MIT – usage libre pour la démonstration/formation.

---

## 12) Références

- Apache Hop : https://hop.apache.org/
- MariaDB : https://mariadb.org/
- Adminer : https://www.adminer.org/
