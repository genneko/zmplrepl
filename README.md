# zmplrepl
[SZ]imple ZFS Replicator

## Install
Install this program on both sending and receiving hosts.

1. Clone this repo in a directory of your choice.
    ```
    $ cd ~/app
    $ git clone git@github.com:genneko/zmplrepl.git
    or
    $ git clone https://github.com/genneko/zmplrepl.git
    ```

2. Add the cloned directory to your $PATH.
    ```
    PATH="$PATH:$HOME/app/zmplrepl"
    ```

## Quick Guide
Assume you want to replicate zroot/data and all its descendant datasets on the sending host to the receiving host's backup/data.
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
    ```
    repl@r$ sudo zfs create -u backup/data
    ```

4. Grant appropriate permissions on the source/destination dataset to the sending and receiving users.
    ```
    repl@s$ sudo zfs allow -u repl send,snapshot,hold zroot/data
    repl@r$ sudo zfs allow -u repl receive,create,mount,mountpoint,compression,recordsize,atime,canmount backup/data
    ```

5. Take an initial snapshot on the sending host.  
Then send it as a full replication stream to the receiving host.
    ```
    repl@s$ zfs snapshot -r zroot/data@first
    repl@s$ zmplrepl -R zroot/data r:backup
    ```

6. Afterwards, run zmplrepl again without -R after taking a snapshot.
    ```
    repl@s$ zfs snapshot -r zroot/data@second
    repl@s$ zmplrepl zroot/data r:backup
    ```

7. If you want to mount the replicated datasets on the receiving host, run the following command after examing and correcting properties such as mountpoint.
    ```
    repl@r$ zfs list -o name,mountpoint -r backup/data
    repl@r$ zfs set mountpoint=... (If required)
    repl@r$ zfs list -o name -H -r backup/data | sudo xargs -n1 zfs mount
    ```

    Optionally, you can make the datasets readonly to keep someone from accidentally changing their contents.
    ```
    repl@r$ sudo zfs set readonly=on backup/data
    ```

## Usage
```
zmplrepl [-nvpIFRz] [-k KEYFILE] [-S RE]
         [-f fromSnap] srcDs[@toSnap] [host:]dstDs(base)
zmplrepl [-nvpIFz] -s [-k KEYFILE] [-S RE]
         [-f fromSnap] srcDs[@toSnap] [host:]dstDs
zmplrepl [-h]

  -n: Dry-run (zfs send -nv).
  -v: Be verbose. Can specify as -vv to be more verbose (zfs send -v).
  -p: Preserve properties on sender (zfs send -p). -R implies -p.
  -I: Use dense mode (zfs send -I) when sending incremental stream.
  -F: Force full stream.
      By default, zmplrepl sends incremental stream if both sender and
      receiver have a common snapshot, while zmplrepl sends full stream
      if there is no common snapshot for both sides.
  -R: Send full replication stream (zfs send -R) based on the srcDs.
      By default, zmplrepl sends indivudual stream for each dataset
      under the srcDs.
  -S: Specify an extended regular expression to filter snapshot list.
  -z: Short-hand for -S "zfs-auto-snap_(daily|weekly|monthly)".
  -k: Specify a SSH private key to run zmplhelper on the receiving host.
  -f: Specify a fromSnap for incremental stream.
      By default, the latest common snapshot for both sides are used.
  -s: Use one-to-one stream. When using -s, specify an absolute dataset
      path for the dstDs which corresponds to the srcDs. Otherwise,
      dstDs should be the base (container) dataset for a replication.
      Here are some examples.
        zmplrepl -s zroot/data/a/b/c r:backup/c
	  -> 'zroot/data/a/b/c' is replicated to r's 'backup/c'.
	zmplrepl zroot/data/x/y/z r:backup
	  -> 'zroot/data/x/y/z' is replicated to r's 'backup/data/x/y/z'.
  -h: Show this usage and exit.
```
