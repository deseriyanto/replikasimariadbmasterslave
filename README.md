Untuk mengkonfigurasi replikasi Master-Slave pada MariaDB, Anda perlu melakukan beberapa langkah konfigurasi di kedua server (Master dan Slave). Berikut adalah langkah-langkahnya:

### 1. Konfigurasi pada Server Master (172.16.88.127)

#### a. Edit File Konfigurasi MariaDB
Buka file konfigurasi MariaDB (biasanya terletak di `/etc/my.cnf` atau `/etc/mysql/mariadb.conf.d/50-server.cnf`) dan tambahkan atau pastikan konfigurasi berikut ada di bawah bagian `[mysqld]`:

```ini
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-do-db=v2_ikafh_db
bind-address=0.0.0.0
```

- `server-id=1`: Ini adalah ID unik untuk server Master. Pastikan ID ini berbeda dengan server Slave.
- `log-bin=mysql-bin`: Mengaktifkan binary logging yang diperlukan untuk replikasi.
- `binlog-do-db=v2_ikafh_db`: Menentukan database mana yang akan direplikasi.
- `bind-address=0.0.0.0`: Mengizinkan koneksi dari semua IP. Anda bisa membatasi ini ke IP Slave jika diperlukan.

#### b. Restart MariaDB
Setelah mengedit file konfigurasi, restart MariaDB untuk menerapkan perubahan:

```bash
sudo systemctl restart mariadb
```

#### c. Buat User Replikasi
Masuk ke MariaDB sebagai root dan buat user khusus untuk replikasi:

```sql
CREATE USER 'replica_user'@'172.16.19.141' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'172.16.19.141';
FLUSH PRIVILEGES;
```

#### d. Lock Database dan Catat Posisi Binary Log
Lock database untuk memastikan tidak ada perubahan selama proses setup replikasi:

```sql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

Catat nilai `File` dan `Position` dari output `SHOW MASTER STATUS;`. Anda akan membutuhkannya untuk konfigurasi Slave.

#### e. Backup Database
Backup database yang akan direplikasi:

```bash
mysqldump -u root -p v2_ikafh_db > v2_ikafh_db.sql
```

#### f. Unlock Database
Setelah backup selesai, unlock database:

```sql
UNLOCK TABLES;
```

### 2. Konfigurasi pada Server Slave (172.16.19.141)

#### a. Edit File Konfigurasi MariaDB
Buka file konfigurasi MariaDB dan tambahkan atau pastikan konfigurasi berikut ada di bawah bagian `[mysqld]`:

```ini
[mysqld]
server-id=2
relay-log=mysql-relay-bin
read-only=1
```

- `server-id=2`: Ini adalah ID unik untuk server Slave. Pastikan ID ini berbeda dengan server Master.
- `relay-log=mysql-relay-bin`: Mengaktifkan relay log.
- `read-only=1`: Mengatur server Slave dalam mode read-only untuk mencegah perubahan data di Slave.

#### b. Restart MariaDB
Restart MariaDB untuk menerapkan perubahan:

```bash
sudo systemctl restart mariadb
```

#### c. Restore Database
Restore database yang telah di-backup dari Master:

```bash
mysql -u root -p v2_ikafh_db < v2_ikafh_db.sql
```

#### d. Konfigurasi Replikasi
Masuk ke MariaDB sebagai root dan konfigurasi replikasi:

```sql
CHANGE MASTER TO
MASTER_HOST='172.16.88.127',
MASTER_USER='replica_user',
MASTER_PASSWORD='password',
MASTER_LOG_FILE='mysql-bin.000001', -- Ganti dengan nilai File dari SHOW MASTER STATUS di Master
MASTER_LOG_POS=123; -- Ganti dengan nilai Position dari SHOW MASTER STATUS di Master

START SLAVE;
```

#### e. Periksa Status Replikasi
Periksa status replikasi untuk memastikan semuanya berjalan dengan baik:

```sql
SHOW SLAVE STATUS\G
```

Pastikan nilai `Slave_IO_Running` dan `Slave_SQL_Running` adalah `Yes`. Jika ada masalah, periksa log error untuk detail lebih lanjut.

### 3. Testing Replikasi
- Lakukan perubahan data di database `v2_ikafh_db` pada server Master.
- Periksa apakah perubahan tersebut terlihat di server Slave.

Dengan mengikuti langkah-langkah di atas, Anda seharusnya dapat mengkonfigurasi replikasi Master-Slave pada MariaDB dengan sukses.
