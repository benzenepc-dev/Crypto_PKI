# Reproduction temporaire pour captures propres

But : rejouer les étapes clés de l'installation EJBCA dans un dossier **temporaire et jetable**
(`C:\Users\Ahmad\Downloads\ejbca-demo-tmp`), pour prendre des captures d'écran en live, commande
par commande. À la fin : on supprime ce dossier temporaire et on continue sur la vraie
installation (`C:\ejbca-tools`), qui n'est pas touchée par cette reproduction.

**Terminal à utiliser : Git Bash** (clic droit dans un dossier → "Git Bash Here", ou lance
`git-bash.exe`). C'est celui qui a servi pour la vraie installation — les scripts `.sh`
(`ejbca.sh`) ne marchent qu'avec bash, pas cmd/PowerShell.

Exécute chaque bloc **un par un**, capture (Win+Maj+S) après chaque résultat affiché, avant de
passer au suivant.

---

## Étape 1 — Vérifier JDK 17

```bash
export JAVA_HOME="/c/Program Files/Eclipse Adoptium/jdk-17.0.20.101-hotspot"
export PATH="$JAVA_HOME/bin:$PATH"
java -version
```
📸 `01-jdk17-java-version.png`

## Étape 2 — Extraire Ant (déjà téléchargé) et vérifier

```bash
mkdir -p /c/Users/Ahmad/Downloads/ejbca-demo-tmp
cd /c/Users/Ahmad/Downloads/ejbca-demo-tmp
unzip -q /c/Users/Ahmad/Downloads/ejbca-setup/apache-ant.zip
export ANT_HOME="/c/Users/Ahmad/Downloads/ejbca-demo-tmp/apache-ant-1.10.18"
export PATH="$ANT_HOME/bin:$PATH"
ant -version
```
📸 `02-ant-version.png`

## Étape 3 — PostgreSQL : service + base (déjà en place, on vérifie juste)

```bash
powershell -Command "Get-Service postgresql-x64-17"
```
📸 `03-postgresql-service-running.png`

```bash
export PGPASSWORD=postgres
"/c/Program Files/PostgreSQL/17/bin/psql.exe" -h localhost -U postgres -c "\l" | grep ejbca
```
📸 `04-postgresql-database-ejbca.png`

## Étape 4 — Extraire le code source EJBCA (déjà téléchargé)

```bash
cd /c/Users/Ahmad/Downloads/ejbca-demo-tmp
unzip -q /c/Users/Ahmad/Downloads/ejbca-setup/ejbca-ce.zip
mv Keyfactor-ejbca-ce-* ejbca-src
ls ejbca-src
```
📸 `05-ejbca-source-extracted.png` (optionnel)

## Étape 5 — Extraire WildFly (déjà téléchargé)

```bash
cd /c/Users/Ahmad/Downloads/ejbca-demo-tmp
unzip -q /c/Users/Ahmad/Downloads/ejbca-setup/wildfly.zip
ls wildfly-39.0.1.Final
```

## Étape 6 — Config EJBCA (database.properties / install.properties)

```bash
cd /c/Users/Ahmad/Downloads/ejbca-demo-tmp/ejbca-src
cat > conf/database.properties << 'EOF'
datasource.jndi-name=EjbcaDS
database.name=postgres
database.url=jdbc:postgresql://127.0.0.1:5432/ejbca
database.driver=org.postgresql.Driver
database.username=ejbca
database.password=ejbca
EOF
cat > conf/install.properties << 'EOF'
ca.name=ManagementCA
ca.dn=CN=Master-SSI Management CA,O=Master-SSI-2026,C=SN
ca.tokentype=soft
ca.tokenpassword=null
ca.keytype=RSA
ca.keyspec=4096
ca.signaturealgorithm=SHA256WithRSA
ca.validity=3650
ca.policy=null
EOF
cat conf/database.properties
```
📸 `06-config-database-properties.png`

## Étape 7 — Build EJBCA (`ant deployear`)

⚠️ Cette étape peut prendre plusieurs minutes même avec le cache Ivy déjà rempli — normal, on
attend cette fois avec le chrono affiché à l'écran pour la capture finale.

```bash
export APPSRV_HOME="/c/Users/Ahmad/Downloads/ejbca-demo-tmp/wildfly-39.0.1.Final"
export EJBCA_HOME="/c/Users/Ahmad/Downloads/ejbca-demo-tmp/ejbca-src"
cd "$EJBCA_HOME"
ant clean
ant deployear
```
📸 `07-ant-build-successful.png` (dernières lignes, `BUILD SUCCESSFUL`)

## Nettoyage final (après toutes les captures)

```bash
rm -rf /c/Users/Ahmad/Downloads/ejbca-demo-tmp
```

---

## Et après ?

Une fois ces captures faites, on **reprend l'installation réelle** (`C:\ejbca-tools`), qui elle
n'a jamais été touchée pendant cette reproduction : WildFly y tourne toujours, la Management CA
et le SuperAdmin y existent toujours. Prochaine étape réelle : télécharger et importer le
certificat `.p12` du SuperAdmin depuis `https://localhost:8443/ejbca/ra/`.
