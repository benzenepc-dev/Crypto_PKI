# Guide — Migration HTTP → HTTPS avec le certificat EJBCA

Branche concernée : `webserver-https-migration`

## 1. Générer la CSR côté serveur (avant EJBCA)

```bash
mkdir -p certs
openssl genrsa -out certs/server.key 2048
openssl req -new -key certs/server.key -out certs/server.csr \
  -subj "/CN=pki-demo.local/O=Master-SSI-2026/C=FR" \
  -addext "subjectAltName=DNS:pki-demo.local,DNS:localhost,IP:127.0.0.1"
```

Transmettre `certs/server.csr` au membre en charge d'EJBCA (branche `pki-ejbca-setup`),
qui la soumet dans la RA Web (mode "génération de clé côté serveur" / CSR upload) et renvoie :
- `server-cert.pem` (certificat émis)
- `chain.pem` (Sub CA + Root CA)
- `root-ca.pem` (Root CA seule, pour le truststore client)

Placer ces fichiers dans `certs/`.

## 2. Étape 1 — démo HTTP (avant migration)

Voir `infra/webserver/nginx/http-only.conf` ou `infra/webserver/apache/http-only.conf`.
Servir une page simple sur le port 80, capturer une preuve (`curl -v http://...`) montrant
l'absence de chiffrement.

## 3. Étape 2 — migration vers HTTPS

### Nginx

Utiliser `infra/webserver/nginx/https.conf` :

```bash
docker run --rm -d --name web-demo \
  -p 80:80 -p 443:443 \
  -v "$(pwd)/infra/webserver/nginx/https.conf:/etc/nginx/conf.d/default.conf:ro" \
  -v "$(pwd)/certs:/etc/nginx/certs:ro" \
  -v "$(pwd)/infra/webserver/site:/usr/share/nginx/html:ro" \
  nginx:latest
```

### Apache (alternative)

Utiliser `infra/webserver/apache/https.conf` avec l'image `httpd:latest` (activer `mod_ssl`).

## 4. Vérifications

```bash
# Redirection HTTP -> HTTPS
curl -v http://localhost/

# Handshake TLS et chaîne de certificats
openssl s_client -connect localhost:443 -servername pki-demo.local -showcerts

# Requête HTTPS en faisant confiance à notre Root CA
curl -v --cacert certs/root-ca.pem --resolve pki-demo.local:443:127.0.0.1 \
  https://pki-demo.local/
```

Vérifier :
- Le certificat présenté correspond bien à celui émis par la CA (CN/SAN, émetteur).
- `curl` réussit sans erreur de confiance grâce à `--cacert root-ca.pem`.
- Sans `--cacert`, `curl` doit refuser (preuve que ce n'est pas une CA publique reconnue).
- HTTP (port 80) redirige bien vers HTTPS (code 301/308).

## 5. Durcissement (bonus)

- Limiter à TLS 1.2/1.3 uniquement.
- Ajouter l'en-tête `Strict-Transport-Security: max-age=31536000; includeSubDomains`.
- Scanner avec `testssl.sh` ou `nmap --script ssl-enum-ciphers -p 443 localhost`.
