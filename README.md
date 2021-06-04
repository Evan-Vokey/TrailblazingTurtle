
# Userportal
This portal is intended to give users a view of their cluster use, including job level performance.

## UI

Each user can see their current uses on the cluster and a few hours in the past. All stats on the jobs in the past week are available. The users can also see the aggregated use of the users in the same group.

<a href="docs/user.png"><img src="docs/user.png" alt="Stats per user" width="100"/></a>
<a href="docs/job.png"><img src="docs/job.png" alt="Stats per job" width="100"/></a>
<a href="docs/accountstats.png"><img src="docs/accountstats.png" alt="Stats per account" width="100"/></a>

### Storage
Each user can see their current storage allocations and who within their group are using the group quota.

![HSM](docs/quota.png)

Info about HSM state (Tape) are also available.

![HSM](docs/hsm.png)

### Top users
These pages are only available to staff and are meant to visualize poor cluster utilization:

* Largest compute users, CPU cores and GPUs
* Jobs on large memory nodes
* Top users on Lustre

<a href="docs/top_compute.png"><img src="docs/top_compute.png" alt="Top compute user (CPU)" width="100"/></a>
<a href="docs/top_compute_gpu.png"><img src="docs/top_compute_gpu.png" alt="Top compute user(GPU)" width="100"/></a>
<a href="docs/top_largemem.png"><img src="docs/top_largemem.png" alt="Jobs on large memory nodes" width="100"/></a>
<a href="docs/top_lustre.png"><img src="docs/top_lustre.png" alt="Top users on Lustre" width="100"/></a>

## Design
Jobs and filesystems metrics are stored in Prometheus, 3 exporters are essentials to get this data:

### slurm-job-exporter
[slurm-job-exporter](https://github.com/guilbaults/slurm-job-exporter) is used to capture informations from cgroup managed by slurm on each compute nodes. This gather CPU, memory and GPU utilization.

### lustre\_exporter
[lustre\_exporter](https://github.com/HewlettPackard/lustre_exporter) capture information on Lustre MDS and OSS but will only use \$SLURM\_JOBID as a tag on the metrics.

### lustre\_exporter\_slurm
[lustre\_exporter\_slurm](https://github.com/guilbaults/lustre_exporter_slurm) is used as a proxy between Prometheus and lustre_exporter to improve the content of the tags. This will match the \$SLURM\_JOBID to a job in slurm and will add the username and slurm account in the tags. 

### Integration
The Django portal will also access various MySQL databases to gather some informations. Timeseries are stored in Prometheus for better performance. 

![Architecture diagram](docs/userportal.png)

## Test environment
To test localy without having to use the SSO:

`REMOTE_USER=sigui4@computecanada.ca affiliation=staff@computecanada.ca python manage.py runserver`

## Production

RPMs required for production

* `shibboleth`
* `python36-virtualenv` 
* `python3-mod_wsgi`
* `openldap-devel`
* `gcc`
* `mariadb-devel`

A file in the python env need to be patched, check the diff in `ldapdb.patch`

Static files are handled by Apache and need to be collected since python will not serve them:

```
python manage.py collectstatic
```

## API
An API is available to modify resources in the database. A local superuser need to be created:

```
python manage.py createsuperuser
```

The token can be created with:

```
manage.py drf_create_token
```

## Slurm prolog script
The script `slurm_prolog/slurm_prolog_userportal.py` can be used to add the submited script to the database of the portal. This should run as a slurm prolog script. This script use the API to push to job script and will need a user/token.

## Upgrades
When SQL models are modified, the automated migration script need to run once:
```
python manage.py migrate
```

### Apache virtualhost config

```
<VirtualHost *:443>
  ServerName userportal.int.ets1.calculquebec.ca

  ## Vhost docroot
  DocumentRoot "/var/www/userportal"
  ## Alias declarations for resources outside the DocumentRoot
  Alias /static "/var/www/userportal-static"
  Alias /favicon.ico "/var/www/userportal-static/favicon.ico"

  ## Directories, there should at least be a declaration for /var/www/userportal

  <Directory "/var/www/userportal/">
    AllowOverride None
    Require all granted
  </Directory>

  <Directory "/var/www/userportal/static">
    AllowOverride None
    Require all granted
  </Directory>

  <Location "/secure">
    Require shib-session
    AuthType shibboleth
    ShibRequestSetting requireSession 1
  </Location>

  <Location "/Shibboleth.sso">
    Require all granted
    SetHandler shib
  </Location>

  ## Logging
  ErrorLog "/var/log/httpd/userportal.int.ets1.calculquebec.ca_error_ssl.log"
  ServerSignature Off
  CustomLog "/var/log/httpd/userportal.int.ets1.calculquebec.ca_access_ssl.log" combined

  ## SSL directives
  SSLEngine on
  SSLCertificateFile      "/etc/pki/tls/certs/userportal.int.ets1.calculquebec.ca.crt"
  SSLCertificateKeyFile   "/etc/pki/tls/private/userportal.int.ets1.calculquebec.ca.key"
  WSGIApplicationGroup %{GLOBAL}
  WSGIDaemonProcess userportal python-home=/var/www/userportal-env python-path=/var/www/userportal/
  WSGIProcessGroup userportal
  WSGIScriptAlias / "/var/www/userportal/userportal/wsgi.py"
  ## Shibboleth
</VirtualHost>
```

## Translation
Create the .po files for french: `python manage.py makemessages -l fr`

Update all message files for all languages: `python manage.py makemessages -a`

Compile messages: `python manage.py compilemessages`

