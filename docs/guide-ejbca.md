# Guide — Déployer EJBCA et émettre un certificat serveur

Branche concernée : `pki-ejbca-setup`

## 1. Lancer EJBCA

```bash
cd infra/ejbca
docker compose up -d
docker compose logs -f ejbca
```

Attendre le message indiquant qu'EJBCA est prêt, puis récupérer l'URL (par défaut) :
`https://localhost:8443/ejbca/adminweb/` (Admin Web) et `https://localhost:8443/ejbca/ra/` (RA Web).

## 2. Créer le SuperAdmin

1. Aller sur **RA Web** → `Request new certificate` → `Make New Request`.
2. Template : `ENDUSER`, génération de clé "By the CA", RSA 2048.
3. Common Name : `SuperAdmin`, définir un username/mot de passe (ex. `superadmin` / `foo123`).
4. Télécharger le certificat au format **PKCS#12** (.p12).
5. Importer le .p12 dans le navigateur (gestionnaire de certificats client).
6. Dans **Admin Web** → `System Configuration` → `Roles and Access Rules`, ajouter ce certificat
   au rôle **Super Administrator Role**.
7. Se reconnecter à Admin Web en présentant ce certificat client → accès complet.

## 3. Créer la Root CA

Dans **Admin Web** → `Certification Authorities` :

1. **Certificate Profile** : cloner `ROOTCA`, adapter (ex. RSA 4096, validité 30 ans).
2. **Crypto Token** : créer un crypto token logiciel (soft keystore), générer les clés
   (signature, chiffrement, testkey).
3. **CA** : `Add CA`, choisir le crypto token, définir le DN (ex.
   `CN=Master-SSI Root CA,O=Master-SSI-2026,C=FR`), la validité, le profil de certificat,
   la durée d'expiration de la CRL.

## 4. (Optionnel) Créer une Sub CA

Même procédure que la Root CA, en cochant "Signed by" = Root CA créée précédemment,
profil `SUBCA`, validité plus courte (ex. 10 ans).

## 5. Émettre un certificat serveur

1. **Certificate Profile** : cloner `SERVER`, vérifier les extensions (Key Usage,
   Extended Key Usage = serverAuth, SAN autorisé).
2. **End Entity Profile** : cloner un profil permettant de saisir CN + SAN (DNS name).
3. Dans **RA Web**, `Make New Request` :
   - CA : Sub CA (ou Root CA si pas de Sub CA).
   - Common Name : nom du serveur de démo (ex. `pki-demo.local`).
   - SAN : `DNS:pki-demo.local,DNS:localhost,IP:127.0.0.1`.
   - Génération de clé : "On server" (fournir un CSR généré côté serveur web) — voir
     [guide-https.md](guide-https.md) pour générer la CSR avec OpenSSL.
4. Récupérer le certificat émis (PEM) + la chaîne (Root/Sub CA) pour les remettre à la
   branche `webserver-https-migration`.

## 6. Récupérer les artefacts pour les autres membres

Placer les fichiers suivants dans `infra/ejbca/output/` (créé localement, non versionné avec les
clés privées réelles — voir `.gitignore`) :

- `root-ca.pem` — certificat public de la Root CA (à distribuer aux clients de test).
- `server-cert.pem` — certificat serveur émis.
- `chain.pem` — chaîne complète (server + Sub CA + Root CA).

> ⚠️ Ne jamais committer de clé privée réelle dans le repo Git, même pour la démo.
