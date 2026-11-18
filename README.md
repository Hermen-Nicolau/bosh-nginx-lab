# BOSH Lab – NGINX Deployment and Runtime Config Add-on

In this lab you deploy your own one-instance NGINX server with BOSH on the OMA Eng foundation. Then you give it a custom login banner with a BOSH **runtime config** (add-on) that uses the os-conf release. **Placement rules** make sure the banner lands on your deployment and nobody else's. You prove it by SSHing in and seeing your own text.

**Time:** about 45 minutes

## Before you start

- **Your names come from the instructor.** You'll receive your **deployment name**, **runtime config name** and **add-on name**, plus the **AZ**, **network** and **VM type** to use. Use them exactly as given. Unique names are what keep you from overwriting a classmate's work.
- **Access problems?** If you can't reach the jumpbox or the Ops Manager VM, contact **Benny Chacko** or **Venkat Gutta**.
- **You're working as a BOSH admin.** The Ops Manager VM logs you in to the BOSH Director with full rights over every deployment on it, TPCF included. Always pass `-d <your-deployment-name>`, and never deploy, recreate or delete a deployment that isn't yours.

## What you'll use

| Item | What it is |
| --- | --- |
| [nginx-release](https://github.com/cloudfoundry-community/nginx-release) 1.21.6 | Provides the `nginx` job. `nginx_conf` (required) is the whole nginx.conf, and `pre_start` (optional) runs before nginx starts and writes the web page |
| [os-conf-release](https://github.com/cloudfoundry/os-conf-release) 23.0.0 | Provides OS tweaks as BOSH jobs. Its `login_banner` job sets the text shown at SSH login |
| `templates/nginx.yml` | The deployment manifest template |
| `templates/runtime-config.yml` | The runtime config template |

The instructor has already uploaded both releases to the Director. Copies of the templates are in the `bootcamp` folder on the Ops Manager VM.

---

## Step 1 – Set up your workspace (5 min)

1. SSH to the jumpbox, then to the Ops Manager VM (lab access steps from the Day 1 deck), and load the BOSH environment with `source vmware/env.sh`.
2. In the `bootcamp` folder, **create your own folder**, named after your username, and work only inside it.
3. Copy both templates from the `bootcamp` folder into your folder.

```bash
cd ~/bootcamp
mkdir <your-username> && cd <your-username>
cp ../nginx.yml ../runtime-config.yml .
```

4. Look around the Director before you change anything. Find:
   - which deployments already exist
   - which stemcells are uploaded (you need `ubuntu-jammy`)
   - that the `nginx` and `os-conf` releases are there, and at which versions
   - the AZ, network and VM type names in the cloud config

> **Think about:** Which three things does every BOSH deployment need?

---

## Step 2 – Edit your NGINX manifest (8 min)

Open your copy of `nginx.yml` in an editor (`vi` or `nano`) and **replace every `<placeholder>` by hand**:

| Placeholder | Replace with |
| --- | --- |
| `<deployment-name>` | Your deployment name from the instructor |
| `<az>` | The AZ from the instructor |
| `<vm-type>` | The VM type from the instructor |
| `<network>` | The network from the instructor |
| `<your-name>` | Your name, for the web page |

For reference, here's the complete template:

```yaml
---
name: <deployment-name>

releases:
- name: nginx
  version: "1.21.6"

stemcells:
- alias: default
  os: ubuntu-jammy
  version: latest

update:
  canaries: 1
  max_in_flight: 1
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000

instance_groups:
- name: nginx
  instances: 1
  azs: [<az>]
  vm_type: <vm-type>
  stemcell: default
  networks:
  - name: <network>
  jobs:
  - name: nginx
    release: nginx
    properties:
      nginx_conf: |
        worker_processes  1;
        error_log /var/vcap/sys/log/nginx/error.log info;
        events {
          worker_connections 1024;
        }
        http {
          include /var/vcap/packages/nginx/conf/mime.types;
          default_type application/octet-stream;
          sendfile on;
          keepalive_timeout 65;
          server {
            listen 80;
            server_name _;
            access_log /var/vcap/sys/log/nginx/access.log;
            root /var/vcap/data/nginx/document_root;
            index index.html;
          }
        }
      pre_start: |
        #!/bin/bash -ex
        NGINX_DIR=/var/vcap/data/nginx/document_root
        mkdir -p $NGINX_DIR
        cat > $NGINX_DIR/index.html <<HTML
        <html><body><h1>Hello from <your-name>'s BOSH-deployed nginx</h1></body></html>
        HTML
```

**Before you deploy**, make sure you can explain what each section does: `name`, `releases`, `stemcells`, `update`, `instance_groups`, and the job's `properties`. Then double-check that no `<placeholder>` is left, and that the indentation under each `|` is unchanged. YAML is strict about spaces.

---

## Step 3 – Deploy, inspect and log in (10 min)

**Deploy** your manifest:

```bash
bosh -d <deployment-name> deploy nginx.yml
```

Read the diff BOSH shows before you confirm.

**Inspect it.** Using the BOSH CLI, find:
- your deployment in the list of deployments
- your instance, its state, and the processes running on it
- your VM's IP, AZ, VM type, and its CPU, memory and disk usage
- the manifest BOSH stored for your deployment (the intended state)
- your deploy task in the recent tasks

**Test NGINX.** From the Ops Manager VM, `curl` your VM's IP. You should get your "Hello from…" page. If the IP isn't reachable from the Ops Manager VM, test from inside the VM in the next step.

**SSH in** to your instance:

```bash
bosh -d <deployment-name> ssh nginx/0
```

**Note exactly what the login screen looks like.** You'll change it in Step 5. Then, inside the VM, become root and find:
- the nginx process as monit sees it
- the nginx job's files under `/var/vcap/jobs`
- the web page, by curling `localhost`
- your request in the nginx access log under `/var/vcap/sys/log`

> **Think about:** You never installed nginx on this VM. Where did it come from? What would BOSH do if this VM disappeared?

---

## Step 4 – Edit your runtime config (8 min)

A runtime config adds jobs to VMs without touching their deployment manifests. **Placement rules** (`include` / `exclude`) decide which VMs an add-on lands on.

Open your copy of `runtime-config.yml` and **replace every `<placeholder>` by hand**:

| Placeholder | Replace with |
| --- | --- |
| `<addon-name>` | Your add-on name from the instructor |
| `<your unique banner text>` | A banner only you would write. Keep it indented under `text: \|` |
| `<deployment-name>` | Your deployment name. This is the placement rule |

For reference, here's the complete template:

```yaml
---
releases:
- name: os-conf
  version: 23.0.0

addons:
- name: <addon-name>
  jobs:
  - name: login_banner
    release: os-conf
    properties:
      login_banner:
        text: |
          <your unique banner text>
  include:
    deployments:
    - <deployment-name>
```

Three rules:
- **Keep `version: 23.0.0`.** Never use `latest` in a runtime config. Every runtime config on the Director must agree on the os-conf version.
- **The `include` rule must name your deployment only.** Without it, your banner goes to every deployment on the Director at its next deploy, TPCF included.
- **Upload it under your own runtime config name.** Without a name, you'd replace the Director's default runtime config.

**Upload it** using your runtime config name:

```bash
bosh update-config --type runtime --name <runtime-config-name> runtime-config.yml
```

Then confirm yours is listed among the runtime configs on the Director, and that its content is what you wrote.

---

## Step 5 – Apply it and check your banner (8 min)

Uploading a runtime config changes nothing on running VMs. It's applied on the **next deploy** of each deployment it matches.

1. **Redeploy** your deployment with the same manifest as before. This time, the diff should show your add-on, the `os-conf` release and the `login_banner` job being added, even though your manifest didn't change.
2. **SSH in again.** Your banner text should appear at login. Compare it with what you noted in Step 3.
3. Inside the VM, find where the `login_banner` job now lives next to `nginx`.
4. **Prove the placement rule worked:** compare login screens with a neighbour. You should each see only your own banner.

> **Think about:** Everyone in the class uploaded a runtime config with the same `login_banner` job. Why didn't they collide? What could happen to the whole foundation if a broken runtime config were left on the Director?

---

## Step 6 – Clean up (3 min)

Remove your runtime config **first**, because it's the piece that can affect other deployments. Then delete your deployment. Check the names twice before confirming.

```bash
bosh delete-config --type runtime --name <runtime-config-name>
bosh -d <deployment-name> delete-deployment
```

Confirm neither shows up any more in the runtime configs or the deployments.

---

## When you're stuck

| What you see | Where to look |
| --- | --- |
| Deploy fails: release or version doesn't exist | Compare your manifest's release name and version with `bosh releases` |
| Deploy fails: AZ, network or VM type not found | Names must match the cloud config exactly, and they're case-sensitive |
| Deploy fails with a YAML error | Indentation, especially under `nginx_conf: \|`, `pre_start: \|` and `text: \|`. Is any `<placeholder>` left? |
| Instance fails and nginx won't start | SSH in and read the nginx logs under `/var/vcap/sys/log/nginx/` |
| `update-config` rejects your file | Check `version: 23.0.0` (not `latest`) and the YAML indentation |
| Deploy fails with an os-conf version conflict | Someone's runtime config uses another os-conf version. Tell the instructor |
| No banner after redeploying | Is the deployment name in your `include` rule exactly right? Did you redeploy after uploading? |
| Your banner shows on someone else's VM | Their `include` rule names your deployment. Tell the instructor |
| Can't reach the jumpbox or Ops Manager VM | Contact Benny Chacko or Venkat Gutta |
