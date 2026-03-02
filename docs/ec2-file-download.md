# EC2 Connection, Folder Backup, SQL Dump, and Download Guide

![alt text](assets/images/image-2.png)

## 1. Connecting to an EC2 Instance

### Method 1: EC2 Instance Connect (Browser SSH)
Follow these steps to establish a connection:
1. Open the **AWS Console**.
2. Go to: **EC2** → **Instances**.
3. Select your running instance.
4. Click **Connect**.
5. Choose **EC2 Instance Connect**.
6. Click **Connect**.

You now have terminal access inside the browser.

### Method 2: SSH Using a PEM Key (From Local Machine)
Use the following command from your terminal:
```bash
ssh -i my-key.pem ubuntu@12.34.56.78
```
**Requirements:**
*   The correct `.pem` file.
*   Port 22 open in the **Security Group**.
*   A public IPv4 address available.

## 2. Checking MySQL Server Status
Verify if the database is running using this command:
```bash
sudo systemctl status mysql
```
**Expected Output:**
*   Active: **active (running)**.
*   Status: **Server is operational**.

**If MySQL Is Not Running:**
Start the service manually:
```bash
sudo systemctl start mysql
```

## 3. Locating Project and Database Credentials
Assume your project folder is located at `/var/www/html/sampleapp`.

Navigate to the project directory:
```bash
cd /var/www/html/sampleapp
```
View the environment file:
```bash
cat .env
```
Look for the following variables:
```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=sampledb
DB_USERNAME=sampleuser
DB_PASSWORD=strongpassword
```
These credentials are used to create the database backup.

## 4. Creating a Database Backup (SQL Dump)

### Dump a Specific Database
Run this command to create a targeted backup:
```bash
mysqldump -u sampleuser -p sampledb > sampledb.sql
```
This creates `sampledb.sql` in the current directory.

### Dump All Databases
Run this command to back up everything:
```bash
mysqldump -u sampleuser -p --all-databases > fulldbbackup.sql
```

## 5. Creating a ZIP Backup of the Project Folder
Navigate to the parent directory:
```bash
cd /var/www/html
```
Create the ZIP file:
```bash
zip -r sampleapp-backup.zip sampleapp
```
**Explanation of parameters:**
*   `zip` → Creates the archive.
*   `-r` → Recursive flag (includes all subfolders).
*   `sampleapp-backup.zip` → The output file name.
*   `sampleapp` → The folder being zipped.

## 6. Move Backup Files to Home Directory
Move the files for easy download:
```bash
sudo cp sampleapp-backup.zip /home/ubuntu/
sudo cp sampledb.sql /home/ubuntu/
```
Verify the transfer:
```bash
ls -lh /home/ubuntu
```
**Expected Output:**
*   `sampleapp-backup.zip`.
*   `sampledb.sql`.

## 7. Downloading Files from EC2

### Method 1: Temporary Python HTTP Server

**Step 1 — Open Port 8000 (Temporary)**
In **Security Group** → **Inbound rules**, add:
*   **Type**: Custom TCP.
*   **Port**: 8000.
*   **Source**: 0.0.0.0/0.

Save changes.

**Step 2 — Start Server**
Run the server with these commands:
```bash
cd /home/ubuntu
python3 -m http.server 8000
```
The server runs at: `http://0.0.0.0:8000/`.

**Step 3 — Download in Browser**
Open these links to download:
*   `http://12.34.56.78:8000/sampleapp-backup.zip`.
*   or `http://12.34.56.78:8000/sampledb.sql`.

**Step 4 — Stop Server**
Press **CTRL + C**.

**Step 5 — Remove Port 8000**
Go back to: **EC2** → **Security Groups** → **Edit inbound rules**.
Remove port 8000.

### Method 2: Secure Download Using SCP
From your local machine, run:
```bash
scp -i my-key.pem ubuntu@12.34.56.78:/home/ubuntu/sampleapp-backup.zip .
scp -i my-key.pem ubuntu@12.34.56.78:/home/ubuntu/sampledb.sql .
```
This downloads files securely without opening extra ports.

## 8. Common Errors and Solutions

**Error: Access denied for user ‘root’@‘localhost’**
*   **Cause**: Root password unknown, or production uses a separate database user.
*   **Fix**: Use database credentials from the `.env` file.

**Error: Permission denied (publickey)**
*   **Cause**: Wrong or missing `.pem` key.
*   **Fix**: Use correct key file and ensure port 22 is open.

## 9. Safe Production Practices
*   Do not leave temporary ports open.
*   Do not use the **root** user unnecessarily.
*   Always keep a Project ZIP backup and SQL database dump.
*   Store backups locally.
*   Remove temporary services after use.

## 10. Final Backup Checklist
After completion, you should have locally:
*   `sampleapp-backup.zip`.
*   `sampledb.sql`.

**This Ensures:**
*   Full project recovery.
*   Full database recovery.
*   Production safety.