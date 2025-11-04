
## How To Use
The following is broken into chunks incase some is not needed.



## Script:
```
[BACKUP]
source_dirs=/home/user/docs,/var/log,/etc/nginx
backup_dir=/backups
compression=gzip
retention_days=7

[ENCRYPTION]
gpg_key_id=your-gpg-key-id
encrypt=true

[CLOUD]
s3_bucket=my-backup-bucket
region=us-east-1
upload=true

[NOTIFICATION]
email=admin@example.com
smtp_server=smtp.example.com:587
smtp_user=user
smtp_pass=pass
Main Script (abrs.sh)
Below is the complete, commented Bash script. Save it as abrs.sh and make it executable (chmod +x abrs.sh).
```

```
#!/bin/bash

# Automated Backup and Restore System (ABRS)
# Version: 1.0
# Author: [Your Name]
# Description: Creates encrypted backups, manages retention, uploads to S3, and notifies on status.

set -euo pipefail  # Exit on error, undefined vars, pipe failures

# Logging function
log() {
    local level="$1"
    shift
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$level] $@"
}

# Load configuration from INI file
load_config() {
    local config_file="${1:-config.ini}"
    if [[ ! -f "$config_file" ]]; then
        log "ERROR" "Config file $config_file not found."
        exit 1
    fi

    # Parse INI sections using awk (simple parser for demo; use a library for production)
    declare -A config
    while IFS='=' read -r key value; do
        key=$(echo "$key" | xargs)  # Trim whitespace
        value=$(echo "$value" | xargs)
        if [[ "$key" =~ ^\[([^]]+)\]$ ]]; then
            current_section="${BASH_REMATCH[1]}"
        elif [[ -n "$key" && -n "$value" && "$current_section" ]]; then
            config["${current_section}_${key}"]="$value"
        fi
    done < <(grep -v '^#' "$config_file" | grep -v '^$')

    # Export for use
    export BACKUP_SOURCE_DIRS=(${config[BACKUP_source_dirs]:-})
    export BACKUP_DIR="${config[BACKUP_backup_dir]:-/backups}"
    export COMPRESSION="${config[BACKUP_compression]:-gzip}"
    export RETENTION_DAYS="${config[BACKUP_retention_days]:-7}"
    export GPG_KEY_ID="${config[ENCRYPTION_gpg_key_id]:-}"
    export ENCRYPT="${config[ENCRYPTION_encrypt]:-true}"
    export S3_BUCKET="${config[CLOUD_s3_bucket]:-}"
    export AWS_REGION="${config[CLOUD_region]:-us-east-1}"
    export UPLOAD="${config[CLOUD_upload]:-true}"
    export NOTIFY_EMAIL="${config[NOTIFICATION_email]:-}"
    export SMTP_SERVER="${config[NOTIFICATION_smtp_server]:-}"
    export SMTP_USER="${config[NOTIFICATION_smtp_user]:-}"
    export SMTP_PASS="${config[NOTIFICATION_smtp_pass]:-}"
}

```


# Create backup archive
```
create_backup() {
    local timestamp=$(date '+%Y%m%d_%H%M%S')
    local archive_name="backup_${timestamp}.tar.${COMPRESSION}"
    local archive_path="${BACKUP_DIR}/${archive_name}"

    # Ensure backup dir exists
    mkdir -p "$BACKUP_DIR"

    # Build tar command
    local tar_cmd="tar -cf -"
    if [[ "$COMPRESSION" == "gzip" ]]; then
        tar_cmd+=" -z"
    elif [[ "$COMPRESSION" == "bzip2" ]]; then
        tar_cmd+=" -j"
    fi

    # Add source directories
    for dir in "${BACKUP_SOURCE_DIRS[@]}"; do
        if [[ ! -d "$dir" ]]; then
            log "WARN" "Source directory $dir does not exist. Skipping."
            continue
        fi
        tar_cmd+=" \"$dir\""
    done

    # Create and compress
    log "INFO" "Creating backup: $archive_path"
    eval "$tar_cmd" | {
        if [[ "$ENCRYPT" == "true" && -n "$GPG_KEY_ID" ]]; then
            gpg --batch --yes --encrypt --recipient "$GPG_KEY_ID" -o "$archive_path.gpg"
        else
            cat > "$archive_path"
        fi
    } || {
        log "ERROR" "Backup creation failed."
        return 1
    }

    log "INFO" "Backup created successfully: $archive_path"
    echo "$archive_path"  # Return path for further processing
}
```
# Upload to S3 (if enabled)
```
upload_to_s3() {
    local archive_path="$1"
    if [[ "$UPLOAD" != "true" || -z "$S3_BUCKET" ]]; then
        log "INFO" "S3 upload disabled."
        return 0
    fi

    log "INFO" "Uploading to S3: s3://$S3_BUCKET/$(basename "$archive_path")"
    AWS_DEFAULT_REGION="$AWS_REGION" aws s3 cp "$archive_path" "s3://$S3_BUCKET/" || {
        log "ERROR" "S3 upload failed."
        return 1
    }
    log "INFO" "Upload successful."
}
```
# Manage retention (delete old backups)
```
manage_retention() {
    local cutoff_date=$(date -d "${RETENTION_DAYS} days ago" '+%Y%m%d') 2>/dev/null || {
        log "ERROR" "Date calculation failed."
        return 1
    }

    log "INFO" "Removing backups older than $RETENTION_DAYS days."
    find "$BACKUP_DIR" -name "backup_*.tar.*" -o -name "backup_*.tar.*.gpg" | while read -r file; do
        local file_date=$(basename "$file" | sed -E 's/backup_([0-9]{8})_.*$/\1/')
        if [[ "$file_date" -lt "$cutoff_date" ]]; then
            rm -f "$file"
            log "INFO" "Deleted old backup: $file"
        fi
    done
}
```
# Send notification email

```
send_notification() {
    local status="$1"
    local message="$2"
    if [[ -z "$NOTIFY_EMAIL" ]]; then
        return 0
    fi

    local subject="ABRS Backup: $status - $(date)"
    local body="Status: $status\nMessage: $message\nTimestamp: $(date)"

    # Use mail with SMTP if configured (requires msmtp or similar; fallback to local mail)
    if [[ -n "$SMTP_SERVER" ]]; then
        echo -e "$body" | msmtp --host="$SMTP_SERVER" --port=$(echo "$SMTP_SERVER" | cut -d: -f2) \
            --auth=on --user="$SMTP_USER" --password="$SMTP_PASS" "$NOTIFY_EMAIL"
    else
        echo -e "Subject: $subject\n\n$body" | mail -s "$subject" "$NOTIFY_EMAIL"
    fi || log "WARN" "Notification failed."
}
```
# Restore function (called with --restore flag)
```
restore_backup() {
    local archive_path="$1"
    local target_dir="${2:-/tmp/restore}"
    if [[ ! -f "$archive_path" ]]; then
        log "ERROR" "Archive $archive_path not found."
        exit 1
    fi

    mkdir -p "$target_dir"
    log "INFO" "Restoring to $target_dir"

    local temp_file=$(mktemp)
    if [[ "$archive_path" == *.gpg ]]; then
        gpg --batch --yes --decrypt "$archive_path" > "$temp_file" || {
            log "ERROR" "Decryption failed."
            rm -f "$temp_file"
            exit 1
        }
    else
        cp "$archive_path" "$temp_file"
    fi

    tar -xf "$temp_file" -C "$target_dir" || {
        log "ERROR" "Extraction failed."
        rm -f "$temp_file"
        exit 1
    }

    rm -f "$temp_file"
    log "INFO" "Restore completed successfully."
}
```
# Main execution

```
main() {
    local mode="${1:-backup}"  # Default to backup; use --restore for restore
    local config_file="${2:-config.ini}"

    load_config "$config_file"
    trap 'log "ERROR" "Script interrupted. Cleaning up..."; exit 1' INT TERM

    case "$mode" in
        backup)
            local archive_path
            if archive_path=$(create_backup); then
                upload_to_s3 "$archive_path"
                manage_retention
                send_notification "SUCCESS" "Backup completed: $archive_path"
            else
                send_notification "FAILED" "Backup creation failed."
                exit 1
            fi
            ;;
        --restore)
            restore_backup "${2:-}" "${3:-}"
            ;;
        *)
            log "ERROR" "Invalid mode: $mode. Use 'backup' or '--restore <archive> [target_dir]'."
            exit 1
            ;;
    esac

    log "INFO" "ABRS execution completed."
}
```
# Run main if called directly
```

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi

```

## Usage Instructions
1.	Backup Execution: 
./abrs.sh backup config.ini

2.	 Restore Execution:
./abrs.sh --restore /backups/backup_20231023_120000.tar.gz.gpg /tmp/my-restore

3.	 Scheduling: Add to crontab (crontab -e):
0 2 * * * /path/to/abrs.sh backup /path/to/config.ini >> /var/log/abrs.log 2>&1
(note Runs daily at 2:00 AM.)





```