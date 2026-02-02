# Ansible playbook for BAR Live Services

> [!WARNING]
> This repo is work in progress, not finished, nothing to see here yet.

This is an [Ansible](https://en.wikipedia.org/wiki/Ansible_(software)) playbook for setting up central BAR Live Services server.

## Usage

**TODO**

### Dependencies

Make sure required collections are installed by running:

```
ansible-galaxy collection install -r requirements.yml
```

If you pull repo and there are changes to that fille, you need to rerun the command to make sure you pick up latest changes.

## Local testing

### Setup

We use Incus for local testing. Make sure you have it installed and initialized following [the official getting started docs](https://linuxcontainers.org/incus/docs/main/tutorial/first_steps/).

To create a new container and initialize it via cloud-init, run the following command:

```
touch .incus-integration-on && \
chmod 0600 test.ssh.key && \
incus launch images:debian/trixie/cloud bar-live-srv-test < test.incus.yml && \
incus exec bar-live-srv-test -- cloud-init status --wait
```

Then test that it works for ansible:

```
ansible dev -m shell -a 'uname -a'
```

The test container runs a few different services, and depends on a few `*.bar-live-srv.local` domain names pointing at the incus container so they are independently routable from web browser. Easiest is to add a new entry in `/etc/hosts` like:

```
{incus_container_ip} ok.bar-live-srv.local
```

you can add/update it with:

```
ansible ,localhost -b -K -m lineinfile -a "path=/etc/hosts regexp='.*bar-live-srv.*' line='$(ansible-inventory --host test | jq -r '.ansible_host') ok.bar-live-srv.local'"
```

### Usage

Now you can use all the playbooks and roles as usual, just make sure you are targeting the `dev` inventory group or `test` host. For example:

```
ansible-playbook -l dev play.yml
```

You can ssh into it with something like:

```
ssh -i test.ssh.key ansible@$(ansible-inventory --host test | jq -r '.ansible_host')
```

Or enter directly into root container shell with:

```
incus exec bar-live-srv-test -- /bin/bash
```

#### Monitoring host

To configure sending metrics to local monitoring host set up with [ansible-monitoring](https://github.com/beyond-all-reason/ansible-monitoring) playbook, run:

```
MON_HOST_IP=$(incus list -f csv -c 4 bar-mon-test | grep -E 'eth|enp' | cut -d' ' -f1)
ansible-playbook -l test play.yaml --diff -t monitoring \
  -e "{ configure_monitoring: true, monitoring_metrics_write_host_ip: $MON_HOST_IP }"
```

### Cleanup

To stop and remove the container:

```
incus stop bar-live-srv-test && incus delete bar-live-srv-test
```
