Below is a consolidated collection of 15 Linux shell scripts covering a wide range of topics—from integrating with cloud services for DevOps to everyday system, network, and text processing tasks. Five additional use cases have been added that leverage network tools, as well as utilities like `awk`, `find`, and `sed` in genuine scenarios.

---

## 1. `backup_logs.sh`

Archives log files from a given directory into a compressed backup and deletes backups older than a set number of days.

```bash
#!/bin/bash
# backup_logs.sh - Archive log files and clean up old backups.
# Usage: ./backup_logs.sh [log_directory] [backup_directory] [days_to_keep]
#
# Example:
#   ./backup_logs.sh /var/log /backup/logs 7

set -euo pipefail

LOG_DIR="${1:-/var/log}"
BACKUP_DIR="${2:-/backup/logs}"
DAYS_TO_KEEP="${3:-7}"

mkdir -p "$BACKUP_DIR"

timestamp=$(date +%Y%m%d%H%M%S)
archive_file="${BACKUP_DIR}/logs_backup_${timestamp}.tar.gz"

echo "Archiving logs from $LOG_DIR to $archive_file"
tar -czf "$archive_file" -C "$LOG_DIR" .
echo "Backup complete."

echo "Deleting backups older than $DAYS_TO_KEEP days"
find "$BACKUP_DIR" -type f -name 'logs_backup_*.tar.gz' -mtime +"$DAYS_TO_KEEP" -exec rm {} \;
echo "Old backups deleted."
```

---

## 2. `system_resource_monitor.sh`

Monitors CPU and memory usage. If usage exceeds specified thresholds, it prints warning messages—ideal for routine health checks.

```bash
#!/bin/bash
# system_resource_monitor.sh - Monitor CPU and memory usage.
# Usage: ./system_resource_monitor.sh [cpu_threshold] [memory_threshold]
#
# Example:
#   ./system_resource_monitor.sh 80 80

set -euo pipefail

CPU_THRESHOLD="${1:-80}"
MEM_THRESHOLD="${2:-80}"

cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print 100 - $8}')
mem_usage=$(free | awk '/Mem/ {printf("%.0f"), $3/$2 * 100}')

echo "Current CPU Usage: ${cpu_usage}%"
echo "Current Memory Usage: ${mem_usage}%"

if (( $(echo "$cpu_usage > $CPU_THRESHOLD" | bc -l) )); then
    echo "Warning: High CPU usage at ${cpu_usage}%"
fi

if [ "$mem_usage" -gt "$MEM_THRESHOLD" ]; then
    echo "Warning: High Memory usage at ${mem_usage}%"
fi
```

---

## 3. `renew_ssl_cert.sh`

A genuine DevOps use case—in this script, an SSL certificate is automatically renewed via Certbot. Ensure Certbot is installed and properly configured on your server.

```bash
#!/bin/bash
# renew_ssl_cert.sh - Renew SSL certificates using certbot.
# Usage: ./renew_ssl_cert.sh [domain] [email]
#
# Example:
#   ./renew_ssl_cert.sh example.com admin@example.com

set -euo pipefail

DOMAIN="${1:?You must provide a domain name.}"
EMAIL="${2:?You must provide an email address for notices.}"

if ! command -v certbot &> /dev/null; then
    echo "certbot is not installed. Please install it before running this script."
    exit 1
fi

echo "Renewing SSL certificate for ${DOMAIN}"
sudo certbot certonly --standalone -d "$DOMAIN" --non-interactive --agree-tos --email "$EMAIL"
echo "Certificate renewal attempted. Please check certbot logs for details."
```

---

## 4. `docker_health_check.sh`

Checks the health of all running Docker containers and automatically restarts any container that is marked as “unhealthy”. Useful in containerized environments.

```bash
#!/bin/bash
# docker_health_check.sh - Check Docker container health and restart unhealthy ones.
# Usage: ./docker_health_check.sh

set -euo pipefail

if ! command -v docker &> /dev/null; then
    echo "Docker command not found. Please ensure Docker is installed."
    exit 1
fi

containers=$(docker ps -q)
if [ -z "$containers" ]; then
    echo "No running containers found."
    exit 0
fi

for container in $containers; do
    health_status=$(docker inspect --format='{{.State.Health.Status}}' "$container" 2>/dev/null || echo "no healthcheck")
    echo "Container ${container} health: ${health_status}"
    if [ "$health_status" == "unhealthy" ]; then
        echo "Restarting container ${container}"
        docker restart "$container"
    fi
done
```

---

## 5. `system_update.sh`

Automates system package updates and security patches. Great for routine maintenance on a Linux server.

```bash
#!/bin/bash
# system_update.sh - Update system packages and install security patches.
# Usage: ./system_update.sh

set -euo pipefail

echo "Updating package list..."
sudo apt-get update -y

echo "Upgrading installed packages..."
sudo apt-get upgrade -y

echo "System update complete."
```

---

## 6. `disk_usage_monitor.sh`

Monitors disk usage for the root filesystem. If usage exceeds a given threshold, a warning is printed to help prevent storage shortages.

```bash
#!/bin/bash
# disk_usage_monitor.sh - Check disk usage and alert if usage exceeds threshold.
# Usage: ./disk_usage_monitor.sh [threshold_percentage]
#
# Example:
#   ./disk_usage_monitor.sh 90

set -euo pipefail

THRESHOLD="${1:-90}"

disk_usage=$(df / | tail -1 | awk '{print $5}' | tr -d '%')
echo "Current disk usage on /: ${disk_usage}%"

if [ "$disk_usage" -ge "$THRESHOLD" ]; then
    echo "Warning: Disk usage is at ${disk_usage}%, which exceeds the threshold of ${THRESHOLD}%."
fi
```

---

## 7. `service_auto_restart.sh`

Monitors a specified system service (e.g., Apache, Nginx) and restarts it if it isn’t running—helping ensure high availability.

```bash
#!/bin/bash
# service_auto_restart.sh - Check and restart a service if it is not active.
# Usage: ./service_auto_restart.sh [service_name]
#
# Example:
#   ./service_auto_restart.sh apache2

set -euo pipefail

SERVICE="${1:?You must provide a service name (e.g., apache2)}"

SERVICE_STATUS=$(systemctl is-active "$SERVICE")
echo "Service ${SERVICE} status: ${SERVICE_STATUS}"

if [ "$SERVICE_STATUS" != "active" ]; then
    echo "Service ${SERVICE} is not running. Attempting to restart..."
    sudo systemctl restart "$SERVICE"
    echo "Service ${SERVICE} restarted."
else
    echo "Service ${SERVICE} is running normally."
fi
```

---

## 8. `mysql_backup.sh`

Backs up MySQL databases using `mysqldump` and compresses the output. Adjust the credentials and backup directory as needed.

```bash
#!/bin/bash
# mysql_backup.sh - Backup MySQL databases and compress dumps.
# Usage: ./mysql_backup.sh [mysql_user] [mysql_password] [backup_directory]
#
# Example:
#   ./mysql_backup.sh root password /backup/mysql

set -euo pipefail

USER="${1:?You must provide a MySQL username}"
PASSWORD="${2:?You must provide a MySQL password}"
BACKUP_DIR="${3:-/backup/mysql}"
DATE=$(date +%Y%m%d%H%M%S)

mkdir -p "$BACKUP_DIR"

# Retrieve databases, excluding system databases.
databases=$(mysql -u "$USER" -p"$PASSWORD" -e "SHOW DATABASES;" 2>/dev/null | tr -d "| " | grep -v Database | grep -vE "information_schema|performance_schema|mysql|sys")

for db in $databases; do
    backup_file="${BACKUP_DIR}/${db}_${DATE}.sql.gz"
    echo "Backing up database ${db} to ${backup_file}"
    mysqldump -u "$USER" -p"$PASSWORD" "$db" | gzip > "$backup_file"
done

echo "MySQL backup complete."
```

---

## 9. `user_inactive_cleanup.sh`

Lists user accounts that have not logged in for a defined number of days. This is useful for audits or cleanup tasks.

```bash
#!/bin/bash
# user_inactive_cleanup.sh - List users who haven't logged in within a given timeframe.
# Usage: ./user_inactive_cleanup.sh [days]
#
# Example:
#   ./user_inactive_cleanup.sh 90

set -euo pipefail

DAYS="${1:-90}"

echo "Users inactive for more than ${DAYS} days:"
lastlog -b "$DAYS"
```

---

## 10. `crontab_backup.sh`

Backs up the current user’s crontab entries to a specified directory for auditing or future restoration.

```bash
#!/bin/bash
# crontab_backup.sh - Backup the current crontab entries.
# Usage: ./crontab_backup.sh [backup_directory]
#
# Example:
#   ./crontab_backup.sh /backup/crontab

set -euo pipefail

BACKUP_DIR="${1:-/backup/crontab}"
DATE=$(date +%Y%m%d%H%M%S)

mkdir -p "$BACKUP_DIR"
output_file="${BACKUP_DIR}/crontab_backup_${DATE}.txt"

echo "Backing up crontab entries to ${output_file}"
crontab -l > "$output_file" 2>/dev/null || echo "No crontab for current user"
echo "Crontab backup complete."
```

---

## 11. `ping_monitor.sh`

A networking use case that pings a list of hosts (provided in a file) to verify connectivity—helpful for monitoring the reachability of critical servers.

```bash
#!/bin/bash
# ping_monitor.sh - Checks network connectivity by pinging a list of hosts.
# Usage: ./ping_monitor.sh [hosts_file]
#
# Example:
#   ./ping_monitor.sh hosts.txt
#
# hosts.txt should contain one host per line (e.g., 8.8.8.8)

set -euo pipefail

HOSTS_FILE="${1:-hosts.txt}"

if [ ! -f "$HOSTS_FILE" ]; then
  echo "Hosts file '$HOSTS_FILE' not found."
  exit 1
fi

while IFS= read -r host; do
    # Skip empty lines
    if [[ -z "$host" ]]; then continue; fi
    echo "Pinging $host..."
    if ping -c 1 -W 2 "$host" &> /dev/null; then
        echo "Host $host is reachable."
    else
        echo "Host $host is NOT reachable."
    fi
done < "$HOSTS_FILE"
```

---

## 12. `log_parser_awk.sh`

Demonstrates the use of `awk` to parse a log file and count occurrences of common log levels (INFO, WARN, ERROR).

```bash
#!/bin/bash
# log_parser_awk.sh - Parse a log file to count occurrences of log levels (INFO, WARN, ERROR) using awk.
# Usage: ./log_parser_awk.sh [log_file]
#
# Example:
#   ./log_parser_awk.sh /var/log/syslog

set -euo pipefail

LOG_FILE="${1:-/var/log/syslog}"

if [ ! -f "$LOG_FILE" ]; then
    echo "Log file '$LOG_FILE' does not exist."
    exit 1
fi

echo "Parsing log file: $LOG_FILE"
awk '
/INFO/ { count_info++ }
/WARN/ { count_warn++ }
/ERROR/ { count_error++ }
END {
    print "Log Level Summary:"
    print "INFO: " (count_info+0)
    print "WARN: " (count_warn+0)
    print "ERROR: " (count_error+0)
}
' "$LOG_FILE"
```

---

## 13. `find_large_files.sh`

Uses the `find` command to search for files larger than a specified size within a directory—helpful for identifying storage hogs.

```bash
#!/bin/bash
# find_large_files.sh - Find files larger than a specified size within a directory.
# Usage: ./find_large_files.sh [directory] [size]
#
# Example:
#   ./find_large_files.sh /var/log 50M

set -euo pipefail

SEARCH_DIR="${1:-.}"
SIZE_THRESHOLD="${2:-100M}"

echo "Searching for files larger than $SIZE_THRESHOLD in $SEARCH_DIR"
find "$SEARCH_DIR" -type f -size +"${SIZE_THRESHOLD}" -exec ls -lh {} \; | awk '{ print $9 ": " $5 }'
```

---

## 14. `config_replace_sed.sh`

Uses `sed` to update a configuration parameter in a configuration file. This can be used to automate config file changes across deployments.

```bash
#!/bin/bash
# config_replace_sed.sh - Replace a configuration parameter's value in a config file using sed.
# Usage: ./config_replace_sed.sh [config_file] [parameter] [new_value]
#
# Example:
#   ./config_replace_sed.sh app.conf max_connections 200

set -euo pipefail

CONFIG_FILE="${1:?You must provide a configuration file}"
PARAMETER="${2:?You must provide the parameter name}"
NEW_VALUE="${3:?You must provide the new value}"

if [ ! -f "$CONFIG_FILE" ]; then
    echo "Configuration file $CONFIG_FILE not found."
    exit 1
fi

echo "Updating parameter '$PARAMETER' in $CONFIG_FILE to new value: $NEW_VALUE"
# Assumes the config file has lines in the form: parameter=value
sed -i.bak "s/^\(${PARAMETER}=\).*/\1${NEW_VALUE}/" "$CONFIG_FILE"
echo "Update complete. Backup file created as ${CONFIG_FILE}.bak"
```

---

## 15. `http_log_summary.sh`

Analyzes an Apache HTTP access log to provide a summary of HTTP status codes using `awk` and `sed` for a clean output—useful for monitoring web server health.

```bash
#!/bin/bash
# http_log_summary.sh - Summarize Apache HTTP access log status codes using awk and sed.
# Usage: ./http_log_summary.sh [access_log]
#
# Example:
#   ./http_log_summary.sh /var/log/apache2/access.log

set -euo pipefail

ACCESS_LOG="${1:-/var/log/apache2/access.log}"

if [ ! -f "$ACCESS_LOG" ]; then
    echo "Access log file '$ACCESS_LOG' not found."
    exit 1
fi

echo "Generating HTTP status code summary from $ACCESS_LOG:"
awk '{
    # In common log format the status code is typically the 9th field.
    status = $9
    count[status]++
}
END {
    for (s in count) {
        printf "Status Code %s: %d requests\n", s, count[s]
    }
}' "$ACCESS_LOG" | sed 's/^/  /'
```

---

## README.md

```markdown
# Linux Scripts Collection for DevOps & System Administration

This repository contains **15 Linux shell scripts** covering a wide range of important topics. These include cloud integrations for DevOps (such as container health checks and automated SSL certificate renewals) as well as daily system tasks like system monitoring, backups, log analysis, network connectivity checks, and text processing with tools like `awk`, `find`, and `sed`.

## Scripts Overview

1. **backup_logs.sh**  
   Archives log files from a designated directory into a compressed backup and removes backups older than a specified number of days.

2. **system_resource_monitor.sh**  
   Monitors CPU and memory usage and prints warnings if usage exceeds defined thresholds.

3. **renew_ssl_cert.sh**  
   Automatically renews SSL certificates using Certbot to ensure that TLS certificates are always up-to-date.

4. **docker_health_check.sh**  
   Checks the health of running Docker containers and restarts any that are marked as “unhealthy”. Ideal for containerized environments.

5. **system_update.sh**  
   Automates package updates and security patch installations to maintain your Linux system's integrity.

6. **disk_usage_monitor.sh**  
   Monitors disk usage and alerts when the usage exceeds a specified threshold—helpful in preventing storage issues.

7. **service_auto_restart.sh**  
   Monitors a given system service (e.g., Apache, Nginx) and restarts it if it is not running, ensuring high service availability.

8. **mysql_backup.sh**  
   Backs up MySQL databases using `mysqldump` and compresses the output, excluding system databases.

9. **user_inactive_cleanup.sh**  
   Lists user accounts that haven’t logged in for a specified number of days, helping with account audits and cleanups.

10. **crontab_backup.sh**  
    Backs up current crontab entries to a file, useful for keeping a record or restoring scheduled jobs.

11. **ping_monitor.sh**  
    Pings a list of hosts (from a file) to verify network connectivity, ensuring that critical servers are reachable.

12. **log_parser_awk.sh**  
    Parses a log file using `awk` to count occurrences of INFO, WARN, and ERROR messages—ideal for quick log analysis.

13. **find_large_files.sh**  
    Uses the `find` command to locate files larger than a specified size in a directory, aiding in storage management.

14. **config_replace_sed.sh**  
    Updates configuration parameters in a file using `sed`—this automates configuration changes across environments.

15. **http_log_summary.sh**  
    Generates a summary report of HTTP status codes from an Apache access log using `awk` and `sed` for monitoring web server activity.

## Usage

### Prerequisites

- A Linux or Unix-like operating system with the Bash shell.
- Some scripts require additional tools:
  - `certbot` (for SSL certificate renewal).
  - Docker (for container health checks).
  - MySQL client utilities (`mysql`, `mysqldump`) for database backups.
- Appropriate permissions (e.g., use of `sudo`) where necessary.

### Running the Scripts

1. **Make the script executable:**

   ```bash
   chmod +x script_name.sh
   ```

2. **Execute the script with any needed arguments:**

   ```bash
   ./script_name.sh [arguments]
   ```

   For example, to run the disk usage monitor with a 90% threshold:

   ```bash
   ./disk_usage_monitor.sh 90
   ```

## Best Practices

- **Error Handling:**  
  Each script uses `set -euo pipefail` to catch errors early.
  
- **Modular Design:**  
  Scripts allow overriding default parameters via command-line arguments.

- **Logging and Feedback:**  
  Informative messages are printed during execution to aid troubleshooting.

- **Security Considerations:**  
  Ensure that credentials and sensitive information are handled securely before deploying in production.

## Contributing

Feel free to fork this repository, contribute improvements, or add additional scripts based on your system administration or DevOps needs. Feedback and contributions are highly appreciated!

## License

This project is licensed under the MIT License.
```

---

This comprehensive set of 15 scripts, along with the README documentation, provides a strong starting point for automating tasks, monitoring systems, and addressing genuine DevOps challenges on Linux environments. Enjoy customizing and expanding upon these scripts to meet your unique operational needs!
