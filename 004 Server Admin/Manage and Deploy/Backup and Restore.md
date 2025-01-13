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

