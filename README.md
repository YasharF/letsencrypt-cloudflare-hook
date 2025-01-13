# CloudFlare hook for dehydrated

This is a hook for the [Let's Encrypt](https://letsencrypt.org/) ACME client [dehydrated](https://github.com/dehydrated-io/dehydrated) that allows you to use [CloudFlare](https://www.cloudflare.com/) DNS records to respond to `dns-01` challenges. This script requires Python and as well as your CloudFlare account e-mail and API key (as environment variables).

## Table of Contents
- [System Prerequisites](#system-prerequisites)
- [Installation](#installation-and-usage-without-a-python-virtual-environment)
- [Configuration](#configuration)
- [Usage](#usage)
- [Using Virtual Environments and Scripting](#installation-with-python-virtual-envs-and-bash-script-for-quick-re-runs)
- [Testing](#testing)

## System Prerequisites

- Python 3.x
- pip
- git

### WSL, Ubuntu, and potentially Debian prerequisites
You may need to install the following two packages in addition to Python 3 if you run into issues during the Installation:

```
sudo apt install python3-pip python-is-python3
```


## Installation and usage without a Python virtual environment
It is highly recommanded to use Python Virtual Environmnets instead of the installation and usage steps in this section.  See [Installation with Python virtual envs and bash script for quick re-runs](#installation-with-python-virtual-envs-and-bash-script-for-quick-re-runs) for more information.

Clone the code repo:
```
$ cd ~
$ git clone https://github.com/dehydrated-io/dehydrated
$ cd dehydrated
$ mkdir hooks
$ git clone https://github.com/SeattleDevs/letsencrypt-cloudflare-hook hooks/cloudflare
```

Install required python packages:
```
$ pip3 install -r hooks/cloudflare/requirements.txt
```


### Configuration

An API token from your Cloudflare account is required. If you are currently using an email and your Cloudflare account's global API Key for authentication, please migrate to using an API token. API tokens are more secure as they follow the principles of least privilege and reduce the potential impact of malicious actors if your account key is compromised. To create an API token for this hook, go to https://dash.cloudflare.com/profile/api-tokens (API token page under your Cloudflare "My Profile"). Then, click on "Create API Token" to start.

-  Type a name for your API token for easy identification, such as "Dehydrated Let's Encrypt Hook"
-  Add Read permission for Zone -> Zone
-  Add Edit permission for Zone -> DNS
-  If you have multiple domains in your account, you can use Zone inclusion or exclusion to limit the API token's access to them
-  It is highly recommended to limit the API access to your IP address or subnet using the Client IP Address Filtering
-  Once you have completed the above steps, press the "Continue to summary" button and then "Create Token."

Your account's CloudFlare email and API key are expected to be in the environment, you can set the enviornment variable if you need to in bash using:

```
$ export CF_API_TOKEN='zzle8imobfxpg50sdb3c'
```

You can supply multiple API keys by separating them with one or more spaces. API keys will be tried in the order given, until one is found that has permissions for the relevant domain.
Leading, trailing, and extra spaces are ignored, so you can vertically align credential pairs for easy reading:

```
$ export CF_API_TOKEN='zzle8imobfxpg50sdb3c txjo6e779dxhpa8yofft'
```

Optionally, you can specify the DNS servers to be used for propagation checking via the `CF_DNS_SERVERS` environment variable. The following are the IPs for the Cloudflare DNS Servers in case if you would like to use them instead of your client's default DNS configs:

```
$ export CF_DNS_SERVERS='1.1.1.1 1.0.0.1'
```

If you experience problems with DNS propagation, increasing the time (in seconds) this hook waits for things to settle down after setting the DNS records, may help. The default is 10.

```
$ export CF_SETTLE_TIME='30'
```

If you want more information about what is going on while the hook is running:

```
$ export CF_DEBUG='true'
```

Alternatively, these statements can be placed in `dehydrated/config`, which is automatically sourced by `dehydrated` on startup:

```
echo "export CF_API_TOKEN=zzle8imobfxpg50sdb3c" >> config
echo "export CF_DEBUG=true" >> config
```


### Usage

```
$ ./dehydrated -c -d example.com -t dns-01 -k 'hooks/cloudflare/hook.py'
#
# !! WARNING !! No main config file found, using default config!
#
Processing example.com
 + Signing domains...
 + Creating new directory /home/user/dehydrated/certs/example.com ...
 + Generating private key...
 + Generating signing request...
 + Requesting challenge for example.com...
 + CloudFlare hook executing: deploy_challenge
 + DNS not propagated, waiting 30s...
 + DNS not propagated, waiting 30s...
 + Responding to challenge for example.com...
 + CloudFlare hook executing: clean_challenge
 + Challenge is valid!
 + Requesting certificate...
 + Checking certificate...
 + Done!
 + Creating fullchain.pem...
 + CloudFlare hook executing: deploy_cert
 + ssl_certificate: /home/user/dehydrated/certs/example.com/fullchain.pem
 + ssl_certificate_key: /home/user/dehydrated/certs/example.com/privkey.pem
 + Done!
```


## Installation with Python virtual envs and bash script for quick re-runs
The following will install dehydrated in `cert_workspace` folder under your home path.  The last line sets up a Python virtual env named dehydrated_env.

```
$ cd ~
$ mkdir cert_workspace
$ cd cert_workspace
$ git clone https://github.com/dehydrated-io/dehydrated
$ cd dehydrated
$ mkdir hooks
$ git clone https://github.com/SeattleDevs/letsencrypt-cloudflare-hook hooks/cloudflare
$ python3 -m venv dehydrated_env
```

### Initialize the python environment
Activate the virtual env and install dependencies:

```
source dehydrated_env/bin/activate 
$ (dehydrated_env) pip3 install -r hooks/cloudflare/requirements.txt
```

### Usage with a bash script
You can take a shortcut by creating a bash script (such as `domaincert.sh` in `~/cert_workspace`) as following to generate your certificate quickly since you need to regenerate your certificates once every 90 days. Replace CF_EMAIL (your Cloudflare email), CF_KEY (your Cloudflare API Key), and DOMAIN variables with your own info. The following assumes that the bash script is stored where you created the git clone folder for dehydrated to reduce the chances of accidentally checking in the bash script into a git repo because it is a good security practice not to store credentials in git repos.

```
#!/bin/bash

export CF_API_TOKEN='zzle8imobfxpg50sdb3c'
export DOMAIN='my.domain.com'

export CF_DNS_SERVERS='1.1.1.1 1.0.0.1'
# export CF_DEBUG='true'

dehydrated/dehydrated -c -d $DOMAIN  -t dns-01 -k 'dehydrated/hooks/cloudflare/hook.py'

cp dehydrated/certs/$DOMAIN/privkey.pem $DOMAIN.letsencrypt.key
cp dehydrated/certs/$DOMAIN/fullchain.pem $DOMAIN.letsencrypt.crt

exit 0
```

Assuming that you created a bash script as `~/cert_workspace/domaincert.sh` , you can run it with the following. Note that the first time you run dehydrated, it may prompt you to accept its terms.  Please read and follow with any such instructions that it may provide, and then re-run your `domaincert.sh` script to generate the certificate.

```
$ (dehydrated_env) cd ~/cert_workspace
$ (dehydrated_env) ./domaincert.sh
```


### Re-run (every 90 days or 8x days)
This is what you need to re-run before the 90 day expiration of your certificate to generate a new certificate assuming that you set up your installation as above.

```
$ cd ~/cert_workspace/dehydrated
$ source dehydrated_env/bin/activate
$ (dehydrated_env) cd ~/cert_workspace
$ (dehydrated_env) ./domaincert.sh
```

### Update and Re-run
If you want to update the scripts to the latest version (i.e. because Python, Let's Encrypt or Cloudflare changed their APIs causing errors when you re-run the script, or if there is a security update), run the following:

```
$ cd ~/cert_workspace/dehydrated
$ git pull
$ cd hooks/cloudflare
$ git pull
$ source dehydrated_env/bin/activate
$ (dehydrated_env) pip3 install -r hooks/cloudflare/requirements.txt
```

and then re-generate your certifcate as before:

```
$ (dehydrated_env) cd ~/cert_workspace
$ (dehydrated_env) ./domaincert.sh
```


## Testing
### Unit Tests
If you are making changes to the code, run the unit tests with `tox` to make sure your changes aren't breaking the hook.

```
$ (dehydrated_env) cd hooks/cloudflare
$ (dehydrated_env) pip install tox
$ (dehydrated_env) tox
```

### Integration Testing
If you are making changes to the code, you can do a full test with real calls through dehydrated and Let's Encrypt's test servers. The test servers won't issue a valid certificate, but have a higher rate limit which allows you to test your hook changes without using up your production quota.

1. If you haven't used Let's Encrypt's test server before, you will need to accept its terms of service. Assuming you are aware of the terms and would like to accept them, you can do so using the following command:
```
$ ./dehydrated --register --accept-terms --ca letsencrypt-test
```

2. You need to add `--ca letsencrypt-test --force --force-validation` to the dehydrated parameters when calling it. This ensures the use of the test servers, tells dehydrated to ignore the 30-day protection limit, and skips previously cached domain verifications with Cloudflare so the verification process is re-run.
```
$ ./dehydrated -c --ca letsencrypt-test --force --force-validation -d example.com -t dns-01 -k 'hooks/cloudflare/hook.py'
```

## Further reading
If you want some prose to go with the code, check out the relevant blog post here: [From StartSSL to Let's Encrypt, using CloudFlare DNS](http://kappataumu.com/articles/letsencrypt-cloudflare-dns-01-hook.html).
