# Journal d'installation — EJBCA en natif (sans Docker)

Ce fichier trace, étape par étape, l'installation réelle d'EJBCA effectuée par Ahmad Diop,
avec les commandes exécutées et les résultats. Il servira de base factuelle pour remplir
le rapport Word (`GROUPE-2/Rapport_PKI_EJBCA_GROUPE-2.docx`, sections 4 et 5) : à chaque
étape marquée 📸, **Ahmad prend lui-même** une capture de la seule fenêtre concernée
(terminal ou navigateur) au moment opportun, et l'ajoute dans `docs/captures/`.

> ⚠️ Pas de capture automatique de l'écran entier : ça risque d'inclure des fenêtres/contenus
> sans rapport avec l'installation. Chaque capture doit être prise manuellement et ne montrer
> que la fenêtre pertinente (Alt+Impr écran sous Windows, ou l'outil Capture d'écran).

## Contexte et décision technique

- Le sujet demande une PKI avec **EJBCA**, mais ne demande pas Docker.
- Vérification faite dans `cours_Cryptographie-v2.pdf` (chapitres 4 et 5) : le cours ne
  mentionne jamais Docker ; les TP du cours utilisent OpenSSL/GPG/KeyTools en natif.
  EJBCA n'est cité (p.53) que comme exemple de logiciel de PKI, sans méthode d'installation imposée.
- Sur la machine d'Ahmad, Docker Desktop a échoué avec l'erreur **"Virtualization support not
  detected"** : VT-x est désactivé dans le BIOS/UEFI (confirmé aussi via `wsl --status`).
  Reconfigurer le BIOS n'était pas souhaité aujourd'hui.
- **Décision** : installer EJBCA Community **nativement sur Windows** (JDK + Ant + WildFly +
  MariaDB), ce qui ne nécessite aucune virtualisation — seulement des installations logicielles
  classiques.

## Prérequis constatés sur la machine (avant installation)

| Composant | État constaté | Action |
|---|---|---|
| Java (JDK) | OpenJDK 25 (Temurin) déjà installé | Trop récent pour EJBCA/WildFly → JDK 17 installé en plus |
| Apache Ant | Absent | Installé (voir ci-dessous) |
| Base de données | Absent | **PostgreSQL 17** installé (MariaDB abandonné : le mirroir officiel bloquait les téléchargements avec un challenge anti-bot/Cloudflare) |
| WildFly | Absent | Installé (voir ci-dessous) |
| Docker | Installé mais bloqué (pas de virtualisation) | Abandonné pour ce poste |

**Note technique** : EJBCA supporte officiellement PostgreSQL comme base de données
(au même titre que MariaDB), ce choix n'est donc pas une entorse au sujet.

## Étapes réalisées

### 1. Installation de JDK 17 (Eclipse Temurin)

Commande :
```
winget install --id EclipseAdoptium.Temurin.17.JDK --silent
```

Statut : ✅ fait — installé dans `C:\Program Files\Eclipse Adoptium\jdk-17.0.20.101-hotspot`
(coexiste avec le JDK 25 déjà présent).

📸 Capture à prendre : fenêtre terminal avec `java -version` pointant vers JDK 17.

### 2. Installation d'Apache Ant

Téléchargement direct (le binaire officiel Apache, pas de paquet winget) :
```
curl -sL -o apache-ant.zip https://dlcdn.apache.org/ant/binaries/apache-ant-1.10.18-bin.zip
```
Extrait dans `C:\ejbca-tools\apache-ant-1.10.18`.

Statut : ✅ fait

### 3. Installation de la base de données : PostgreSQL 17 (remplace MariaDB)

MariaDB a été abandonné : le mirroir officiel `downloads.mariadb.org` et `winget`
renvoyaient tous deux une erreur (challenge anti-bot Cloudflare / HTTP 403).
EJBCA supportant officiellement PostgreSQL, on a basculé dessus sans impact sur le sujet.

```
winget install --id PostgreSQL.PostgreSQL.17 --silent
```

Création de la base et de l'utilisateur dédiés à EJBCA :
```sql
CREATE USER ejbca WITH PASSWORD 'ejbca';
CREATE DATABASE ejbca OWNER ejbca ENCODING 'UTF8';
GRANT ALL PRIVILEGES ON DATABASE ejbca TO ejbca;
```

Statut : ✅ fait — service `postgresql-x64-17` démarré, base `ejbca` créée.

📸 Captures à prendre : fenêtre `Get-Service postgresql-x64-17` (service Running) et fenêtre
terminal montrant la création de la base (`CREATE DATABASE ejbca`).

### 4. Récupération du code source EJBCA Community

```
curl -sL -o ejbca-ce.zip https://api.github.com/repos/Keyfactor/ejbca-ce/zipball/r9.3.7
```

Statut : ✅ téléchargé (version r9.3.7, dernière release stable de Keyfactor/ejbca-ce)

### 5. Installation de WildFly (serveur d'application Java EE)

```
curl -sL -o wildfly.zip https://github.com/wildfly/wildfly/releases/download/39.0.1.Final/wildfly-39.0.1.Final.zip
```
Extrait dans `C:\ejbca-tools\wildfly-39.0.1.Final`.

Statut : ✅ fait

### 6. Configuration des fichiers de propriétés EJBCA

Créés à partir des `.sample` fournis :
- `conf/database.properties` : PostgreSQL, base `ejbca`, utilisateur `ejbca`.
- `conf/install.properties` : Management CA `CN=Master-SSI Management CA,O=Master-SSI-2026,C=SN`,
  RSA 4096, SHA256WithRSA, validité 10 ans.

Statut : ✅ fait

### 7. Variables d'environnement de build (JDK 17 / Ant / WildFly)

Fichier `C:\ejbca-tools\ejbca-env.sh` centralisant `JAVA_HOME`, `ANT_HOME`, `APPSRV_HOME`
(pointant vers WildFly) et `EJBCA_HOME`.

Statut : ✅ fait

### 8. Build EJBCA avec Ant (`ant clean` puis `ant deployear`)

```
ant clean
ant deployear
```

Statut : ✅ **BUILD SUCCESSFUL** (27 min 40 s, téléchargement des dépendances Maven/Ivy inclus).
L'EAR `ejbca.ear` a été automatiquement copié dans
`C:\ejbca-tools\wildfly-39.0.1.Final\standalone\deployments\`.

📸 Capture à prendre : fin du build Ant affichant `BUILD SUCCESSFUL`.

### 9. Configuration du datasource PostgreSQL dans WildFly

- Pilote JDBC PostgreSQL (`postgresql-42.7.4.jar`, téléchargé depuis Maven Central) déployé en
  hot-deploy dans `standalone/deployments/`.
- Mémoire heap augmentée dans `bin/standalone.conf.bat` : `-Xms512M -Xmx2048M`.
- Datasource JNDI `java:/EjbcaDS` créé via `jboss-cli.bat` :
  ```
  data-source add --name=ejbcads --jndi-name=java:/EjbcaDS \
    --driver-name=postgresql-42.7.4.jar \
    --connection-url=jdbc:postgresql://127.0.0.1:5432/ejbca \
    --user-name=ejbca --password=ejbca ...
  :reload
  ```

Statut : ✅ fait — WildFly a rechargé sa config, `EjbcaDS` est bien lié
(`WFLYJCA0001: Bound data source [java:/EjbcaDS]`), le redéploiement de `ejbca.ear` est reparti.

### 10. Démarrage de WildFly

`ant install` n'existe plus dans cette version d'EJBCA (r9.3.7) — la target a été supprimée.
Utilisation de `bin/ejbca.sh` (CLI EJB) à la place, après un fix nécessaire :

- **Bug corrigé** : `bin/ejbca.sh` lançait `-Dlog4j.configuration=C:/...` sans préfixe `file:`,
  ce qui fait planter le parsing d'URL sous Windows (`C:` interprété comme un protocole).
  Corrigé en ajoutant `file:` devant le chemin dans le script.
- **Bug corrigé** : `dist/ejbca-ejb-cli/jboss-ejb-client.properties` pointait vers le port
  legacy `4447` (JBoss AS7/EAP6), qui n'existe plus sur WildFly 39 (remoting HTTP-upgrade sur
  le port 8080). Reconfiguré avec `port=8080` et `protocol=http-remoting`.

Statut : ✅ WildFly démarre proprement (`WFLYSRV0025: ... started ...`), `ejbca.ear` déployé.

### 11. Création de la Management CA (CLI)

```
./bin/ejbca.sh ca init ManagementCA "CN=Master-SSI Management CA,O=Master-SSI-2026,C=SN" \
  soft foo123 4096 RSA 3650 null SHA256WithRSA -superadmincn SuperAdmin
```

Statut : ✅ confirmé en base (table `cadata`, CA `ManagementCA` statut actif) et le rôle
"Super Administrator Role" a bien été créé avec un membre matchant `CN=SuperAdmin` signé par
cette CA (tables `roledata` / `rolememberdata`).

📸 Capture à prendre : terminal montrant la commande `ca init` exécutée.

### 12. Création de l'end entity SuperAdmin (CLI)

```
./bin/ejbca.sh ra addendentity --username SuperAdmin --password foo123 --dn "CN=SuperAdmin" \
  --caname ManagementCA --type 1 --token P12
```

⚠️ Syntaxe à double-tiret (`--username`) obligatoire, la syntaxe à simple-tiret ou positionnelle
échoue silencieusement (exit code 3) sur cette version.

Statut : ✅ confirmé en base (table `userdata`, `SuperAdmin` en statut NEW, tokentype P12).

### 13. Génération du certificat P12 SuperAdmin

La commande CLI `./bin/ejbca.sh batch` n'a rien généré (chemin de sortie probablement mal résolu,
échec silencieux). **Solution retenue : enrôlement via la RA Web**, qui est de toute façon la
méthode standard recommandée par EJBCA :

1. Ouvrir `https://localhost:8443/ejbca/ra/`
2. "Make New Request" → "Use username and enrollment code"
3. Username `SuperAdmin`, code `foo123`
4. Télécharger le certificat au format PKCS#12

Statut : 🔄 en cours (à faire par Ahmad dans le navigateur)

📸 Capture à prendre : formulaire d'enrôlement RA Web + confirmation de téléchargement du .p12.

### 14. Import du certificat SuperAdmin et accès à l'Admin Web

Statut : ⏳ à faire — importer le .p12 dans le navigateur, puis se reconnecter sur
`https://localhost:8443/ejbca/adminweb/` en présentant ce certificat client.

📸 Capture à prendre : Admin Web accessible avec le compte SuperAdmin authentifié.

## Difficultés rencontrées

*(à compléter au fur et à mesure — ex. incompatibilité de version JDK, erreur de datasource,
port déjà utilisé...)*

## Notes pour le rapport Word

- Section "Phase d'installation" du rapport : remplacer la partie Docker Compose par cette
  installation native, en expliquant le choix (absence de virtualisation, mais conformité au
  sujet qui ne mentionne pas Docker).
- Garder les captures listées ci-dessus dans l'ordre pour la section 4/5 du rapport.
