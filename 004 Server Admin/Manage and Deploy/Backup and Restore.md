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
