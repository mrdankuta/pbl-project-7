# Project 7 - Tooling Website Solution

---

## Step 1 - Setup NFS Server

- Launch a new EC2 instance `nfs-server`
- Create 3 Volumes of 10GB each within the same Availability Zone as the previously created instance
- Attach each volume to the `nfs-server` instance
- Connect to `nfs-server` instance via terminal
- Inspect devices by using the `lsblk` command
- Create a single partition on each attached disks using the `gdisk` utility:
    ```
    sudo gdisk /dev/xvd[f,g,h]
    ```
- Use `lsblk` to view devices and new partitions
- Install `lvm2` package:
    ```
    sudo yum install lvm2
    ```
- Check for available partitions:
    ```
    sudo lvmdiskscan
    ```
- Mark each partition previously created as physical volumes to be used by `lvm`:
    ```
    sudo pvcreate /dev/xvdf1
    sudo pvcreate /dev/xvdg1
    sudo pvcreate /dev/xvdh1
    ```
- Use the `pvs` command to verify and view newly created physical volumes:
    ```
    sudo pvs
    ```
- Add all 3 physical volumes to a volume group called `nfs-vg`:
    ```
    sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
    ```
- Use the `vgs` command to verify and view newly created volume group:
    ```
    sudo vgs
    ```
- Create three logical volumes using the `lvcreate` utility. Create `lv-apps` (to be used by webservers), `lv-logs` (to be used by webserver logs), and `lv-opt` (to be used by Jenkins server).
    ```
    sudo lvcreate -n lv-apps -L 10G nfs-vg
    sudo lvcreate -n lv-logs -L 10G nfs-vg
    sudo lvcreate -n lv-opt -L 9G nfs-vg
    ```
- Use the `lvs` command to verify and view newly created logical volumes:
    ```
    sudo lvs
    ```
- Verify entire setup:
    ```
    sudo vgdisplay -v #view complete setup - VG, PV, and LV
    sudo lsblk
    ```
- Format logical volumes with `xfs` filesystem:
    ```
    sudo mkfs -t xfs /dev/nfs-vg/lv-apps
    sudo mkfs -t xfs /dev/nfs-vg/lv-logs
    sudo mkfs -t xfs /dev/nfs-vg/lv-opt
    ```
    ![XFS Formating](./images/001-mkfs-formated.png)
- Create directories for logical volumes mount points:
    ```
    sudo mkdir /mnt/apps
    sudo mkdir /mnt/logs
    sudo mkdir /mnt/opt
    ```
- Mount `lv-apps` on `/mnt/apps`, `lv-logs` on `/mnt/logs`, and `lv-opt` on `/mnt/opt`:
    ```
    sudo mount /dev/nfs-vg/lv-apps /mnt/apps/
    sudo mount /dev/nfs-vg/lv-logs /mnt/logs/
    sudo mount /dev/nfs-vg/lv-opt /mnt/opt/
    ```
- Install NFS server:
    ```
    sudo yum -y update
    sudo yum install nfs-utils -y
    ```
- Configure NFS server to start on reboot and ensure it is running:
    ```
    sudo systemctl start nfs-server.service
    sudo systemctl enable nfs-server.service
    sudo systemctl status nfs-server.service
    ```
- Configure permissions to allow Web servers to read, write and execute files on NFS:
    ```
    sudo chown -R nobody: /mnt/apps
    sudo chown -R nobody: /mnt/logs
    sudo chown -R nobody: /mnt/opt

    sudo chmod -R 777 /mnt/apps
    sudo chmod -R 777 /mnt/logs
    sudo chmod -R 777 /mnt/opt

    sudo systemctl restart nfs-server.service
    ```
- Configure access to NFS for only clients within the same subnet by editing `/etc/exports`:
    ```
    sudo vi /etc/exports
    ```
- Insert the following code:
    ```
    /mnt/apps 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash)
    /mnt/logs 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash)
    /mnt/opt 172.31.80.0/20(rw,sync,no_all_squash,no_root_squash)
    ```
- Export:
    ```
    sudo exportfs -arv
    ```
    ![Export NFS Config](./images/002-exorting-nfs-mnt.png)
- Check port used by NFS:
    ```
    rpcinfo -p | grep nfs
    ```
- In EC2, open ports `TCP 111`, `UDP 111`, and `UDP 2049` to allow NFS server to be accessible from client.
    ![Opening NFS Ports](images/003-nfs-ports.png)



## Configure Database Server

- Update `apt` repository:
    ```
    sudo apt update
    ```
- Install MySQL Server Software: 
    ```
    sudo apt install mysql-server
    ``` 
- Open port `3306` on `mysql-server` by adding inbound rule in EC2. Allow access only to `mysql-client`.
- Set root user password: 
    ```
    ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'DeFPassyWord_1';
    ```
- Exit MySQL Shell: 
    ```
    exit
    ```
- Remove insecure default setting with pre-installed security script. Start script: 
    ```
    sudo mysql_secure_installation
    ```
- Follow and respond to prompts to setup `root` password & other preferences.
- Login with `-p` flag to prompt for password and verify everything works well.
    ```
    sudo mysql -p
    ```
- Create a new role:
    ```
    CREATE ROLE 'webservers';
    ```
- Create a new database:
    ```
    CREATE DATABASE 'tooling';
    ```
- Assign privileges to `tooling` for `webservers` role:
    ```
    GRANT ALL PRIVILEGES ON tooling.* TO 'webservers';
    FLUSH PRIVILEGES;
    ```
- Create new user and assign `webservers` role:
    ```
    CREATE USER IF NOT EXISTS 'webaccess'@'172.31.80.0/20' IDENTIFIED WITH mysql_native_password BY 'mypass001' DEFAULT ROLE webservers;
    ```