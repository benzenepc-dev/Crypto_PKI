# Plan de tests final — Validation de la PKI et de la migration HTTPS

Dernière étape du projet (branche `docs-rapport-tests`, Mame Fama Dieng), réalisée en équipe
avec Ahmad Diop (CA/EJBCA) et Papa Mamadou Bâ (serveur web). Tous les échanges réseau passent
par **Tailscale** (voir [guide-tailscale-tests-reseau.md](guide-tailscale-tests-reseau.md)).

## Prérequis avant de commencer

- [ ] Les 3 membres sont connectés sur le même réseau Tailscale (`tailscale status` montre les
      3 machines chez chacun)
- [ ] Ahmad : EJBCA/WildFly tourne et écoute sur `0.0.0.0:8443` (vérifié via `netstat -an | grep 8443`)
- [ ] Papa : Nginx tourne avec le certificat signé par la CA, écoute sur `0.0.0.0:80` et `0.0.0.0:443`
- [ ] Fama : dispose du dépôt Git cloné sur sa VM Kali (`~/Desktop/Crypto_PKI/`), avec le dossier
      `pki-exports/` contenant `root-ca.pem`, `chain.pem`, `server-cert.pem`

Remplacer dans toutes les commandes ci-dessous :
- `<IP_AHMAD>` par l'IP Tailscale d'Ahmad (ex. `100.98.55.7`)
- `<IP_PAPA>` par l'IP Tailscale de Papa
- `<IP_FAMA>` par l'IP Tailscale de Fama (sa VM Kali, ex. `100.115.119.123`)

---

## Test 1 — Accès HTTP non chiffré (avant redirection)

**Principe** : sans TLS, toutes les données circulent en clair sur le réseau ; n'importe qui
interceptant le trafic (sniffing) peut les lire.

**Objectif** : prouver que le port 80 du serveur de Papa ne sert jamais de contenu en clair et
redirige systématiquement vers HTTPS.

**Qui** : Fama, depuis sa VM Kali.

```bash
curl -v http://<IP_PAPA>/
```

**Résultat attendu** : code de réponse **301** ou **308**, avec un en-tête `Location:` pointant
vers `https://...`. Aucun contenu HTML ne doit être renvoyé en clair sur le port 80.

📸 Capture à prendre : sortie complète de la commande (montrant le code 301/308).

---

## Test 2 — Connexion HTTPS avec confiance en la CA

**Principe** : un client qui possède le certificat racine d'une CA peut vérifier toute la chaîne
de confiance jusqu'au certificat présenté par le serveur.

**Objectif** : prouver que le certificat émis par la Management CA d'Ahmad est valide et permet
d'établir une connexion HTTPS de confiance vers le serveur de Papa.

**Qui** : Fama, depuis sa VM Kali.

```bash
curl -v --cacert ~/Desktop/Crypto_PKI/pki-exports/root-ca.pem \
  --resolve pki-demo.local:443:<IP_PAPA> \
  https://pki-demo.local/
```

**Résultat attendu** : réponse **200 OK**, aucun avertissement de certificat, la ligne
`SSL certificate verify ok` apparaît dans la sortie verbeuse.

📸 Capture à prendre : sortie complète montrant `HTTP/1.1 200` ou `HTTP/2 200` et la vérification
du certificat réussie.

---

## Test 3 — Rejet sans la CA importée (preuve que c'est une CA privée)

**Principe** : contrairement à une CA publique (déjà intégrée dans tous les systèmes
d'exploitation et navigateurs), une CA privée n'est reconnue que si on lui fait confiance
explicitement (import du certificat racine).

**Objectif** : démontrer concrètement la différence de comportement entre CA privée et CA
publique.

**Qui** : Fama, depuis sa VM Kali.

Sans faire confiance à la CA (sans `--cacert`) :
```bash
curl -v --resolve pki-demo.local:443:<IP_PAPA> https://pki-demo.local/
```
**Résultat attendu** : échec avec l'erreur
`SSL certificate problem: unable to get local issuer certificate`.

Comparaison avec une CA publique reconnue :
```bash
curl -sI https://www.google.com/
```
**Résultat attendu** : réussite immédiate, **sans aucune option spéciale** — preuve que les CA
publiques sont préinstallées dans le magasin de confiance du système, contrairement à une CA
privée comme la vôtre.

📸 Capture à prendre : les deux résultats côte à côte (échec sur pki-demo.local, succès sur
google.com).

---

## Test 4 — Inspection manuelle de la chaîne de certificats

**Principe** : `openssl s_client` simule une connexion TLS complète et affiche exactement les
certificats envoyés par le serveur, sans passer par un navigateur.

**Objectif** : vérifier en détail le contenu de la chaîne de certificats (sujet, émetteur,
validité).

**Qui** : Fama, depuis sa VM Kali.

```bash
openssl s_client -connect <IP_PAPA>:443 -servername pki-demo.local -showcerts </dev/null
```

**Résultat attendu** : dans la sortie, vérifier :
- `subject=CN=pki-demo.local` (le certificat serveur)
- `issuer=CN=Master-SSI Management CA,O=Master-SSI-2026,C=SN`
- Deux certificats affichés dans la chaîne (serveur + CA)

📸 Capture à prendre : la section `Certificate chain` de la sortie.

---

## Test 5 — Révocation de certificat et vérification CRL

**Principe** : un certificat compromis, expiré prématurément ou plus utilisé doit pouvoir être
invalidé avant sa date d'expiration normale. La CRL (Certificate Revocation List) est la liste
publiée par la CA des certificats révoqués.

**Objectif** : démontrer le cycle de vie complet d'un certificat : émission → usage → révocation
→ publication CRL → rejet par un client vigilant.

**Étape 1 — Révocation (Ahmad)**, dans l'Admin Web (`https://<IP_AHMAD>:8443/ejbca/adminweb/`) :
1. Menu `RA Functions` → `Search End Entities`
2. Rechercher `pkidemo-server`
3. Cliquer sur le résultat → bouton **Revoke**
4. Choisir une raison (ex. "Cessation of Operation") → confirmer

**Étape 2 — Vérification de la CRL (Ahmad)** :
1. Menu `CA Functions` → `Certification Authorities` → `ManagementCA`
2. Vérifier la date de dernière publication de la CRL (doit être récente)
3. Récupérer l'URL de téléchargement de la CRL (visible dans les détails de la CA, ou via) :
   ```
   https://<IP_AHMAD>:8443/ejbca/publicweb/webdist/certdist?cmd=crl&issuer=CN=Master-SSI+Management+CA,O=Master-SSI-2026,C=SN
   ```

**Étape 3 — Téléchargement de la CRL et vérification (Fama, VM Kali)** :
```bash
curl -sk "https://<IP_AHMAD>:8443/ejbca/publicweb/webdist/certdist?cmd=crl&issuer=CN=Master-SSI+Management+CA,O=Master-SSI-2026,C=SN" \
  -o managementca.crl.der
openssl crl -inform DER -in managementca.crl.der -outform PEM -out managementca.crl.pem
openssl crl -in managementca.crl.pem -noout -text | grep -A2 "Revoked Certificates"
```
**Résultat attendu** : le numéro de série du certificat de `pkidemo-server` apparaît dans la
liste des certificats révoqués.

Vérification finale de rejet :
```bash
openssl verify -crl_check \
  -CAfile ~/Desktop/Crypto_PKI/pki-exports/root-ca.pem \
  -CRLfile managementca.crl.pem \
  ~/Desktop/Crypto_PKI/pki-exports/server-cert.pem
```
**Résultat attendu** : `error 23 at 0 depth lookup: certificate revoked` (preuve que le
certificat révoqué est bien rejeté).

📸 Captures à prendre : la révocation dans l'Admin Web, la CRL mise à jour, et le résultat
`certificate revoked` d'openssl.

---

## Test 6 — Audit de sécurité TLS indépendant

**Principe** : un audit mené par une personne différente de celle qui a configuré le serveur
évite les biais de configuration et révèle objectivement les faiblesses éventuelles.

**Objectif** : vérifier quels protocoles et suites cryptographiques le serveur de Papa accepte
réellement, indépendamment de sa propre configuration déclarée.

**Qui** : Fama, depuis sa VM Kali (elle n'a pas configuré le serveur elle-même).

```bash
sudo apt update && sudo apt install testssl.sh -y
testssl.sh --ip <IP_PAPA> https://pki-demo.local:443
```

Alternative plus rapide avec nmap :
```bash
nmap --script ssl-enum-ciphers -p 443 <IP_PAPA>
```

**Résultat attendu** :
- TLS 1.2 et/ou TLS 1.3 acceptés
- SSLv3, TLS 1.0, TLS 1.1 refusés (protocoles obsolètes)
- Pas de suites de chiffrement faibles (RC4, DES, suites NULL ou EXPORT)

📸 Capture à prendre : le résumé des protocoles/ciphers de testssl.sh ou nmap.

---

## Test 7 — Comparatif CA privée vs CA publique (synthèse)

**Principe** : ce test résume visuellement, en une seule séquence, pourquoi une CA privée
nécessite une configuration de confiance explicite alors qu'une CA publique fonctionne "out of
the box".

**Objectif** : conclusion pédagogique de la partie tests du rapport.

**Qui** : Fama, depuis sa VM Kali.

```bash
echo "=== CA PUBLIQUE (Google), aucune option requise ==="
curl -sI https://www.google.com/ | head -1

echo "=== CA PRIVEE, sans fichier de confiance : echec attendu ==="
curl -sI --resolve pki-demo.local:443:<IP_PAPA> https://pki-demo.local/ 2>&1 | head -3

echo "=== CA PRIVEE, avec root-ca.pem : succes ==="
curl -sI --cacert ~/Desktop/Crypto_PKI/pki-exports/root-ca.pem \
  --resolve pki-demo.local:443:<IP_PAPA> https://pki-demo.local/ | head -1
```

📸 Capture à prendre : les trois résultats dans le même terminal, pour un visuel direct dans le
rapport.

---

## Récapitulatif — qui fait quoi

| Test | Acteur principal | Machine |
|---|---|---|
| 1. HTTP → redirection | Fama | VM Kali |
| 2. HTTPS avec confiance | Fama | VM Kali |
| 3. Rejet sans CA / comparatif | Fama | VM Kali |
| 4. Inspection openssl | Fama | VM Kali |
| 5. Révocation + CRL | Ahmad (révocation) puis Fama (vérification) | Admin Web (Ahmad) + VM Kali (Fama) |
| 6. Audit TLS indépendant | Fama | VM Kali |
| 7. Comparatif final | Fama | VM Kali |

## Une fois tous les tests faits

- [ ] Rassembler toutes les captures dans `docs/captures/` (numérotation `39-...` et suivantes)
- [ ] Rédiger la section "Tests et validation" du rapport avec ces résultats
- [ ] Mettre à jour `docs/repartition-des-taches.md` (cocher les tâches terminées)
- [ ] Passer à la compilation finale du rapport, du PPT et de la vidéo
