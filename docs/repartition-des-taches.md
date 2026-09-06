# Répartition des tâches

## Ahmad Diop — branche `pki-ejbca-setup`

- [ ] Docker Compose EJBCA + MariaDB (`infra/ejbca/docker-compose.yml`, déjà initialisé)
- [ ] Création du SuperAdmin
- [ ] Création de la Root CA (profil, crypto token, DN)
- [ ] (Bonus) Création d'une Sub CA
- [ ] Création des profils de certificat/End Entity pour un certificat serveur
- [ ] Émission du certificat serveur à partir de la CSR fournie par le membre 2
- [ ] Export des artefacts (`root-ca.pem`, `chain.pem`, `server-cert.pem`) pour le membre 2
- [ ] Donner à Mame Fama un accès EJBCA (compte RA/Admin) pour qu'elle puisse réaliser
      elle-même la démonstration de révocation (voir sa section ci-dessous)
- [ ] Rédiger la section 5 du rapport (installation + configuration EJBCA, avec ses propres
      captures d'écran) — voir `Rapport_PKI_EJBCA_GROUPE-2.docx`

Référence : [../docs/guide-ejbca.md](guide-ejbca.md)

## Papa Mamadou — branche `webserver-https-migration`

- [ ] Générer la CSR (`certs/server.csr` + `server.key`)
- [ ] Monter un serveur web de démo en HTTP simple
- [ ] Récupérer le certificat émis par le membre 1
- [ ] Reconfigurer le serveur en HTTPS (Nginx ou Apache, configs déjà présentes dans
      `infra/webserver/`)
- [ ] Mettre en place la redirection HTTP → HTTPS
- [ ] Durcissement TLS (protocoles, ciphers, HSTS)
- [ ] Vérifications de base (`openssl s_client`, `curl --cacert`)
- [ ] Rédiger la section 6 du rapport (migration HTTP→HTTPS, avec ses propres captures)

Référence : [../docs/guide-https.md](guide-https.md)

## Mame Fama — branche `docs-rapport-tests`

Rôle technique à part entière : audit de sécurité indépendant + démonstration de révocation,
en plus de la compilation finale du rapport. Elle doit manipuler EJBCA et le serveur elle-même,
pas seulement rédiger ce que les deux autres ont fait.

- [ ] **Démonstration de révocation** (hands-on, dans EJBCA) : révoquer le certificat serveur
      émis par Ahmad, publier/rafraîchir la CRL, puis prouver côté client que le certificat
      révoqué est rejeté (`openssl verify -crl_check`, ou nouvelle tentative `curl`/navigateur).
      Voir scénario 5 ci-dessous.
- [ ] **Audit de sécurité TLS indépendant** du serveur migré par Papa Mamadou :
      `testssl.sh https://pki-demo.local` et/ou
      `nmap --script ssl-enum-ciphers -p 443 pki-demo.local`, analyse des résultats
      (protocoles/ciphers acceptés, note de sécurité, recommandations).
- [ ] **Comparatif CA privée vs CA publique** : tester l'accès à un site public HTTPS
      (ex. `curl -v https://google.com`) vs notre site (avec/sans `--cacert`), expliquer
      dans le rapport pourquoi le comportement diffère (magasins de confiance système).
- [ ] Exécuter et documenter les scénarios de test 1 à 4 ci-dessous (captures à l'appui).
- [ ] Compiler le rapport final : rédiger l'introduction, le contexte/objectifs, les
      difficultés rencontrées (à collecter auprès des 3), les perspectives et la conclusion ;
      intégrer les sections 5 et 6 écrites par Ahmad et Papa.
- [ ] Mettre à jour et finaliser le PowerPoint à partir du rapport compilé.
- [ ] Enregistrer la vidéo de démonstration (`VIDEO/script-video-demo.md`), y compris la
      démo de révocation qu'elle a réalisée.

### Scénarios de test à documenter

1. Accès HTTP avant migration → aucune confidentialité (capture Wireshark ou `curl -v`).
2. Accès HTTPS après migration, avec `--cacert root-ca.pem` → connexion de confiance.
3. Accès HTTPS sans faire confiance à la Root CA → erreur de certificat (preuve que ce n'est
   pas une CA publique).
4. Redirection automatique HTTP → HTTPS (code 301/308).
5. Révocation d'un certificat dans EJBCA → apparition dans la CRL → refus côté client
   (réalisée et documentée par Mame Fama, plus obligatoire dans ce rapport).

## Workflow Git

1. Chacun travaille sur sa branche (`git checkout -b <branche>` depuis `main`).
2. Commits réguliers et explicites.
3. Pull Request vers `main` en fin de tâche, relecture par au moins un autre membre.
4. `main` doit toujours contenir une version qui fonctionne (démo reproductible).
