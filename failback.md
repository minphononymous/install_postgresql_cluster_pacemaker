> Confirm the failed DB is up and Pacemaker service is started.

****On db1 (Failback Node)***
```bash
systemctl status pacemaker
pcs status
```
> Reset Error Counts
```bash
pcs resource cleanup pg_group
```
> Perform the Failback (Move the Service)
```bash
pcs resource move pg_group db01
watch -n 1 pcs status
```
Clear the Constraint
```bash
pcs resource clear pg_group
pcs status
```
