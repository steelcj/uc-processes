---
title: Set up a Production-Ready Droplet
description: Create a new Droplet with our recommended configuration for improved security, reliability, and monitoring.
product: Droplets
url: https://docs.digitalocean.com/products/droplets/getting-started/recommended-droplet-setup/
last_updated: "2026-08-07"
---

> **For AI agents:** The documentation index is at [https://docs.digitalocean.com/llms.txt](https://docs.digitalocean.com/llms.txt). Markdown versions of pages use the same URL with `index.html.md` in place of the HTML page (for example, append `index.html.md` to the directory path instead of opening the HTML document).

# Set up a Production-Ready Droplet

DigitalOcean Droplets are Linux-based virtual machines (VMs) that run on top of virtualized hardware. Each Droplet you create is a new server you can use, either standalone or as part of a larger, cloud-based infrastructure.

When you first create a Droplet, we recommend configuring it for security and usability in a way that makes scaling and integration with other products simpler in the future. Our recommended setup for an Ubuntu Droplet has the following:

- **Improved security**: SSH key authentication for a sudo non-`root` user, no password-based access to `root`, and a cloud firewall to restrict access to SSH only.
- **Reliability and usability**: Automatic backups to prevent data loss in emergencies, and networking features like VPC and IPv6 support with no manual configuration.
- **Capacity and scaling information**: The DigitalOcean metrics agent to understand your resource usage and make more informed decisions on when and how to scale.

After you set up one Droplet with our recommended setup, configuring subsequent Droplets with the same setup only requires selecting options on the [Droplet creation page](https://cloud.digitalocean.com/droplets/new).

You can use Droplets with this setup to host a website, scale out from a single Droplet to multiple Droplets with a [load balancer](https://docs.digitalocean.com/products/networking/load-balancers/index.html.md), or add [object storage](https://docs.digitalocean.com/products/spaces/index.html.md) to serve assets.

## Before You Start

Choose whether you want to use [the DigitalOcean Control Panel](https://cloud.digitalocean.com) in a browser or [`doctl`](https://docs.digitalocean.com/reference/doctl/index.html.md), the DigitalOcean command-line interface, from a terminal.

The Control Panel visually guides you through creation and configuration and lets you get started without setting up additional tools. `doctl` lets you work from the command line.

### Using the browser-based Control Panel

If you don’t already have a DigitalOcean account, sign up now and log in to the [Control Panel](https://cloud.digitalocean.com).

## Step 1: Create and Upload SSH Keys

Our recommended setup uses SSH keys for authentication when logging into Droplets because password-based authentication is less secure. After you upload your SSH public key to your DigitalOcean account, you can add it automatically to any new Droplets you create, which avoids manually adding or configuring them.

### How do I do this?

If you don’t have an SSH key pair, create one using OpenSSH, which is included on Linux, macOS, and Windows Subsystem for Linux:

```bash
ssh-keygen
```

Your key pair is saved in the location prompted, which by default is `~/.ssh/` on Linux and `/Users/your_username/.ssh` on Windows and macOS. Copy the contents of your public key, which is named `id_ed25519.pub` by default. On macOS, you can copy the key directly to your clipboard by running the following command:

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

The Windows and Linux versions of the command depend on your specific distribution, subsystem, or command-line shell.

In the left menu of the Control Panel, click **Settings**, then click the **Security** tab at the top of the page to go to the [team security settings page](https://cloud.digitalocean.com/account/security). The **SSH keys** section lists any keys already added to the team.

![Security tab showing a list of SSH keys with their names and fingerprints.](https://docs.digitalocean.com/screenshots/teams/ssh-keys.903f80218faa4686b6bf6565b1d8ad2d67f89b22834d8577c66b2ecb1f52b9b8.png)

Click **Add SSH Key** to open the **New SSH key** window.

Copy your public key into the **Public Key** field. Enter a name in the **Key Name** field, which you use identify this key in the DigitalOcean Control Panel. We recommend using the name of the machine you copied the public key from.

## Get more detail on creating and uploading SSH keys.

The following articles have more detailed explanations of this step:

[How to Create SSH Keys with OpenSSH on macOS or Linux](https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/create-with-openssh/index.html.md): Use OpenSSH to create new SSH keys on macOS, Linux, or Windows Subsystem for Linux.

[How to Create SSH Keys with PuTTY on Windows](https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/create-with-putty/index.html.md): Use PuTTY to create SSH keys on Windows systems without Bash.

[How to Manage SSH Public Keys on DigitalOcean Teams](https://docs.digitalocean.com/platform/teams/how-to/upload-ssh-keys/index.html.md): Add public SSH keys to a DigitalOcean team to be able to automatically configure SSH key authentication during Droplets creation.

## Step 2: Create and Configure the Droplet

Our recommended setup for Droplets includes enabling several features: VPC (private networking), IPv6, monitoring, and backups.

- [VPC](https://docs.digitalocean.com/products/networking/vpc/index.html.md) creates a private network interface accessible only by resources within the same account or team. It’s free and increases security and decreases bandwidth costs for resources that communicate using it. Enabling it later requires manual network configuration and rebooting the Droplet.
- [IPv6](https://docs.digitalocean.com/products/networking/ipv6/index.html.md) enables an additional 16 IP addresses for the Droplet. It’s free and enabling it later requires manual network configuration and rebooting the Droplet.
- [Monitoring](https://docs.digitalocean.com/products/monitoring/index.html.md) is a metrics visualization service that adds [additional graphs](https://docs.digitalocean.com/products/droplets/how-to/track-performance/index.html.md) to the Control Panel (like CPU load, RAM usage, and disk usage) and the ability to set up [alert policies](https://docs.digitalocean.com/products/monitoring/how-to/manage-alerts/index.html.md#create-control). It’s free and enabling it from the start avoids manual setup and lets you understand your resource usage to make more informed decisions on when and how to scale.
- [Backups](https://docs.digitalocean.com/products/backups/index.html.md) are automatic, system-level disk images of Droplets taken weekly, daily, or multiple times a day. Backups give you a way to revert a Droplet to an older state or recreate Droplets, protecting you against data loss.

Our setup also uses [user data](https://docs.digitalocean.com/products/droplets/how-to/provide-user-data/index.html.md), which is data that cloud-init consumes during the Droplet’s first boot to perform tasks or run scripts. The user data script in this tutorial implements two security measures:

- Disables password-based login to the Droplet, making it accessible with SSH keys only.
- Creates a sudo non-root user for day-to-day use. The root user has broad privileges that you don’t need for many tasks. Using a sudo non-root user decreases the risk of making destructive changes by accident and still lets you escalate privileges when necessary.

### How do I do this?

From the Control Panel, click **Create** at the top to open the create menu, then click **Droplet** to open [the Droplet create page](https://cloud.digitalocean.com/droplets/new). Configure the new Droplet with the following options:

1. In **Choose an image**, under the **OS** tab, choose a current LTS version of Ubuntu, such as Ubuntu 24.04.
2. In **VPC Network**, choose the default VPC.
3. In **Networking**, check **Enable IPv6**. In **Monitoring**, leave **Improved Metrics and monitoring** enabled.
4. In **Additional Options**, enable **Startup scripts**. In the text box that opens, copy and paste the following `cloud-config` script. Customize the emphasized line to set the username.

User data `cloud-config` script

```bash
#!/bin/bash
set -euo pipefail

USERNAME=sammy # TODO: Customize the sudo non-root username here

# Create user and immediately expire password to force a change on login
useradd --create-home --shell "/bin/bash" --groups sudo "${USERNAME}"
passwd --delete "${USERNAME}"
chage --lastday 0 "${USERNAME}"

# Create SSH directory for sudo user and move keys over
home_directory="$(eval echo ~${USERNAME})"
mkdir --parents "${home_directory}/.ssh"
cp /root/.ssh/authorized_keys "${home_directory}/.ssh"
chmod 0700 "${home_directory}/.ssh"
chmod 0600 "${home_directory}/.ssh/authorized_keys"
chown --recursive "${USERNAME}":"${USERNAME}" "${home_directory}/.ssh"

# Disable root SSH login with password
sed --in-place 's/^PermitRootLogin.*/PermitRootLogin prohibit-password/g' /etc/ssh/sshd_config
if sshd -t -q; then systemctl restart sshd fi
```

5. In **Choose Authentication Method**, select **SSH keys**, and choose one or more keys. These keys give you access to the root user, and the user data script adds these keys to the sudo non-root user and disable password authentication.
6. In **Tags**, create a tag that matches what you’re using the Droplet for, like `webserver`. You need to use this tag to apply cloud firewalls in the next step.
7. In the **Enable Backups** section, check **Enable automated backups**.

Once you’ve selected all of the options, click **Create Droplet**.

## Get more detail on creating Droplets.

The following articles have more detailed explanations of this step:

[How to Create a Droplet](https://docs.digitalocean.com/products/droplets/how-to/create/index.html.md): Create Droplets and customize the image, plan, authentication method, and quantity of Droplets you want.

## Step 3: Create a Cloud Firewall

Firewalls place a barrier between your servers and other machines on the network to protect them from external attacks. [DigitalOcean Cloud Firewalls](https://docs.digitalocean.com/products/networking/firewalls/index.html.md) are a free, stateful firewall service for Droplets. They block all traffic that isn’t expressly permitted by a rule.

You can apply cloud firewalls to individual Droplets by name or to one or more Droplets by [tag](https://docs.digitalocean.com/products/droplets/how-to/tag/index.html.md). Our setup uses tags. When you add a tag to a cloud firewall, any Droplets with that tag are automatically included in the firewall configuration, including new Droplets that you tag during creation.

To start, we recommend the following default firewall rules:

- Restrict all inbound traffic except for SSH connections to the Droplet on port 22.
- Allow all outbound traffic to any destination on any port. Many fundamental services rely on outbound communication, and these defaults make it easier to set up a new Droplet without introducing restrictions that could cause expected problems.

In the long term, we recommend [organizing firewalls by role](https://docs.digitalocean.com/products/networking/firewalls/concepts/organization/index.html.md), so you can create custom firewalls for your specific use case.

### How do I do this?

From the Control Panel, click **Create** at the top to open the create menu, then click **Firewall** to open [the firewall create page](https://cloud.digitalocean.com/networking/firewalls/new). Configure the cloud firewall with the following options:

1. In **Name**, enter `inbound-ssh-only`.
2. In **Inbound Rules**, leave the single default rule for **SSH**.
3. In **Outbound Rules**, keep the default rules, which permit all traffic to any destination on any port.
4. In **Apply to Droplets**, add the tag you created with the new Droplet. When you create additional Droplets, adding the same tag to them automatically adds them to this cloud firewall as well, simplifying scaling in the future.

Once you’ve selected all of the options, click **Create Firewall**.

## Get more detail on firewall creation and rules.

The following articles have more detailed explanations of this step:

[How to Create Firewalls](https://docs.digitalocean.com/products/networking/firewalls/how-to/create/index.html.md): Create a cloud firewall to restrict network traffic to and from specified Droplets.

[How to Configure Firewall Rules](https://docs.digitalocean.com/products/networking/firewalls/how-to/configure-rules/index.html.md): Create, modify, or delete firewall rules to restrict Droplets’ inbound and outbound traffic based on ports and sources.

[How to Add and Remove Droplets from Firewalls](https://docs.digitalocean.com/products/networking/firewalls/how-to/manage-droplets/index.html.md): Add Droplets to a firewall by name or by tag to apply the firewall’s rules.

## Summary

After you set up one Droplet with our recommended setup, setting up future ones is simpler because you don’t need to repeat most of the steps. You only need to complete these steps once:

- Creating an SSH key pair.
- Uploading your public key to your DigitalOcean account.
- Creating the cloud firewall.

To create additional Droplets with the same setup, the only step is choosing its configuration options on the [Droplet creation page](https://cloud.digitalocean.com/droplets/new):

1. Enable the same features (VPC, IPv6, monitoring, and backups).
2. Choose your SSH key.
3. Paste [the `cloud-config` script](https://docs.digitalocean.com/products/droplets/getting-started/recommended-droplet-setup/recommended-droplet-setup.sh) in user data.
4. Add the tag for the cloud firewall.

If you use `doctl`, the DigitalOcean command line interface, you can [create a Droplet with all of these options](#doctl) in a single command:

```shell
doctl compute droplet create TODO-NAME --tag-names TODO-TAG-NAME \
    --image ubuntu-24-04-x64 --region nyc3 --size s-2vcpu-2gb \
    --ssh-keys TODO-KEY-FINGERPRINT --user-data-file TODO-PATH-TO-FILE \
    --enable-ipv6 --enable-monitoring --enable-private-networking --enable-backups
```

### Using the doctl CLI

If you don’t already have a DigitalOcean account, sign up now.

[Install `doctl` using the GitHub repository’s instructions](https://github.com/digitalocean/doctl), which recommends native package managers:

```shell
# On macOS:
brew install doctl
```

```shell
# On Snap-supported systems, like Ubuntu:
sudo snap install doctl
sudo snap connect doctl:ssh-keys :ssh-keys # Enable support for doctl compute ssh
sudo snap connect doctl:kube-config # Enable support for kubectl
```

Then, on the [**Applications & API** page of the Control Panel](https://cloud.digitalocean.com/account/api/tokens), create a **Personal access token** for the DigitalOcean API with read and write access.

Give `doctl` access to your DigitalOcean account:

```shell
doctl auth init --context examplename
```

Enter the API token when prompted. Using `--context` identifies your account by naming the authentication context. You can list and switch between multiple authenticated accounts with `doctl auth list` and `doctl auth switch`, respectively.

## Get more detail on `doctl` setup, personal access tokens, and `doctl auth` commands.

The following articles have more detailed explanations of this step:

[doctl Command Line Interface (CLI)](https://docs.digitalocean.com/reference/doctl/index.html.md): Manage your DigitalOcean resources from the command line with doctl, our open-source command line interface (CLI).

[How to Create a Personal Access Token](https://docs.digitalocean.com/reference/api/create-personal-access-token/index.html.md): Create a personal access token for use with the DigitalOcean API.

[doctl auth init](https://docs.digitalocean.com/reference/doctl/reference/auth/init/index.html.md): This command allows you to initialize doctl with a token that allows it to query and manage your account details and resources. The command requires and API token to authenticate, which you can generate in the control panel at [https://cloud.digitalocean.com/account/api/tokens](https://cloud.digitalocean.com/account/api/tokens). The `--context` flag allows you to add authentication for multiple accounts and then switch between them as needed. Provide a case-sensitive name for the context and then enter the API token you want use for that context when prompted. You can switch authentication contexts using `doctl auth switch`, which re-initializes doctl. You can also provide the `--context` flag when using any doctl command to specify the auth context for that command. This enables you to use multiple DigitalOcean accounts with doctl, or tokens that have different authentication scopes. If the `--context` flag is not specified, doctl creates a default authentication context named `default`. You can use doctl without initializing it by adding the `--access-token` flag to each command and providing an API token as the argument.

[doctl auth list](https://docs.digitalocean.com/reference/doctl/reference/auth/list/index.html.md): List named authentication contexts that you created with `doctl auth init`. To switch between the contexts use `doctl auth switch --context <name>`, where `<name>` is one of the contexts listed. To create new contexts, see the help for `doctl auth init`.

[doctl auth switch](https://docs.digitalocean.com/reference/doctl/reference/auth/switch/index.html.md): This command allows you to switch between authentication contexts you’ve already created. To see a list of available authentication contexts, call `doctl auth list`. For details on creating an authentication context, see the help for `doctl auth init`.

## Step 1: Create and Upload SSH Keys

Our recommended setup uses SSH keys for authentication when logging into Droplets because password-based authentication is less secure. After you upload your SSH public key to your DigitalOcean account, you can add it automatically to any new Droplets you create, which avoids manually adding or configuring them.

### How do I do this?

If you don’t have an SSH key pair, create one using OpenSSH, which is included on Linux, macOS, and Windows Subsystem for Linux:

```shell
ssh-keygen
```

Your key pair is saved in the location prompted, which by default is `~/.ssh/` on Linux and `/Users/your_username/.ssh` on Windows and macOS. Copy the contents of your public key, which is named `id_ed25519.pub` by default.

Use `doctl compute ssh-key import` to upload the key to your account. Specify the public key file and a name for the key.

```shell
doctl compute ssh-key import TODO-KEY-NAME --public-key-file ~/.ssh/id_ed25519.pub
```

If you saved your SSH key to a location other than the default, use that path for `--public-key-file`.

## Get more detail on creating SSH keys and `doctl ssh-key` commands.

The following articles have more detailed explanations of this step:

[doctl compute ssh-key import](https://docs.digitalocean.com/reference/doctl/reference/compute/ssh-key/import/index.html.md): Use this command to add a new SSH key to your account, using a local public key file. Note that importing a key to your account will not add it to any Droplets

[doctl compute ssh-key create](https://docs.digitalocean.com/reference/doctl/reference/compute/ssh-key/create/index.html.md): Use this command to add a new SSH key to your account. Specify a `<key-name>` for the key, and set the `--public-key` flag to a string with the contents of the key. Note that creating a key will not add it to any Droplets.

[How to Create SSH Keys with OpenSSH on macOS or Linux](https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/create-with-openssh/index.html.md): Use OpenSSH to create new SSH keys on macOS, Linux, or Windows Subsystem for Linux.

[How to Create SSH Keys with PuTTY on Windows](https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/create-with-putty/index.html.md): Use PuTTY to create SSH keys on Windows systems without Bash.

## Step 2: Create and Configure the Droplet

Our recommended setup for Droplets includes enabling several features: VPC (private networking), IPv6, monitoring, and backups.

- [VPC](https://docs.digitalocean.com/products/networking/vpc/index.html.md) creates a private network interface accessible only by resources within the same account or team. It’s free and increases security and decreases bandwidth costs for resources that communicate using it. Enabling it later requires manual network configuration and rebooting the Droplet.
- [IPv6](https://docs.digitalocean.com/products/networking/ipv6/index.html.md) enables an additional 16 IP addresses for the Droplet. It’s free and enabling it later requires manual network configuration and rebooting the Droplet.
- [Monitoring](https://docs.digitalocean.com/products/monitoring/index.html.md) is a metrics visualization service that adds [additional graphs](https://docs.digitalocean.com/products/droplets/how-to/track-performance/index.html.md) to the Control Panel (like CPU load, RAM usage, and disk usage) and the ability to set up [alert policies](https://docs.digitalocean.com/products/monitoring/how-to/manage-alerts/index.html.md). It’s free and enabling it from the start avoids manual setup and lets you understand your resource usage to make more informed decisions on when and how to scale.
- [Backups](https://docs.digitalocean.com/products/backups/index.html.md) are automatic, system-level disk images of Droplets taken weekly, daily, or multiple times a day. Backups give you a way to revert a Droplet to an older state or recreate Droplets, protecting you against data loss.

Our setup also uses [user data](https://docs.digitalocean.com/products/droplets/how-to/provide-user-data/index.html.md), which is data that cloud-init consumes during the Droplet’s first boot to perform tasks or run scripts. The user data script in this tutorial implements two security measures:

- Disables password-based login to the Droplet, making it accessible with SSH keys only.
- Creates a sudo non-root user for day-to-day use. The root user has broad privileges that you don’t need for many tasks. Using a sudo non-root user decreases the risk of making destructive changes by accident and still lets you escalate privileges when necessary.

### How do I do this?

First, save [the `cloud-config` script](https://docs.digitalocean.com/products/droplets/getting-started/recommended-droplet-setup/recommended-droplet-setup.sh) locally:

User data `cloud-config` script

```bash
#!/bin/bash
set -euo pipefail

USERNAME=sammy # TODO: Customize the sudo non-root username here

# Create user and immediately expire password to force a change on login
useradd --create-home --shell "/bin/bash" --groups sudo "${USERNAME}"
passwd --delete "${USERNAME}"
chage --lastday 0 "${USERNAME}"

# Create SSH directory for sudo user and move keys over
home_directory="$(eval echo ~${USERNAME})"
mkdir --parents "${home_directory}/.ssh"
cp /root/.ssh/authorized_keys "${home_directory}/.ssh"
chmod 0700 "${home_directory}/.ssh"
chmod 0600 "${home_directory}/.ssh/authorized_keys"
chown --recursive "${USERNAME}":"${USERNAME}" "${home_directory}/.ssh"

# Disable root SSH login with password
sed --in-place 's/^PermitRootLogin.*/PermitRootLogin prohibit-password/g' /etc/ssh/sshd_config
if sshd -t -q; then systemctl restart sshd fi
```

You can customize the username of the sudo non-root user on the emphasized line.

Next, use `doctl compute droplet create` to create the Droplet.

```shell
doctl compute droplet create TODO-NAME --tag-names TODO-TAG-NAME \
    --image ubuntu-24-04-x64 --region nyc3 --size s-2vcpu-2gb \
    --ssh-keys TODO-KEY-FINGERPRINT --user-data-file TODO-PATH-TO-FILE \
    --enable-ipv6 --enable-monitoring --enable-private-networking --enable-backups
```

Replace the `TODO-` values with your values. Choose a name for the Droplet and create a tag that matches what you’re using the Droplet for, like `webserver`. You need to use this tag to apply cloud firewalls in the next step. Specify the fingerprint of the SSH key you want to use and the relative path to the saved user data file. You can customize the given [datacenter region](https://docs.digitalocean.com/platform/regional-availability/index.html.md#available-datacenters) and [Droplet size](https://docs.digitalocean.com/products/droplets/details/pricing/index.html.md#droplet-sizes).

## Get more detail on Droplet metadata and the `doctl` create command.

The following articles have more detailed explanations of this step:

[How to Access Information about a Droplet using the Metadata API](https://docs.digitalocean.com/products/droplets/how-to/access-metadata/index.html.md): Use the Droplet metadata service to programmatically query a Droplet for information about itself.

[doctl compute droplet create](https://docs.digitalocean.com/reference/doctl/reference/compute/droplet/create/index.html.md): Creates a new Droplet on your account. The command requires values for the `--size`, and `--image` flags. To retrieve a list of size slugs, use the `doctl compute size list` command. To retrieve a list of image slugs, use the `doctl compute image list` command. If you do not specify a region, the Droplet is created in the default region for your account. If you do not specify any SSH keys, we email a temporary password to your account’s email address.

## Step 3: Create a Cloud Firewall

Firewalls place a barrier between your servers and other machines on the network to protect them from external attacks. [DigitalOcean Cloud Firewalls](https://docs.digitalocean.com/products/networking/firewalls/index.html.md) are a free, stateful firewall service for Droplets. They block all traffic that isn’t expressly permitted by a rule.

You can apply cloud firewalls to individual Droplets by name or to one or more Droplets by [tag](https://docs.digitalocean.com/products/droplets/how-to/tag/index.html.md). Our setup uses tags. When you add a tag to a cloud firewall, any Droplets with that tag are automatically included in the firewall configuration, including new Droplets that you tag during creation.

To start, we recommend the following default firewall rules:

- Restrict all inbound traffic except for SSH connections to the Droplet on port 22.
- Allow all outbound traffic to any destination on any port. Many fundamental services rely on outbound communication, and these defaults make it easier to set up a new Droplet without introducing restrictions that could cause expected problems.

In the long term, we recommend [organizing firewalls by role](https://docs.digitalocean.com/products/networking/firewalls/concepts/organization/index.html.md), so you can create custom firewalls for your specific use case.

### How do I do this?

Create a firewall named `inbound-ssh-only`, specifying the tag you used for the new Droplet:

```shell
doctl compute firewall create --name "inbound-ssh-only" \
    --tag-names TODO-TAG-NAME \
    --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0" \
    --outbound-rules "protocol:icmp,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:all,address:0.0.0.0/0,address:::/0 protocol:udp,ports:all,address:0.0.0.0/0,address:::/0"
```

## Get more detail on `doctl compute firewall create`.

The following articles have more detailed explanations of this step:

[doctl compute firewall create](https://docs.digitalocean.com/reference/doctl/reference/compute/firewall/create/index.html.md): Creates a cloud firewall. This command must contain at least one inbound or outbound access rule.

## Summary

After you set up one Droplet with our recommended setup, setting up future ones is simpler because you don’t need to repeat most of the steps.

You only need to complete these steps once:

- Creating an SSH key pair
- Uploading your public key to your DigitalOcean account
- Creating the cloud firewall

To create additional Droplets with the same setup, the only step is choosing its configuration options:

1. Enable the same features (private networking, IPv6, monitoring, and backups).
2. Choose your SSH key.
3. Paste [the `cloud-config` script](https://docs.digitalocean.com/products/droplets/getting-started/recommended-droplet-setup/recommended-droplet-setup.sh) in user data.
4. Add the tag for the cloud firewall.

```shell
doctl compute droplet create TODO-NAME --tag-names TODO-TAG-NAME \
    --image ubuntu-24-04-x64 --region nyc3 --size s-2vcpu-2gb \
    --ssh-keys TODO-KEY-FINGERPRINT --user-data-file TODO-PATH-TO-FILE \
    --enable-ipv6 --enable-monitoring --enable-private-networking --enable-backups
```

## What’s Next?

After this initial setup, you can use your Droplet to host a website, scale out from a single Droplet to multiple Droplets with a [load balancer](https://docs.digitalocean.com/products/networking/load-balancers/index.html.md), or add [object storage](https://docs.digitalocean.com/products/spaces/index.html.md) to serve assets.