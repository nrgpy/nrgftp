# ![NRGPy](https://www.gravatar.com/avatar/6282094b092c756acc9f7552b164edfe?s=24) nrgpy FTP server

a Docker-based FTP server for accumulating files from a LOGR-S Solar Data Logger. this tool
is a derivative of the original created by Andrew Stilliard (see [credits](#credits) for more info).

__note this requires that Docker https://docs.docker.com/get-docker/ 
and GIT https://git-scm.com/download are installed on your computer__

## build it

this process will take about 15 minutes

__note that you will not need to add `sudo` before all docker commands if you're running windows__

``` bash
git clone https://github.com/nrgpy/nrgftp
cd nrgftp
sudo docker build . -t nrgpy/nrgftp
```

## run it

the following creates a Docker container for FTP access for a user/password = nrguser/s0l@r1sgre@t

it also creates two volumes for user info and data, so persistences of these is in place if the
container restarts for any reason

please adjust for your usage

``` bash
sudo docker volume create --name nrgpy-ftp-db-volume && \
sudo docker volume create --name nrgpy-ftp-file-volume && \
sudo docker run --rm -d --name nrgpy_ftp_server -p 21:21 -p 30000-30049:30000-30049 -e FTP_USER_NAME=nrguser -e FTP_USER_PASS=s0l@r1sgre@t! -e FTP_USER_HOME=/home/ftpusers/nrguser -v nrgpy-ftp-db-volume:/etc/pure-ftpd/passwd -v nrgpy-ftp-file-volume:/home/ftpusers nrgpy/nrgftp bash /run.sh -c 30 -C 10 -l puredb:/etc/pure-ftpd/pureftpd.pdb -E -j -R -P localhost -p 30000:30059 
```

You can also pass ADDED_FLAGS as an env variable to add additional options such as --tls to the pure-ftpd command.  
e.g. ` -e "ADDED_FLAGS=--tls=2" `

## FTP access via client

### LOGR-S setup

... here goes LOGR-S settings

### filezilla

- host: localhost (or IP/server name if accessing remotely)
- username: nrguser
- password: s0l@r1sgre@t!
- port: 21

### terminal from the host machine

```bash
ftp -p localhost 21
```

## access to server via terminal

``` bash
sudo docker exec -it ftpd_server /bin/bash
```

### creating a runtime FTP user

To create a user on the ftp container, use the following environment variables: `FTP_USER_NAME`, `FTP_USER_PASS` and `FTP_USER_HOME`.

`FTP_USER_HOME` is the root directory of the new user.

Example usage:

``` bash
sudo docker run -e FTP_USER_NAME=bob -e FTP_USER_PASS=12345 -e FTP_USER_HOME=/home/bob nrgpy/nrgftp
```

If you wish to set the `UID` & `GID` of the FTP user, use the `FTP_USER_UID` & `FTP_USER_GID` environment variables.

## using different passive ports
------------------------------

To use passive ports in a different range (*eg*: `10000-10009`), use the following setup:

``` bash
sudo docker run -e FTP_PASSIVE_PORTS=10000:10009 --expose=10000-10009 -p 21:21 -p 10000-10009:10000-10009
```

You may need the `--expose=` option, because default passive ports exposed are `30000` to `30009`.

### example usage once inside

create an ftp user: `e.g. bob with chroot access only to /home/ftpusers/bob`

```bash
pure-pw useradd bob -f /etc/pure-ftpd/passwd/pureftpd.passwd -m -u ftpuser -d /home/ftpusers/bob
```

*no restart should be needed.*

*if you have any trouble with volume permissions due to the **uid** or **gid** of the created user you can change the **-u** flag for the uid you would like to use and/or specify **-g** with the group id as well. For more information see issue [#35](https://github.com/stilliard/docker-pure-ftpd/issues/35#issuecomment-325583705).*

more info on usage here: https://download.pureftpd.org/pure-ftpd/doc/README.Virtual-Users

## max clients

by default we set 20 max clients at once, but you can increase this by using the following environment variable `FTP_MAX_CLIENTS`, e.g. to `FTP_MAX_CLIENTS=50` (or by editing the `run.sh` file) and then also increasing the number of public ports opened from `FTP_PASSIVE_PORTS=30000:30009` `FTP_PASSIVE_PORTS=30000:30099`. You'll also want to open those ports when running docker run.
In addition you can specify the maximum connections per ip by setting the environment variable `FTP_MAX_CONNECTIONS`. By default the value is 20.

## all Pure-ftpd flags available:

https://linux.die.net/man/8/pure-ftpd

## logs

To get verbose logs add the following to your `docker run` command:

``` bash
-e "ADDED_FLAGS=-d -d"
```

then the logs will be redirected to the stdout of the container and captured by the docker log collector.
You can watch them with `docker logs -f ftpd_server`

or, if you exec into the container you could watch over the log with `tail -f /var/log/messages`

want a transfer log file? add the following to your `docker run` command:

```bash
-e "ADDED_FLAGS=-O w3c:/var/log/pure-ftpd/transfer.log"
```

## default pure-ftpd options explained

``` bash
/usr/sbin/pure-ftpd # path to pure-ftpd executable
-c 5 # --maxclientsnumber (no more than 5 people at once)
-C 5 # --maxclientsperip (no more than 5 requests from the same ip)
-l puredb:/etc/pure-ftpd/pureftpd.pdb # --login (login file for virtual users)
-E # --noanonymous (only real users)
-j # --createhomedir (auto create home directory if it doesnt already exist)
-R # --nochmod (prevent usage of the CHMOD command)
-P $PUBLICHOST # IP/Host setting for PASV support, passed in your the PUBLICHOST env var
-p 30000:30009 # PASV port range (10 ports for 5 max clients)
-tls 1 # Enables optional TLS support
```

for more information please see `man pure-ftpd`, or visit: https://www.pureftpd.org/

## why so many ports opened?

This is for PASV support, please see: [#5 PASV not fun :)](https://github.com/stilliard/docker-pure-ftpd/issues/5)

## changing a password

e.g. to change the password for user "bob":

``` bash
pure-pw passwd bob -f /etc/pure-ftpd/passwd/pureftpd.passwd -m
```

## TLS

if you want to enable tls (for ftps connections), you need to have a valid
certificate. You can get one from one of the certificate authorities that you'll
find when googling this topic. The certificate (containing private key and
certificate) needs to be at:

``` bash
/etc/ssl/private/pure-ftpd.pem
```

use docker volumes to get the certificate there at runtime. The container will
automatically enable optional TLS when it detect the file at this location.

you can also self-sign a certificate, which is certainly the easiest way to
start out. Self signed certificates come with certain drawbacks, but it might
be better to have a self signed one than none at all.

here's how to create a self-signed certificate from within the container:

```bash
mkdir -p /etc/ssl/private
openssl dhparam -out /etc/ssl/private/pure-ftpd-dhparams.pem 2048
openssl req -x509 -nodes -newkey rsa:2048 -sha256 -keyout \
    /etc/ssl/private/pure-ftpd.pem \
    -out /etc/ssl/private/pure-ftpd.pem
chmod 600 /etc/ssl/private/*.pem
```

### automatic TLS certificate generation

if `ADDED_FLAGS` contains `--tls` (e.g. --tls=1 or --tls=2) and file `/etc/ssl/private/pure-ftpd.pem` does not exists
it is possible to generate self-signed certificate if `TLS_CN`, `TLS_ORG` and `TLS_C` are set.

keep in mind that if no volume is set for `/etc/ssl/private/` directory generated
certificates won't be persisted and new ones will be generated on each start.

you can also pass `-e "TLS_USE_DSAPRAM=true"` for faster generated certificates
though this option is not recommended for production.

please check out the [TLS docs here](https://download.pureftpd.org/pub/pure-ftpd/doc/README.TLS).

### TLS with cert and key file for Let's Encrypt

Let's Encrypt provides two separate files for certificate and keyfile. The [Pure-FTPd TLS encryption](https://download.pureftpd.org/pub/pure-ftpd/doc/README.TLS) documentation suggests to simply concat them into one file. 
So you can simply provide the Let's Encrypt cert ``/etc/ssl/private/pure-ftpd-cert.pem`` and key ``/etc/ssl/private/pure-ftpd-key.pem`` via Docker Volumes and let them get auto-concatenated into ``/etc/ssl/private/pure-ftpd.pem``.

or concat them manually with

``` bash
cat /etc/letsencrypt/live/<your_server>/cert.pem /etc/letsencrypt/live/<your_server>/privkey.pem > pure-ftpd.pem
```

## credits

very special thanks to Andrew Stilliard for doing all the hard work for this tool. please visit his site for more information, and to give him a high five and buy him some coffee!!

[![Andrew's docker-pure-ftpd Github][https://github.com/stilliard/docker-pure-ftpd]

## license

MIT licence
