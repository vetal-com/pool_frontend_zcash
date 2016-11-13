###### Description

This is ZCash mining pool.

* poolcore - common pool library for accounting and statistics handling
* pool_frontend_zcash - frontend, support native ZMQ and stratum protocols
* poolrestapi - standalone module for REST api support
* ngxrest - nginx module for REST api, fastcgi analogue

###### Устанавливаем зависимости

sudo apt-get install cmake libssl-dev libsodium-dev libpcre3-dev libleveldb-dev libboost-all-dev libgmp-dev libprotobuf-dev protobuf-compiler libjansson-dev

###### Переходим в root. Скачиваем исходники.

```
sudo su
cd
git clone https://github.com/zcash/zcash
git clone https://github.com/config4star/config4cpp
git clone https://github.com/google/flatbuffers
git clone https://github.com/eXtremal-ik7/libp2p -b version/0.3
git clone https://github.com/eXtremal-ik7/poolcore
git clone https://github.com/eXtremal-ik7/pool_frontend_zcash
git clone https://github.com/eXtremal-ik7/poolrestapi
git clone https://github.com/eXtremal-ik7/ngxrest
git clone https://github.com/eXtremal-ik7/pooljs
wget https://nginx.org/download/nginx-1.11.5.tar.gz
tar -xzf nginx-1.11.5.tar.gz
```

###### Собираем ZCash daemon, если отсутствует.

Как собрать, тут -> ZCash github page: https://github.com/zcash/zcash/wiki/1.0-User-Guide

###### Собираем необходиое для пула 
```
cd /root/config4cpp
make -j$(nproc)

cd /root/flatbuffers
mkdir build
cd build
cmake ..
make -j$(nproc)
sudo make install

cd /root/libp2p
mkdir x86_64-Linux
cd x86_64-Linux
cmake ../src
make -j$(nproc)

cd /root/poolcore
mkdir x86_64-Linux
cd x86_64-Linux
cmake ../src -DROOT_SOURCE_DIR=/root -DZCASH_ENABLED=1
make -j$(nproc)

cd /root/pool_frontend_zcash
mkdir x86_64-Linux
cd x86_64-Linux
cmake ../src -DROOT_SOURCE_DIR=/root
make -j$(nproc)

cd /root/poolrestapi
mkdir x86_64-Linux
cd x86_64-Linux
cmake ../src -DROOT_SOURCE_DIR=/root
make -j$(nproc)

cd /root/nginx-1.11.5
./configure --prefix=NGINX_INSTALL_DIRECTORY --add-module=/root/ngxrest
make -j$(nproc)
make install
```

###### Патчим ZCash daemon

```
cd /root/zcash
patch -p0 < /pool_frontend_zcash/pool.diff
```


Меняем строку в /root/zcash/src/Makefile:

```
--- src/Makefile        2016-11-08 13:36:41.033699973 +0300
+++ src/Makefile        2016-11-08 13:37:20.889665298 +0300
@@ -1380,7 +1380,7 @@
 LIBSNARK_LIBS = -lsnark
 LIBTOOL = $(SHELL) $(top_builddir)/libtool
 LIBTOOL_APP_LDFLAGS = 
-LIBZCASH_LIBS = -lsnark -lgmp -lgmpxx -lboost_system-mt -lcrypto -lsodium -fopenmp
+LIBZCASH_LIBS = -lsnark -lgmp -lgmpxx -lboost_system-mt -lcrypto -lsodium -fopenmp -L/root/poolcore/x86_64-Linux/zcash -lpoolrpczcash -L/root/libp2p/x86_64-Linux/p2p -lp2p -L/root/libp2p/x86_64-Linux/asyncio -lasyncio-0.3 -lrt
 LIPO = 
 LN_S = ln -s
 LRELEASE = 
```

Пересобираем пропатченого zcashd:
``` 
make -j$(nproc)
```

Запускаем src/zcashd -p2pport=12201. Port 12201 is used by internal pool protocol. Also, you can use poolrpccmd utility for interact with daemon by command line.

###### Настройка конфига пула
```
pool_frontend_zcash {
  isMaster = "true";
  poolFee = "1";
  poolFeeAddr = "t1ZsLopJKzyuaqmeC2Y6cx1351G13st2sWN";
  walletAddrs = ["p2p://127.0.0.1:12201"];
  localAddress = "p2p://127.0.0.1:13301";
  walletAppName = "pool_rpc";
  poolAppName = "pool_frontend_zcash";
  requiredConfirmations = "10";
  defaultMinimalPayout = "0.01";
  dbPath = "/home/xpm/pool.zcash";
  keepRoundTime = "3";
  keepStatsTime = "2";
  confirmationsCheckInterval = "7";
  payoutInterval = "30";
  balanceCheckInterval = "3";
  statisticCheckInterval = "1";

  zmqclientHost = "coinsforall.io";
  zmqclientListenPort = "6668";
  zmqclientWorkPort = "60200";

  pool_zaddr = "zcWDMjkEyRPQAFjs5RWEzoqXG9BipLRWr4S9bC4A44g7eBv6ZhnqrWWc5M5MyNn4pt9vP2PEvK9NkUwerQbewEcEWz8ZL4f";
  pool_taddr = "t1ZRsyK4pkBq7WD4gnF8r3nCYSQT19Zhh3C";
}
```

* isMaster - alltimes "true", slave mode for distribute share calculating not implemented
* poolFee - pool fee in percents
* poolFeeAddr - address for pool fee
* walletAddrs - list of zcash daemons in pool cluster. Now tested only with one daemon. Port 12201 must be taken from -p2pport command line argument of zcash daemon (see 4).
* localAddress - backend address and port. Used by poolrestapi module and poolrpccmd utility
* walletAppName - alltimer "pool_rpc"
* poolAppName - alltimes "pool_frontend_zcash"
* requiredConfirmations - minimal confirmations for block before payout (pool can't make payout for orphans)
* defaultMinimalPayout - default minimal payout for new accounts
* dbPath - path for pool database, must be exists
* keepRoundTime - how much days pool keep shares
* keepStatsTime - statistic (performance, number of GPUs, temperatures) keep time in minutes
* confirmationsCheckInterval - interval for check new mined blocks confirmations number
* payoutInterval - payout interval in minutes
* balanceCheckInterval - balance check interval in minutes
* statisticCheckInterval - statistic check interval in minutes (should be 1)

* zmqclientHost - native zeromq protocol host name
* zmqclientListenPort - navite protocol port
* zmqclientWorkPort - extra port for native protocol (pool uses port N and N+1)

* pool_zaddr - only for ZCash, pool Z-Address for receive coinbase funds. Must be created by zcash-cli z_getnewaddress
* pool_taddr - only for ZCash, address used for make payouts. Must be created by zcash-cli getnewaddress

###### Настройка модуля poolrestapi
```
poolrestapi {
  listenAddress = "cxxrestapi://127.0.0.1:19999";
  coins = ["xpm", "zcash"];
}

xpm {
  frontends = ["p2p://127.0.0.1:13300"];
  poolAppName = "pool_frontend_xpm";
}

zcash {
  frontends = ["p2p://127.0.0.1:13301"];
  poolAppName = "pool_frontend_zcash";
}
```

* listenAddress - address and port for poolrestapi. Used by nginx module for fast interaction (like fastcgi).
* coins - coin list. Name used as URL part for HTTP REST queries.

* frontends - pool frontend addresses, must be taken from 'localAddress' of pool frontend configuration file
* poolAppName - must be taken from 'poolAppName' of pool frontend configuration file

###### Настройка nginx

This is fragment of nginx.conf file:
```
    upstream api_backend {
        server     127.0.0.1:19999;
        keepalive  32;
    }

    server {
        listen       *:80 reuseport;
        server_name  pool;

        location / {
            root   html;
            index  index.html index.htm;
            expires 5s;
        }

        location /api {
            cxxrest_pass api_backend;
        }

        location ~* ^.+\.(jpg|jpeg|gif|png|ico)$ {
          expires 3d; # 3 days
        }
    }
```

* upstream api_backend:
 * server - poolrestapi address, must be taken from poolrestapi configuration file (see p.6)
  
* server/location /api:
 * cxxrest_pass - module name and link to upstream api_backend
  
Full nginx configuration file example you can find in poolrestapi repository

###### Запуск ZCash daemon
```
src/zcashd -p2pport=12201
```

После запуска можно протестировать через poolrpccmd utility (находится в папке poolcore)

```
cd /root/poolcore/x86_64-Linux/poolrpccmd
./poolrpccmd p2p://127.0.0.1:12201 getInfo
```

```
 * coin: ZEC
```

```
./poolrpccmd p2p://127.0.0.1:12201 getBlockTemplate
```

```
getBlockTemplate call duration 0.307ms
 * nbits: 486813521
 * prevhash: 0000000122760d5a6e162e08e5b7f94b9d631081bc50cafd2e9e402c405eebed
 * hashreserved: 0000000000000000000000000000000000000000000000000000000000000000
 * merkle: d803756fa90508f6e94d274236ebd158b35d79329997dbef6fa576d8830f5b58
 * time: 1478536889
 * extraNonce: 462
 * equilHashK = 9
 * equilHashN = 200
```
###### Запуск пула

```
cd /root/pool_frontend_zcash/x86_64-Linux
./pool_frontend_zcash ~/.poolcfg/zcash.cfg
```

~/.poolcfg/zcash.cfg - pool configuration file (see p. 4)

###### Запуск poolrestapi and nginx

```
cd /root/poolrestapi/x86_64-Linux
./poolrestapi ~/.poolcfg/poolrestapi.cfg
```

###### Copy web client part and launch nginx

```
cd YOUR_BUILD_DIRECTORY/pooljs/coins-for-all/webapp
cp -r * NGINX_INSTALL_DIRECTORY/html
NGINX_INSTALL_DIRECTORY/sbin/nginx
```
