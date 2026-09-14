# Checklist finale — Répartition des tâches

Approche actuelle : **installation native** (JDK + Ant + WildFly + PostgreSQL), sans Docker.
Voir le détail technique dans [journal-installation-ejbca-natif.md](journal-installation-ejbca-natif.md).

## Ahmad Diop — branche `pki-ejbca-setup`

- [x] Installer JDK 17, Apache Ant, PostgreSQL 17
- [x] Télécharger et compiler EJBCA Community (`ant deployear` → BUILD SUCCESSFUL)
- [x] Installer et configurer WildFly 39 (datasource PostgreSQL, mémoire heap)
- [x] Créer la Management CA (`ca init`)
- [x] Créer le rôle Super Administrator + le compte SuperAdmin (`ra addendentity`)
- [ ] Télécharger et importer le certificat SuperAdmin (.p12) dans le navigateur — **en cours**
- [ ] Se connecter à l'Admin Web en SuperAdmin
- [ ] Créer un profil de certificat "SERVER" + un profil d'End Entity pour le serveur web de démo
- [ ] Émettre le certificat serveur (à partir de la CSR de Papa, ou générer un couple clé/CSR)
- [ ] Exporter `root-ca.pem` + `chain.pem` + `server-cert.pem` pour Papa
- [ ] Prendre les captures manquantes (install EJBCA, Admin Web connecté, émission du certificat)
- [ ] Rédiger sa section du rapport (installation + configuration EJBCA)

## Papa Mamadou — branche `webserver-https-migration`

Repart de zéro (l'ancienne approche Docker/Nginx a été abandonnée avec le reste).

- [ ] Choisir et installer un serveur web en natif (Apache, Nginx ou IIS — pas de Docker)
- [ ] Monter une page de démo en HTTP simple (port 80)
- [ ] Générer une CSR (clé privée + demande) pour le serveur
- [ ] Envoyer la CSR à Ahmad, récupérer le certificat signé + la chaîne de la CA
- [ ] Configurer HTTPS avec le certificat (port 443)
- [ ] Mettre en place la redirection HTTP → HTTPS
- [ ] Durcir TLS (protocoles/ciphers modernes, en-tête HSTS)
- [ ] Vérifier avec `openssl s_client` / `curl`
- [ ] Prendre ses captures et rédiger sa section du rapport

## Mame Fama — branche `docs-rapport-tests`

- [ ] Démonstration de révocation d'un certificat + vérification CRL dans EJBCA
- [ ] Audit de sécurité TLS indépendant (`testssl.sh` / `nmap`) sur le serveur de Papa
- [ ] Comparatif CA privée vs CA publique (`curl` sur un vrai site HTTPS vs le nôtre)
- [ ] Exécuter et documenter les scénarios de test :
  1. Accès HTTP avant migration (non chiffré)
  2. Accès HTTPS avec confiance en la Root CA
  3. Rejet de la connexion sans la Root CA importée
  4. Redirection automatique HTTP → HTTPS
  5. Révocation d'un certificat → refus côté client

## Commun / Équipe (à la fin)

- [ ] Rassembler toutes les captures dans `docs/captures/` et les insérer dans le Word
- [ ] Finaliser `GROUPE-2/Rapport_PKI_EJBCA_GROUPE-2.docx` (remplacer les placeholders restants)
- [ ] Finaliser `GROUPE-2/Presentation_PKI_EJBCA_GROUPE-2.pptx`
- [ ] Enregistrer la vidéo de démonstration
- [ ] Zipper `GROUPE-2/` et l'envoyer à **nds.ucad.fst.lacgaa@gmail.com**
