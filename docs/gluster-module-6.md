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

The ansible script **module6**  home will do the following:

1. Stop and delete the volumes "repvol" and "distvol" if they exist.
2. Split up the existing cluster into 2 separate clusters
3. Create a ssh key for "root" and copy it to "rhgs4" since password-less root access is needed later on for the geo-replication.
4. Copy required materials to **rhgs4**

Run the ansible script
```bash
ansible-playbook -i ~/materials/ansible/inventory ~/materials/ansible/module6.yaml
```

Once the script has finished, two separate clusters are set up


|local         | remote     |
|--------------|------------|
|rhgs1         | rhgs4      |
|rhgs2         | rhgs5      |
|rhgs3         | rhgs6      |


We will set up a geo-replication with the master being in the "local" cluster and the slave in the "remote" cluster.


### CREATE MASTER AND SLAVE VOLUMES

On **rhgs1** create the master volume using the mastervol.conf file with gdeploy
```bash
gdeploy -c ~/materials/gdeploy/mastervol.conf
```
  

Also on **rhgs1** create the slave volume using the slavevol.conf file with gdeploy
```bash
gdeploy -c ~/materials/gdeploy/slavevol.conf
```


### CREATE THE GEO-REPLICATION SESSION

First a common pem pub file needs to be created, back on **rhgs1**

```bash
sudo gluster system:: execute gsec_create
```
```
Common secret pub file present at /var/lib/glusterd/geo-replication/common_secret.pem.pub
```

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

```
MASTER NODE    MASTER VOL    MASTER BRICK                  SLAVE USER    SLAVE SLAVE NODE    STATUS     CRAWL STATUS       LAST_SYNCED  
----------------------------------------------------------------------------------------------------------------------------------------------------------
rhgs1          mastervol     /rhgs/brick_xvdc/mastervol    root rhgs4::slavevol    rhgs5         Active     Changelog Crawl    2018-03-06 15:08:22          
rhgs2          mastervol     /rhgs/brick_xvdc/mastervol    root rhgs4::slavevol    rhgs4         Passive    N/A                N/A        
```

**NOTE** It might take a moment to reach this state, you will possibly see a status of "Initializing" shortly. 
Please wait until the above state has been reached before you move on.

### CLIENT ACCESS

On **client1** mount the geo-replicated volume from **rhgs1** and create files on it:

```bash
ssh student@client1
```
```bash
sudo mkdir -p /rhgs/client/native/georep
```
```bash
sudo mount -t glusterfs rhgs1:mastervol /rhgs/client/native/georep
```
```bash
sudo mkdir -p /rhgs/client/native/georep/mydir
```
```bash
sudo chmod 777 /rhgs/client/native/georep/mydir
```

Now create 50 files
```bash
for i in {001..050}; do echo hello$i > /rhgs/client/native/georep/mydir/file$i;
done
```
```bash
exit
```


## DISASTER RECOVERY

### SIMULATE A DATACENTER OUTAGE

We need to turn the master off completely, so that the slave can take over its functions. Stop the glusterd on **rhgs1** AND **rhgs2** since it's a replicated volume

```bash
sudo systemctl stop glusterd
```
And also kill the glusterfsd and python processes
```bash
sudo pkill -9 glusterfsd
```
```bash
sudo pkill -9 python
```
```bash
sudo pkill -9 glusterfs
```

Do the same on **rhgs2**

```bash
ssh student@rhgs2
```
```bash                                                                          
sudo systemctl stop glusterd 
```                                                                              
And also kill the glusterfsd and python processes 
```bash                                                                                                                                           
sudo pkill -9 glusterfsd 
```
```bash
sudo pkill -9 python
```     
```bash
sudo pkill -9 glusterfs
```
```bash
exit
```

### CHECK CLIENT ACCESS

On **client1** check if the files are still accessible
```bash
ssh student@client1
```
```bash
ls -l /rhgs/client/native/georep/mydir | wc -l
```
```
ls: cannot access /rhgs/client/native/georep/mydir: Transport endpoint is not
connected
0
```
```bash
exit
```


### PROMOTE THE SLAVE TO TEMPORARY MASTER

Now that rhgs1 and rhgs2 are dead, we need to make **rhgs4** the new, temporary,
master. On **rhgs4** run

```bash
ssh student@rhgs4
```
```bash
sudo gluster volume set slavevol geo-replication.indexing on
```
```bash
sudo gluster volume set slavevol changelog on
```
```bash
exit
```

### CHANGE CLIENT TO USE THE TEMPORARY MASTER

On **client1** umount the volume from rhgs1 which is no longer accessible and use the slavevol from rhgs4

```bash
ssh student@client1
```
```bash
sudo umount /rhgs/client/native/georep
```
```bash
sudo mount -t glusterfs rhgs4:slavevol /rhgs/client/native/georep
```

Check the contents of /rhgs/client/native/georep/mydir
```bash
ls /rhgs/client/native/georep/mydir | wc -l
```
``50``



