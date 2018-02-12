# **THIS LAB IS A WORK IN PROGRESS**
# Lab Guide <br/> Gluster Test Drive Module 6 <br/> Geo-Replication

## LAB AGENDA

Welcome to the Gluster Test Drive Module 6 - Geo-Replication

- Set up geo-replication between two sites, local and remote
- Display status information
- Disaster recovery: Promote slave to master
- Failback: Bring master and slave back to their initial state

## GETTING STARTED

If you have not already done so, click the <img src="http://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl02-working-ebs/media/image005.png"> button in the navigation bar above to launch your lab. If you are prompted for a token, use the one distributed to you (or credits you've purchased).

> **NOTE** It may take **up to 10 minutes** for your lab systems to start up before you can access them.

## LAB SETUP

### CONNECT TO THE LAB

connect to the **rhgs1** server instance, using its public IP address from the **Addl. Info** tab to the right. 
```bash
ssh student@<rhgs1PublicIP>
```

## GEO-REPLICATION PREREQUISITES

If you have gone through Module 2 before, it's necessary to revert some of the steps taken in there. 

The cleanup.sh script in student's home will do the following:

1. Stop and delete the volumes "repvol" and "distvol"
2. Remove the nodes "rhgs4", "rhgs5" and "rhgs6" from the pool of trusted nodes
3. Create a ssh key for "root" and copy it to "rhgs4" since password-less root access is needed later on for the geo-replication.

### MANUAL STEPS

If you can't for some reason run the cleanup.sh script or don't want to, here are the manual steps.


Stop and delete the volumes "repvol" and "distvol"
```bash
sudo gluster volume stop distvol
sudo gluster volume delete distvol
```

```bash
sudo gluster volume stop repvol
sudo gluster volume delete distvol
```

Remove the nodes "rhgs4", "rhgs5" and "rhgs6" from the pool of trusted nodes

```bash
sudo gluster peer detach rhgs4
sudo gluster peer detach rhgs5
sudo gluster peer detach rhgs6
```

 Create a ssh key for "root" and copy it to "rhgs4"
```bash
sudo ssh-keygen -t rsa -f /root/.ssh/id.rsa -q -P ""
sudo ssh-copy-id root@rhgs4
```

The password for root is **"Redhat18"**


### SET UP THE REMOTE SITE

Build a second pool of trusted nodes on **rhgs4**
```bash
sudo gluster peer probe rhgs5
sudo gluster peer probe rhgs6
```

After these steps there are two clusters, consisting of 3 nodes each:

|local         | remote     |
|--------------|------------|
|rhgs1         | rhgs4      |
|rhgs2         | rhgs5      |
|rhgs3         | rhgs6      |


We will set up a geo-replication with the master being in the "local" cluster and the slave in the "remote" cluster.


### CREATE MASTER AND SLAVE VOLUMES

On **rhgs1** create the master volume using the mastervol.conf file with
gdeploy
```bash
gdeploy -c ~/mastervol.conf
```
  

On **rhgs4** create the slave volume using the slavevol.conf file with gdeploy
```bash
gdeploy -c ~/slavevol.conf
```


### CREATE THE GEO-REPLICATION SESSION

First a common pem pub file needs to be created

```bash
sudo gluster system:: execute gsec_create
```
``Common secret pub file present at /var/lib/glusterd/geo-replication/common_secret.pem.pub``

The next step is to create the actual replication session, using the pem file created above
  


```bash
sudo gluster volume geo-replication mastervol rhgs4::slavevol create push-pem
```
``Creating geo-replication session between mastervol & rhgs4::slavevol has been successful`` 


### START THE GEO-REPLICATION SESSION

```bash
sudo gluster volume geo-replication mastervol rhgs4::slavevol start
```
``Starting geo-replication session between mastervol & rhgs4::slavevol has been successful ``

Check if the deployment works correctly:

```bash
sudo gluster volume geo-replication mastervol rhgs4::slavevol status
```

``MASTER NODE    MASTER VOL    MASTER BRICK                  SLAVE USER    SLAVE SLAVE NODE    STATUS     CRAWL STATUS       LAST_SYNCED                    ``
``----------------------------------------------------------------------------------------------------------------------------------------------------------``
``rhgs1          mastervol     /rhgs/brick_xvdc/mastervol    root rhgs4::slavevol    rhgs5         Active     Changelog Crawl    2018-01-31 09:10:31        ``
``rhgs2          mastervol     /rhgs/brick_xvdc/mastervol    root rhgs4::slavevol    rhgs4         Passive    N/A                N/A                        ``

**NOTE** It might take a moment to reach this state, you will possibly see a status of "Initializing" shortly. 

### Client access

On **client1** mount the geo-replicated volume from **rhgs1** and create files on it:

```bash
sudo mkdir /rhgs/client/native/georep
sudo mount -t glusterfs rhgs1:clientvol /rhgs/client/native/georep
sudo mkdir /rhgs/client/native/georep/mydir
sudo chmod 777 /rhgs/client/native/georep/mydir
```

Now create 50 files
```bash
for i in {000..050}; do echo hello$i > /rhgs/client/native/georep/mydir/file$i;
done
```


## DISASTER RECOVERY

### SIMULATE A DATACENTER OUTAGE

We need to turn the master off completely, so that the slave can take over its
functions. Stop the glusterd on **rhgs1**
```bash
sudo systemctl stop glusterd
```

And also kill the glusterfsd and python processes
```bash
sudo pkill glusterfsd
sudo pkill python
```

### CHECK CLIENT ACCESS

On **client1** check if the files are still accessible
```bash
ls -l /rhgs/client/native/georep/mydir | wc -l
```

### PROMOTE THE SLAVE TO TEMPORARY MASTER

Now that rhgs1 and rhgs2 are dead, we need to make **rhgs4** the new, temporary,
master

```bash
sudo gluster volume set slavevol geo-replication.indexing on
sudo gluster volume set slavevol changelog on
```

### CHANGE CLIENT TO USE THE TEMPORARY MASTER

On **client1** umount the volume from rhgs1 which is no longer accessible and use the slavevol from rhgs4

```bash
sudo umount /rhgs/client/native/georep
sudo mount -t glusterfs rhgs4:slavevol /rhgs/client/native/georep
```

Check the contents of /rhgs/client/native/georep/mydir
```bash
ls -l /rhgs/client/native/georep/mydir | wc -l
```
``100``


