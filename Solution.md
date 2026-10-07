You are already logged in: seeing

```text
snowadmin@blue-shift-team-05:~$
```

means your terminal is running commands **on the exam’s Linux server**.

There are three separate parts:

| Part | Purpose |
|---|---|
| WireGuard | Connects your computer to the exam network |
| `192.168.14.25` | Your team’s Linux server, accessed through SSH |
| `http://192.168.14.8:8000/` | The competition’s web panel, opened in your browser |

The web panel’s role is now clear from your instructions. Open it with the supplied team credentials and read the rules, tasks, and scoring requirements first. Do not treat that panel as a target to investigate.

Your practical objective is to **find and fix vulnerabilities while keeping the shop working**. Here is the order I would follow.

### 1. Read the rules and inspect the server

Keep your current SSH session open. Run these commands **one at a time**:

```bash
whoami
id
sudo -l
pwd
ls -la
sudo docker ps
sudo ss -tlnp
```

They show your account, permissions, current directory, running containers, and listening network ports.

From the earlier output, your server contains:

- `allyoucansnow`: the shop.
- `warehouse-api`: the warehouse and simulated partner services.
- `snowshop-db`: the database.

Check the shop before changing anything:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1/
curl -sS http://127.0.0.1:8080/health
```

Also open `http://192.168.14.25/` in your browser and try normal shop functions.

### 2. Make a fresh backup

The earlier `opt.bak` may contain the intact search page, so preserve it. Create a separate backup:

```bash
sudo bash <<'SH'
set -eu
task_backup="/root/exam-backup-$(date +%Y%m%d-%H%M%S)"
install -d -m 700 "$task_backup"
cp -a /opt/docker-apps "$task_backup/"
cp -a /opt/scripts "$task_backup/"
cp -a /etc/ssh "$task_backup/"
printf 'Backup saved at: %s\n' "$task_backup"
SH
```

Record the printed path. This copies application files and SSH settings; **it is not a backup of the running database**.

### 3. Read the relevant files

```bash
sudo cat /opt/scripts/export_ordini.sh
sudo cat /opt/docker-apps/warehouse-api/app.py
sudo cat /opt/docker-apps/allyoucansnow/app/cerca.php
sudo cat /opt/docker-apps/allyoucansnow/app/scheda.php
sudo cat /opt/docker-apps/allyoucansnow/rebuild.sh
sudo docker inspect allyoucansnow --format '{{json .Mounts}}'
```

Some files contain credentials. Remove passwords and keys before sharing their output.

Based on the code you previously provided, these are the main findings:

| Finding | Practical fix |
|---|---|
| `cerca.php` joins user input directly into SQL | Use a prepared statement, which passes input separately from the database command |
| Its “no results” message displays input directly | Escape the input with `htmlspecialchars` |
| `scheda.php` includes a user-selected path | Allow only approved document names and display static files with `readfile` |
| Export script was writable by everyone | Restrict ownership and permissions according to its actual execution account |
| `/internal/config` returns a secret API key | Remove the secret from responses and protect internal routes while updating legitimate callers |

### 4. Fix the search page first

Your earlier Base64 command failed, so **check whether `cerca.php` is damaged before editing it**:

```bash
sudo sed -n '1,120p' /opt/docker-apps/allyoucansnow/app/cerca.php
sudo ls -l /root/opt.bak/docker-apps/allyoucansnow/app/cerca.php
```

If the current file is damaged and the backup is intact, restore **only that file**:

```bash
sudo cp -a /root/opt.bak/docker-apps/allyoucansnow/app/cerca.php \
  /opt/docker-apps/allyoucansnow/app/cerca.php
```

Then edit it:

```bash
sudo nano /opt/docker-apps/allyoucansnow/app/cerca.php
```

PHP code belongs **inside this file**, not directly at the shell prompt. The detailed replacements are in the [English repair document](sandbox:/workspace/scratch/d08cc259f1ea/server_security_fixes.docx).

### 5. Apply and verify each change

The `Mounts` output tells you whether editing the host file also changes the running container. If the application is copied into its image, you need to rebuild using the existing deployment process. Read `rebuild.sh` before executing it.

Check PHP syntax in the running container:

```bash
sudo docker exec allyoucansnow php -l /var/www/allyoucansnow/cerca.php
sudo docker exec allyoucansnow php -l /var/www/allyoucansnow/scheda.php
```

These commands check the **container’s current files**. If you have not deployed your edits yet, they may check the old version.

After deployment, test ordinary searches, product pages, images, checkout, and tracking. A successful syntax check alone does not prove the application works.

### 6. Handle the compromised password carefully

If changing passwords is forbidden, your options depend on the rules: restrict SSH access to approved sources or switch to verified key-based login if permitted.

**Do not disable password login, lock `snowadmin`, or change firewall rules until you know how your teammates and the competition checker connect.** Test a second working connection before changing SSH access.

If an attacker uses the same account and has `sudo`, file permissions alone cannot separate their access from yours.

Your immediate next step is: **read the web panel’s rules, run the inspection commands in step 1, and check the current `cerca.php`.** That establishes what is running and whether the earlier failed edit needs recovery before further hardening.
