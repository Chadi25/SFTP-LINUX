# 🧾 README - Infrastructure SFTP avec partage de données centralisé

## 📌 Objectif

Mettre en place un serveur SFTP sécurisé avec :

- 🔒 utilisateurs chrootés  
- 📁 accès à des dossiers partagés individuels ou communs  
- 💾 stockage des données sur un disque de **1 To**  
- ♻️ restauration automatisée des données depuis un ancien serveur FileZilla  

---

## 🔧 Structure de l’infra

- **Répertoire chroot des users :** `/sftp/<user>/home`  
- **Stockage centralisé :** `/mnt/sftp_data/shared_data/<dossier_partagé>`  
- **Montage bind pour chaque dossier partagé** vers le chroot de l'utilisateur  
- **Contrôle d'accès fin** via ACL (lecture seule ou lecture/écriture)  
- **Montage réseau** pour récupération des anciennes données via `/mnt/ftp_win`  

---

## 🧩 Étapes principales (automatisées avec scripts)

### 1. 📥 Récupérer la config de l’ancien serveur

- Copier le fichier `FileZilla Server.xml`  
- Lancer :

```bash
python3 analyze_filezilla_sftp_map.py
```

➡️ Affiche les utilisateurs et les dossiers qu’ils utilisaient  
➡️ Permet de **construire manuellement** `output_utilisateurs.csv` :

```
utilisateur;dossier_partage;droit
egx;egx;rw
fkc;validated_drawings;r
...
```

---

### 2. 🧪 Vérification du disque de données

```bash
df -h /mnt/sftp_data
```

➡️ Doit montrer un disque de ~1 To monté sur `/mnt/sftp_data` (pas `/` !)

---

### 3. 🚀 Lancer la création des utilisateurs et répertoires

```bash
bash init_users_and_folders.sh
```

📌 Ce script :

- crée tous les utilisateurs dans `/sftp/<user>`  
- crée les répertoires partagés dans `/mnt/sftp_data/shared_data/`  
- applique les bons propriétaires et droits  

---

### 4. 🔗 Créer les montages (bind) et unités systemd

```bash
bash bind_shared_folders_with_units.sh
```

📌 Ce script :

- crée les **montages bind** entre `/mnt/sftp_data/shared_data/<dossier>` → `/sftp/<user>/home/<dossier>`  
- crée automatiquement des unités `.mount` (`sftp-<user>-home-<share>.mount`)  
- applique les **ACL** : lecture seule (`r`) ou lecture/écriture (`rw`) selon le CSV  

> 🔄 Relance sécurisée : ce script nettoie automatiquement les anciens montages avant de tout reconstruire.

---

### 5. ♻️ Restaurer les données depuis l’ancien serveur

Vérifie d’abord le montage réseau :

```bash
ls /mnt/ftp_win
```

Puis lance la synchronisation :

```bash
bash restore_sftp_data.sh
```

📌 Ce script :

- lit `output_utilisateurs.csv`  
- copie avec `rsync` les données du répertoire `/mnt/ftp_win/<partage>` vers `/mnt/sftp_data/shared_data/<partage>`  
- applique les droits `sftpusers` + bon propriétaire  

➡️ **Peut être relancé** sans casser les accès (pas de doublons, pas d’écrasement indésirable)

---

## 🧪 Vérifications finales

```bash
mount | grep /sftp/
```

➡️ Tous les bind mounts doivent être listés

```bash
df -h /mnt/sftp_data
```

➡️ Vérifie que les données sont bien sur le disque de 1 To (et non sur `/`)

```bash
ls -l /sftp/<user>/home/<partage>
```

➡️ Vérifie les droits, et que l'utilisateur peut lire (et écrire si `rw`)

---

## 🧱 Réaliser **manuellement** (équivalent sans scripts)

### 1. Créer l'utilisateur

```bash
sudo useradd -m -d /sftp/egx -s /sbin/nologin -G sftpusers egx
sudo mkdir -p /sftp/egx/home
sudo chown root:root /sftp/egx
sudo chmod 755 /sftp/egx
sudo chown egx:sftpusers /sftp/egx/home
```

### 2. Créer le dossier de données

```bash
sudo mkdir -p /mnt/sftp_data/shared_data/egx
sudo chown root:sftpusers /mnt/sftp_data/shared_data/egx
```

### 3. Monter le dossier (bind)

```bash
sudo mkdir -p /sftp/egx/home/egx
sudo mount --bind /mnt/sftp_data/shared_data/egx /sftp/egx/home/egx
```

### 4. Créer l’unité systemd (recommandé)

`/etc/systemd/system/sftp-egx-home-egx.mount` :

```ini
[Unit]
Description=Bind mount for EGX
After=network.target

[Mount]
What=/mnt/sftp_data/shared_data/egx
Where=/sftp/egx/home/egx
Type=none
Options=bind

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sftp-egx-home-egx.mount
```

### 5. Appliquer les droits (ACL)

Lecture seule :

```bash
sudo setfacl -Rm u:egx:rx /mnt/sftp_data/shared_data/egx
sudo setfacl -dRm u:egx:rx /mnt/sftp_data/shared_data/egx
```

Lecture/écriture :

```bash
sudo setfacl -Rm u:egx:rwx /mnt/sftp_data/shared_data/egx
sudo setfacl -dRm u:egx:rwx /mnt/sftp_data/shared_data/egx
```

---

## 📎 Fichiers importants du projet

| Fichier                             | Description                                                  |
| ----------------------------------- | ------------------------------------------------------------ |
| `output_utilisateurs.csv`           | Liste des utilisateurs, partages et droits                   |
| `analyze_filezilla_sftp_map.py`     | Analyse FileZilla XML et aide à remplir le CSV               |
| `init_users_and_folders.sh`         | Crée les users + chroot + dossiers partagés                  |
| `bind_shared_folders_with_units.sh` | Crée les montages bind + unités systemd + ACL                |
| `restore_sftp_data.sh`              | Copie les données de l’ancien serveur dans les bons dossiers |

---

## ✅ Résultat final

- Les utilisateurs **ne voient que leurs dossiers**  
- Les accès sont **sécurisés et chrootés**  
- Les données sont **stockées uniquement** sur le **disque de 1 To**  
- L’infrastructure est **persistante au redémarrage**  
- Et toute la mise en place peut être **entièrement automatisée** ou refaite à la main si besoin
