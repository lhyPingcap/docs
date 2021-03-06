---
title: TiKV Control User Guide
category: tools
---

# TiKV Control User Guide

TiKV Control (`tikv-ctl`) is a command line tool of TiKV, used to manage the cluster. When you compile TiKV, the `tikv-ctl` command is also compiled at the same time. If the cluster is deployed using Ansible, the `tikv-ctl` binary file exists in the corresponding `tidb-ansible/resources/bin` directory. If the cluster is deployed using the binary, the `tikv-ctl` file is in the `bin` directory together with other files such as `tidb-server`, `pd-server`, `tikv-server`, etc.
## General options

`tikv-ctl` provides two operation modes:

- Remote mode: use the `--host` option to accept the service address of TiKV as the argument

    For this mode, if SSL is enabled in TiKV, `tikv-ctl` also needs to specify the related certificate file. For example:

    ```
    $ tikv-ctl --ca-path ca.pem --cert-path client.pem --key-path client-key.pem --host 127.0.0.1:21060 <subcommands>
    ```

- Local mode: use the `--db` option to specify the local TiKV data directory path

Unless otherwise noted, all commands supports both the remote mode and the local mode.

Additionally, `tikv-ctl` has two simple commands `--to-hex` and `--to-escaped`, which are used to make simple changes to the form of the key.

Generally, use the `escaped` form of the key. For example:

```bash
$ tikv-ctl --to-escaped 0xaaff
\252\377
$ tikv-ctl --to-hex "\252\377"
AAFF
```

> **Note:** When you specify the `escaped` form of the key in a command line, it is required to enclose it in double quotes. Otherwise, bash eats the backslash and a wrong result is returned.

## Subcommands, some options and flags

This section describes the subcommands that `tikv-ctl` supports in detail. Some subcommands support a lot of options. For all details, run `tikv-ctl --help <subcommand>`.

### View information of the Raft state machine

Use the `raft` subcommand to view the status of the Raft state machine at a specific moment. The status information includes two parts: three structs (**RegionLocalState**, **RaftLocalState**, and **RegionApplyState**) and the corresponding Entries of a certain piece of log.

Use the `region` and `log` subcommands to obtain the above information respectively. The two subcommands both support the remote mode and the local mode at the same time. Their usage and output are as follows:

```bash
$ tikv-ctl --host 127.0.0.1:21060 raft region -r 2
region id: 2
region state key: \001\003\000\000\000\000\000\000\000\002\001
region state: Some(region {id: 2 region_epoch {conf_ver: 3 version: 1} peers {id: 3 store_id: 1} peers {id: 5 store_id: 4} peers {id: 7 store_id: 6}})
raft state key: \001\002\000\000\000\000\000\000\000\002\002
raft state: Some(hard_state {term: 307 vote: 5 commit: 314617} last_index: 314617)
apply state key: \001\002\000\000\000\000\000\000\000\002\003
apply state: Some(applied_index: 314617 truncated_state {index: 313474 term: 151})
```

### View the Region size

Use the `size` command to view the Region size:

```bash
$ tikv-ctl --db /path/to/tikv/db size -r 2
region id: 2
cf default region size: 799.703 MB
cf write region size: 41.250 MB
cf lock region size: 27616
```

### Scan to view MVCC of a specific range

The `--from` and `--to` options of the `scan` command accept two escaped forms of raw key, and use the `--show-cf` flag to specify the column families that you need to view.

```bash
$ tikv-ctl --db /path/to/tikv/db scan --from 'zm' --limit 2 --show-cf lock,default,write
key: zmBootstr\377a\377pKey\000\000\377\000\000\373\000\000\000\000\000\377\000\000s\000\000\000\000\000\372
         write cf value: start_ts: 399650102814441473 commit_ts: 399650102814441475 short_value: "20"
key: zmDB:29\000\000\377\000\374\000\000\000\000\000\000\377\000H\000\000\000\000\000\000\371
         write cf value: start_ts: 399650105239273474 commit_ts: 399650105239273475 short_value: "\000\000\000\000\000\000\000\002"
         write cf value: start_ts: 399650105199951882 commit_ts: 399650105213059076 short_value: "\000\000\000\000\000\000\000\001"
```

### View MVCC of a given key

Similar to the `scan` command, the `mvcc` command can be used to view MVCC of a given key.

```bash
$ tikv-ctl --db /path/to/tikv/db mvcc -k "zmDB:29\000\000\377\000\374\000\000\000\000\000\000\377\000H\000\000\000\000\000\000\371" --show-cf=lock,write,default
key: zmDB:29\000\000\377\000\374\000\000\000\000\000\000\377\000H\000\000\000\000\000\000\371
         write cf value: start_ts: 399650105239273474 commit_ts: 399650105239273475 short_value: "\000\000\000\000\000\000\000\002"
         write cf value: start_ts: 399650105199951882 commit_ts: 399650105213059076 short_value: "\000\000\000\000\000\000\000\001"
```

In this command, the key is also the escaped form of raw key.

### Print a specific key value

To print the value of a key, use the `print` command.

### Compact data manually

Use the `compact` command to manually compact TiKV data. If you specify the `--from` and `--to` options, then their flags are also in the form of escaped raw key. You can use the `--db` option to specify the RocksDB that you need to compact. The optional values are `kv` and `raft`.

```bash
$ tikv-ctl --db /path/to/tikv/db compact -d kv
success!
```

### Set a Region to tombstone

The `tombstone` command is usually used in circumstances where the sync-log is not enabled, and some data written in the Raft state machine is lost caused by power down.

In a TiKV instance, you can use this command to set the status of some Regions to Tombstone. Then when you restart the instance, those Regions are skipped. Those Regions need to have enough healthy replicas in other TiKV instances to be able to continue writing and reading through the Raft mechanism.

```bash
pd-ctl>> operator add remove-peer <region_id> <peer_id>
$ tikv-ctl --db /path/to/tikv/db tombstone -p 127.0.0.1:2379 -r 2
success!
```

> **Note:**
>
> - This command only supports the local mode.
> - The argument of the `--pd/-p` option specifies the PD endpoints without the `http` prefix. Specifying the PD endpoints is to query whether PD can securely switch to Tombstone. Therefore, before setting a PD instance to Tombstone, you need to take off the corresponding Peer of this Region on the machine in `pd-ctl`.

### Force Region to recover the service from multiple replicas failure

Use the `unsafe-recover remove-fail-stores` command to remove the failed machines from the peers list of all Regions. Then after you restart TiKV, these Regions can continue to provide services using the other healthy replicas. This command is usually used in circumstances where multiple TiKV stores are damaged or deleted.

```bash
$ tikv-ctl --db /path/to/tikv/db unsafe-recover remove-fail-stores 3,4,5
success!
```

> **Note:** This command only supports the local mode. It prints `success!` when successfully run.

### Send a `consistency-check` request to TiKV

Use the `consistency-check` command to execute a consistency check among replicas in the corresponding Raft of a specific Region. If the check fails, TiKV itself panics. If the TiKV instance specified by `--host` is not the Region leader, an error is reported.

```bash
$ tikv-ctl --host 127.0.0.1:21060 consistency-check -r 2
success!
$ tikv-ctl --host 127.0.0.1:21061 consistency-check -r 2
DebugClient::check_region_consistency: RpcFailure(RpcStatus { status: Unknown, details: Some("StringError(\"Leader is on store 1\")") })
```

> **Note:**
>
> - This command only supports the remote mode.
> - Even if this command returns `success!`, you need to check whether TiKV panics. This is because this command is only a proposal that requests a consistency check for the leader, and you cannot know from the client whether the whole check process is successful or not.

### Print the Regions where the Raft state machine corrupts

To avoid checking the Regions while TiKV is started, you can use the `tombstone` command to set the Regions where the Raft state machine reports an error to Tombstone. Before running this command, use the `bad-regions` command to find out the Regions with errors, so as to combine multiple tools for automated processing.

```bash
$ tikv-ctl --db /path/to/tikv/db bad-regions
all regions are healthy
```

If the command is successfully executed, it prints the above information. If the command fails, it prints the list of bad Regions. Currently, the errors that can be detected include the mismatches between `last index`, `commit index` and `apply index`, and the loss of Raft log. Other conditions like the damage of snapshot files still need further support.

### View Region properties

- To view in local the properties of Region 2 on the TiKV instance that is deployed in `/path/to/tikv`:

    ```bash
    $ tikv-ctl --db /path/to/tikv/data/db region-properties -r 2
    ```

- To view online the properties of Region 2 on the TiKV instance that is running on `127.0.0.1:20160`:

    ```bash
    $ tikv-ctl --host 127.0.0.1:20160 region-properties -r 2
    ```
