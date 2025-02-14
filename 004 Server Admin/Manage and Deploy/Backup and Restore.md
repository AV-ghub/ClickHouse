## [Backup and Restore](https://clickhouse.com/docs/en/operations/backup)
### [Backup to a local disk](https://clickhouse.com/docs/en/operations/backup#backup-to-a-local-disk)
#### Configure a backup destination
In the examples below you will see the backup destination specified like Disk('backups', '1.zip').    
To prepare the destination add a file to **/etc/clickhouse-server/config.d/backup_disk.xml** specifying the backup destination.    
For example, this file defines disk named backups and then adds that disk to the backups > allowed_disk list:
```
<clickhouse>
    <storage_configuration>
        <disks>
            <backups>
                <type>local</type>
                <path>/var/lib/clickhouse/backups/</path>
            </backups>
        </disks>
    </storage_configuration>
    <backups>
        <allowed_disk>backups</allowed_disk>
        <allowed_path>/var/lib/clickhouse/backups/</allowed_path>
    </backups>
</clickhouse>
```
> **Path should be absolute!!!**

> **Set chown on catalog & all copyied files!!!**

> **Check for syntax errors here**   
> Finally, check that the preprocessed configuration is being written and contains the changes you've made. It is located in the preprocessed_configs in the data directory, e.g., /var/lib/clickhouse/preprocessed_configs.   
> [src](https://github.com/ClickHouse/ClickHouse/issues/54966#issuecomment-1732803282)
> if some exists - you`ll see `em as they are loaded by clickhouse-server inside preprocessed_configs text

> **Restart service** via ```systemctl restart clickhouse-server```

#### Additional resources
[Clickhouse backup to disk problem backups.allowed_disk configuration parameter is not set](https://stackoverflow.com/questions/77296018/clickhouse-backup-to-disk-problem-backups-allowed-disk-configuration-parameter)   
You need to add the <storage_configuration> to the config.d\backup_disk.xml
```
<clickhouse>
    <storage_configuration>
        <disks>
            <backups>
                <type>local</type>
                <path>/backups/</path>
            </backups>
        </disks>
    </storage_configuration>
    <backups>
        <allowed_disk>backups</allowed_disk>
        <allowed_path>/backups/</allowed_path>
    </backups>
</clickhouse>
```

