# DevNation Tech Talk: Ansible Playbook Awesomeness

This directory contains the example content that came as a result of the
DevNation Tech Talk entitled "[Ansible Playbook Awesomeness](https://developers.redhat.com/devnation/tech-talks/ansible-playbook-awesomeness/)"

## Ansible to Automate the Hybrid Cloud

Ansible is an automation tool with the capability of automating many different
tasks across all the major cloud providers, in this example we demonstrate
using Ansible to satisfy a [Hybrid Cloud](https://en.wikipedia.org/wiki/Cloud_computing#Hybrid_cloud)
use case. We're going to create an instance in both [AWS EC2](https://aws.amazon.com/ec2/)
and [Microsoft Azure](https://azure.microsoft.com/en-us/) to simulate managing
a group of web app nodes across multiple infrastructure footprints logically
as one.

I will note that I am using [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux),
version 8.2 at the time of this writing, in order to run the demo and Ansible
version 2.9.7.

There's a little bit of preparation work needed in order to run this demo.

### Prepare the runtime environment

First thing first, enable the Ansible channel on your RHEL 8 system (assuming
your CPU architecture is in fact x86_64, more information [here](https://access.redhat.com/articles/3174981)).
Then we'll install both Ansible and the python virtualenv utilities in order to
setup a python virtualenv to pull the necessary dependencies for running against
the cloud providers:

```
$ subscription-manager repos --enable=ansible-2-for-rhel-8-x86_64-rpms
$ yum install ansible python3-virtualenv
```

We also need to ensure to install the container management tooling as we will
be using a [podman](https://podman.io/) to [manage our containers](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/index)
in a rootless, daemonless, true userspace fashion.

Something to note here is that the container management tooling in RHEL 8 is
provided as an [AppStream Module](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/installing_managing_and_removing_user-space_components/using-appstream_using-appstream#application-streams_using-appstream)
so we'll be using the `module` subcommand for yum:

```
$ yum module install container-tools
```

At this point we can pull the Red Hat Enterprise Linux [Universal Base Image](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image) from the Red Hat public container registry:

We'll be using the `ubi-init` version which comes with the [systemd](https://systemd.io/)
running inside, making it a prime candidate to simulate a linux system for
example purposes.
```
$ podman pull registry.access.redhat.com/ubi8/ubi-init
```

Now go ahead and run it with the name `ubi8` in order to match the inventory
located here:
```
$ podman run --rm -d --name ubi8 registry.access.redhat.com/ubi8/ubi-init
```

Next we need to setup the python dependencies required here in a [Python
virtualenv](https://virtualenv.pypa.io/en/stable/) as defined by the
[Ansible Public Cloud Guides](https://docs.ansible.com/ansible/latest/scenario_guides/cloud_guides.html)
for AWS and Azure, respectively.

We'll also take care of a bookkeeping issue with SELinux python bindings mixing
with python virtualenvs.

```
$ virtualenv devnation-virtualenv
$ ln -s /usr/lib64/python3.6/site-packages/selinux/ devnation-virtualenv/lib64/python3.6/site-packages/selinux
$ ln -s /usr/lib64/python3.6/site-packages/_selinux.cpython-36m-x86_64-linux-gnu.so devnation-virtualenv/lib64/python3.6/site-packages/_selinux.cpython-36m-x86_64-linux-gnu.so
$ ln -s /usr/lib64/python3.6/site-packages/semanage.py devnation-virtualenv/lib64/python3.6/site-packages/semanage.py
$ ln -s /usr/lib64/python3.6/site-packages/_semanage.cpython-36m-x86_64-linux-gnu.so devnation-virtualenv/lib64/python3.6/site-packages/_semanage.cpython-36m-x86_64-linux-gnu.so
$ source devnation-virtualenv/bin/activate
$ pip install 'ansible[azure]' boto boto3
```

At this point, you will need to configure the [Ansible AWS Authentication](https://docs.ansible.com/ansible/latest/scenario_guides/guide_aws.html#authentication)
using your preferred method and the [Ansible Azure Authentication](https://docs.ansible.com/ansible/latest/scenario_guides/guide_azure.html#authenticating-with-azure) procedures.

### Run the exmample playbooks.

First we'll provision the AWS resource and then the Azrue resource, I recommend
changing the respective variables in each to something that is meaningful to you
personally in your environment.

```
$ ansible-playbook provision_aws.yml
$ ansible-playbook privision_azure.yml
```

Something to note is that you can run these in two different terminals in order
to run them in parallel if you like.

Also, these playbooks will add host information to our `inventory.ini` file
which we will pass to the `lamp.yml` playbook in order to simulate the culmination
of automating a Hybrid Cloud burst scaling event followed by an application
deployment cross-infrastructure-footprint.

```
$ ansible-playbook lamp.yml -i inventory.ini
```

Now to get out of the [Python virtualenv](https://virtualenv.pypa.io/en/stable/)
you created, just run the command `deactivate`.

### Notes

This is an example and effectively a simluation of what Ansible is capable of
but is by no means a scenario that you should consider a production use case.
Also, the prep work of setting up the dependencies in a virtualenv, doing 
parallel deployment, secrets/credentials, fine grain access control, and so 
much more are things you can utilize [Ansible Tower](https://www.ansible.com/products/tower)
to address in a real world production environment.

