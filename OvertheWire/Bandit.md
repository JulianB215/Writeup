# Writeup — OverTheWire : Bandit (Level 0 → 33)

> **Note** : les mots de passe sont volontairement tronqués (`abcd••••`) pour ne pas spoiler le challenge, conformément à l'esprit des règles d'OverTheWire. Les clés SSH privées ne sont pas reproduites non plus.

## Introduction

**Bandit** est le wargame d'initiation d'[OverTheWire](https://overthewire.org/wargames/bandit/). Il apprend les bases de la ligne de commande Linux, de l'administration système et de quelques notions de sécurité (SSH, SSL/TLS, cron, setuid, git…).

Chaque niveau se résout sur la machine distante, et le mot de passe trouvé sert à se connecter au niveau suivant :

```bash
ssh banditX@bandit.labs.overthewire.org -p 2220
```

- `ssh` : connexion sécurisée à une machine distante.
- `-p 2220` : le serveur écoute sur le port 2220 et non sur le port SSH standard (22).

---

## Sommaire

| Niveau | Notion principale |
|---|---|
| [0 → 1](#level-0--1) | `ls`, `cat` |
| [1 → 2](#level-1--2) | Fichier nommé `-` |
| [2 → 3](#level-2--3) | Fichier avec espaces et tirets |
| [3 → 4](#level-3--4) | Fichier caché |
| [4 → 5](#level-4--5) | Identifier un fichier lisible |
| [5 → 6](#level-5--6) | `find` avec taille |
| [6 → 7](#level-6--7) | `find` avec propriétaire/groupe |
| [7 → 8](#level-7--8) | `grep` |
| [8 → 9](#level-8--9) | `sort` + `uniq` |
| [9 → 10](#level-9--10) | `strings` + `grep` |
| [10 → 11](#level-10--11) | Base64 |
| [11 → 12](#level-11--12) | ROT13 avec `tr` |
| [12 → 13](#level-12--13) | Décompressions successives |
| [13 → 14](#level-13--14) | Authentification SSH par clé |
| [14 → 15](#level-14--15) | `nc` (netcat) |
| [15 → 16](#level-15--16) | SSL/TLS avec `openssl` |
| [16 → 17](#level-16--17) | Scan de ports avec `nmap` |
| [17 → 18](#level-17--18) | `diff` |
| [18 → 19](#level-18--19) | Contourner un `.bashrc` piégé |
| [19 → 20](#level-19--20) | Binaire setuid |
| [20 → 21](#level-20--21) | Serveur netcat + setuid |
| [21 → 22](#level-21--22) | Cron (1) |
| [22 → 23](#level-22--23) | Cron (2) |
| [23 → 24](#level-23--24) | Cron + script shell |
| [24 → 25](#level-24--25) | Brute-force d'un pincode |
| [25 → 26](#level-25--26) | Évasion via `more` / `vim` |
| [26 → 27](#level-26--27) | Binaire setuid |
| [27 → 28](#level-27--28) | Git clone |
| [28 → 29](#level-28--29) | Historique git |
| [29 → 30](#level-29--30) | Branches git |
| [30 → 31](#level-30--31) | Tags git |
| [31 → 32](#level-31--32) | Git push |
| [32 → 33](#level-32--33) | Évasion d'un shell restreint |

---

## Level 0 → 1

**Objectif** : le mot de passe est dans un fichier `readme` dans le répertoire personnel.

```bash
ls -la
cat readme
```

- `ls -la` : liste **tous** les fichiers (`-a`, y compris cachés) au format détaillé (`-l`). On repère ainsi `readme` et ses permissions.
- `cat readme` : affiche le contenu du fichier, qui contient le mot de passe.

**Mot de passe** : `6y2k••••`

---

## Level 1 → 2

**Objectif** : le mot de passe est dans un fichier nommé `-`.

```bash
ls -la
cat ./-
```

- Un simple `cat -` ne fonctionnerait pas : pour la plupart des commandes, `-` désigne l'entrée standard (stdin) ou introduit une option.
- On préfixe donc par `./` pour indiquer explicitement un **chemin relatif** vers un fichier du répertoire courant.

**Mot de passe** : `PK8f••••`

---

## Level 2 → 3

**Objectif** : le fichier s'appelle `--spaces in this filename--`.

```bash
ls -la
cat "--spaces in this filename--"      # échec
cat ./"--spaces in this filename--"    # succès
```

- Premier essai avec des guillemets : `cat` interprète le nom comme une option à cause des `--` du début (`unexpected argument`).
- Les guillemets règlent le problème des **espaces**, mais pas celui du **tiret initial**.
- Solution : on ajoute `./` devant, comme au niveau précédent. (L'astuce proposée par l'erreur, `cat -- "..."`, fonctionne aussi : `--` signifie « fin des options ».)

**Mot de passe** : `7ZZ2••••`

---

## Level 3 → 4

**Objectif** : le mot de passe est dans un fichier caché du dossier `inhere`.

```bash
cd inhere
ls -la
cat ./"...Hiding-From-You"
```

- `ls` seul n'affiche pas les fichiers commençant par un point. Il faut `-a` (inclus dans `-la`).
- Le nom commence par `...` : on le protège par `./` et des guillemets pour éviter toute mauvaise interprétation.

**Mot de passe** : `xzTX••••`

---

## Level 4 → 5

**Objectif** : trouver le seul fichier lisible par un humain dans `inhere`.

```bash
cd inhere/
ls -la
cat ./-file00
...
cat ./-file07
```

- Tous les fichiers commencent par `-`, d'où le `./`.
- J'ai testé les fichiers un par un : les fichiers `-file00` à `-file06` affichent des caractères binaires illisibles. Seul `-file07` contient du texte.

> **Astuce** : afficher du binaire peut « casser » le terminal ; la commande `reset` le remet en état. Plus propre : `file ./-file*` indique directement le type de chaque fichier (le seul `ASCII text` est le bon).

**Mot de passe** : `6C7h••••`

---

## Level 5 → 6

**Objectif** : fichier dans `inhere` (sous-dossiers compris), lisible, de **1033 octets**, non exécutable.

```bash
find ./ -size -1034c -and -size +1032c
cd maybehere07
cat ./.file2
```

- `find ./` : parcourt récursivement le dossier courant.
- `-size -1034c -and -size +1032c` : taille strictement inférieure à 1034 octets **et** strictement supérieure à 1032, soit exactement 1033 octets (le suffixe `c` = octets).
- Un seul résultat : `./maybehere07/.file2`, qu'on lit ensuite.

**Mot de passe** : `pXa2••••`

---

## Level 6 → 7

**Objectif** : fichier quelque part sur le serveur, propriétaire `bandit7`, groupe `bandit6`, 33 octets.

```bash
find / -size 33c -user bandit7 -group bandit6 2>/dev/null
cat /var/lib/dpkg/info/bandit7.password
```

- `find /` : recherche sur tout le système de fichiers.
- `-size 33c`, `-user bandit7`, `-group bandit6` : critères de l'énoncé.
- `2>/dev/null` : redirige les erreurs (`Permission denied`, très nombreuses en cherchant depuis `/`) vers la « poubelle » pour ne garder que le résultat utile.

**Mot de passe** : `Bmnn••••`

---

## Level 7 → 8

**Objectif** : le mot de passe est à côté du mot `millionth` dans `data.txt`.

```bash
grep "millionth" data.txt
```

- `grep` filtre les lignes contenant un motif. Le fichier est énorme : inutile de le lire à la main.

**Mot de passe** : `VR1l••••`

---

## Level 8 → 9

**Objectif** : la seule ligne qui n'apparaît qu'une seule fois dans `data.txt`.

```bash
sort data.txt | uniq -u
```

- `sort` : trie les lignes. **Obligatoire** car `uniq` ne détecte que les doublons **adjacents**.
- `uniq -u` : n'affiche que les lignes **uniques** (celles qui n'ont pas de doublon).

**Mot de passe** : `Ejm0••••`

---

## Level 9 → 10

**Objectif** : une des rares chaînes lisibles de `data.txt`, précédée de plusieurs `=`.

```bash
strings data.txt | grep "=="
```

- `data.txt` est un fichier binaire : `cat` afficherait surtout du bruit.
- `strings` extrait uniquement les suites de caractères imprimables.
- `grep "=="` ne garde que les lignes contenant plusieurs `=`. Le mot de passe apparaît en dernier.

**Mot de passe** : `B0s2••••`

---

## Level 10 → 11

**Objectif** : `data.txt` contient des données encodées en base64.

```bash
base64 -d data.txt
```

- `base64 -d` (`--decode`) décode le contenu. Le base64 est un **encodage**, pas un chiffrement : aucune clé n'est nécessaire.

**Mot de passe** : `pYfO••••`

---

## Level 11 → 12

**Objectif** : les lettres de `data.txt` ont subi une rotation de 13 positions (ROT13).

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

- `tr` remplace des caractères par d'autres, un à un.
- On mappe `A-Z` vers `N-ZA-M` (décalage de 13) et pareil pour les minuscules. Comme l'alphabet compte 26 lettres, ROT13 est son propre inverse : l'appliquer chiffre comme déchiffre.

**Mot de passe** : `GROo••••`

---

## Level 12 → 13

**Objectif** : `data.txt` est un *hexdump* d'un fichier compressé de nombreuses fois. On travaille dans un dossier temporaire (`/tmp/...`) car le répertoire personnel est en lecture seule.

```bash
xxd -r data.txt > data
file data
```

- `xxd -r` reconstruit le fichier binaire à partir de son dump hexadécimal.
- `file` identifie le **vrai type** d'un fichier, indépendamment de son nom. C'est l'outil clé de ce niveau : à chaque étape on l'utilise pour savoir quoi faire ensuite.

Ensuite, on répète la boucle **`file` → renommer avec la bonne extension → décompresser** :

| Type détecté par `file` | Renommage | Décompression |
|---|---|---|
| gzip | `mv data data.gz` | `gzip -d data.gz` |
| bzip2 | `mv data data.bz2` | `bzip2 -d data.bz2` |
| tar | `mv data data.tar` | `tar xf data.tar` |

- Le renommage est nécessaire car `gzip` et `bzip2` exigent la bonne extension pour décompresser.
- `tar xf` extrait l'archive et produit un nouveau fichier (`data5.bin`, `data6.bin`, `data8.bin`…) : on relance `file` dessus.
- Chaîne rencontrée : gzip → bzip2 → gzip → tar → tar → bzip2 → tar → gzip → texte ASCII.

Quand `file` répond `ASCII text`, on lit le résultat :

```bash
strings data
```

> Quelques erreurs de manipulation dans mon historique (`gzip: data.gz: No such file`, `file data6` sans l'extension `.bin`) : elles viennent simplement d'un mauvais nom de fichier, à corriger avec `ls`.

**Mot de passe** : `qQYQ••••`

---

## Level 13 → 14

**Objectif** : pas de mot de passe, mais une **clé SSH privée** (`sshkey.private`) permettant de se connecter à `bandit14`.

```bash
# Sur bandit13
cat sshkey.private
exit

# Sur ma machine locale
mkdir /tmp/overthewire
cd /tmp/overthewire
echo "-----BEGIN OPENSSH PRIVATE KEY----- ... " > key
chmod 600 key
ssh bandit14@bandit.labs.overthewire.org -p 2220 -i key

# Sur bandit14
cat /etc/bandit_pass/bandit14
```

- Je copie le contenu de la clé dans un fichier local `key`.
- `chmod 600 key` : SSH **refuse** une clé privée dont les permissions sont trop ouvertes (lisible par d'autres utilisateurs). `600` = lecture/écriture pour le propriétaire uniquement.
- `ssh -i key` : s'authentifie avec la clé (*identity file*) à la place d'un mot de passe.
- Une fois sur `bandit14`, `/etc/bandit_pass/bandit14` est lisible par cet utilisateur.

**Mot de passe** : `aaWe••••`

---

## Level 14 → 15

**Objectif** : envoyer le mot de passe actuel au port 30000 de `localhost`.

```bash
nc localhost 30000
```

- `nc` (netcat) ouvre une connexion TCP brute vers un hôte et un port, et permet d'envoyer/recevoir du texte à la main.
- Une fois connecté, je tape le mot de passe de `bandit14`, le serveur répond `Correct!` suivi du mot de passe suivant.

**Mot de passe** : `pbLY••••`

---

## Level 15 → 16

**Objectif** : même chose, mais sur le port 30001 **avec chiffrement SSL/TLS**.

```bash
openssl s_client -connect localhost:30001
```

- `nc` ne sait pas parler TLS. `openssl s_client` est un client TLS en ligne de commande.
- Il affiche le certificat (auto-signé, d'où l'avertissement `self-signed certificate`, sans importance ici) puis laisse une session interactive où l'on tape le mot de passe de `bandit15`.
- Si l'on obtient `KEYUPDATE` ou `RENEGOTIATING`, c'est que la ligne commence par `R` ou `K` : on peut utiliser l'option `-quiet` (voir niveau suivant).

**Mot de passe** : `kS0H••••`

---

## Level 16 → 17

**Objectif** : trouver, parmi les ports 31000-32000, le serveur qui parle SSL et qui renvoie les identifiants suivants.

```bash
nmap localhost -p 31000-32000
openssl s_client -connect localhost:31790 -quiet
```

- `nmap` scanne les ports ouverts. Cinq ports répondent : 31046, 31518, 31691, 31790, 31960.
- Parmi eux, il faut identifier ceux qui parlent TLS, puis celui qui donne autre chose qu'un simple écho de ce que l'on envoie. Le port **31790** est le bon : je lui envoie le mot de passe de `bandit16`.
- `-quiet` supprime les informations de session pour garder une sortie lisible et évite les commandes spéciales.
- Le serveur renvoie une **clé privée SSH** (pas un mot de passe).

On la sauvegarde localement, comme au niveau 13 :

```bash
echo "-----BEGIN OPENSSH PRIVATE KEY----- ..." > key16
chmod 600 key16
ssh bandit17@bandit.labs.overthewire.org -p 2220 -i key16
cat /etc/bandit_pass/bandit17
```

**Mot de passe** : `pWXM••••`

---

## Level 17 → 18

**Objectif** : deux fichiers `passwords.old` et `passwords.new` ; le mot de passe est la seule ligne modifiée.

```bash
diff passwords.old passwords.new
```

- `diff` compare deux fichiers ligne par ligne et indique que seule la ligne 42 diffère.
- La ligne préfixée par `<` vient du premier fichier (`passwords.old`), celle préfixée par `>` du second (`passwords.new`). Le mot de passe est dans `passwords.new`, donc la ligne `>`.

**Mot de passe** : `OQxX••••`

---

## Level 18 → 19

**Objectif** : le `.bashrc` a été modifié pour **déconnecter** immédiatement à l'ouverture d'une session SSH. Le mot de passe est dans `readme`.

```bash
ssh bandit18@bandit.labs.overthewire.org -p 2220 cat readme
```

- Quand on passe une commande à `ssh`, elle est exécutée directement sur la machine distante, **sans ouvrir de shell interactif**.
- On contourne donc le `.bashrc` piégé (la déconnexion est dans le shell de connexion) et on lit `readme` sans problème. Le mot de passe de `bandit18` est demandé au passage.

**Mot de passe** : `KpsO••••`

---

## Level 19 → 20

**Objectif** : utiliser un binaire **setuid** (`bandit20-do`) pour lire `/etc/bandit_pass/bandit20`.

```bash
strings bandit20-do
ls -la
./bandit20-do cat /etc/bandit_pass/bandit20
```

- `strings` sur le binaire montre un message d'usage : `Run a command as another user. Example: %s whoami`.
- `ls -la` révèle les permissions `-rwsr-x---` : le **`s`** est le bit **setuid**. Le programme s'exécute avec les droits de son propriétaire (`bandit20`) et non de celui qui le lance.
- Il suffit donc de lui faire exécuter `cat` sur le fichier de mot de passe.

**Mot de passe** : `4pIj••••`

---

## Level 20 → 21

**Objectif** : un binaire setuid (`suconnect`) se connecte à un port local, lit une ligne et, si c'est le mot de passe de `bandit20`, renvoie celui de `bandit21`. Il faut donc **fournir le serveur nous-mêmes**.

```bash
nc -lvnp 31045
./suconnect 31045
```

- `nc -l` : mode écoute (*listen*). `-v` verbeux, `-n` pas de résolution DNS, `-p` port.
- Dans un second terminal (autre session SSH), je lance `./suconnect 31045` qui se connecte à mon serveur.
- Dans la fenêtre du serveur netcat, je tape le mot de passe de `bandit20`. Le programme le valide et renvoie le mot de passe suivant.

**Mot de passe** : `bW9k••••`

---

## Level 21 → 22

**Objectif** : un programme tourne régulièrement via **cron**.

```bash
cd /etc/cron.d
ls
strings cronjob_bandit22
cat /usr/bin/cronjob_bandit22.sh
cat /tmp/t7O6••••
```

- `/etc/cron.d/` contient les tâches planifiées. `cronjob_bandit22` indique `* * * * * bandit22 /usr/bin/cronjob_bandit22.sh` : exécution **chaque minute** par l'utilisateur `bandit22`.
- J'ai consulté `man 5 crontab` pour comprendre la syntaxe (les 5 étoiles = minute, heure, jour, mois, jour de semaine).
- Le script copie le mot de passe de `bandit22` dans un fichier `/tmp/...` et le rend lisible (`chmod 644`). Il suffit de lire ce fichier.

**Mot de passe** : `RYVu••••`

---

## Level 22 → 23

**Objectif** : même principe avec un script un peu plus complexe.

```bash
cat /etc/cron.d/cronjob_bandit23
cat /usr/bin/cronjob_bandit23.sh
echo I am user bandit23 | md5sum | cut -d ' ' -f 1
cat /tmp/8ca3••••
```

- Le script calcule le nom du fichier cible : `echo I am user $myname | md5sum | cut -d ' ' -f 1`, avec `$myname` = utilisateur qui exécute le script, c'est-à-dire **bandit23**.
- Je ne peux pas lancer le script en tant que bandit23, mais je peux **reproduire le calcul** en remplaçant `$myname` par `bandit23` : cela me donne le nom du fichier `/tmp/...` où le mot de passe est copié.
- `cut -d ' ' -f 1` retire le `-` que `md5sum` ajoute en fin de ligne.

**Mot de passe** : `gKXD••••`

---

## Level 23 → 24

**Objectif** : un cron exécute et supprime tous les scripts placés dans `/var/spool/bandit24/foo`. Il faut écrire **son propre script**.

```bash
cat /etc/cron.d/cronjob_bandit24
cat /usr/bin/cronjob_bandit24.sh

cat > /tmp/monscript.sh << 'EOF'
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/password24.txt
chmod 644 /tmp/password24.txt
EOF

chmod +x /tmp/monscript.sh
cp /tmp/monscript.sh /var/spool/bandit24/foo/
cat /tmp/password24.txt
```

- Le script du cron exécute, **en tant que bandit24**, chaque fichier du dossier `foo` appartenant à `bandit23`, puis le supprime.
- Je crée donc un script qui copie le mot de passe de `bandit24` dans un fichier `/tmp` que je peux lire, puis le rend lisible (`chmod 644`), car le fichier sera créé par `bandit24`.
- `<< 'EOF'` (*heredoc*) permet d'écrire un fichier multi-lignes directement depuis le terminal. Les guillemets autour de `EOF` empêchent le shell d'interpréter des variables dans le contenu.
- `chmod +x` rend le script exécutable : sans cela, le cron ne pourrait pas le lancer.
- Après une minute maximum (le cron tourne chaque minute), le fichier `/tmp/password24.txt` apparaît.

> **Remarque** : sur ce niveau j'ai aussi tenté `ls` dans `foo` (refusé : on n'a pas le droit de lister ce dossier) et une commande `chmod` mal écrite (`chmod foo 600`, les arguments étaient inversés). Sans conséquence.

**Mot de passe** : `hVQM••••`

---

## Level 24 → 25

**Objectif** : un démon sur le port 30002 donne le mot de passe de `bandit25` si on lui envoie le mot de passe de `bandit24` + un **pincode à 4 chiffres**. Il faut tester les 10 000 combinaisons.

```bash
for i in $(seq -w 0000 9999); do
    echo "hVQM•••• $i"
done | nc localhost 30002 > /tmp/resultat.txt

grep -v Wrong /tmp/resultat.txt
```

- `seq -w 0000 9999` génère les nombres de 0000 à 9999 avec des **zéros de remplissage** (`-w`, largeur égale) : indispensable car le pincode peut commencer par des zéros.
- La boucle écrit une ligne `mot_de_passe pincode` pour chaque combinaison. Toute la sortie est envoyée **dans une seule connexion** netcat (l'énoncé précise qu'on n'a pas besoin de rouvrir une connexion à chaque essai) et enregistrée dans un fichier.
- `grep -v Wrong` : affiche toutes les lignes qui **ne contiennent pas** `Wrong`, c'est-à-dire la seule réponse correcte.

**Mot de passe** : `SoHf••••`

---

## Level 25 → 26

**Objectif** : se connecter à `bandit26`, dont le shell n'est pas `/bin/bash` mais un programme qui empêche d'obtenir un terminal classique. Il faut trouver comment s'en échapper.

**Démarche :**

1. Dans `bandit25`, on trouve une clé SSH privée `bandit26.sshkey`. On se connecte avec `ssh -i bandit26.sshkey bandit26@... -p 2220`.
2. La connexion se ferme immédiatement : le shell de `bandit26` affiche un texte via **`more`** puis quitte.
3. Astuce : **réduire la fenêtre du terminal** à une seule ligne, pour que `more` ne puisse pas afficher tout le texte et reste en mode interactif (pagination) au lieu de quitter.
4. Dans `more`, la touche **`v`** ouvre l'éditeur **vim** sur le fichier affiché.
5. Dans vim, on définit le shell et on le lance :

```vim
:set shell=/bin/bash
:shell
```

6. On obtient un vrai shell bash :

```bash
cat /etc/bandit_pass/bandit26
```

- `more` est un pager, donc il ne s'arrête que s'il y a plus de texte que de lignes disponibles : d'où la fenêtre minuscule.
- vim permet d'exécuter des commandes shell : c'est cette fonctionnalité qui permet de sortir du shell restreint.

**Mot de passe** : `jHdv••••`

---

## Level 26 → 27

**Objectif** : « Good job getting a shell ! Now hurry and grab the password for bandit27 ! » — on est dans le shell de `bandit26` obtenu au niveau précédent.

```bash
ls
strings bandit27-do
./bandit27-do cat /etc/bandit_pass/bandit27
```

- `ls` montre un binaire `bandit27-do` : même mécanisme que `bandit20-do` (setuid), ici avec les droits de `bandit27`.
- `strings` confirme qu'il exécute la commande donnée en argument.
- On lui fait donc afficher le mot de passe.

**Mot de passe** : `STJL••••`

---

## Level 27 → 28

**Objectif** : cloner un dépôt git hébergé sur la machine Bandit, **depuis sa machine locale**.

```bash
git clone ssh://bandit27-git@bandit.labs.overthewire.org:2220/home/bandit27-git/repo
cd repo
ls
cat README
```

- `git clone` copie le dépôt distant en local. L'URL contient l'utilisateur (`bandit27-git`), le serveur, le **port 2220** et le chemin du dépôt.
- Le mot de passe demandé est celui de `bandit27`.
- Le mot de passe du niveau suivant est dans le `README` du dépôt.

**Mot de passe** : `y8Yd••••`

---

## Level 28 → 29

**Objectif** : même chose, mais le mot de passe a été masqué dans le dépôt.

```bash
git clone ssh://bandit28-git@bandit.labs.overthewire.org:2220/home/bandit28-git/repo
cd repo
cat README.md
git log
git log -p
```

- Le `README.md` actuel affiche `password: xxxxxxxxxx` : l'information a été retirée.
- `git log` liste les commits : l'un d'eux s'appelle `fix info leak`, ce qui est un indice.
- `git log -p` affiche le contenu (`-p` = *patch*) de chaque commit. Dans le commit `add missing data`, on voit la ligne avec le mot de passe, **supprimée** ensuite. Git garde tout l'historique : supprimer une information dans un commit ne l'efface pas du dépôt.

**Mot de passe** : `Em7e••••`

---

## Level 29 → 30

**Objectif** : le README indique `<no passwords in production!>` : le mot de passe est ailleurs.

```bash
git clone ssh://bandit29-git@bandit.labs.overthewire.org:2220/home/bandit29-git/repo
git branch -a
git checkout dev
git log -p
```

- Le dépôt a **plusieurs branches**. `git branch -a` liste toutes les branches, y compris distantes (`remotes/origin/dev`).
- Le message « no passwords in production » suggère que le mot de passe est dans une branche de développement : on passe sur `dev` avec `git checkout dev`.
- `git log -p` montre le commit `add data needed for development` qui ajoute le mot de passe dans le README.

**Mot de passe** : `jq9D••••`

---

## Level 30 → 31

**Objectif** : le dépôt semble contenir seulement un fichier vide.

```bash
cat README.md
git log
git branch -a
git tag
git show secret
```

- `README.md` : `just an empty file... muahaha`.
- `git log` : un seul commit, rien d'intéressant. `git branch -a` : uniquement `master`.
- Dernière piste : les **tags**, marqueurs nommés sur des objets git. `git tag` révèle un tag `secret`.
- `git show secret` affiche son contenu : le mot de passe.

**Mot de passe** : `82Nk••••`

---

## Level 31 → 32

**Objectif** : pousser (`push`) sur le dépôt distant un fichier `key.txt` contenant `May I come in?` sur la branche `master`.

```bash
cat README.md
nano key.txt                    # contenu : May I come in?
git add key.txt                 # refusé : fichier ignoré
git add key.txt -f
git config --global user.email "..."
git config --global user.name "..."
git commit -m "ajout key"
git push origin master
```

- `nano key.txt` : crée le fichier avec le contenu demandé.
- `git add key.txt` échoue : un `.gitignore` ignore ce fichier. L'option `-f` (*force*) permet de l'ajouter quand même.
- `git commit` échoue la première fois : git a besoin d'une identité (`user.name` / `user.email`) qu'on configure avec `git config`.
- `git push origin master` : envoie le commit. Un *hook* côté serveur valide le contenu, et le message de succès affiche le mot de passe, même si la fin du push est signalée comme rejetée (`pre-receive hook declined`) : ce rejet n'a pas d'importance, le mot de passe est déjà affiché dans les lignes `remote:`.

**Mot de passe** : `pWuj••••`

---

## Level 32 → 33

**Objectif** : « Another escape ». Le shell de connexion transforme tout ce que l'on tape en **majuscules** (c'est un « uppershell »), donc aucune commande classique (`ls`, `cat`…) ne fonctionne, puisque Linux est sensible à la casse.

```bash
$0
python3 -c 'import pty; pty.spawn("/bin/bash")'
cat /etc/bandit_pass/bandit33
```

- `$0` est une variable shell qui contient le nom du programme en cours d'exécution (ici le shell). Comme elle ne contient aucune lettre, la mise en majuscule ne la modifie pas : elle est développée normalement et lance un shell (`sh`), qui n'est plus soumis à la conversion en majuscules.
- Le shell obtenu est minimal ; pour en avoir un plus confortable, on lance `bash` dans un pseudo-terminal avec Python (`pty.spawn`).
- Le binaire de l'uppershell étant setuid `bandit33`, le shell hérite de cet utilisateur, ce qui permet de lire `/etc/bandit_pass/bandit33`.

**Mot de passe** : `u4P2••••`

---

## Conclusion

Ce parcours couvre les bases essentielles :

- **Navigation et fichiers** : `ls`, `cat`, `find`, `grep`, `file`, `strings`, `diff`, `sort`, `uniq`, `tr`.
- **Encodage et compression** : base64, ROT13, hexdump, gzip, bzip2, tar.
- **Réseau** : `ssh` (mot de passe, clé, commande à distance), `nc`, `openssl s_client`, `nmap`.
- **Élévation de privilèges légère** : binaires setuid, tâches cron, scripts.
- **Évasion de shells restreints** : `more` → `vim`, `$0`.
- **Git** : `clone`, `log -p`, `branch`, `tag`, `push`, hooks.

**À retenir** : lire attentivement l'énoncé et les messages d'erreur, utiliser `man` / `--help`, et ne jamais supposer qu'une information supprimée a disparu (historique git, fichiers `/tmp`, etc.).
