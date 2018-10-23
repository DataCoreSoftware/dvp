# DataCore Docker Volume Plugin


### Prerequisites

> DataCore Server
- DataCore SANSymphony or HVSAN version 10.0 PSP7 U2 or later.
- DataCore REST server version 1.08 or later.

> Docker Hosts 

- Docker version 18.06 or later, Community or Enterprise Editions.
- If using iSCSI, then the iSCSI initiator must have already been installed.
- If using FibreChannel, then connectivity must already be established between the host node and storage server over the FC fabric.
- Multipathing should be enabled if mirrored virtual disks will be used. Please refer to the guide for configuring linux hosts [here](http://datacore.custhelp.com/app/answers/detail/a_id/1546/session/L2F2LzEvdGltZS8xNTM3NDczNTY1L2dlbi8xNTM3NDczNTY1L3NpZC9mVUc3NWszSjdDendFZG5ZJTdFemJOQVRQQjZnc0JfaDR6emxQakd3SU1USlYlN0VDWHZiNVlPY1FtM1ZyN2loeGVvUzdFOFFIRlVrY0VVTzlBN2VnNHo1JTdFOE9kcEdKcUlIOHdETFR4dVpQRl9QXyU3RTB5TWtydDlNanFaQSUyMSUyMQ%3D%3D).

> Constraints
- Windows hosts are currently not supported even if running linux workloads.

### Installation
> Preparation for iSCSI
- Connect to SANsymphony with the iSCSI initiator using `iscsiadm`. For example, `iscsiadm -m discovery -t sendtargets -I default -p <IPaddress> --op new`. Be sure to replace *IPaddress* with the address of target port you want to connect to. Do this for each target port you want to connect to, then log in to all ports with `iscsiadm -m node --login`.

> DataCore Plugin
    
- Install the plugin:
    ```
    # docker plugin install --alias datacore --disable datacoresoftware/dvp:1.0.0
    ```
    The plugin will request privileges for the following, which must be approved for it to work:
    ```
    Plugin "datacoresoftware/dcsdvp" is requesting the following privileges:
     - network: [host]
     - mount: [/dev]
     - mount: [/sys]
     - mount: [/var/run]
     - mount: [/lib/modules]
     - mount: [/etc/iscsi]
     - allow-all-devices: [true]
     - capabilities: [CAP_IPC_LOCK CAP_IPC_OWNER CAP_NET_ADMIN CAP_SYS_ADMIN CAP_MKNOD CAP_SYS_MODULE]
    ```

> Configuration
- Before enabling the plugin for use, you must set configuration parameters for the name of the storage server, rest server, credentials and virtual disk template name:
    ```
    # docker plugin set datacore DCSSVR=<IPaddress/FQDN of SANsymphony server>
    # docker plugin set datacore DCSREST=<IPaddress/FQDN of SANsymphony REST server>
    # docker plugin set datacore DCSUNAME=<Username of account to connect to SANsymphony with>
    # docker plugin set datacore DCSPWORD=<Password for the username>
    # docker plugin set datacore DCSTEMPL=<Virtual disk temlate name to use when creating persistent volumes>
    # docker plugin set datacore DCSNODENAME=`hostname`
    # docker plugin set datacore DOCKERVER=`docker --version`
    ```
    
- Enable the plugin
    ```
    # docker plugin enable datacore
    ```

### Using the plugin
- The command line syntax to create a persistent volume is:
    ```
    docker volume create -d datacore --name <NameOfVolume> -o <Option>=<Value>
    ```
    Where Option can be any of the following:
    
    - `size` specifies the size of the persistent volume to create, for example `-o size=100GB`
    - `fromSnap` specifies that the new persistent volume should be created from a snapshot, for example `-o fromSnap=NameOfSnap`
    - `rollback` specifies that the new persistent volume should be created from a point in time of an existing volume. Requires one of the additional parameters below:
        - `rollbacktime` specifies that the new persistent volume will be created based on a number of seconds in the past from the current time, for example, `-o rollback=original-volume -rollbacktime=300` will create a new persistent volume that contains the data on the volume `original-volume` from 300 seconds from the present.
        - `rollbackutc` specifies that the new persistent volume will be created based on an absolute time, for example `-o rollback=original-volume -o rollbackutc="2018-08-01T00:00:00Z"` will create the new persistent volume with the data from original-volume as it existed on midnight 08/01/2018.
    - `template` allows you to specifiy a different template to create a new persistent volume from a different virtual disk template than the one specified at configuration time.

### Examples

- Simple Volume Creation: Create a persistent volume named *test-vol1* with size *5GB*
    ```
    docker volume create -d datacore --name test-vol1 -o size=5GB
    ```
    
- Create a volume based on an already existing snapshot of a persistent volume:
    ```
    docker volume create -d datacore --name new-vol -o fromSnap=snapshot-name
    ```
    
- Create a volume based on a point in time from an already existing volume. In this example, a new persistent volume will be created based on the data that was present 300 seconds ago in the volume named existing-data.
    ```
    docker volume create -d datacore --name some-time-ago -o rollback=existing-data -o rollbacktime=300
    ```
    
- Create a volume based on a point in time from an already existing volume. In this example, a new persistent volume will be created based on the data that was present at an absolute time, expressed in UTC, in the volume named existing-data.
    ```
    docker volume create -d datacore --name some-time-ago -o rollback=existing-data -o rollbackutc="2018-08-01T00:00:00Z"
    ```
