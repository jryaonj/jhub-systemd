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
