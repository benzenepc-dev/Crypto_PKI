# Section rapport — Installation et configuration d'EJBCA (Ahmad Diop)

Installation **native**, sans Docker (non exigé par le sujet ni par le cours — voir
[recherche-ecosysteme-pki-2026.md](recherche-ecosysteme-pki-2026.md)). Toutes les commandes
ci-dessous sont reproductibles telles quelles par n'importe quel autre groupe sous Windows.

## 1. Prérequis

| Outil | Version | Rôle |
|---|---|---|
| JDK | 17 (Temurin) | Compiler et exécuter EJBCA/WildFly (une version plus récente, ex. 25, n'est pas compatible) |
| Apache Ant | 1.10.18 | Compiler EJBCA depuis les sources |
| PostgreSQL | 17 | Base de données de la CA (alternative officiellement supportée à MariaDB) |
| WildFly | 39.0.1.Final | Serveur d'application Java EE qui héberge EJBCA |

## 2. Installation des prérequis

```bash
# JDK 17
winget install --id EclipseAdoptium.Temurin.17.JDK --silent

# PostgreSQL 17
winget install --id PostgreSQL.PostgreSQL.17 --silent

# Apache Ant (pas de paquet winget officiel, téléchargement direct)
curl -sL -o apache-ant.zip https://dlcdn.apache.org/ant/binaries/apache-ant-1.10.18-bin.zip
unzip -q apache-ant.zip -d C:\ejbca-tools

# WildFly
curl -sL -o wildfly.zip https://github.com/wildfly/wildfly/releases/download/39.0.1.Final/wildfly-39.0.1.Final.zip
unzip -q wildfly.zip -d C:\ejbca-tools
```

📸 `01-jdk17-java-version.png` — vérification :
```bash
export JAVA_HOME="/c/Program Files/Eclipse Adoptium/jdk-17.0.20.101-hotspot"
export PATH="$JAVA_HOME/bin:$PATH"
java -version
```

📸 `02-ant-version.png` :
```bash
export ANT_HOME="/c/ejbca-tools/apache-ant-1.10.18"
export PATH="$ANT_HOME/bin:$PATH"
ant -version
```

## 3. Base de données PostgreSQL

```sql
CREATE USER ejbca WITH PASSWORD 'ejbca';
CREATE DATABASE ejbca OWNER ejbca ENCODING 'UTF8';
GRANT ALL PRIVILEGES ON DATABASE ejbca TO ejbca;
```

📸 `03-postgresql-service-running.png` : `Get-Service postgresql-x64-17` → statut `Running`.
📸 `04-postgresql-database-ejbca.png` : `\l` dans `psql` montrant la base `ejbca`.

## 4. Récupération et compilation d'EJBCA Community

```bash
curl -sL -o ejbca-ce.zip https://api.github.com/repos/Keyfactor/ejbca-ce/zipball/r9.3.7
unzip -q ejbca-ce.zip
```

Créer `conf/database.properties` :
```properties
datasource.jndi-name=EjbcaDS
database.name=postgres
database.url=jdbc:postgresql://127.0.0.1:5432/ejbca
database.driver=org.postgresql.Driver
database.username=ejbca
database.password=ejbca
```

Créer `conf/install.properties` :
```properties
ca.name=ManagementCA
ca.dn=CN=Master-SSI Management CA,O=Master-SSI-2026,C=SN
ca.tokentype=soft
ca.tokenpassword=null
ca.keytype=RSA
ca.keyspec=4096
ca.signaturealgorithm=SHA256WithRSA
ca.validity=3650
ca.policy=null
```

📸 `06-config-database-properties.png`

Compilation :
```bash
export JAVA_HOME="/c/Program Files/Eclipse Adoptium/jdk-17.0.20.101-hotspot"
export ANT_HOME="/c/ejbca-tools/apache-ant-1.10.18"
export APPSRV_HOME="/c/ejbca-tools/wildfly-39.0.1.Final"
export PATH="$JAVA_HOME/bin:$ANT_HOME/bin:$PATH"
cd ejbca-src
ant clean
ant deployear
```

⏱️ Première compilation : ~28 minutes (téléchargement des dépendances Maven/Ivy). Les
compilations suivantes sont bien plus rapides grâce au cache Ivy local.

📸 `07-ant-build-successful.png` — doit afficher `BUILD SUCCESSFUL`. L'EAR `ejbca.ear` est
automatiquement copié dans `wildfly-39.0.1.Final/standalone/deployments/`.

## 5. Configuration de WildFly

Pilote JDBC PostgreSQL déployé en hot-deploy :
```bash
curl -sL -o postgresql-driver.jar https://repo1.maven.org/maven2/org/postgresql/postgresql/42.7.4/postgresql-42.7.4.jar
cp postgresql-driver.jar wildfly-39.0.1.Final/standalone/deployments/postgresql-42.7.4.jar
touch wildfly-39.0.1.Final/standalone/deployments/postgresql-42.7.4.jar.dodeploy
```

Mémoire heap (`bin/standalone.conf.bat`) :
```bat
set "JBOSS_JAVA_SIZING=-Xms512M -Xmx2048M"
```

Démarrage de WildFly (Windows : lancer via `Start-Process`, pas directement en pipe, sinon le
process ne se détache pas correctement) :
```powershell
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17.0.20.101-hotspot"
Start-Process -FilePath "C:\ejbca-tools\wildfly-39.0.1.Final\bin\standalone.bat" `
  -RedirectStandardOutput "C:\ejbca-tools\wildfly-stdout.log" `
  -RedirectStandardError "C:\ejbca-tools\wildfly-stderr.log" -WindowStyle Hidden
```

📸 `08-wildfly-started.png` : dans `wildfly-stdout.log`, chercher `WFLYSRV0025` (`started in
...ms`, sans erreur — la première tentative de déploiement affiche `WFLYSRV0026` **avec erreur**
tant que le datasource n'existe pas, c'est normal, voir étape suivante).

Création du datasource JNDI `EjbcaDS` via `jboss-cli` :
```bash
"/c/ejbca-tools/wildfly-39.0.1.Final/bin/jboss-cli.bat" --connect --file="add-datasource.cli"
```
avec `add-datasource.cli` :
```
data-source add --name=ejbcads --jndi-name=java:/EjbcaDS --driver-name=postgresql-42.7.4.jar \
  --connection-url=jdbc:postgresql://127.0.0.1:5432/ejbca --user-name=ejbca --password=ejbca \
  --use-ccm=false --min-pool-size=5 --max-pool-size=150 --pool-prefill=true \
  --transaction-isolation=TRANSACTION_READ_COMMITTED --check-valid-connection-sql="SELECT 1" \
  --background-validation=true --background-validation-millis=60000 \
  --exception-sorter-class-name=org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLExceptionSorter
:reload
```

📸 `09-wildfly-datasource-bound.png` : ligne `WFLYJCA0001: Bound data source [java:/EjbcaDS]`
dans le log, suivie du redéploiement réussi de `ejbca.ear`.

## 6. Création de la Management CA et du SuperAdmin (CLI EJBCA)

⚠️ **Deux bugs rencontrés et corrigés** (spécifiques à WildFly 39 + Windows), voir
[journal-installation-ejbca-natif.md](journal-installation-ejbca-natif.md) section 10 pour le
détail complet.

1. `bin/ejbca.sh` utilisait `-Dlog4j.configuration=C:/...` sans préfixe `file:`, ce qui fait
   planter le parsing d'URL sous Windows. **Fix** : ajouter `file:` devant le chemin.
2. `dist/ejbca-ejb-cli/jboss-ejb-client.properties` pointait vers le port legacy `4447`
   (JBoss AS7/EAP6). WildFly 39 utilise le remoting HTTP-upgrade sur le port 8080. **Fix** :
   `remote.connection.default.port=8080` et `remote.connection.default.protocol=http-remoting`.

Création de la CA (signée elle-même, RSA 4096) et du rôle SuperAdmin :
```bash
./bin/ejbca.sh ca init ManagementCA "CN=Master-SSI Management CA,O=Master-SSI-2026,C=SN" \
  soft foo123 4096 RSA 3650 null SHA256WithRSA -superadmincn SuperAdmin
```

📸 `10-ca-init-db-proof.png` : preuve en base (la commande CLI n'affiche rien en console à
cause d'un logger cassé, mais l'opération réussit bel et bien) :
```sql
SELECT name, status FROM cadata;
SELECT rolename FROM roledata;
```

Création du compte utilisateur SuperAdmin (End Entity) :
```bash
./bin/ejbca.sh ra addendentity --username SuperAdmin --password foo123 --dn "CN=SuperAdmin" \
  --caname ManagementCA --type 1 --token P12
```

⚠️ Syntaxe à **double tiret** obligatoire (`--username`), la syntaxe positionnelle ou à simple
tiret échoue silencieusement (exit code 3) sur cette version d'EJBCA.

📸 `11-superadmin-endentity-db.png` :
```sql
SELECT username, status, tokentype FROM userdata WHERE username='SuperAdmin';
```

## 7. Récupération du certificat SuperAdmin et connexion à l'Admin Web

La commande CLI `./bin/ejbca.sh batch` (génération automatique du .p12) a échoué silencieusement.
**Solution retenue** — l'enrôlement par la RA Web, qui est de toute façon la méthode standard :

1. Ouvrir `https://localhost:8443/ejbca/ra/`
2. Menu **Enrôlement** → **Enrôlement avec un nom d'utilisateur**
3. Username `SuperAdmin`, code d'enrôlement `foo123`
4. Choisir un algorithme de clé sérieux (**RSA 2048** minimum — pas la courbe ECC par défaut
   proposée, `sect163r2`/B-163, obsolète et trop faible)
5. Télécharger au format **PKCS#12**

📸 `12-ra-web-enrollment.png`

6. Importer le `.p12` dans le navigateur (gestionnaire de certificats)
7. Se reconnecter sur `https://localhost:8443/ejbca/adminweb/` en présentant ce certificat client

📸 `13-adminweb-superadmin-connected.png`

## 8. Suite (émission du certificat serveur pour Papa)

- Créer un profil de certificat `SERVER` (Key Usage / Extended Key Usage = serverAuth)
- Créer/adapter un profil d'End Entity autorisant la saisie du CN + SAN
- Récupérer la CSR de Papa (branche `webserver-https-migration`) et émettre le certificat signé
- Exporter `root-ca.pem`, `chain.pem`, `server-cert.pem` pour la suite de la démo HTTPS
