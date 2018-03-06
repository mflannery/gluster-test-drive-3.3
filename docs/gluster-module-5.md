# Lab Guide <br/> Gluster Test Drive Module 5 <br/> Snapshot Operations and Administration

## LAB AGENDA

Welcome to the Gluster Test Drive Module 5 - Snapshots

- Learn how snapshots are created and deleted
- List available snapshot and get information about them
- Access and restore snapshots
- Configure snapshot behaviour

## ABOUT SNAPSHOTS

Snapshots are **point-in-time copies** of volumes, which can be used to protect data. They are read-only by default so data on them can't be accidentally deleted, corrupted or modified. 

The most important features of snapshots are:

- Crash consistency
- Online snapshots
- Quorum based

Each snapshot creates as many bricks as the original volume has.

## GETTING STARTED

If you have not already done so, click the <img src="http://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl02-working-ebs/media/image005.png"> button in the navigation bar above to launch your lab. If you are prompted for a token, use the one distributed to you (or credits you've purchased).

> **NOTE** It may take **up to 10 minutes** for your lab systems to start up before you can access them.

## LAB SETUP

### CONNECT TO THE LAB

connect to the **rhgs1** server instance, using its public IP address from the
**Addl. Info** tab to the right. 
```bash
ssh student@<rhgs1PublicIP>
```

## OPTIONAL: CREATE THE DISTVOL VOLUME

If you have not already done so as part of **Module 2**, deploy the ``distvol`` volume, using the provided gdeploy configuration file.

```bash
gdeploy -c ~/materials/gdeploy/distvol.conf
```
Confirm the volume configuration.
```bash
sudo gluster volume info distvol
```

```bash
Volume Name: distvol
Type: Distribute
Volume ID: 957b4f60-4c2e-401f-b13c-558fd9df0c45
Status: Started
Snapshot Count: 0
Number of Bricks: 6
Transport-type: tcp
Bricks:
Brick1: rhgs1:/run/gluster/snaps/d1fbe5166ad54fb7b0917e8cf5d5ba00/brick1/distvol
Brick2: rhgs2:/run/gluster/snaps/d1fbe5166ad54fb7b0917e8cf5d5ba00/brick2/distvol
Brick3: rhgs3:/run/gluster/snaps/d1fbe5166ad54fb7b0917e8cf5d5ba00/brick3/distvol
Brick4: rhgs4:/run/gluster/snaps/d1fbe5166ad54fb7b0917e8cf5d5ba00/brick4/distvol
Brick5: rhgs5:/run/gluster/snaps/d1fbe5166ad54fb7b0917e8cf5d5ba00/brick5/distvol
Brick6: rhgs6:/run/gluster/snaps/d1fbe5166ad54fb7b0917e8cf5d5ba00/brick6/distvol
Options Reconfigured:
transport.address-family: inet
nfs.disable: off
features.quota: off
features.inode-quota: off
features.quota-deem-statfs: off
```

Make sure the ``nfs.disable`` attribute is set to off. In case it's not, run
```bash
sudo gluster volume set distvol nfs.disable off
```

## SNAPSHOT PREREQUISITES

The snapshots will consists of the same number of bricks as the original
volumes. By default bricks communicate via privileged ports, but the number of
such ports is limited to 1024. In order to support up to 256 snapshots per
volume the glusterd must be allowed to use non-privileged ports.

This can either be done manually by following the steps below or by running an
ansible playbook. If you choose to use ansible, please run

```bash
ansible-playbook -i materials/ansible/inventory materials/ansible/module5.yaml
```

To achieve the same manually, please follow these steps:

Permit insecure ports for the volume for which snapshots will be used:
```bash
# sudo gluster volume set distvol server.allow-insecure on
```

Edit ``/etc/glusterfs/glusterd.vol`` in every cluster node, and add the
following line:
```bash
option rpc-auth-allow-insecure on
```

Then restart the glusterd service to make make the changes effective:
```bash
# sudo systemctl restart glusterd
```


## NFS CLIENT ACCESS
A Gluster volume can be accessed through multiple standard client protocols, as well as through specialized methods including the OpenStack Swift protocol and a direct API.  
                                                                                       
For many common use cases, the well-established NFS protocol is used for ease of implementation and compatibility with existing applications and architectures. For some use cases, the NFS protocol may also offer a performance benefit over other access methods. 
                                                                                       
You can connect to the **client1** system via `ssh` directly from the rhgs1 system.    
                                                                                       
```bash                                                                                
ssh student@client1                                                                    
```                                                                                    
                                                                                       
Here, on the RHEL client, you will mount via NFS the Gluster **distvol** volume you created above.                                                                            
                                                                                       
```bash                                                                                
sudo mkdir -p /rhgs/client/nfs/distvol 
```
```bash
sudo mount -t nfs rhgs1:/distvol /rhgs/client/nfs/distvol 
```                                                                                    
                                                                                       
Check the mount and observe the output.                                                
                                                                                       
```bash                                                                                
df -h /rhgs/client/nfs/distvol 
```                                                                                    
                                                                                       
``Filesystem      Size  Used Avail Use% Mounted on``                                   
``rhgs1:/distvol   60G  198M   60G   1% /rhgs/client/nfs/distvol``                     
                                                                                       
```bash                                                                                
mount | grep distvol 
```                                                                                    
                                                                                       
```
rhgs1:/distvol on /rhgs/client/nfs/distvol type nfs (rw,relatime,vers=3,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=10.100.1.11,mountvers=3,mountport=38465,mountproto=tcp,local_lock=none,addr=10.100.1.11)```
```

                                                                                       
Create and set permissions on a subdirectory to hold your data.                        
                                                                                       
                                                                                       
```bash                                                                                
sudo mkdir /rhgs/client/nfs/distvol/mydir 
```
```bash
sudo chmod 777 /rhgs/client/nfs/distvol/mydir 
```                                                                                    
                                                                                       
Add 100 files to the directory.                                                        
                                                                                       
```bash                                                                                
for i in {001..100}; do echo hello$i > /rhgs/client/nfs/distvol/mydir/file$i; done 
```                                                                                    
                                                                                                                                                                             
List the directory, counting its contents to confirm the 100 files written.                                                                                                  
                                                                                                                                                                             
```bash                                                                                                                                                                      
ls /rhgs/client/nfs/distvol/mydir/ | wc -l 
```                                                                                                                                                                          
                                                                                                                                                                             
``100``                                                                                                                                                                      
    
## CREATION OF SNAPSHOTS

Now that the volume is populated with your 100 data files take a snapshot.
Logout from **client1**
```
exit
```
Create the snapshot on **rhgs1**

```bash
sudo gluster snapshot create snap1 distvol no-timestamp description "populated volume"
```
``snapshot create: success: Snap snap1 created successfully``


## RETRIEVE INFORMATION ABOUT SNAPSHOTS

Check the list of available snapshots
```bash
sudo gluster snapshot list
```
``
snap1
``

There is more information available for this snapshot

```bash
sudo gluster snapshot info volume distvol
```
```
Volume Name               : distvol        
Snaps Taken               : 1              
Snaps Available           : 255            
        Snapshot                  : snap1                      
        Snap UUID                 : bc9abd1b-6239-49ec-aa0c-d0e56d619dd1               
        Description               : populated volume                                   
        Created                   : 2018-01-30 10:47:28                                
        Status                    : Stopped    
```


## ACCESS AND RESTORE SNAPSHOTS

Snaphosts of volumes can only be accessed via FUSE (native) mounts. 
In order to do that snapshots must be activated.

```bash
sudo gluster snapshot activate snap1
```
``Snapshot activate: snap1: Snap activated successfully``

Connect to **client1** using ssh and create a directory for the snapshots to be mounted:
```bash
ssh student@client1
```
```bash
sudo mkdir -p /rhgs/snaps/snap1
```

Mount the snapshot using
```bash
sudo mount -t glusterfs rhgs1:/snaps/snap1/distvol /rhgs/snaps/snap1
```

Check the contents of the **read-only** snapshot
```bash
ls /rhgs/snaps/snap1/mydir | wc -l 
```
```
100
```

Now delete all the files in ``/rhgs/client/nfs/distvol/mydir/``

```bash
rm -rf /rhgs/client/nfs/distvol/mydir/*
```
```bash
ls /rhgs/client/nfs/distvol/mydir/ | wc -l
```
```
0
```

There are no files left, all 100 files have been deleted. 

Check if the files are still available in the snapshot **snap1**
```bash
ls /rhgs/snaps/snap1/mydir/ | wc -l
```
```
100
```

Go back to **rhgs1** and restore the original volume from the snapshot. 
For this step the volume must be stopped

```bash
exit
```

```bash
sudo gluster volume stop distvol 
```
```
Stopping volume will make its data inaccessible. Do you want to continue (y/n) y
volume stop: distvol: success
```

Now restore the volume from the snapshot ``snap1``
```bash
sudo gluster snapshot restore snap1
```
```
Restore operation will replace the original volume with the snapshotted volume.
Do you still want to continue? (y/n) y
```

``Snapshot restore: snap1: Snap restored successfully``

Start the volume: 
```bash
sudo gluster volume start distvol
```

Go back to **client1** and check the contents of ``/rhgs/client/nfs/distvol/mydir/``
```bash
ssh student@client1
```
```bash
ls /rhgs/client/nfs/distvol/mydir/ | wc -l
```
``100``

All the files that have been deleted have been restored from the snapshot that was taken before the deletion.

IMPORTANT: Once the snapshot is restored it will be deleted. If you plan on further possible restores, create a new snapshot immediately.

## CONFIGURE SNAPSHOT BEHAVIOUR

Since snapshots can easily fill up the available space on the gluster nodes, there are a few tunable parameters to prevent that from happening:

- **snap-max-hard-limit**: Once the snapshot count reaches this limit, no further snapshots can be created. The range is from 1 up to 256.
- **snap-max-soft-limit**: This parameter is a percentage value and defaults to 90%.  It works together with the auto-delete feature. So once this soft-limit is reached, the system wil delete the oldest snapshot. If auto-delete is disabled, it will display a warning. 
- **auto-delete**: If enabled, this option will work with snap-max-soft-limit as described above. **Note:** This is a global option and can not be set on a per-volume basis.

Log out from **client1**
```bash
exit
```

Display the configuration values:
```bash
sudo gluster snapshot config distvol
```

```
Snapshot System Configuration:
snap-max-hard-limit : 256
snap-max-soft-limit : 90%
auto-delete : disable
activate-on-create : disable

Snapshot Volume Configuration:

Volume : distvol
snap-max-hard-limit : 256
Effective snap-max-hard-limit : 256
Effective snap-max-soft-limit : 230 (90%)
```

Set the maximum number of snapshots for ``distvol`` to 3
```bash
sudo gluster snapshot config distvol snap-max-hard-limit 3
```

```
Changing snapshot-max-hard-limit will limit the creation of new snapshots if they exceed the new limit.
Do you want to continue? (y/n) y
snapshot config: snap-max-hard-limit for distvol set successfully
```


Now create 3 snapshots for distvol:
```bash
for i in {1..3}; do sudo gluster snapshot create snap$i distvol no-timestamp force; done
```
```
snapshot create: success: Snap snap1 created successfully 
snapshot create: success: Snap snap2 created successfully 
snapshot create: success: Snap snap3 created successfully 
Warning: Soft-limit of volume (distvol) is reached. Snapshot creation is not possible once hard-limit is reached. 
```

If we create a 4th snapshot, we cross the pre-defined limit:
```bash
sudo gluster snapshot create snap4 distvol no-timestamp force
```
```
snapshot create: failed: The number of existing snaps has reached the effective maximum limit of 3, for the volume (distvol). Please delete few snapshots before taking further snapshots. 
Snapshot command failed
```


