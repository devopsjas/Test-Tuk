Bhai, bilkul! ❤️ Main tumhe ek reusable CRM Deployment Runbook de raha hoon, jise tum har baar use kar sakte ho jab developer latest ZIP bheje — chahe frontend update ho, backend update ho, ya database schema change ho.

Tumhara existing production setup hi use karenge: `crm.tuktuk.live`, EC2, PM2, Nginx, MySQL aur S3. Har deployment mein pehle backup, phir changes inspect, phir deployment aur testing.

# CRM Deployment Runbook — Step by Step

1\. Backup & safety checks

Live application, database aur uploads ki current state safe rakho.

2\. New ZIP inspect karo

Frontend, backend, dependencies aur SQL files ke changes identify karo.

3\. Correct deployment karo

Frontend, backend aur database ke liye alag procedure follow karo.

4\. Verify & rollback-ready raho

Live website, APIs, data aur logs check karo.

## Step 1 — Current production status check karo

EC2 server par SSH login karke ye commands run karo:

```
pm2 status
pm2 show crm-api
curl -sS -o /dev/null -w "CRM HTTP: %{http_code}\n" https://crm.tuktuk.live
```

Expected HTTP status `200` hai, lekin login-protected routes ka status alag ho sakta hai.

Production details:

| Item                   | Current configuration                          |
| ---------------------- | ---------------------------------------------- |
| Live application       | `/home/ubuntu/crm-app-new`                     |
| Frontend               | `/home/ubuntu/crm-app-new/dist`                |
| Backend entry point    | `/home/ubuntu/crm-app-new/server/src/index.js` |
| Backend process        | `crm-api`                                      |
| Production environment | `/home/ubuntu/crm-app-new/server/.env`         |
| Backup S3 bucket       | `crm-tuktuk-backups-352306493926`              |
| Database               | `yyjcpl_crm`                                   |
| Website                | `https://crm.tuktuk.live`                      |

Important: Ye production environment hai. Har deployment mein existing `.env`, database credentials, uploads aur Nginx configuration ko preserve karna hai.

## Step 2 — Current application aur database ka backup lo

### 2A. Frontend rollback backup

```
cd /home/ubuntu/crm-app-new

sudo cp -a dist "dist-backup-$(date +%Y%m%d-%H%M%S)"

ls -ld /home/ubuntu/crm-app-new/dist-backup-*
```

Backup ka path note kar lo. Verification hone tak delete mat karna.

### 2B. Database backup verify karo

Tumhare server par database backup already automated hai. Deployment ke liye latest backup confirm karne ke liye:

```
aws s3 ls s3://crm-tuktuk-backups-352306493926/mysql/yyjcpl_crm/ --recursive --region ap-south-1 | tail -n 10
```

Agar database change karna hai, to change se just pehle ek fresh backup banana preferable hai. Existing script use kar sakte ho:

```
sudo /usr/local/bin/crm-db-backup.sh
```

Output mein `CRM DB backup completed` aur S3 upload success confirm karo.

### 2C. Uploads backup verify karo

```
sudo tail -n 10 /var/log/crm-uploads-backup.log
```

Ye confirm karta hai ki scheduled uploads backup ke latest runs successful rahe hain. Production uploads folder hai:

```
/home/ubuntu/crm-app-new/server/uploads
```

## Step 3 — Developer ki new ZIP server par upload karo

Windows PowerShell mein developer ki ZIP ke actual path ke saath command run karo. Example:

```
scp -i "C:\Users\jaswa\Downloads\crm-server-key.pem" "C:\Users\jaswa\Downloads\yyjcpl-crm-aws (1).zip" ubuntu@3.109.90.44:/home/ubuntu/crm-update-latest.zip
```

ZIP ka naam har baar different ho sakta hai. Isliye actual filename verify karna.

Server par:

```
ls -lh /home/ubuntu/crm-update-latest.zip
unzip -l /home/ubuntu/crm-update-latest.zip | head -60
```

Is stage par live application mein koi change nahi hota.

## Step 4 — ZIP ko temporary folder mein extract karo

```
mkdir -p /home/ubuntu/crm-update-latest
unzip -oq /home/ubuntu/crm-update-latest.zip -d /home/ubuntu/crm-update-latest
```

Verify:

```
find /home/ubuntu/crm-update-latest -maxdepth 3 -type f | sort | head -100
```

Agar `appears to use backslashes as path separators` warning aaye, to extraction ke baad actual file paths verify karna. Tumhari previous ZIP successfully extract hui thi.

### Ab changes identify karo

Frontend references:

```
echo "=== CURRENT ==="
grep -Eo 'src="[^"]+|href="[^"]+' /home/ubuntu/crm-app-new/dist/index.html

echo "=== NEW ==="
grep -Eo 'src="[^"]+|href="[^"]+' /home/ubuntu/crm-update-latest/dist/index.html
```

Backend source differences:

```
diff -ru /home/ubuntu/crm-app-new/server/src /home/ubuntu/crm-update-latest/server/src
```

Root package differences:

```
diff -u /home/ubuntu/crm-app-new/package.json /home/ubuntu/crm-update-latest/package.json
```

Backend package differences:

```
diff -u /home/ubuntu/crm-app-new/server/package.json /home/ubuntu/crm-update-latest/server/package.json
```

SQL files bhi identify karo:

```
find /home/ubuntu/crm-update-latest -type f \( -iname "*.sql" -o -iname "*migration*" \)
```

### `.env` ko kabhi blindly replace mat karna

New ZIP mein `.env` ho to usko live `.env` par copy mat karna. Developer ki file mein different database name, user, password ya JWT secret ho sakta hai.

Secrets display karne ke bajay sirf differences ko carefully inspect karo. Agar credentials accidentally expose ho jaayein, unhe chat mein paste mat karna.

# Step 5 — Changes ke type ke hisaab se deployment

## Option A — Sirf frontend update

Tumne latest deployment mein isi procedure ko successfully use kiya tha.

Pehle fresh frontend backup lo (Step 2A). Phir:

```
cp -a /home/ubuntu/crm-update-latest/dist/. /home/ubuntu/crm-app-new/dist/
```

Verify:

```
grep -Eo 'src="[^"]+|href="[^"]+' /home/ubuntu/crm-app-new/dist/index.html
```

Uske baad browser mein [CRM website](https://crm.tuktuk.live) kholo aur `Ctrl + F5` press karo.

Check karo:

- Login
- Dashboard aur navigation
- Updated logo/images
- Relevant forms aur buttons
- Data load/save operations

Frontend-only update mein aam taur par PM2 restart ki zaroorat nahi hoti.

Note: Agar frontend ke saath API contract ya backend changes bhi hain, to sirf frontend deploy karna sufficient nahi hoga.

## Option B — Backend update

Backend ke liye sirf files copy karna kaafi nahi; pehle differences aur dependencies inspect karni hain.

Current PM2 setup:

```
Script: /home/ubuntu/crm-app-new/server/src/index.js
Working directory: /home/ubuntu/crm-app-new/server
```

Deploy karne se pehle:

1. Current backend source aur package files ka rollback backup lo.
2. New `server/src` ke changes review karo.
3. `.env` aur `server/uploads` preserve karo.
4. `server/package.json` aur lock file changes check karo.
5. Agar dependencies change hui hain, to compatible lock file ke saath dependency installation plan banao.
6. Database/API compatibility check karo.
7. Tabhi reviewed backend files replace karo.

Tumhare current production backend mein:

```
app.listen(port, "127.0.0.1", ...)
```

hai. Ye Nginx reverse proxy ke saath localhost par listen karta hai. Isko blindly:

```
app.listen(port, ...)
```

se replace mat karna, kyunki binding behavior change ho sakta hai.

### Backend deploy hone ke baad

Agar backend files ya dependencies successfully update hui hain, tab:

```
pm2 restart crm-api
pm2 status
pm2 logs crm-api --lines 50
```

Logs mein errors, database connection failures aur application startup confirm karo. API health endpoint available ho to usko bhi test karo.

Agar startup fail ho, baar-baar restart mat karna. Logs dekho aur rollback plan use karo.

## Option C — Database/schema update

Bhai, ye sabse sensitive update hai. SQL file milne ka matlab ye nahi ki use directly production mein execute kar dena chahiye.

Pehle:

1. SQL file ka complete content inspect karo.
2. Check karo ki woh schema creation, `ALTER TABLE`, data updates, ya destructive `DROP`/`DELETE` statements contain karti hai.
3. Fresh database backup lo aur S3 upload success verify karo.
4. Ideally staging/test database par migration run karo.
5. Migration ke baad application code compatibility confirm karo.
6. Production mein approved SQL statements hi run karo.

Database backup script:

```
sudo /usr/local/bin/crm-db-backup.sh
```

Successful backup ke baad hi approved migration process shuru karo.

Database restore aur schema migration do alag operations hain. Backup ko restore karna existing data overwrite kar sakta hai; migration ko carefully review karna zaroori hai.

## Option D — Database credentials/configuration change

Agar developer ne DB host, database name, credentials ya environment variables change karne ko kaha hai:

- Pehle reason confirm karo.
- Production `.env` ka protected backup lo.
- Existing database aur permissions verify karo.
- New credentials ko separately validate karo.
- Application ko tabhi restart karo jab updated configuration ready aur verified ho.

Production `.env` ko ZIP se overwrite mat karna.

# Step 6 — Deployment ke baad final verification

Har deployment ke baad:

```
pm2 status
curl -sS -o /dev/null -w "HTTP %{http_code}\n" https://crm.tuktuk.live
```

Phir browser mein login, navigation, relevant forms, data operations aur images test karo.

Agar backend ya database changes hue hain, to sirf homepage open hona enough nahi hai. API aur changed functionality bhi verify karni hogi.

## Step 7 — Rollback plan

Agar frontend update fail ho:

1. Deployment se pehle note kiya hua `dist-backup-...` folder locate karo.
2. Current broken `dist` ko alag folder mein preserve karo.
3. Backup ko `dist` mein restore karo.
4. Browser cache refresh karke verify karo.

Backend rollback mein matching previous source files, dependencies aur configuration restore karni hogi. Database migration rollback alag process hai aur har migration automatically reversible nahi hoti.

Temporary ZIP/extraction folder tabhi delete karo jab deployment successful ho, testing complete ho aur rollback backup confirm ho. Database backups aur existing uploads ko kabhi cleanup command se remove mat karna.

## Step 8 — S3 backup policy

Tumhare current setup mein:

| Backup type     | Current approach                   |
| --------------- | ---------------------------------- |
| Database        | Automated, daily 07:30 AM IST      |
| Uploads         | Automated, daily 08:00 AM IST      |
| Application ZIP | Manual upload to S3 `application/` |

Application code ka automated backup abhi set nahi kiya hai; tum manual ZIP upload karna prefer karte ho. Ye theek hai, bas production deployment se pehle ek clean, verified application archive rakhna.

Manual application ZIP mein `.env`, passwords, private keys, aur unnecessary uploads include mat karna. S3 mein existing backups ko overwrite/delete karne se bhi bachna.

## Har baar mujhse deployment karwane ka easy tareeka

Jab developer nayi ZIP de, mujhe bas ye message bhej dena:

Bhai, developer ne CRM ki new ZIP di hai. Isko production server par safely deploy karna hai. Pehle ZIP inspect karwao, current backup verify karwao, frontend/backend/database changes identify karo, production .env aur uploads preserve rakho, aur changes ke type ke hisaab se step-by-step commands do. Ek time par ek step dena, output verify karna, phir next step. Production database migration bina review aur backup ke execute mat karna. Deployment ke baad website, API aur relevant functionality verify karwana. Rollback backup preserve rakhna.

Is tarah har baar tumhe poori history dobara explain nahi karni padegi. Hum pehle identify karenge ki ZIP mein exactly kya change hua hai, phir usi ke hisaab se deployment karenge.
