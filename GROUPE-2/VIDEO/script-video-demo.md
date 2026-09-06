# Script de la vidéo de démonstration (5–8 minutes)

À enregistrer avec OBS Studio, Windows Xbox Game Bar (Win+G) ou tout logiciel de capture d'écran.
Placer le fichier final ici : `GROUPE-2/VIDEO/demo-pki-ejbca-groupe-2.mp4`.

## Plan de la vidéo

1. **Intro (30s)** — À l'oral, face caméra ou voix off : présenter le sujet, le groupe, les 3 membres
   et leurs rôles.

2. **Démarrage de la PKI (1 min)**
   - Montrer le terminal : `docker compose up -d` dans `infra/ejbca/`.
   - Montrer `docker compose ps` (conteneurs up).
   - Ouvrir l'interface EJBCA dans le navigateur.

3. **Configuration de la CA (1–2 min)**
   - Montrer rapidement l'Admin Web : la Root CA créée, le profil de certificat serveur.
   - Montrer la RA Web et le certificat serveur déjà émis (ou émettre en direct si le temps le
     permet).

4. **État HTTP avant migration (30s)**
   - `curl -v http://pki-demo.local/` → montrer l'absence de chiffrement (pas de TLS).

5. **Migration et test HTTPS (2 min)**
   - Montrer le fichier de config HTTPS (Nginx/Apache) avec le certificat.
   - Relancer/recharger le serveur web.
   - `curl -v --cacert root-ca.pem https://pki-demo.local/` → succès.
   - Ouvrir `https://pki-demo.local/` dans le navigateur → cadenas, détails du certificat
     (émetteur = notre Root CA).

6. **Preuve de la chaîne de confiance (1 min)**
   - Refaire le test HTTPS **sans** `--cacert` (ou sans avoir importé la Root CA dans le
     navigateur) → montrer l'erreur de confiance.
   - Expliquer pourquoi (CA privée, pas dans les magasins de confiance publics).

7. **Redirection HTTP → HTTPS (30s)**
   - `curl -v http://pki-demo.local/` après migration → montrer le code 301/308 vers https://.

8. **(Bonus) Révocation (1 min)** — si réalisé.

9. **Conclusion (30s)** — Résumer ce qui a été démontré.

## Conseils

- Zoomer le terminal/navigateur pour que le texte soit lisible dans la vidéo.
- Couper les temps morts au montage (ou simplement enchaîner les actions sans pause).
- Exporter en MP4, résolution 1080p si possible, taille raisonnable pour l'envoi par email.
