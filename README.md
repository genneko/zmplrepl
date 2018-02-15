# zmplrepl
[SZ]imple ZFS Replicator

## Install
Clone this repo in a directory of your choice. Optionally, add the cloned directory to your $PATH.
```
$ cd ~/app
$ git clone git@github.com:genneko/zmplrepl.git
or
$ git clone https://github.com/genneko/zmplrepl.git
```

## Quick Guide
Assume you want to replicate zroot/data and all descendant datasets on the sending host to the receiving host's backup/data such like:
```
  [sender(s)]          [receiver(r)]
  zroot/data       --> backup/data
  zroot/data/doc   --> backup/data/doc
  zroot/data/photo --> backup/data/photo
  zroot/data/video --> backup/data/video
```

You can achieve this by taking the following steps.

1. Create a dedicated user on both sending and receiving hosts.  
It's optional but recommended in production environment.

2. Generate a SSH public key for the sending user.  
Then put it into the receiving user's ~/.ssh/authorized_keys.

3. Create a base dataset with -u (unmounted) on the receiving host.  
If you are using property-based backup solution such as zfstools, you might want to explicitly disable it on the dataset.
    ```
    repl@r$ sudo zfs create -u backup/data
    repl@r$ sudo zfs set com.sun:auto-snapshot=false backup/data
    ```

4. Grant appropriate permissions on the source/destination dataset to the sending and receiving users.
    ```
    repl@s$ sudo zfs allow -u repl send,snapshot,hold zroot/data
    repl@r$ sudo zfs allow -u repl receive,create,mount,mountpoint,compression,recordsize backup/data
    ```

5. Take an initial snapshot on the sending host.  
Then send it as a full replication stream to the receiving host.
    ```
    repl@s$ zfs snapshot -r zroot/data@first
    repl@s$ zmplrepl -R zroot/data r:backup
    ```

6. Afterwards, run the same command without -R again to synchronize.
    ```
    repl@s$ zmplrepl zroot/data r:backup
    ```

7. If you want to mount the replicated datasets on the receiving host, run the following command.
    ```
    repl@r$ zfs list -o name -H -r backup/data | sudo xargs -n1 zfs mount
    ```

    Optionally, you can make the datasets readonly to keep someone from accidentally changing their contents.
    ```
    repl@r$ sudo zfs set readonly=on backup/data
    ```

## Usage
```
  zmplrepl [-nvpiF] [-R] [-f fromSnap] srcDs[@toSnap] [host:]dstDs(base)
  zmplrepl [-nvpiF] -s [-f fromSnap] srcDs[@toSnap] [host:]dstDs
  zmplrepl [-gh]

  -n: Dry-run (zfs send -nv).
  -v: Be verbose.
  -p: Preserve properties on sender (zfs send -p). -R implies -p.
  -i: Use sparse mode (zfs send -i) when sending incremental stream.
  -F: Force full stream.
      By default, zmplrepl sends incremental stream if both sender and
      receiver have a common snapshot, while zmplrepl sends full stream
      if there is no common snapshot for both sides.
  -R: Send full replication stream (zfs send -R) based on the srcDs.
      By default, zmplrepl sends indivudual stream for each dataset
      under the srcDs.
  -f: Specify a fromSnap for incremental stream.
      By default, the latest common snapshot for both sides are used.
  -s: Use one-to-one stream. Set dstDs to a absolute destination dataset
      path for the srcDs when using -s.
      Otherwise, dstDs should be the base dataset for replication.
      Here are some examples.
        zmplrepl -s zroot/data/a/b/c r:backup/c
          -> 'zroot/data/a/b/c' is replicated to r's 'backup/c'.
        zmplrepl zroot/data/x/y/z r:backup
          -> 'zroot/data/x/y/z' is replicated to r's 'backup/data/x/y/z'.
  -g: Show quick guide and exit.
  -h: Show this usage and exit.
```
