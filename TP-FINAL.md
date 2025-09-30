# TP OpenSSL

## Contexte & environnement réel du TP

- Réalisé le: 30/09/2025 (session de révision sécurité système)
- Plateforme: WSL2 Ubuntu 22.04 sur machine perso (Windows 11)
- Version OpenSSL utilisée: `OpenSSL 3.0.x` (vérifiée avec `openssl version`)
- Petit imprévu rencontré au début: j’avais oublié d’ajouter le SAN, le navigateur ignorait le CN et affichait une alerte supplémentaire.

J’essaie de garder les commandes minimalistes. Là où j’hésitais (ex: durée exacte du serveur), je me suis aligné sur la limite < 398 jours supportée par les principaux navigateurs.

> Environnement visé : shell Bash Linux ou WSL. OpenSSL récent (>=1.1.1). Noms internes utilisés :
> `app.lab.securecorp.internal`, `www.app.lab.securecorp.internal`, `pki.lab.securecorp.internal`, `ocsp.lab.securecorp.internal`
>
> Ajouter si besoin dans /etc/hosts :
>
> ```text
> 127.0.0.1 app.lab.securecorp.internal www.app.lab.securecorp.internal pki.lab.securecorp.internal ocsp.lab.securecorp.internal
> ```

## 1. Structure initiale & préparation

```bash
mkdir -p pki/{certs,crl,newcerts,private,csr,ocsp,server}
chmod 700 pki/private
touch pki/index.txt
echo 1000 > pki/serial
echo 1000 > pki/crlnumber
```

Fichier d’attributs optionnel (je l’ai laissé vide pour ne pas me bloquer sur des doublons CN) : `pki/index.txt.attr`.

---

## 2. Configuration OpenSSL

### 2.2 Modèle fourni (annexe adapté en `openssl.cnf` pour `demoCA/`)

Version plus « académique » issue du modèle. Je l’ai gardée telle quelle sauf adaptation des domaines.

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = ./demoCA
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
crlnumber         = $dir/crlnumber
certificate       = $dir/cacert.pem
private_key       = $dir/private/cakey.pem
default_md        = sha256
default_days      = 365
default_crl_days  = 30
x509_extensions   = usr_cert
policy            = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 2048
prompt              = no
default_md          = sha256
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca

[ req_distinguished_name ]
C  = FR
ST = IDF
L  = Paris
O  = ACME Corp
CN = ACME Corp Root CA

[ v3_ca ]
basicConstraints       = critical, CA:true, pathlen:0
keyUsage               = critical, keyCertSign, cRLSign
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer

[ usr_cert ]
basicConstraints       = CA:false
keyUsage               = critical, DigitalSignature, KeyEncipherment
extendedKeyUsage       = serverAuth
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
subjectAltName         = @alt_names

[ crl_ext ]
authorityKeyIdentifier = keyid:always

[ ocsp ]
basicConstraints       = CA:false
keyUsage               = critical, DigitalSignature
extendedKeyUsage       = OCSPSigning
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

[ alt_names ]
DNS.1 = app.lab.securecorp.internal
DNS.2 = www.app.lab.securecorp.internal
IP.1  = 127.0.0.1
```

### 2.3 Différences et choix

| Élément        | Version minimale (pki/)  | Modèle fourni (demoCA/) | Motif / commentaire                                      |
| -------------- | ------------------------ | ----------------------- | -------------------------------------------------------- |
| Répertoire     | ./pki                    | ./demoCA                | Convention personnelle vs modèle sujet                   |
| Fichiers CA    | ca.cert.pem / ca.key.pem | cacert.pem / cakey.pem  | Nommage plus explicite côté perso                        |
| Durée root     | 3650 (argument)          | 365 (config)            | J’ai forcé longue durée pour illustrer pratique courante |
| Profil serveur | server_cert              | usr_cert                | Deux noms pour même logique d’usage TLS                  |
| Section ocsp   | Ajoutée plus tard        | Prévue d’entrée         | Approche incrémentale vs complète                        |
| CRL params     | Ajout manuel             | default_crl_days 30     | Paramètre absent dans version courte                     |
| Politique DN   | state absente            | state exigée            | Simplification au début                                  |
| Alt names      | Oui                      | Oui                     | Indispensable (navigateurs)                              |

En résumé : j’ai d’abord compris chaque brique avec une config épurée, puis vérifié que je pouvais reproduire avec la structure attendue par le sujet.

### 2.4 Extensions de distribution (CDP / AIA)

Non incluses dans la version minimale pour rester focalisé sur les bases. En production :

- crlDistributionPoints : URL(s) de récupération automatique de la CRL.
- authorityInfoAccess : point(s) OCSP (statut ciblé) et parfois CA issuers.

Sans ces extensions les clients peuvent ne pas vérifier la révocation.

Exemple d’ajout (optionnel) :

```ini
[ usr_cert ]
crlDistributionPoints = URI:http://pki.lab.securecorp.internal/crl/ca.crl.pem
authorityInfoAccess   = OCSP;URI:http://ocsp.lab.securecorp.internal
```

Je les intégrerais après validation fonctionnelle locale.

---

## 3. Création de la CA racine

### 3.1 Clé privée CA

```bash
openssl genrsa -aes256 -out pki/private/ca.key.pem 4096
chmod 600 pki/private/ca.key.pem
```

### 3.2 Certificat auto-signé (root)

```bash
openssl req -config pki/openssl.cnf \
  -key pki/private/ca.key.pem \
  -new -x509 -days 3650 -sha256 -extensions v3_ca \
  -out pki/certs/ca.cert.pem
chmod 644 pki/certs/ca.cert.pem
```

### 3.3 Vérification rapide

```bash
openssl x509 -noout -text -in pki/certs/ca.cert.pem | head -n 30
```

Sortie:

```bash
Issuer: C=FR, O=DemoSecurity, CN=DemoSecurity Root CA
Subject: C=FR, O=DemoSecurity, CN=DemoSecurity Root CA
X509v3 Basic Constraints: critical
    CA:TRUE, pathlen:0
X509v3 Key Usage: critical
    Certificate Sign, CRL Sign
```

---

## 4. Certificat serveur

### 4.1 Génération clé privée serveur

```bash
openssl genrsa -out pki/server/server.key.pem 2048
chmod 600 pki/server/server.key.pem
```

### 4.2 CSR

Je génère une CSR où je force le SAN via un mini fichier de config dédié (pratique quand le `openssl.cnf` principal ne doit pas être modifié à chaque demande).

```bash
cat > pki/csr/server.csr.cnf <<'EOF'
[ req ]
default_bits       = 2048
default_md         = sha256
prompt             = no
distinguished_name = dn
req_extensions     = req_ext

[ dn ]
C=FR
O=DemoSecurity
CN=app.lab.securecorp.internal

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = app.lab.securecorp.internal
DNS.2 = www.app.lab.securecorp.internal
IP.1  = 127.0.0.1
EOF

openssl req -new -key pki/server/server.key.pem \
  -out pki/csr/server.csr.pem \
  -config pki/csr/server.csr.cnf
```

### 4.3 Signature par la CA

```bash
openssl ca -config pki/openssl.cnf \
  -extensions server_cert \
  -days 397 -notext -md sha256 \
  -in pki/csr/server.csr.pem \
  -out pki/server/server.cert.pem
chmod 644 pki/server/server.cert.pem
```

### 4.4 Vérification des usages

```bash
openssl x509 -in pki/server/server.cert.pem -noout -text | grep -E "(Key Usage|Extended Key Usage|Subject Alternative Name)" -A1
```

Sortie :

```bash
X509v3 Key Usage: critical
    Digital Signature, Key Encipherment
X509v3 Extended Key Usage:
    TLS Web Server Authentication
X509v3 Subject Alternative Name:
  DNS:app.lab.securecorp.internal, DNS:www.app.lab.securecorp.internal, IP Address:127.0.0.1
```

> Note : pas d’EKU `clientAuth` ajouté ici pour limiter l’usage du certificat. Un besoin d’authentification mutuelle justifierait un certificat dédié.

---

## 5. Révocation + CRL

Avant de révoquer, j’ai regardé le contenu de `pki/index.txt` pour voir le format :

```bash
V 350927095900Z  - 1000 unknown /C=FR/O=DemoSecurity/CN=app.lab.securecorp.internal
```

Après la commande de révocation, la ligne passe à l’état `R` avec une date :

```bash
R 250930100500Z 350927095900Z 1000 unknown /C=FR/O=DemoSecurity/CN=app.lab.securecorp.internal
```

(Les dates sont celles simulées dans le TP – en vrai elles dépendront de l’horloge système.)

### 5.1 Révocation (exemple volontaire)

```bash
openssl ca -config pki/openssl.cnf -revoke pki/server/server.cert.pem
```

Révocation avec raison explicite :

```bash
openssl ca -config pki/openssl.cnf -revoke pki/server/server.cert.pem -crl_reason keyCompromise
```

Sortie:

```bash
Using configuration from pki/openssl.cnf
Enter pass phrase for pki/private/ca.key.pem:
Revoking Certificate 1000.
Data Base Updated
```

### 5.2 Générer la CRL

```bash
openssl ca -config pki/openssl.cnf -gencrl -out pki/crl/ca.crl.pem
```

### 5.3 Contrôle de présence dans la CRL

```bash
openssl crl -in pki/crl/ca.crl.pem -noout -text | grep -i 'Revoked Certificates' -A5
```

Sortie (extrait) :

```bash
Revoked Certificates:
    Serial Number: 1000
        Revocation Date: Sep 30 10:05:00 2025 GMT
```

### 5.4 Distribution (principe)

- Hébergée via HTTP(s) sur un point interne : `http://pki.lab.securecorp.internal/crl/ca.crl.pem`.
- Serveur Web (Nginx/Apache) + permission lecture.
- Mise à jour régulière automatisée (cron) : régénération + publication atomique.
- Optionnel : signature timestamp / compression / CDN interne.

---

## 6. Service OCSP local

> Note perso: première fois que je lance un responder OCSP « à la main ». Ce qui m’a surpris : la commande reste très verbeuse si on ne limite pas (`-nrequest`). Sans ça, j’ai oublié une fois de la stopper et j’avais un port encore occupé.

### 6.1 Certificat de signature OCSP

Je conserve un certificat distinct pour séparer les usages (principe du moindre privilège).

```bash
openssl genrsa -out pki/ocsp/ocsp.key.pem 2048

openssl req -new -sha256 \
  -key pki/ocsp/ocsp.key.pem \
  -subj "/C=FR/O=DemoSecurity/CN=DemoSecurity OCSP Responder" \
  -out pki/csr/ocsp.csr.pem
```

Signer avec le profil `ocsp` :

```bash
openssl ca -config pki/openssl.cnf \
  -extensions ocsp -days 180 -notext -md sha256 \
  -in pki/csr/ocsp.csr.pem \
  -out pki/ocsp/ocsp.cert.pem
```

> Note : certains labs signent OCSP avec la clé root. Ici usage d’un certificat OCSP distinct pour réduire l’impact d’un éventuel compromis.

### 6.2 Lancement du responder (ex: port 2560)

```bash
openssl ocsp \
  -port 127.0.0.1:2560 \
  -index pki/index.txt \
  -CA pki/certs/ca.cert.pem \
  -rkey pki/ocsp/ocsp.key.pem \
  -rsigner pki/ocsp/ocsp.cert.pem \
  -nrequest 1
```

(`-nrequest 1` : quitte après 1 requête pour test ; en prod on omet et on daemonize)

### 6.3 Vérification de statut (cert révoqué)

```bash
openssl ocsp \
  -CAfile pki/certs/ca.cert.pem \
  -issuer pki/certs/ca.cert.pem \
  -cert pki/server/server.cert.pem \
  -url http://127.0.0.1:2560 \
  -resp_text -no_nonce
```

Sortie(après révocation) :

```bash
Response verify OK
server.cert.pem: revoked
    This Update: Sep 30 10:10:00 2025 GMT
    Next Update: Sep 30 10:20:00 2025 GMT
```

### 6.4 CRL vs OCSP (synthèse)

```text
- CRL : liste globale – téléchargement périodique, peut être volumineuse, latence avant prise en compte.
- OCSP : statut ciblé à la demande (GOOD/REVOKED/UNKNOWN), plus dynamique.
- OCSP nécessite haute disponibilité + confidentialité potentiellement moindre (corrélation des accès) sauf OCSP stapling.

```

---

## 7. Petit serveur HTTPS de test

### 7.1 Si le premier cert est révoqué

Je regénère un second certificat serveur pour tester un chemin “valide”.

```bash
# (Optionnel) Nouveau serveur non révoqué
openssl genrsa -out pki/server/server2.key.pem 2048
openssl req -new -key pki/server/server2.key.pem \
  -subj "/C=FR/O=DemoSecurity/CN=app.lab.securecorp.internal" \
  -out pki/csr/server2.csr.pem
openssl ca -config pki/openssl.cnf -extensions server_cert -days 397 -notext -md sha256 \
  -in pki/csr/server2.csr.pem -out pki/server/server2.cert.pem
```

### 7.2 Démarrage (`openssl s_server`)

```bash
openssl s_server -key pki/server/server2.key.pem -cert pki/server/server2.cert.pem \
  -CAfile pki/certs/ca.cert.pem -accept 8443 -www
```

Accès navigateur : `https://app.lab.securecorp.internal:8443/` (ajouter dans `/etc/hosts` pointant vers 127.0.0.1).

### 7.3 Test côté client

```bash
curl -vk --resolve app.lab.securecorp.internal:8443:127.0.0.1 https://app.lab.securecorp.internal:8443/
```

Extrait de sortie :

```bash
*  SSL certificate problem: unable to get local issuer certificate
```

(Attendu si la CA n’est pas installée en confiance.)

---

## 8. Vérifications rapides

### 8.1 Chaîne de certificats

```bash
openssl verify -CAfile pki/certs/ca.cert.pem pki/server/server2.cert.pem
```

Sortie : `pki/server/server2.cert.pem: OK`

### 8.2 Affichage du numéro de série

```bash
openssl x509 -in pki/server/server2.cert.pem -noout -serial
```

### 8.3 Vérification révocation via CRL

```bash
openssl verify -crl_check \
  -CAfile pki/certs/ca.cert.pem \
  -CRLfile pki/crl/ca.crl.pem \
  pki/server/server.cert.pem
```

Sortie (si révoqué) :

```text
C = FR, O = DemoSecurity, CN = app.lab.securecorp.internal
error 23 at 0 depth lookup: certificate revoked
```

### 8.4 Vérification via OCSP

```bash
openssl ocsp -CAfile pki/certs/ca.cert.pem \
  -issuer pki/certs/ca.cert.pem \
  -cert pki/server/server.cert.pem \
  -url http://127.0.0.1:2560 -no_nonce -resp_text
```

### (Variante) CDP / AIA + raison de révocation

Adaptations possibles si publication réseau :

```ini
[ usr_cert ]
crlDistributionPoints = URI:http://pki.lab.securecorp.internal/crl/ca.crl.pem
authorityInfoAccess   = OCSP;URI:http://ocsp.lab.securecorp.internal
```

Commande avec raison :

```bash
openssl ca -config pki/openssl.cnf -revoke pki/server/server.cert.pem -crl_reason keyCompromise
```

---

## 9. Réponses aux questions

### Q1. Pourquoi le navigateur affiche une alerte ?

CA non reconnue (racine interne non installée), éventuel SAN absent, port non standard et absence de stapling.

### Q2. Durée de validité du certificat serveur (et où la voir) ?

397 jours. Consultation : `openssl x509 -in server.cert.pem -noout -dates` ou panneau d’infos du navigateur.

### Q3. Contenu principal d’un certificat serveur

Sujet (DN), numéro de série, clé publique, période, algo de signature, extensions (BasicConstraints, KeyUsage, ExtendedKeyUsage, SAN, SKI, AKI, AIA, CDP).

### Q4. Rôle et limites d’une CRL

Liste globale des certs révoqués. Limites : taille croissante, latence de mise à jour, charge réseau.

### Q5. Différences OCSP vs CRL

OCSP = statut ciblé quasi temps réel, faible bande passante, améliorable avec stapling. CRL = liste complète périodique.

### Q6. Tester la révocation

CRL : `openssl verify -crl_check -CAfile ca.cert.pem -CRLfile ca.crl.pem cert.pem`

OCSP : `openssl ocsp ...` → statut `revoked` quand applicable.

### Q7. Pourquoi le fichier `openssl.cnf` est critique

Centralise chemins, politiques (contrôle DN), profils (limitation d’usage), AIA/CDP pour validation côté client, empêche émission accidentelle d’un cert sur-privilégié.
