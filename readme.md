## Ansible-Proxmox-Backup-Client

This role will install and configure the Proxmox Backup Client on Debian a based OS.
Find Proxmox Backup Client documentation here. [Proxmox Backup Client](https://pbs.proxmox.com/docs/backup-client.html).

### Vars
| Variable | Required | Default | Description|
|-----------|----------|----------|----------|
| proxmox_backup | Yes | | Dict containing your configuration for this role |
| .server | Yes | | Dict containing the connection information for your Proxmox Backup Server. |
| .server.user | Yes | | Username to access the PBS, ex. backup@pbs |
| .server.pass | Yes | | Password or Token for the username provided. |
| .server.host | Yes | | Hostname/FQDN/IP of the PBS. |
| .server.datastore | Yes | | Datastore to store the backups. |
| .server.port | No | 8007 | Custom port of the PBS API. |
| .server.fingerprint | No | | Fingerprint of the server cert, found on the dashboard. |
| .client | Yes | | Dict containing the client configuration for the PBC. |
| .client.archives | Yes | | Array of items to backup. |
| .client.archives.name | Yes | | Name of the archive file to upload, cannot have the '/' character. |
| .client.archives.path | Yes | | Path to backup. |
| .client.archives.format | Yes | | Common types are .pxar for file archives, and .img for block device images. |
| .client.encrypt | No | | Dict containing client side encryption settings. |
| .client.encrypt.enable | Yes | No | Enable or Disable client side encryption. |
| .client.encrypt.pass | No | | Password for the Encryption Key, if password protected. |
| .client.encrypt.key | Yes | ~/.config/proxmox-backup/encryption-key.json | Path to store the encryption key. |
| .client.encrypt.master | No | | Path to local master-public.pem to copy to the client |
| .client.output | No | | Dict containing stdout log formatting options. |
| .client.output.format | No | text | Possible values are text, json, json-pretty. |
| .client.output.border | No | | If set, will not render table borders. |
| .client.output.header | No | | If set, will not render table headers. |
| .client.include_devices | No | | Array of mount points to include, MPs are skipped by default. |
| .client.skip_lost_found | No | No | skip lost+found directories. |
| .client.schedule.calendar | No | '\*-\*-\* 4:00:00' | Systemd Timer OnCalendar string, Default 4am Daily. |
| .client.schedule.runifmissed | No | No | Trigger service if the last start time was missed. |
| .client.exclude | No | | Array containting settings for .pxarexclude files. |
| .client.exclude[#].path | Yes | | File path of .pxarexclude (Parent Folder). |
| .client.exclude[#].enabled | Yes | | Create the exclusion file, delete if no. |
| .client.exclude[#].rules | Yes | | Array of exclusion rules, can include an explicit inclusion. |

### Default
```
default_config:
  client:
    encrypt:
      enable: false
      key: ~/.config/proxmox-backup/encryption-key.json
    output:
      format: text
    skip_lost_found: no
    schedule:
      calendar: '*-*-* 4:00:00' # Daily at 4am
      runifmissed: no
```

## Examples
### All level Group Vars (YAML format).
```
pbs_global_server:
    user: backup@pbs
    pass: ProxmoxBackupPassword
    host: pbs.example.com
    datastore: backups
    fingerprint: 7f:f1:ee:...:3d:de:2b
pbs_global_client:
    skip_lost_found: yes
    exclude:
    - path: /
        rules:
        - /boot/*
        - /tmp/*
    - path: /var/
        rules: 
        - /run/*
        - /log/*.[0-9]
        - /log/*.[0-9].*
        - /log/**/*.[0-9]
        - /log/**/*.[0-9].*
    archives:
    - name: root
        path: /
        format: .pxar
    - name: boot
        path: /boot
        format: .pxar
```
### Host Level Vars (YAML format).
```
pbs_client:
archives:
    - name: ssd
    path: /ssd
    format: .pxar
    - name: data
    path: /Data
    format: .pxar
exclude:
    - path: /ssd/qbit/
    rules:
        - /incomplete/*
    - path: /ssd/media/
    rules:
        - /_transcode/temp/*
        - /_transcode/log/*
        - /converted/*.jpg
        - /converted/*.mp4
        - /.Trash*
```
### Merge Group defaults with Host overrides
pbs_global_client is an ALL group_var.
pbs_client is a host_var.
combine(pbs_client, recursive=true) will merge dictionaries. The dict inside the combine will take precedence over the piped dict. Can be chained.
```
    # Merge the arrays so one doesn't override the other.
  - name: Merge PBS Client Arrays
    set_fact:
      pbs_client_merged:
          archives: "{{ pbs_global_client.archives + pbs_client.archives }}"
          exclude: "{{ pbs_global_client.exclude + pbs_client.exclude }}"
  
  - name: Setup Proxmox Backup Client
    include_role:
      name: proxmox-backup-client
    vars:
      proxmox_backup:
        server: "{{ pbs_global_server }}"
        client: "{{ pbs_global_client | combine(pbs_client, recursive=true) | combine(pbs_client_merged, recursive=true) }}"
```