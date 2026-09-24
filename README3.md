| Requirement                                            | Status                           |
| ------------------------------------------------------ | -------------------------------- |
| CRM ko AWS par run karna                               | ✅ Complete                       |
| `crm-aws.tuktuk.live` working                          | ✅ Complete                       |
| HTTPS/SSL                                              | ✅ Complete                       |
| Nginx reverse proxy                                    | ✅ Complete                       |
| Node/PM2 service                                       | ✅ Online                         |
| MySQL database AWS par                                 | ✅ Working                        |
| MySQL ko public access se protect karna                | ✅ Complete                       |
| Node port 3000 ko public access se protect karna       | ✅ Complete                       |
| Daily DB backup                                        | ✅ Configured — 2 AM              |
| Backup S3 mein                                         | ✅ Complete                       |
| S3 backup encryption                                   | ✅ KMS encrypted                  |
| Backup bucket private/versioned                        | ✅ Complete                       |
| Backup restore test                                    | ✅ Successfully tested            |
| Server root EBS encryption                             | ✅ Complete                       |
| 20 GB root disk retained                               | ✅ Complete                       |
| Old root disk rollback ke liye                         | ✅ Detached & preserved           |
| Developer ke future ZIP/DB update ke liye server ready | ✅ Yes                            |
| Backup retention period                                | ⏳ **Sir se requirement pending** |
| RDS migration                                          | ⏳ **Abhi deferred**              |
| Developer ka **new updated ZIP**                       | ⏳ Developer ko provide karna hai |
| Developer ka **new/latest DB**                         | ⏳ Developer ko provide karna hai |


Baad mein production cutover ke time:
Current AWS URL
crm-aws.tuktuk.live
⬇️
Final production URL
crm.tuktuk.live

Us time DNS/domain ko AWS server par point karna hoga aur SSL ko bhi crm.tuktuk.live ke liye configure/issue karna hoga. Abhi existing Hostinger crm.tuktuk.live ko touch nahi karna hai.
Aur jab crm.tuktuk.live ko AWS par shift karenge, tab new domain ke liye SSL bhi configure kar denge.



Haan bhai 👍 Ab tumhare paas **2 type ke commands** honge:

1. **Live database** mein kya data hai check karna.
2. **S3 backup** ke andar kya data hai check karna.

Main tumhe ek **cheat sheet** de raha hoon. Jab Sir/developer bole "attendance check karo", "clients check karo", "backup mein data hai?" etc., directly relevant command use kar lena.

---

## 1. Live DB — sabhi tables ka row count

Ye tumhara main command hai:

```bash
mysql -h 127.0.0.1 -P 3306 -u yyjcpl_admin -p yyjcpl_crm -e "SELECT TABLE_NAME, TABLE_ROWS FROM information_schema.TABLES WHERE TABLE_SCHEMA='yyjcpl_crm' ORDER BY TABLE_NAME;"
```

Ye **actual records nahi dikhata**, sirf table aur approximate row count.

---

## 2. Live DB — kisi specific table ka exact count

Example `clients`:

```bash
mysql -h 127.0.0.1 -P 3306 -u yyjcpl_admin -p yyjcpl_crm -e "SELECT COUNT(*) AS total_records FROM clients;"
```

### Kisi bhi table ke liye

Bas `clients` ko table name se replace karo:

```text
branches
candidates
clients
employees
employee_attendance
employee_leaves
interviews
leads
profiles
students
transactions
```

Example attendance:

```bash
mysql -h 127.0.0.1 -P 3306 -u yyjcpl_admin -p yyjcpl_crm -e "SELECT COUNT(*) AS total_records FROM employee_attendance;"
```

---

# 3. S3 Backup — kisi table ka data hai ya nahi

Example `clients`:

```bash
aws s3 cp s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/2026-09-23/yyjcpl_crm_2026-09-23_12-18-12.sql.gz - | zcat | grep -c '^INSERT INTO `clients`'
```

Attendance:

```bash
aws s3 cp s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/2026-09-23/yyjcpl_crm_2026-09-23_12-18-12.sql.gz - | zcat | grep -c '^INSERT INTO `employee_attendance`'
```

⚠️ Ye **INSERT statements count** karta hai, exact rows nahi.

---

# 4. Backup mein kaun-kaun se tables hain

```bash
aws s3 cp s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/2026-09-23/yyjcpl_crm_2026-09-23_12-18-12.sql.gz - | zcat | grep -E '^CREATE TABLE' | sed 's/CREATE TABLE `//;s/`.*//'
```

Isse sirf table names aayenge.

---

# 5. Backup mein kitni tables hain

```bash
aws s3 cp s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/2026-09-23/yyjcpl_crm_2026-09-23_12-18-12.sql.gz - | zcat | grep -cE '^CREATE TABLE'
```

Abhi expected result **34** hai.

---

# 6. Backup file valid/corrupt check

Agar file pehle download ki hui hai:

```bash
gzip -t /tmp/crm-backup-check.sql.gz && echo "GZIP OK"
```

`GZIP OK` = backup compressed file valid hai. ✅

---

# 7. S3 mein available backups dekhna

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/ --recursive --human-readable --summarize
```

Isse tumhe **date, filename aur size** dikhega.

---

# 8. Latest backup identify karna

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/ --recursive | tail -1
```

---

## ⭐ Sabse important

Agar developer bole:

**"Attendance mein kitne records hain?"**

Live DB:

```bash
mysql -h 127.0.0.1 -P 3306 -u yyjcpl_admin -p yyjcpl_crm -e "SELECT COUNT(*) AS total_records FROM employee_attendance;"
```

**"Clients mein kitne records hain?"**

```bash
mysql -h 127.0.0.1 -P 3306 -u yyjcpl_admin -p yyjcpl_crm -e "SELECT COUNT(*) AS total_records FROM clients;"
```

**"Backup mein clients ka data hai?"**

```bash
aws s3 cp s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/2026-09-23/yyjcpl_crm_2026-09-23_12-18-12.sql.gz - | zcat | grep -c '^INSERT INTO `clients`'
```

**"Backup mein kya-kya hai?"**

```bash
aws s3 cp s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/2026-09-23/yyjcpl_crm_2026-09-23_12-18-12.sql.gz - | zcat | grep -E '^CREATE TABLE' | sed 's/CREATE TABLE `//;s/`.*//'
```

**Important:** S3 backup ka filename/date future mein change hoga. Pehle `aws s3 ls ...` se latest filename dekh lena, phir command mein us filename ko use karna.

Aur **raw `INSERT INTO` command ka output chat mein paste mat karna**, kyunki usme CRM users/data/password hashes aa sakte hain.


