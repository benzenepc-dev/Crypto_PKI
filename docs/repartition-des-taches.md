# Répartition des tâches

## Ahmad Diop — branche `pki-ejbca-setup`

- [ ] Docker Compose EJBCA + MariaDB (`infra/ejbca/docker-compose.yml`, déjà initialisé)
- [ ] Création du SuperAdmin
- [ ] Création de la Root CA (profil, crypto token, DN)
- [ ] (Bonus) Création d'une Sub CA
- [ ] Création des profils de certificat/End Entity pour un certificat serveur
- [ ] Émission du certificat serveur à partir de la CSR fournie par le membre 2
- [ ] Export des artefacts (`root-ca.pem`, `chain.pem`, `server-cert.pem`) pour le membre 2
- [ ] (Bonus) Démonstration de révocation d'un certificat + CRL

Référence : [../docs/guide-ejbca.md](guide-ejbca.md)

## Papa Mamadou — branche `webserver-https-migration`

- [ ] Générer la CSR (`certs/server.csr` + `server.key`)
- [ ] Monter un serveur web de démo en HTTP simple
- [ ] Récupérer le certificat émis par le membre 1
- [ ] Reconfigurer le serveur en HTTPS (Nginx ou Apache, configs déjà présentes dans
      `infra/webserver/`)
- [ ] Mettre en place la redirection HTTP → HTTPS
- [ ] Durcissement TLS (protocoles, ciphers, HSTS)
- [ ] Vérifications (`openssl s_client`, `curl --cacert`, testssl.sh)

Référence : [../docs/guide-https.md](guide-https.md)

## Mame Fatou — branche `docs-rapport-tests`

- [ ] Rédaction du rapport : concepts PKI/X.509, chaîne de confiance, CRL/OCSP
- [ ] Explication des choix techniques (EJBCA, hiérarchie de CA, algorithmes/tailles de clé)
- [ ] Scénarios de test/validation (voir ci-dessous) + captures d'écran
- [ ] Relecture des README/guides des deux autres branches
- [ ] Support de soutenance (slides) si demandé

### Scénarios de test à documenter

1. Accès HTTP avant migration → aucune confidentialité (capture Wireshark ou `curl -v`).
2. Accès HTTPS après migration, avec `--cacert root-ca.pem` → connexion de confiance.
3. Accès HTTPS sans faire confiance à la Root CA → erreur de certificat (preuve que ce n'est
   pas une CA publique).
4. Redirection automatique HTTP → HTTPS (code 301/308).
5. (Bonus) Révocation d'un certificat dans EJBCA → apparition dans la CRL → refus côté client.

## Workflow Git

1. Chacun travaille sur sa branche (`git checkout -b <branche>` depuis `main`).
2. Commits réguliers et explicites.
3. Pull Request vers `main` en fin de tâche, relecture par au moins un autre membre.
4. `main` doit toujours contenir une version qui fonctionne (démo reproductible).
