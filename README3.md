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

-------------------------------------->>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>


Haan bhai 👍 main tumhe **complete verification commands** ek jagah de deta hoon, taaki future mein scheduled backup check karte time easily use kar sako.

## 1. Database backup — S3 mein latest backup check

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/ --recursive
```

Latest backup identify karne ke liye:

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/ --recursive | tail -5
```

---

## 2. DB backup download

Example 26 Sep backup:

```bash
aws s3 cp s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/2026-09-26/yyjcpl_crm_2026-09-26_02-00-01.sql.gz /tmp/crm-backup.sql.gz
```

Extract:

```bash
gunzip -c /tmp/crm-backup.sql.gz > /tmp/crm-backup.sql
```

---

# 3. Documents backup check

Backup SQL file mein:

```bash
grep -in "Dumping data for table \`documents\`" /tmp/crm-backup.sql
```

Documents ke INSERT records:

```bash
grep -in "INSERT INTO \`documents\`" /tmp/crm-backup.sql
```

---

# 4. Attendance backup check

```bash
grep -in "Dumping data for table \`employee_attendance\`" /tmp/crm-backup.sql
```

Attendance INSERT:

```bash
grep -in "INSERT INTO \`employee_attendance\`" /tmp/crm-backup.sql
```

25 Sep ke records specifically search karne ke liye:

```bash
grep -n "2026-09-25" /tmp/crm-backup.sql
```

---

# 5. Candidates backup check

Table section:

```bash
grep -in "Dumping data for table \`candidates\`" /tmp/crm-backup.sql
```

Candidate INSERT:

```bash
grep -in "INSERT INTO \`candidates\`" /tmp/crm-backup.sql
```

Specific candidate:

```bash
grep -in "aniket Jadhav" /tmp/crm-backup.sql
```

---

# 6. Uploads backup — S3

Ye **database se separate** hai.

Saari uploaded backup files:

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/uploads/ --recursive
```

Latest 20 files:

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/uploads/ --recursive | tail -20
```

Specific test PDF:

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/uploads/ --recursive | grep "cacbf846"
```

Agar output aa gaya, to **Testing for backup wali PDF S3 uploads backup mein present hai**.

---

## 7. Uploads backup ka total count

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/uploads/ --recursive | wc -l
```

---

### Simple yaad rakhne wali cheez

**Database backup:**

```text
S3 → mysql/yyjcpl_crm/
       ↓
       .sql.gz
       ↓
       Documents
       Attendance
       Candidates
       Employees
       etc.
```

**Uploads backup:**

```text
S3 → uploads/
       ↓
       Actual PDF / images / files
```
---------------------------------------------------------------->>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

## 😄🔹 MySQL / SQL commands

Pehle login:

```bash
mysql -h 127.0.0.1 -P 3306 -u yyjcpl_admin -p
```

Password enter karo. Phir:

### 1. Database select

```sql
USE yyjcpl_crm;
```

### 2. Documents — total records

```sql
SELECT COUNT(*) AS total_documents FROM documents;
```

### 3. Documents — records dekhna

```sql
SELECT * FROM documents;
```

### 4. Attendance — 25 Sep 2026 ke Present records

```sql
SELECT *
FROM employee_attendance
WHERE attendance_date = '2026-09-25'
AND status = 'present';
```

### 5. Attendance — names ke saath

```sql
SELECT
    e.full_name,
    e.department,
    e.designation,
    ea.status,
    ea.check_in,
    ea.check_out,
    ea.remark
FROM employee_attendance ea
JOIN employees e ON e.id = ea.employee_id
WHERE ea.attendance_date = '2026-09-25'
AND ea.status = 'present'
ORDER BY e.full_name;
```

### 6. Candidates — 25 Sep ko added

```sql
SELECT
    full_name,
    email,
    phone,
    current_location,
    preferred_location,
    experience_years,
    status,
    source,
    created_at
FROM candidates
WHERE created_at >= '2026-09-25 00:00:00'
AND created_at < '2026-09-26 00:00:00'
ORDER BY created_at;
```

### 7. Candidates — total count

```sql
SELECT COUNT(*) AS total_candidates FROM candidates;
```

### 8. Candidate ka uploaded resume/path dekhna

```sql
SELECT full_name, resume_url, created_at
FROM candidates
WHERE created_at >= '2026-09-25 00:00:00'
AND created_at < '2026-09-26 00:00:00';
```

---

### 🔹 Backup verification ka simple flow

**Live Database:**

```text
MySQL
 ↓
Documents
Attendance
Candidates
 ↓
Records verify
```

**Backup:**

```text
S3
 ↓
.sql.gz
 ↓
Extract
 ↓
SQL file
 ↓
Same records search
```

**Actual PDFs/files:**

```text
S3
 ↓
uploads/
 ↓
Actual PDF/image/file
```

Bas bhai, **SQL database ke records ke liye hai; S3 `uploads/` actual files ke liye hai.**

