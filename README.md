## 🎯 Objectif

Configurer un **serveur SFTP sécurisé**, où chaque utilisateur a un environnement chrooté (`/sftp/USER/home`) et accède à **des partages centralisés** (via `bind mount`) stockés sur un **disque dédié de 1To**.

---

## 🧩 Architecture

+-----------------------------+
| Disque partagé SFTP |
| /mnt/sftp_data/shared_data|
+-------------+---------------+
|
| --bind
v
/sftp/<utilisateur>/home/<partage>

yaml
Toujours afficher les détails

Copier

---

## 🗂️ Arborescence principale

- `/sftp/` : chroot des utilisateurs
- `/mnt/sftp_data/shared_data/` : **lieu centralisé de stockage**
- `/mnt/ftp_win/` : monté temporairement en CIFS, contient les anciennes données (via `//IP/FTP_SHARE`)
- `output_utilisateurs.csv` : mapping `utilisateur;partage;droit (r|rw)`

---

## ✅ Étapes automatiques (avec scripts)

> Ces scripts sont fournis dans le dossier `~/py_sftp` :

| Ordre | Script                             | Rôle principal |
|-------|------------------------------------|----------------|
| 1     | `create_sftp_structure_from_csv.sh`| Crée les chroots utilisateur |
| 2     | `prepare_shared_data_dirs.sh`      | Crée les dossiers dans `/mnt/sftp_data` |
| 3     | `clean_mount_units_strict.sh`      | Supprime doublons de units systemd |
| 4     | `create_bind_mounts_from_csv.sh`   | Crée les `.mount` units + bind mount |
| 5     | `restore_sftp_data.sh`             | Copie les données depuis `/mnt/ftp_win` |
| 6     | `apply_sftp_acl.sh`                | Applique les ACL selon le CSV |

---

## 🛠️ Refaire la configuration à la main (sans script)

### 1. Créer les utilisateurs et leur chroot

```bash
sudo useradd -m -d /sftp/egx -s /sbin/nologin -G sftpusers egx
sudo mkdir -p /sftp/egx/home
sudo chown root:root /sftp/egx
sudo chmod 755 /sftp/egx
sudo chown egx:sftpusers /sftp/egx/home
2. Créer le dossier de données partagées
bash
Toujours afficher les détails

Copier
sudo mkdir -p /mnt/sftp_data/shared_data/egx
sudo chown root:sftpusers /mnt/sftp_data/shared_data/egx
3. Monter en bind le dossier partagé
bash
Toujours afficher les détails

Copier
sudo mkdir -p /sftp/egx/home/egx
sudo mount --bind /mnt/sftp_data/shared_data/egx /sftp/egx/home/egx
4. Créer une unité systemd (facultatif mais recommandé)
ini
Toujours afficher les détails

Copier
# /etc/systemd/system/sftp-egx-home-egx.mount
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
Puis :

bash
Toujours afficher les détails

Copier
sudo systemctl daemon-reload
sudo systemctl enable --now sftp-egx-home-egx.mount
5. Appliquer les ACL (droits d'accès)
Lecture seule :

bash
Toujours afficher les détails

Copier
sudo setfacl -Rm u:egx:rx /mnt/sftp_data/shared_data/egx
sudo setfacl -dRm u:egx:rx /mnt/sftp_data/shared_data/egx
Lecture / Écriture :

bash
Toujours afficher les détails

Copier
sudo setfacl -Rm u:egx:rwx /mnt/sftp_data/shared_data/egx
sudo setfacl -dRm u:egx:rwx /mnt/sftp_data/shared_data/egx
🔁 Mise à jour des données
Si des données sont rajoutées côté ancien serveur :
✅ Tu peux relancer restore_sftp_data.sh sans créer de doublons ni casser les accès.

🧪 Vérification
mount | grep /sftp/ → vérifie que tous les mounts sont actifs

df -h → vérifie que /mnt/sftp_data est bien utilisé (pas /)

✅ Résultat final
Chaque utilisateur voit uniquement ses dossiers, avec les droits définis, et les données sont sur le disque de 1To.

