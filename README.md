# systemwide jupyterhub-systemdspawner deployment

using `Jupyterhub-systemdSpawner` to make full system integration with local linux `PAM.d` userlogin authentication, thus enable linux-user directly using jupyterlab within minimum intervene

all user access to `jupyterhub` is same as SSH terminal login

reference on following project
* [conda-forge/miniforge](https://github.com/conda-forge/miniforge)
* [jupyterhub/systemdspawner](https://github.com/jupyterhub/systemdspawner)

## Prerequisites

**software**
* Ubuntu 20.04/22.04/24.04, or anything support systemd
* Miniforge3, for jupyterhub base venv

## Jupyterhub installation 

### Miniforge3 inst

as default `miniforge3` installation process

[conda-forge/miniforge](https://github.com/conda-forge/miniforge)

using `Miniforge3` for virtual python environment creation

```bash
# download and install base env
curl -LO "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
chmod +x Miniforge3-$(uname)-$(uname -m).sh
# for system-wide support
# choose a global folder, like /opt/miniforge3
bash Miniforge3-$(uname)-$(uname -m).sh
```

### Jhub-env for the target jupyterhub systemd service

create a dedicate `jupyterhub` env for further systemd-service and other stuff

* for any venv 'wish-to-display' in the belowing jupyterlab interface, please add a `nb_conda_kernels` pkg for hooking

```bash
# create jupyterhub venv env within nb_conda_kernels
mamba create -p /opt/jupyterhub/venv jupyterhub jupyterlab jupyterhub-systemdspawner nb_conda_kernels
```
### PAM.d configuration for jupyterhub

For missing `lastlog.so` in new Debian/ubuntu version, please run the following command to escape

```bash
# PAM
cd /etc/pam.d
rsync -a login jupyterhub
# comment line start with pam_lastlog.so, as new version Debian/ubuntu missing it, but jupyterhub service still forcefully try access it thus pop error 
```

### jupyterhub configuration

all necessary configuration file is 

|path|role|description|
|----|----|-----|
|/etc/jupyterhub/jupyterhub_config.py|JupyterHub Service|configuration for Jhub|
|/usr/lib/systemd/system/jupyterhub.service|systemd|Module for Jhub svc|

#### Generate jupyterhub config file

running following command, in `root` permission

* jupyterhub `etc` config

```bash
# generate jhub config then edit
install -d -m 0700 /etc/jupyterhub
/opt/jupyterhub/venv/bin/conda run -p /opt/jupyterhub/venv jupyterhub --generate-config -f /etc/jupyterhub/jupyterhub_config.py
```

make the config as given below,
this is sample configure, make you own one, though

```python
#/etc/jupyterhub/jupyterhub_config.py
c = get_config()  #noqa
c.JupyterHub.authenticator_class = 'jupyterhub.auth.PAMAuthenticator'
c.JupyterHub.hub_ip = '127.0.0.1'
c.JupyterHub.hub_port = 8081
c.JupyterHub.spawner_class = 'systemdspawner.SystemdSpawner'
c.PAMAuthenticator.allow_all = True
c.PAMAuthenticator.service = 'jupyterhub'
c.SystemdSpawner.cmd = ['/opt/miniforge3/bin/conda','run','-p','/opt/jupyterhub/venv','jupyter-labhub',"--ServerApp.root_dir=/"]
c.SystemdSpawner.default_shell = '/bin/bash'
c.SystemdSpawner.disable_user_sudo = False
```

#### Generate jupyterhub systemd file

direct write file as is given belown

* suppose `miniforge3` installed at `/opt/miniforge3` and current `jupyter` venv installed at path: `/opt/jupyterhub`,

make a `/usr/lib/systemd/system/jupyterhub.service` file, content given bas

```ini
[Unit]
Description=JupyterHub multi-user server
After=network.target

[Service]
User=root
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_AUDIT_WRITE CAP_SETGID CAP_SETUID
ExecStart=/opt/miniforge3/bin/conda run -p /opt/jupyterhub/venv \
          jupyterhub --no-ssl --config /etc/jupyterhub/jupyterhub_config.py
Restart=on-failure
User=root
# hub must be able to run systemd-run --uid=<user>
Restart=on-failure
# ulimit -n config, uncomment it if met upper cap
# LimitNOFILE=512000

[Install]
WantedBy=multi-user.target
```

### Service start & stop

Start! just as all other `systemd`

```bash
systemctl daemon-reload
systemctl start jupyterhub
```

then open any browser, access the local ip-port as

```bash
# default ip:port of jupyterhub is [server_ip]:8081, then
xdg-open http://[server_ip]:8081
```

### Per-user postprocess

before/after any user success login via `jupyterhub-spawner` panel, make these following customization for user local folder defination (like venv creation, customized repo, etc)

#### condarc - user-local-venv

for venv create and customized repo

adding following line into user home folder `~/.condarc`
suppose current username is `newbie`, his `$HOME` path is `/nethome/newbie`,

thus config file should locate at `/nethome/newbie/.condarc`

```yaml
channels:
  - conda-forge
show_channel_urls: true
auto_activate_base: false
envs_dirs:
  - /nethome/newbie/envs
```

#### condarc - nearest mirror (chn)

for CHN user, add following repo url into `~/.condarc`

```yaml
channels:
  - conda-forge
show_channel_urls: true
default_channels:
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/main
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/free
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/r
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/pro
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.sustech.edu.cn/anaconda/cloud
  msys2: https://mirrors.sustech.edu.cn/anaconda/cloud
  bioconda: https://mirrors.sustech.edu.cn/anaconda/cloud
  menpo: https://mirrors.sustech.edu.cn/anaconda/cloud
  pytorch: https://mirrors.sustech.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.sustech.edu.cn/anaconda/cloud
  nvidia: https://mirrors.sustech.edu.cn/anaconda-extra/cloud
```

#### misc - how to trigger conda env if user login directly to 'terminal'

if we process inside `JupyterHub`-`JupyterLab` interface, the default `conda`/`mamda` env is auto loaded. while user directly access server via SSH and `terminal`, we could run following command to trigger any conda/mamba installation (base env and command)

* suppose the base conda/mamba/miniforge3 is installed at `/opt/miniforge3` 

```bash
eval "$('/opt/miniforge3/bin/mamba' shell hook --shell bash --root-prefix '/opt/miniforge3')"
```
### self-hosted stuff

for support local-hosting, we need these files

* Authroized or self-assigned SSL certificate for HTTPS protocol
* a local reverse-proxy setup for access intra/extra net, e.x. `NGINX`

#### SSL related stuff

first a brief minimal concept on `modern-internet` basis of 'certificate-chain',
the concised relation ship of these ideas are like

| | authority | certificate | domain | ip address |
|---|---------|-------------|--------|------------|
|authority| verify is such claimed authority **vaild** | two type of cert, one act like a key/seal, other like a lock/signed-documents. authority use the key/seal-like cert to sign certificate, while the **seal**-like one called **root certificate**^1 | - | - |
| certificate | - | user purchase, or gain a certificate^4 or make self-signed one another certificate, thus make a **signatrue-chain**^1 | contain domain info on demand | can also contain IP info, mainly in self-signed one|
| domain | - | - | user purchase domain from **domain register**^2, hosted from provider | domain-hosted provider DNS resolve service^3, can also make a local DNS server to do resolve |
| ip | - | - | reverse resolve, if given such service | - |

1. there are two type of cert, one is full cert, which can both sign/generate another auth-cert and  sign/generate the client-cert, the limited-function client-cert, could only used to verify itself, without the inherit property. this practice of asymmetric encryption thus makes a **chain-of-trust**, for which means the trusted-root-certificate is very important (also vulnerable to, if wrongly trusted then attacking).
2. which we mainly called as **buy a domain**, actually make a **register** on **trusted authority**
3. thus each domain register must give a NS (name service) address setting on each solded `doamin` as the basic service
4. generally called **buy a SSL certificate** (from trusted authority and a long but not everlast valid time), and there're much service on free SSL certificate (like **Let's Encrypt** )within the proof of ownership of typical domain, but within a limited valid time compared with commercial vender products; some auto-script like [acme.sh](https://github.com/acmesh-official/acme.sh) may cope such a shortcome

* you may also figure out the basis of modern internet is based on everywhere **authority**, which is actually by its design and means a lot. but this is another topic.


here we just make a self-signed one. typically every modern LINUX distribution has its own verson of openssl, thus directly running following command at the `workdir`

* support you within root privilege now at `/etc` folder

```bash
# create folder
mkdir -p cert; cd cert
# create domain_san.conf file to set the generating information
# auth/req/domain_cert
cat << EOF > domain_san.conf
[req]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[dn]
C = US
ST = New York
L = Manhattan
O = DemoCommunity
OU = DemoCommunity Infrastracture
CN = infra.democommunity.domain

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = jhub.democommunity.domain
DNS.2 = 10.0.11.22
EOF
# create sign-cert, sign-request, and domain-cert
# ca key
openssl genrsa -out domain.key 2048
# cert sign request to CA, # common name
openssl req -new -key domain.key -out domain.csr -config domain_san.conf
# self-sign, 3650 days, as self-sign nearly always forget to renew -_,-
openssl x509 -req \
    -days 3650 \
    -in domain.csr \
    -signkey domain.key \
    -out domain.crt \
    -extensions req_ext \
    -extfile domain_san.conf
# verify
# openssl x509 -text -noout -in domain.crt
```

after that, you would find `domain.crt` and `domain.key`, which would be used in the below `NGINX` configuration

#### reverse proxy stuff

here we directly use `nginx`, directly install via 

```bash
apt-get install nginx
```

then make a site configure file `jhub` and put it under the directory path `/etc/nginx/sites-available`

* here we make a `server_name ... _;` in cfg , which means the `NGINX` would not first check whether the domain-name is same, and accepte all the access, from the client HTTP request.

```nginx
server {
    listen 443 ssl;
    server_name jhub.democommunity.domain _;

    ssl_certificate /etc/nginx/cert/domain.crt; # Path to your certificate
    ssl_certificate_key /etc/nginx/cert/domain.key; # Path to your private key

    client_max_body_size 10240m;

    location / {
        proxy_pass http://localhost:8000; # JupyterHub's default port
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

after make cfg, copy the generated certificate file from the `/etc/cert/` folder to the configured path in nginx, which is `/etc/nginx/cert/domain.crt` and `/etc/nginx/cert/domain.key`

* remember to check if the user:nginx have the read permission on both file, if not, please add such permission. you can achieve it by directly running `chown`, or make a `acl` wise if `filesystem` support it

```bash
# copy from /etc/ to /etc/nginx
rsync -aviPAXHx /etc/domain.{crt,key} /etc/nginx/cert/
# change ownership
# chown -R nginx /etc/nginx/cert
# or acl cmd
setfacl -m u:nginx:r /etc/nginx/cert/domain.crt
setfacl -m u:nginx:r /etc/nginx/cert/domain.key
```

after that, make a `symlink` to jhub config in `/etc/nginx/sites-enable`, and then restart / reload nginx to make config effect

```bash
# make sure the order is [actual-file] [symbolic-linker]
ln -s /etc/nginx/sites-available/jhub /etc/nginx/sites-enable/jhub
# if nginx not running, 
# systemctl restart nginx
# check if all config is ok
nginx -t
# reload on-site
nginx -s reload
```