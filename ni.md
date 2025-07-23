This is a comprehensive DevOps and Linux test covering **Ansible**, **Jenkins**, **Linux shell scripting**, **SQL**, and **Python programming**. Below is a **complete and accurate breakdown of all tasks**, with explanations and ready-to-use solutions.

---

## ✅ **ANSIBLE TASKS**

---

### 🔹**1. Ansible: Recovery from Hacker Attack**

**Goal:** Delete unauthorized files by comparing SHA1 checksums.

📄 **`playbook.yml`**

```yaml
- name: Remove unauthorized files based on checksum
  hosts: localhost
  become: true
  vars:
    checksums_file: /home/ubuntu/1833145-ansible-recovery-from-hacker-attack-by-eliminating-unauthorized-files/checksums.txt
    scan_dir: /tmp/1833145-ansible-recovery-from-hacker-attack-by-eliminating-unauthorized-files

  tasks:
    - name: Read valid checksums into a list
      slurp:
        src: "{{ checksums_file }}"
      register: checksum_data

    - name: Set valid_checksums list
      set_fact:
        valid_checksums: "{{ checksum_data.content | b64decode | splitlines() }}"

    - name: Find all files in the scan directory
      find:
        paths: "{{ scan_dir }}"
        file_type: file
      register: files_found

    - name: Delete unauthorized files
      block:
        - name: Compute SHA1 and remove if not in valid_checksums
          shell: |
            sha1sum "{{ item.path }}" | awk '{print $1}'
          register: file_checksum
          loop: "{{ files_found.files }}"
          loop_control:
            label: "{{ item.path }}"
          changed_when: false

        - name: Remove unauthorized files
          file:
            path: "{{ item.item.path }}"
            state: absent
          when: item.stdout not in valid_checksums
          loop: "{{ file_checksum.results }}"
```

---

### 🔹**2. Ansible: Deploy Application Stack with Docker**

📄 **`playbook.yml`**

```yaml
- name: Deploy PHP Example Stack
  hosts: localhost
  become: true
  tasks:
    - name: Start db container
      docker_container:
        name: db
        image: mysql:5.7
        state: started
        restart_policy: always
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: example

    - name: Start web container
      docker_container:
        name: web
        image: problemsetters/801133-ansible
        state: started
        restart_policy: always
        published_ports:
          - "8000:80"
        links:
          - db
```

---

### 🔹**3. Ansible: HackerBoard Credentials Setup**

📄 **`playbook.yml`**

```yaml
- name: Setup HackerBoard credentials
  hosts: localhost
  become: true
  vars:
    hackerboard_dir: "{{ lookup('env', 'HACKERBOARD_DIR') }}"
    access_key: "{{ lookup('env', 'HACKERBOARD_ACCESS_KEY') }}"

  tasks:
    - name: Ensure users exist
      user:
        name: "{{ item }}"
        state: present
      loop:
        - hackerboard
        - hackercompany

    - name: Create credentials.ini from template
      template:
        src: credentials.ini.j2
        dest: "{{ hackerboard_dir }}/credentials.ini"
        mode: '0640'
        owner: hackerboard
        group: hackercompany
```

And in your `credentials.ini.j2` template:

```ini
[default]
access_key={{ access_key }}
```

---

## ✅ **JENKINS QUESTIONS**

---

### 🔹 **Missing PowerShell and Git on dynamic node**

**Correct Answer:**
✅ **SSH into the server, install Git and PowerShell, and add it to the tools configuration.**

Because:

* Jenkins dynamic agents don’t persist manual installs via user data or init scripts unless explicitly configured.
* Installing manually and registering in Jenkins tools guarantees availability.

---

### 🔹 **Run command 3 times in Jenkins pipeline**

**Best method:**
✅

```groovy
pipeline {
    agent any
    stages {
        stage("Run step 1") {
            steps {
                script {
                    def executeThreeTimes = {
                        for (int i = 0; i < 3; i++) {
                            sh "my command"
                        }
                    }
                    executeThreeTimes()
                }
            }
        }
    }
}
```

Reason: Reusable, scalable, and concise using Groovy loop in `script` block.

---

### 🔹 **Secure and role-based pipeline access**

✅ **Implement MFA using SAML provider and use Project-based Matrix Authorization Strategy**

Why:

* SAML gives SSO & MFA.
* Project-based Matrix Authorization allows fine-grained control per job/pipeline.

---

## ✅ **LINUX SCRIPTING TASKS**

---

### 🔹 **1. Transaction Log Balance Calculation**

📄 `script.sh`

```bash
#!/bin/bash

awk '
  FNR==NR && $2=="approved" {approved[$1]; next}
  $1 in approved {sum += $2}
  END {print sum > "/tmp/balance.txt"}
' /home/ubuntu/898650-linux-transaction-log-balance-calculation/event.log \
   /home/ubuntu/898650-linux-transaction-log-balance-calculation/transaction.log
```

---

### 🔹 **2. Backup Storage Review (Files > 5MB)**

📄 `script.sh`

```bash
#!/bin/bash

awk '$1 > 5120 {sum += $1}
     END {printf "%.2f MB\n", sum/1024}' /tmp/1671700-linux-backup-storage-review/disk_usage.log
```

---

### 🔹 **3. Monitoring Slow Queries**

📄 `script.sh`

```bash
#!/bin/bash

awk '$4=="SUCCESS" && $3+0 > 2000 {count++}
     END {print count}' /tmp/1672396-linux-monitoring-database-query-performance/db_querieslog
```

---

## ✅ **SQL TASKS**

---

### 🔹 **1. Advertising Net Sellers Report**

```sql
WITH ordered_events AS (
  SELECT e.*, c.name, 
         ROW_NUMBER() OVER () AS rn
  FROM events e
  JOIN campaigns c ON e.campaign_id = c.id
),
grouped AS (
  SELECT *,
         LAG(campaign_id) OVER (ORDER BY rn) != campaign_id AS new_seq
  FROM ordered_events
),
sequences AS (
  SELECT *, SUM(new_seq) OVER (ORDER BY rn) AS seq_id
  FROM grouped
),
aggregated AS (
  SELECT name AS campaign, seq_id,
         SUM(CASE WHEN type='sell' THEN amount ELSE 0 END) AS sell_total,
         COUNT(CASE WHEN type='sell' THEN 1 END) AS sells,
         COUNT(CASE WHEN type='buy' THEN 1 END) AS buys
  FROM sequences
  GROUP BY name, seq_id
  HAVING sells > buys
)
SELECT campaign, COUNT(*) AS netsells_count,
       ROUND(SUM(sell_total), 2) AS netsells_total
FROM aggregated
GROUP BY campaign
ORDER BY netsells_total DESC;
```

---

### 🔹 **2. Online Banking Quarterly Report**

```sql
SELECT
  CONCAT('Q', QUARTER(dt), '''21') AS quarter,
  MAX(CASE WHEN a.iban = 'BH13 AVRQ F92P CVF6 NNYF UP'
           THEN FORMAT(SUM(REPLACE(amount, '$', '') + 0), 2) ELSE NULL END) AS "BH13 AVRQ F92P CVF6 NNYF UP",
  MAX(CASE WHEN a.iban = 'KZ45 2770 7WJ0 XUQ0 KOYH'
           THEN FORMAT(SUM(REPLACE(amount, '$', '') + 0), 2) ELSE NULL END) AS "KZ45 2770 7WJ0 XUQ0 KOYH",
  MAX(CASE WHEN a.iban = 'TN02 7094 9824 2254 5704 7751'
           THEN FORMAT(SUM(REPLACE(amount, '$', '') + 0), 2) ELSE NULL END) AS "TN02 7094 9824 2254 5704 7751"
FROM transactions t
JOIN accounts a ON a.id = t.account_id
WHERE YEAR(dt) = 2021
GROUP BY QUARTER(dt)
ORDER BY QUARTER(dt);
```

---

## ✅ **PYTHON TASKS**

---

### 🔹 **1. Suffix-stripping Stemmer**

```python
def stemmer(text):
    words = text.split()
    result = []
    for word in words:
        for suffix in ['ed', 'ly', 'ing']:
            if word.endswith(suffix):
                word = word[: -len(suffix)]
                break
        if len(word) > 8:
            word = word[:8]
        result.append(word)
    return ' '.join(result)
```

---

### 🔹 **2. Lambda Map**

```python
def lambdaMap(arr):
    return list(map(lambda x: list(map(lambda y: y*y, filter(lambda z: z > 0, x))), arr))
```

---

### 🔹 **3. Partial Function**

```python
def partial(func, *args, **kwargs):
    def inner(*more_args, **more_kwargs):
        final_args = args + more_args
        final_kwargs = {**kwargs, **more_kwargs}
        return func(*final_args, **final_kwargs)
    return inner
```

---

### 🔹 **4. Color Class**

```python
class Color:
    def __init__(self, r, g, b):
        for name, val in zip(['r', 'g', 'b'], [r, g, b]):
            if not (0 <= val <= 255):
                raise ValueError(f"Value of {name} should be between 0 and 255")
        self.r, self.g, self.b = r, g, b

    def __repr__(self):
        return f"Color: #{self.r:02x}{self.g:02x}{self.b:02x}"
```

---

### 🔹 **5. Duck Typing Check**

```python
def process_data(obj):
    if hasattr(obj, 'process') and callable(obj.process):
        return obj.process()
    return None
```

---

If you want me to **create a zip of all these files, or simulate how `sudo solve` would work**, let me know.
