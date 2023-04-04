---
title: Running Zero Copy Cache
layout: post
date: 2023-03-28
tags:
    - zero copy
    - kernell bypass
    - runbook
enableToc: true
---

## What is Zero Copy Cache

## Cornflakes Integration

Currently, the way to run zero copy cache is using Cornflakes, and that is where all my testing happens. This is to make sure that there is a standard application where the cache can be useful. 

### Installing Cornflakes

Ideally, I should be using Deepti's Python Script, but I prefer this method. Go into the install folder, and `cat install.sh`, this gives you a list of commands. Create a new mount on `/mnt/sdb` and run all the commands in that mount. This makes it easy for running the commands at will! Also, one of the commands has `PRIMARY=y` option, just use this. Don't know why, but it is what it is! 

Once installation stops are done, go the root folder, and run `make submodules CONFIG_MLX5=y`, this downloads and compiles all the submodules needed. If you plan to run the redis server, then go into `redis/deps/jemalloc` and run `./autogen.sh`. 

Current status: Use the `mellanox_0cc` branch. Don't use the main branch. 

### Editing the Config file

Run `sudo lshw -c network -businfo` to get a list of all the NICs which are present in the machine. If you're using cloudlab, you might need to activate some of them. Run `ifconfig` to check if something is not yet online. To bring an interface online, running the following commands. 

```

# turn on the interface
$ sudo ip link set dev <ifacename> up

# set the MTU of the interface
$ sudo ip link set dev <ifacename> mtu 9000a

[machine 1] sudo ip addr add dev <ifacename> 192.168.1.1/24 brd +
[machine 2] sudo ip addr add dev <ifacename> 192.168.1.2/24 brd +

```

Now, you can edit the config file in the base folder. 

### Running Redis Server

Compile Step: `make redis CONFIG_MLX=5`

Running, the following command assumes that your cornflakes repo is downloaded into `/mnt/sdb`, you can change it if you want. 

```
sudo LD_LIBRARY_PATH=/mnt/sdb/cornflakes/dpdk-datapath/3rdparty/dpdk/build/lib/x86_64-linux-gnu/:/mnt/sdb/cornflakes/target/release:/mnt/sdb/cornflakes/cf-kv/c/kv-sga-cornflakes-c/target/release \
CONFIG_PATH=/mnt/sdb/cornflakes/example_config.yaml \
RUST_BACKTRACE=full \
RUST_LOG=info \
SERIALIZATION=cornflakes-dynamic \
SERVER_IP=10.10.1.1 \
NUM_VALUES=1 \
NUM_KEYS=1 \
VALUE_SIZE=UniformOverSizes-4096 \
MIN_MEMPOOL_SIZE=131072 \
YCSB_TRACE=/proj/demeter-PG0/deeptir/ycsbc-traces/workloadc-1mil/workloadc-1mil-1-batched.load \
INLINE_MODE=nothing \
/mnt/sdb/cornflakes/redis/src/redis-server /mnt/sdb/cornflakes/redis/redis.conf
```

Running it with the mock traces generated. The mock trace has 1000 keys accessed 10000 times in a uniform fashion. 

```
sudo LD_LIBRARY_PATH=/mnt/sdb/cornflakes/dpdk-datapath/3rdparty/dpdk/build/lib/x86_64-linux-gnu/:/mnt/sdb/cornflakes/target/release:/mnt/sdb/cornflakes/cf-kv/c/kv-sga-cornflakes-c/target/release \
CONFIG_PATH=/mnt/sdb/cornflakes/example_config.yaml \
RUST_BACKTRACE=full \
NUM_REGISTRATIONS=10 \
REGISTER_AT_START=1 \
RUST_LOG=info \
SERIALIZATION=cornflakes-dynamic \
SERVER_IP=10.10.1.1 \
NUM_VALUES=1 \
NUM_KEYS=1 \
VALUE_SIZE=UniformOverSizes-4096 \
MIN_MEMPOOL_SIZE=131072 \
YCSB_TRACE=/proj/demeter-PG0/vish/example_workload/egwrkc-2-batched.load \
INLINE_MODE=nothing \
/mnt/sdb/cornflakes/redis/src/redis-server /mnt/sdb/cornflakes/redis/redis.conf
```


### Running Redis Client

There is no specific redis client, just use the Cornflakes KV Client. Compile Step: `make kv CONFIG_MLX5=y CONFIG_DPDK=y`.

Running, the following command assumes that your cornflakes repo is downloaded into `/mnt/sdb` you can change it if you want. 

```

sudo env LD_LIBRARY_PATH=/mnt/sdb/cornflakes/dpdk-datapath/3rdparty/dpdk/build/lib/x86_64-linux-gnu /mnt/sdb/cornflakes/target/release/ycsb_dpdk \
--config_file /mnt/sdb/cornflakes/example_config.yaml \
--mode client \
--queries /proj/demeter-PG0/deeptir/ycsbc-traces/workloadc-1mil/workloadc-1mil-1-batched.access \
--debug_level info \
--push_buf_type singlebuf \
--value_size UniformOverSizes-4096 \
--rate 10 \
--serialization cornflakes1c-dynamic \
--server_ip 10.10.1.1 \
--our_ip 10.10.1.2 \
--time 15 \
--num_values 1 \
--num_keys 1 \
--num_threads 16 \
--num_clients 1 \
--client_id 0

```

Running the client with the mock traces generated. The mock trace has 1000 keys accessed 10000 times in a uniform fashion. 

```

sudo env LD_LIBRARY_PATH=/mnt/sdb/cornflakes/dpdk-datapath/3rdparty/dpdk/build/lib/x86_64-linux-gnu /mnt/sdb/cornflakes/target/release/ycsb_dpdk \
--config_file /mnt/sdb/cornflakes/example_config.yaml \
--mode client \
--queries /proj/demeter-PG0/vish/example_workload/egwrkc-2-batched.access \
--debug_level info \
--push_buf_type singlebuf \
--value_size UniformOverSizes-4096 \
--rate 1000 \
--serialization cornflakes1c-dynamic \
--server_ip 10.10.1.1 \
--our_ip 10.10.1.2 \
--time 15 \
--num_values 1 \
--num_keys 1 \
--num_threads 16 \
--num_clients 1 \
--client_id 0

```