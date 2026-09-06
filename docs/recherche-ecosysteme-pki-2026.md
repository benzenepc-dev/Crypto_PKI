# Recherche — Écosystème PKI / EJBCA en 2026

Synthèse préparatoire avant répartition des tâches. Sources listées en bas.

## 1. Contexte : pourquoi une PKI privée en 2026

- Les certificats publics (Let's Encrypt, CA publiques) ne conviennent pas aux usages internes
  (intranets, IoT, authentification mutuelle mTLS, signature de code interne) car ils exigent un
  nom de domaine public validable. Une **PKI privée** (EJBCA, step-ca, Vault PKI, Microsoft ADCS)
  reste la norme pour ces cas.
- Tendance 2025-2026 : durée de vie des certificats TLS publics réduite (le CA/Browser Forum a
  voté la réduction progressive vers 47 jours d'ici 2029), ce qui pousse à l'**automatisation**
  de l'émission/renouvellement (ACME, EST, SCEP) même en interne.
- Migration post-quantique amorcée : NIST a standardisé ML-KEM, ML-DSA, SLH-DSA (2024). Les PKI
  modernes commencent à proposer des CA "crypto-agiles" pouvant émettre des certificats hybrides
  ou post-quantiques. Pour ce projet (cours), on reste sur RSA/ECDSA classique, mais c'est un point
  à mentionner dans le rapport comme perspective.

## 2. EJBCA en 2026

- **EJBCA Community** (open source, Apache 2.0) vs **EJBCA Enterprise** (Keyfactor) : la version
  Community suffit largement pour ce projet (CA, profils de certificats, RA Web, Admin Web, CRL/OCSP).
- Dernière version stable citée : EJBCA 9.5.x (2026). Fonctionne en conteneur Docker
  (`keyfactor/ejbca-ce`) avec une base **MariaDB** (ou PostgreSQL) pour stocker CA, certificats, CRL.
- Trois composants d'admin :
  - **Admin Web** (authentification par certificat client — SuperAdmin) : gestion CA, profils,
    rôles, CRL.
  - **RA Web** : interface de demande de certificat (End Entity), plus accessible pour un
    utilisateur/serveur.
  - **Public Web** : téléchargement de CRL, certificat CA public, etc.
- Concepts clés à maîtriser :
  - **Crypto Token** : conteneur de clés (soft keystore en base, ou HSM) utilisé par la CA.
  - **Certificate Profile** : gabarit définissant contraintes (usage clé, extensions, durée de
    validité) — profils par défaut `ROOTCA`, `SUBCA`, `ENDUSER`, `SERVER`, à cloner/adapter.
  - **End Entity Profile** : définit quels champs (CN, SAN, organisation) un demandeur peut fournir.
  - **CA hiérarchique** : Root CA (hors ligne idéalement) → Sub CA (opérationnelle) → certificats
    serveurs. Pour la démo, une CA à 1 ou 2 niveaux suffit.
  - **CRL / OCSP** : mécanismes de révocation, à démontrer si le temps le permet.

## 3. HTTP → HTTPS : bonnes pratiques 2026

- Toujours **rediriger** le trafic HTTP (port 80) vers HTTPS (port 443) au lieu de désactiver
  purement le port 80 (sinon les clients ne reçoivent aucune redirection).
- Configuration TLS recommandée : **TLS 1.2 minimum**, **TLS 1.3 préféré**, désactivation de
  SSLv3/TLS1.0/1.1, suites de chiffrement modernes (ECDHE + AES-GCM ou ChaCha20-Poly1305).
- Ajouter l'en-tête **HSTS** (`Strict-Transport-Security`) une fois HTTPS validé, pour forcer
  les futurs accès en HTTPS.
- Le certificat racine (Root CA) de notre PKI doit être importé dans le **truststore** du client
  de test (navigateur ou `curl --cacert`) pour éviter les erreurs de confiance — normal pour une
  CA privée, à bien expliquer dans le rapport (différence avec CA publique déjà dans les
  truststores système).
- Outils de vérification : `openssl s_client -connect host:443 -showcerts`,
  `curl -v --cacert root-ca.pem https://host/`, `testssl.sh`, l'onglet sécurité du navigateur.

## 4. Sources

- [EJBCA - The Open-Source Certificate Authority](https://www.ejbca.org/)
- [Get started with EJBCA open-source PKI](https://www.ejbca.org/use-cases/get-started-with-ejbca-pki/)
- [Tutorial - Start out with EJBCA Docker container](https://docs.keyfactor.com/ejbca/latest/tutorial-start-out-with-ejbca-docker-container)
- [Tutorial - Create your first Root CA using EJBCA](https://docs.keyfactor.com/ejbca/latest/tutorial-create-your-first-root-ca-using-ejbca)
- [EJBCA and Docker — Streamlining PKI Management and TLS Certificate Issuance (Docker Blog)](https://www.docker.com/blog/ejbca-and-docker-streamlining-pki-management-and-tls-certificate-issuance/)
- [PKI Deployment | Keyfactor Docs](https://docs.keyfactor.com/solution-areas/latest/deployment)
- [The Art of Hacking — Cryptography & PKI (h4cker repo)](https://github.com/The-Art-of-Hacking/h4cker/tree/master/cybersecurity-domains/cryptography-pki/cryptography-and-pki)
  — contient notamment `cert_openssl.md` (guide OpenSSL), `tutorials/` (PKI, TLS/SSL, migration
  post-quantique) et `labs/` (exercices pratiques PKI), utiles comme références complémentaires
  pour le rapport et les tests.
