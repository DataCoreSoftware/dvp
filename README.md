# DataCore Docker Volume Plugin

## Deploying

1. Verify you have met all the requirements.
 - DataCore SANsymphony version 10.0 PSP7 Update 2 or later.
 - DataCore REST Server version 1.08 or later.
 - Docker version 18.03 or later, CE or EE.
 - If using iSCSI, then the iSCSI initiator must have already been installed.
 - If using FibreChannel, then connectivity must already be established between the host node and storage server over the FC fabric.
 - Multipathing should be enabled if mirrored virtual disks will be used. Please refer to the guide for configuring linux hosts [here](http://datacore.custhelp.com/app/answers/detail/a_id/1546/session/L2F2LzEvdGltZS8xNTM3NDczNTY1L2dlbi8xNTM3NDczNTY1L3NpZC9mVUc3NWszSjdDendFZG5ZJTdFemJOQVRQQjZnc0JfaDR6emxQakd3SU1USlYlN0VDWHZiNVlPY1FtM1ZyN2loeGVvUzdFOFFIRlVrY0VVTzlBN2VnNHo1JTdFOE9kcEdKcUlIOHdETFR4dVpQRl9QXyU3RTB5TWtydDlNanFaQSUyMSUyMQ%3D%3D).
 
 **Windows hosts are not currently supported even if running linux workloads.**

2. Start the DataCore plugin.
    ```
    # docker plugin install store/datacoresoftware/dvp:1.0.1 --alias datacore --grant-all-permissions
    ```
    
3. If using iSCSI, connect your docker hosts to each of the SANsymphony nodes you want to consume storage from. For example
    ```
    # iscsiadm -m discovery -t sendtargets -I default -p <FQDN/IPaddress> --op new
    # iscsiadm -m node --login
    ```
    
4. Create the configuration file for the plugin. This file must be located at `/etc/datacore.json`. Be sure to use the correct options for your environment. 
    ```
    cat << EOF > /etc/datacore.json
    {
        "StorageServer": "FQDN or IP address of one of your SANsymphony nodes",
        "RESTServer": "FQDN or IP address of your DataCore REST server",
        "Username": "Username used to connect to StorageServer - must have admin role",
        "Password": "Base64 encoded without padding password for the username",
        "Template": "Default virtual disk template to use when creating persistent storage",
        "Nodename": "`hostname`",
        "DockerVer": "`docker --version`"
    }
    EOF
    ```

## Start using the plugin
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

## Examples

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
