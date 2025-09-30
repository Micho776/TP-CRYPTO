# TP OpenSSL

## 1. Structure initiale & préparation

### 1.1 Arborescence de base de la PKI

```bash
mkdir -p pki/{certs,crl,newcerts,private,csr,ocsp,server}
chmod 700 pki/private
touch pki/index.txt
echo 1000 > pki/serial
echo 1000 > pki/crlnumber
```

Fichier d’attributs optionnel (vide) : `pki/index.txt.attr`

### 1.2 CA minimale (démonstration rapide)

Objectif : montrer la création « brute » d’une clé privée de CA et d’un certificat auto‑signé simple (RSA 2048 / 365 jours) avant la version plus robuste (RSA 4096 / 10 ans) utilisée ensuite dans la section 2.

#### Clé privée de la CA (simple)

```bash
$ openssl genrsa -out Pkey.pem 2048
$ cat Pkey.pem
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDUjQmMahZLNF2K
d8ld80ZbOu2azjTOegyfVCPZ4rmHEozbVabjT91TRsrEzbA6m5OJrYL49eL0zRvq
mFUE6DoqdIXlRTlcy2IyjrOZrZK1FPglK04/mOHwb8nyNpoAwIt65nj0HLvWEyZd
fUT0MVLolvgDhgZk5qPvlfd1p0SXO9ZYQx66074uoTgFFytz3kul3zWX/S2t4xyN
faZS37kjQkTn1JMBeVQ+gNN9L/Bli8NZx70mzL+1owQSMW5fGPXLq5s+7kHywr+p
S4h6IR0+N2ZjN4vFLy1055Ja/YPFgYYsHM66I4bLRcB4T0/qOOaJpk0eChttY8tM
2p70RYTpAgMBAAECggEACTKhOyZMGK0HbzqHyD0CymfeaFiMCHNXoH1vn7oj9Shk
WAl401VdaoEhvSp5ec/JrqeHh2Z8j8NgdeJpq3CxM60fLjC0rsNOWIm8U+Yi9xsV
MeaR2EaxYEo2Hvkl6OpsHsFico3bwwPJITqOhVKtF8uQp/ZgyHUCmxeOQdUfLrwf
9hzpgNk0a6KgTQoDkxdu4LX+BEvzHtDCPzfKWOxAT3CRSQqBYUv1yz8gshsvR7lW
28ViX+gmLdHbI/iUnf1NhPAfBBxLI/4CJ20ZJ53X0EOOz7EwYgTQZ2Ac9l4410MY
tN8yR8ICWDpCED9Wk7C19zR6LJICkJAtc56V12c6AQKBgQDpT+lntC/uA69lEfdj
vN4QDoa1u17rzPeWACH360f73XLFMXhJ6Jb6a3lZGCLi4uiyhTqD30qSIEHjoOfV
Zx04E6jrQ1AJvTQAFaGB3MKChB38slEMdQEZybi8foXEvJLmcNDQwN/531rWJgjW
TxPQBembdvEHvXr33gHpXNmocQKBgQDpOEtOhGjYCRcFKz4OBipM6N6SblBRhCKs
d4KJJl1+1cyLe4ZP+bYFZ7UorlwAnDeYIcyUJyUxvRgagB3a0BeWhU/2Aoptnz96
1/lowwLxjhrwzLPwwuSPf87CgBtiGv3Ks7kcwlrUkq2v9m++TCJpNSn5aVaJhn/u
5I15eVsf+QKBgQCk7nnodqd/UZmXEFlbZ3Nv1GUEaX2ToeTQZC2fLfNIKGbu4abQ
UJ0SUBGLmxVmYNPxB1+zQ5FatXT+rovU/zzXnIZIMeCN0fPFr4Tp4Z6bVzw/m+rR
rJDnowN2NNbpmgka4FuthvuOj4eOZXrPzT0LTHK1FSMUvq0ENiwRxTKU8QKBgF8L
7zz3n5bz1Wro3ahvgMvJV/QLezZNiKgLKKrmdNIdJfhuhiWP2kvHoUAMFzi0cb/R
foOelHz52JnsPr6Pch2JBTZ4gJv+e6t/24iDeW6igH5EnszvUKDe8I+6D+7imy4C
It4Co2vgv2JoJ9BBTQDdhta7xGXV58ufX7zy5V1ZAoGBAKfTUr7WK6FzULqHNigd
iXx/jEXX2AxRSshuAvDoQ+Wc1iBgsuMDJLvBdZtXCYpv1MsZLLsec48hOcgzW0Yk
A9tS8vSdQ3CNTWU6jBvxovfVZDKw+SQCOQByX6rd0AhmR6vFIoff70UvOvLfGjN+
tnYQTO+PmN19Ddp88sukKVFs
-----END PRIVATE KEY-----
```

Je fais une clé privée `Pkey.pem` de 2048 bits (non protégée par mot de passe ici – amélioration plus loin : chiffrement + permissions renforcées).

#### Certificat auto‑signé initial

```bash
$ openssl req -new -key Pkey.pem -out cert.csr
$ openssl x509 -req -days 365 -in cert.csr -signkey Pkey.pem -out certssigh.crt
$ openssl x509 -in certssigh.crt -text -noout
```

Extrait (abrégé) : Version 1 (v1), pas d’extensions (normal), validité 365 jours. Utile uniquement pour un test local, insuffisant pour une PKI structurée (pas de BasicConstraints / KeyUsage).

#### Mise en place arborescence minimale (style ancien script)

```bash
$ mkdir fichierCA
$ cd fichierCA/
$ mkdir certs | mkdir crl | mkdir newcerts | mkdir private
$ touch index.txt
$ echo 1234 > serial
$ echo 1234 > crlnumber
$ ls
certs  crl  crlnumber  index.txt  newcerts  private  serial
$ mv ../Pkey.pem private/cakey.pem
$ mv ../certssigh.crt cacert.pem
$ cd .. && mv fichierCA demoCA
$ ls demoCA
cacert.pem  certs  crl  crlnumber  index.txt  newcerts  private  serial
```

Limites de cette approche : pas de profil d’extensions, clé CA courte (2048), absence de chiffrement de la clé, validité courte, pas de séparation nette des rôles.

### 1.3 Génération rapide d’un certificat serveur (exemple express)

Objectif : montrer le flux minimal (clé + CSR + signature + inspection) avant la version plus propre avec SAN (future section 3 « Certificat serveur »).

#### Clé privée et CSR

```bash
$ openssl genrsa -out serverkey.pem 2048
$ openssl req -new -key serverkey.pem -out server.csr
```

je creer la clef prive du serveur puis un CRS

#### Signature par la CA

```bash
openssl ca -config openssl.cnf -in server.csr -out servercert.pem -batch
Using configuration from openssl.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName           :PRINTABLE:'FR'
stateOrProvinceName   :ASN.1 12:'Some-State'
localityName          :ASN.1 12:'Paris'
organizationName      :ASN.1 12:'Internet Widgits Pty Ltd'
commonName            :ASN.1 12:'server.example.com'
Certificate is to be certified until Sep 30 16:12:40 2026 GMT (365 days)

Write out database with 1 new entries
Database updated
```

#### Vérification des usages (dump brut)

```bash
openssl x509 -in servercert.pem -text -noout
Certificate:
  Data:
    Version: 3 (0x2)
  Serial Number: 4881 (0x1311)
    Signature Algorithm: sha256WithRSAEncryption
    Issuer: C = FR, ST = Some-State, O = Internet Widgits Pty Ltd
    Validity
  Not Before: Sep 30 16:12:40 2025 GMT
  Not After : Sep 30 16:12:40 2026 GMT
    Subject: C = FR, ST = Some-State, O = Internet Widgits Pty Ltd, CN = server.example.com
    Subject Public Key Info:
      Public Key Algorithm: rsaEncryption
        Public-Key: (2048 bit)
        Modulus:
          00:b1:e0:b7:6c:08:c2:fe:6a:21:bd:06:72:61:43:
          51:b5:f6:42:e1:ae:d8:38:55:be:69:9a:fb:4c:4d:
          68:9c:f7:55:2d:3b:fd:a3:91:9e:93:81:bb:9f:b8:
          a8:60:22:bc:a3:ea:3a:33:30:6d:47:6e:d4:c4:07:
          2b:b5:3a:e5:bf:e8:2a:4b:ea:39:4a:02:41:f8:5f:
          9b:11:67:4d:ce:c3:32:9b:63:9f:0d:58:21:83:67:
          93:75:4c:8c:0a:0c:f3:6b:82:a2:0d:85:b3:45:b4:
          bf:2d:68:13:a2:1f:07:fa:f1:6a:bc:33:97:8b:9e:
          d4:f8:e6:d7:09:79:1d:a9:4b:6d:bd:49:4b:3b:c1:
          0a:06:1d:07:e6:95:d1:f2:59:91:41:a1:09:48:45:
          ec:50:a5:c5:02:1e:bc:9a:80:06:53:67:e8:8d:42:
          bc:22:5c:a4:73:d2:a6:ae:cc:95:8e:86:c5:41:72:
          35:a6:9e:f4:37:0d:c4:4e:46:14:6c:4f:8c:35:c1:
          bd:d3:0d:bd:ef:b9:21:fe:d4:ed:8b:0b:c8:db:7a:
          30:76:e7:37:3e:eb:59:ed:a1:83:63:49:35:e2:a8:
          66:d9:0a:2d:08:26:b1:23:bc:58:51:c9:f3:5f:c5:
          ff:f2:16:6d:a2:a0:50:19:5a:36:ca:38:59:e4:93:
          b5:14
        Exponent: 65537 (0x10001)
    X509v3 extensions:
      X509v3 Basic Constraints: critical
        CA:FALSE
      X509v3 Key Usage: critical
        Digital Signature, Key Encipherment
      X509v3 Subject Key Identifier:
        B9:56:4E:B3:7B:09:6A:0A:4C:80:87:73:05:51:E6:DE:23:BE:AF:B1
      X509v3 Authority Key Identifier:
        DirName:/C=FR/ST=Some-State/O=Internet Widgits Pty Ltd
        serial:7A:33:B1:65:B4:49:AB:85:71:FF:1A:6F:5C:CF:69:1B:6A:03:9F:F0
  Signature Algorithm: sha256WithRSAEncryption
  Signature Value:
    1a:44:5b:1b:74:be:5d:5d:1e:75:8b:08:52:ca:d1:d0:cc:43:
    26:6a:ed:71:df:4b:84:b2:a1:7f:8f:d2:d1:d4:84:b9:51:aa:
    54:62:73:ed:27:d0:09:d4:35:f4:b7:08:63:27:79:9c:a6:7f:
    e1:41:d6:be:18:a6:85:57:6a:97:59:c4:71:40:8d:a9:17:b5:
    23:c2:e8:28:8d:09:21:d6:37:f8:85:a4:19:e7:08:68:79:7c:
    c1:47:4e:28:56:85:e2:72:11:36:a7:ec:80:e0:f8:b0:6b:70:
    68:75:3d:67:91:d3:37:08:ed:17:26:26:cf:66:38:cb:dc:8d:
    fd:d4:15:f9:50:d2:5e:26:7a:f3:b3:b8:1a:05:97:b3:63:1d:
    e2:eb:51:c2:29:2b:d9:fc:ef:61:47:67:18:7a:43:f5:a1:3b:
    7c:11:2e:e0:2f:3e:1f:1b:fc:23:a7:a3:3a:27:20:90:ca:85:
    49:67:80:00:46:c4:eb:3f:00:7d:dd:d9:b0:dc:b1:4f:f5:df:
    14:34:e3:d1:ef:39:22:0b:cc:06:ee:47:be:3d:a7:1d:a5:55:
    d1:83:29:fc:4a:84:c0:b5:1c:ee:f3:65:94:08:e6:64:a9:c9:
    e0:2e:8b:d9:99:8b:a6:f7:3c:45:b1:1f:13:15:ea:70:ca:37:
    0f:0c:52:85
```

Commentaires rapides : absence de SAN → usage navigateur limité (on l’ajoutera dans la section dédiée).

---

## 2. Création de la CA racine

### 2.1 Objectif

Mettre en place une CA durable (clé 4096 bits protégée) avec un fichier `pki/openssl.cnf` enrichi contenant des profils d’extensions. Cette partie corrige les limites observées dans les sous‑sections 1.2 (CA minimale) et 1.3 (cert serveur sans SAN) et prépare l’émission contrôlée (Section 3).

### 2.2 Profils essentiels dans `pki/openssl.cnf`

Extraits minimaux (à ajouter si absents) :

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = ./pki
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
crlnumber         = $dir/crlnumber
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem
default_md        = sha256
default_days      = 365
default_crl_days  = 30
policy            = policy_strict
email_in_dn       = no
unique_subject    = no

[ policy_strict ]
C  = match
O  = match
CN = supplied

[ v3_ca ]
basicConstraints       = critical, CA:true, pathlen:0
keyUsage               = critical, keyCertSign, cRLSign
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer

[ usr_cert ]
basicConstraints       = critical, CA:false
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

[ server_cert ]
# Variante explicite (séparation d’usage potentielle)
basicConstraints       = critical, CA:false
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
subjectAltName         = @alt_names

[ ocsp ]
basicConstraints       = critical, CA:false
keyUsage               = critical, digitalSignature
extendedKeyUsage       = OCSPSigning
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

[ crl_ext ]
authorityKeyIdentifier = keyid

# Bloc SAN d'exemple (souvent fourni via un fichier CSR séparé)
[ alt_names ]
DNS.1 = app.lab.securecorp.internal
DNS.2 = www.app.lab.securecorp.internal
IP.1  = 127.0.0.1
```

Justification rapide :

- v3_ca : fixe l'autorité (CA:true) + pathlen:0 pour éviter des intermédiaires non prévus.
- usr_cert : profil générique pour petits services internes (serverAuth suffisant).
- server_cert : duplication contrôlée si besoin de spécialiser (ex : ajouter plus tard d'autres usages ou contraintes SAN).
- ocsp : usage unique OCSPSigning (principe du moindre privilège) + keyUsage digitalSignature.
- SAN : séparé pour ne pas modifier le fichier principal à chaque nouveau nom.
- crl_ext : permet d’insérer l’identifiant d’autorité dans la CRL.

Durées : 365 jours pour les certificats finaux (démonstration). La CA (voir commande 2.4) aura 3650 jours. CRL = 30 jours (valeur pédagogique – en production on réduit parfois à 7 jours voire 1 jour selon exigences de revocation freshness).

### 2.3 Limites des exemples rapides précédents

- Certificat CA v1 (sous‑section 1.2) : aucune extension → peu robuste.
- Clé CA 2048 bits non chiffrée : risque si vol de fichier.
- Cert serveur initial (1.3) sans SAN : rejet navigateur (CN ignoré).
- EKU absent initialement : certains clients tolèrent mais ce n’est plus une bonne pratique.

Ces limites sont levées avec la clé 4096 bits chiffrée + profils v3_ca / server_cert.

### 2.4 Clé privée CA

```bash
openssl genrsa -aes256 -out pki/private/ca.key.pem 4096
chmod 600 pki/private/ca.key.pem
```

### 2.5 Certificat auto-signé (root)

```bash
openssl req -config pki/openssl.cnf \
  -key pki/private/ca.key.pem \
  -new -x509 -days 3650 -sha256 -extensions v3_ca \
  -out pki/certs/ca.cert.pem
chmod 644 pki/certs/ca.cert.pem
```

### 2.6 Vérification rapide

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

## 3. Certificat serveur

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

## 4. Révocation + CRL

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

## 5. Service OCSP local

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

## 6. Petit serveur HTTPS de test

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

## 7. Vérifications rapides

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

## 8. Réponses aux questions

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
