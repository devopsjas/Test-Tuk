Haan bhai. Developer ko CRM AWS EC2 par kaam karne ke liye ek **access/operations cheat sheet** de dete hain. Main password/secret ko chat mein expose nahi karunga; usko server par safely check karne ka command dunga.

## 🖥️ CRM EC2 — Important Details

**EC2 Instance**

* Instance ID: `i-0ffeee0176d5dab98`
* Private IP: `172.31.12.5`
* OS: Ubuntu
* User: `ubuntu`
* CRM domain: `https://crm-aws.tuktuk.live`

### 1. SSH access

Developer ke paas approved SSH private key honi chahiye.

General command:

```bash
ssh -i /path/to/crm-server-key.pem ubuntu@<CURRENT_PUBLIC_IP>
```

⚠️ **Private `.pem` key developer ko openly share nahi karni chahiye.** Agar developer ko access dena hai, better hai uske liye separate SSH key/user setup karein.

---

# 📁 2. CRM Application kaha hai?

Current application:

```bash
/home/ubuntu/crm-app-new/
```

Server code:

```bash
/home/ubuntu/crm-app-new/server/
```

Main entry file:

```bash
/home/ubuntu/crm-app-new/server/src/index.js
```

Application ko manually dekhne ke liye:

```bash
cd /home/ubuntu/crm-app-new
ls -la
```

Server folder:

```bash
cd /home/ubuntu/crm-app-new/server
ls -la
```

---

# ⚙️ 3. PM2 / CRM API

PM2 application name:

```text
crm-api
```

Status:

```bash
pm2 status
```

Logs:

```bash
pm2 logs crm-api
```

Last logs:

```bash
pm2 logs crm-api --lines 100
```

Restart:

```bash
pm2 restart crm-api
```

⚠️ Developer ko `pm2 delete crm-api` ya `pm2 stop crm-api` production/test approval ke bina nahi karna chahiye.

---

# 🗄️ 4. Database kaha hai?

Database **MySQL** mein hai.

Database name:

```text
yyjcpl_crm
```

MySQL server:

```text
127.0.0.1:3306
```

Iska important point:

**Database public internet par exposed nahi hai.**

Server ke andar se MySQL check:

```bash
sudo mysql
```

Databases dekhne:

```sql
SHOW DATABASES;
```

CRM database:

```sql
USE yyjcpl_crm;
```

Tables:

```sql
SHOW TABLES;
```

MySQL se exit:

```sql
EXIT;
```

---

# 🔑 5. Database password

**Database password chat/WhatsApp/document mein plain text mein share mat karna.**

Server par application ki environment/configuration se check karna hai.

Pehle application directory:

```bash
cd /home/ubuntu/crm-app-new/server
```

Environment-related files dekhne:

```bash
ls -la
```

Agar `.env` file hai:

```bash
sudo nano .env
```

⚠️ `.env` ka complete content developer ko paste/share mat karna, kyunki usmein DB password, JWT secrets, API keys etc. ho sakte hain.

Agar PM2 environment variables use kar raha hai:

```bash
pm2 env 0
```

**Is command ka output publicly/chat mein paste mat karna**, kyunki secrets aa sakte hain.

---

# 🌐 6. Nginx configuration

Nginx configs generally:

```bash
/etc/nginx/
```

Sites:

```bash
ls -la /etc/nginx/sites-available/
```

Enabled sites:

```bash
ls -la /etc/nginx/sites-enabled/
```

Nginx configuration test:

```bash
sudo nginx -t
```

Nginx restart:

```bash
sudo systemctl restart nginx
```

Status:

```bash
sudo systemctl status nginx
```

---

# 🔐 7. SSL kaha hai?

Let's Encrypt certificate:

```bash
/etc/letsencrypt/live/crm-aws.tuktuk.live/
```

Certificate status:

```bash
sudo certbot certificates
```

Renewal test:

```bash
sudo certbot renew --dry-run
```

**Certificate/private key files ko developer ko manually share nahi karna hai.**

---

# 💾 8. Database backup

Backup script:

```bash
/usr/local/bin/crm-db-backup.sh
```

Script dekhne:

```bash
sudo cat /usr/local/bin/crm-db-backup.sh
```

Backup schedule:

```bash
sudo cat /etc/cron.d/crm-db-backup
```

Current schedule:

```text
Every day at 2:00 AM
```

Backup log:

```bash
sudo tail -100 /var/log/crm-db-backup.log
```

---

# ☁️ 9. Backup S3 mein kaha jaata hai?

Bucket:

```text
crm-tuktuk-backups-352306493926
```

Path structure:

```text
mysql/yyjcpl_crm/YYYY-MM-DD/
```

Example:

```text
mysql/yyjcpl_crm/2026-09-23/
```

Server se backup list check:

```bash
aws s3 ls s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/ --recursive
```

⚠️ Backup bucket ko public nahi karna hai.

---

# 🔑 10. AWS access

EC2 mein AWS CLI **IAM Role** ke through configured hai.

Check:

```bash
aws sts get-caller-identity
```

AWS account/role information dikhegi.

**AWS access keys `.env` mein create/store karne ki zarurat nahi hai.**

---

# 🔐 11. EBS Root Disk

Current encrypted root volume:

```text
vol-0a2a8724f22840fd3
```

Check:

```bash
lsblk
```

AWS side se encryption check:

```bash
aws ec2 describe-volumes \
  --region ap-south-1 \
  --volume-ids vol-0a2a8724f22840fd3 \
  --query 'Volumes[0].{Encrypted:Encrypted,Size:Size,State:State,KmsKeyId:KmsKeyId}' \
  --output table
```

---

# 🔍 12. Server ke important health commands

Developer ke liye ye commands sabse useful hain:

### All services

```bash
sudo systemctl is-active nginx
sudo systemctl is-active mysql
pm2 status
```

### Ports

```bash
sudo ss -lntp
```

### CRM local API

```bash
curl -I http://127.0.0.1:3000
```

### Public CRM

```bash
curl -I https://crm-aws.tuktuk.live
```

Expected:

```text
HTTP/1.1 200 OK
```

### Disk

```bash
df -h
```

### RAM

```bash
free -h
```

### CPU/processes

```bash
top
```

---

# 🛑 13. Important commands — developer ko careful rehna hai

Ye commands **without approval** use nahi karne:

```bash
sudo systemctl stop mysql
```

```bash
sudo systemctl stop nginx
```

```bash
pm2 stop crm-api
```

```bash
pm2 delete crm-api
```

```bash
sudo reboot
```

Aur especially:

```bash
sudo rm -rf ...
```

Database mein:

```sql
DROP DATABASE ...
```

ya

```sql
DELETE FROM ...
```

**production CRM data ke liye dangerous hain.**

---

## 📌 Developer ke liye short cheat sheet

| Kaam           | Command/Path                                 |
| -------------- | -------------------------------------------- |
| CRM app        | `/home/ubuntu/crm-app-new/`                  |
| Backend        | `/home/ubuntu/crm-app-new/server/`           |
| Main Node file | `server/src/index.js`                        |
| Database       | `yyjcpl_crm`                                 |
| DB host        | `127.0.0.1:3306`                             |
| PM2            | `pm2 status`                                 |
| PM2 logs       | `pm2 logs crm-api`                           |
| Nginx config   | `/etc/nginx/`                                |
| SSL            | `/etc/letsencrypt/live/crm-aws.tuktuk.live/` |
| Backup script  | `/usr/local/bin/crm-db-backup.sh`            |
| Backup cron    | `/etc/cron.d/crm-db-backup`                  |
| Backup log     | `/var/log/crm-db-backup.log`                 |
| S3 backup      | `crm-tuktuk-backups-352306493926`            |
| DB check       | `sudo mysql`                                 |
| AWS identity   | `aws sts get-caller-identity`                |
| Disk           | `df -h` / `lsblk`                            |
| Services       | `sudo systemctl is-active nginx mysql`       |
| CRM health     | `curl -I https://crm-aws.tuktuk.live`        |

koi bhi chij agar server pe bheja hai to powershell mai download mai ja kar ye run karna 

scp -i "C:\Users\jaswa\Downloads\crm-server-key.pem" .\yyjcpl_uploads.zip ubuntu@3.109.90.44:/home/ubuntu/

### 🔴 Sabse important

Developer ko **DB password, `.env`, AWS credentials, KMS keys, SSL private key, Firebase/service-account files** casually share nahi karne hain.

Agar developer ko regular access dena hai, **separate Linux user + separate SSH public key** banana better hai. Isse usko apna access milega aur future mein us access ko independently revoke bhi kar sakte hain.
