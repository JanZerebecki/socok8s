.. _ose-localhost:

Prepare localhost
=================

.. blockdiag::

   blockdiag {
     default_fontsize = 11;
     localhost [label="Prepare localhost"]
     ses [label="Deploy SES"]
     caasp [label="Deploy CaaS Platform"]
     deployer [label="Deploy deployer\n(optional)"]
     enroll_caasp [label="Enroll CaaS Platform Nodes"]

     configure [label="Configure\n Cloud"]
     setup_caasp_workers [label="Setup CaaS Platform\nworker nodes"]
     patch_upstream [label="Apply patches\nfrom upstream\n(for developers)"]
     build_images [label="Build Docker images\n(for developers)"]
     deploy_airship [label="Deploy Airship"]
     deploy_openstack [label="Deploy OpenStack"]

     localhost -> ses;

     group {
       localhost
       color="red"
     }

     group {
       color = "#EEEEEE"
       label = "Setup hosts"
       ses -> caasp;
       caasp -> deployer [folded];
       deployer -> enroll_caasp;
     }

     enroll_caasp -> configure [folded];

     configure -> setup_caasp_workers;

     group {
       color = "#EEEEEE"
       label = "Cloud Deployment"
       setup_caasp_workers -> patch_upstream;
       patch_upstream -> build_images;
       build_images -> deploy_airship [folded];
       setup_caasp_workers -> deploy_airship;
       deploy_airship -> deploy_openstack;
     }
   }


Base software
-------------

Install the following software on your `localhost`:

  * jq
  * ipcalc
  * git
  * python-virtualenv

Optionally, `localhost` can be preinstalled with the following software:

  * ansible>=2.7.0
  * python-openstackclient
  * python-requests
  * python-jmespath
  * python-openstacksdk
  * python-netaddr

Make sure to install the variant of the packages that matches the Python
release that ansible is using. (e.g. on openSUSE Tumbleweed, Ansible is using
Python 3, so install the "python3-" variant of the packages)

If those optional software aren't installed, they will be installed in a
venv in |socok8s_workspace_default|\ `/.ansiblevenv` .

Cloning this repository
-----------------------

To get started, you need to clone this repository. This repository uses
submodules, so you need to get all the code to make sure the playbooks
work.

::

   git clone --recursive https://github.com/SUSE-Cloud/socok8s.git

Alternatively, one can fetch/update the tree of the submodules by
running:

::

   git submodule update --init --recursive

Configure Ansible
-----------------

Use ARA (recommended)
~~~~~~~~~~~~~~~~~~~~~

To use ARA, set the following environment variable before running `run.sh`.

.. code-block:: console

   export USE_ARA='True'

To setup ARA more permanently for your user on `localhost`, create an ansible
configuration file loading ara plugins:

.. code-block:: console

   python -m ara.setup.ansible | tee ~/.ansible.cfg

For more details on ARA's web interface, please read
https://ara.readthedocs.io/en/stable/webserver.html .

Enable mitogen (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~

To improve deployment speed, enable mitogen strategy and connection plugin.
First install mitogen in your venv (e.g. |socok8s_workspace_default|\ `/.ansiblevenv`
or your local ansible environment), then enable it using environment variables.

Alternatively, enable it for all your ansible calls by adding it to your
ansible configuration:

.. we need parsed-literal instead of code-block here. Otherwise the variable substitute does not work
.. parsed-literal::

   cat << EOF >> ~/.ansible.cfg
   strategy_plugins=${HOME}\ |socok8s_workspace_default|\ /.ansiblevenv/lib/python3.6/site-packages/ansible_mitogen/plugins/strategy
   strategy = mitogen_linear
   EOF

For more details on mitogen, please read
https://mitogen.readthedocs.io/en/latest/ansible.html .

Enable pipelining (recommended)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You might want to improve SSH connections by enabling pipelining:

.. code-block:: console

   cat << EOF >> ~/.ansible.cfg
   [ssh_connection]
   pipelining = True
   EOF

.. _deploymechanism:

Defining a workspace
--------------------

`socok8s` might create a :term:`workspace`, install things (eg. Ansible in a
virtualenv) or create resources (eg. OpenStack Heat stacks if the deployment
mechanism is `openstack`). For all of theses operations, a environment variable
called `SOCOK8S_ENVNAME` must be set. This variable must be unique if
multiple environments are installed in parallel.

.. code-block:: console

   export SOCOK8S_ENVNAME='soc-west'


Set the deployment mechanism
----------------------------

This tooling can work with two different mechanisms:

* Bring your own environment
* Deploy everything on top of OpenStack (experimental).

This behavior can be changed by setting the environment variable
`DEPLOYMENT_MECHANISM`. Its default value is "kvm". When you want
to deploy :term:`CaaSP`, :term:`SES` and Containerized OpenStack on top of an
OpenStack environment (for CI for example), run:

.. code-block:: console

   export DEPLOYMENT_MECHANISM='openstack'

.. _configureopenstackdeploymentmechanism:

Configure OpenStack deployment mechanism (experimental)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Make sure your environment have an openstack client configuration file.
For that, you can create the ``~/.config/openstack/clouds.yaml``.

The following is an example if you are running on a "engcloud":

::

   clouds:
     engcloud:
       region_name: CustomRegion
       auth:
         auth_url: https://keystone_url/v3
         username: john # your username here
         password: my-super-secret-password # your password here or add it into secure.yaml
         project_name: cloud
         project_domain_name: default
         user_domain_name: ldap_users # this is just an example, adapt to your needs
       identity_api_version: 3
   ansible:
     use_hostnames: True
     expand_hostvars: False
     fail_on_errors: True

Now pre-create your environment. It is convention here to use your username
as part of the name of objects you create.

Create a keypair on your cloud (named further *engcloud*) using either the
horizon's web interface or OpenStack CLI’s ``openstack keypair create`` for
accessing the instances created. Remember the name of this keypair (which
appears as ``soc-west-key`` in the example below)

Set this for **all** the following scripts in a deployment:

.. code-block:: console

   export SOCOK8S_ENVNAME='soc-west'
   # 'engcloud' is the name in the `clouds.yaml`
   export OS_CLOUD=engcloud
   # Set to the name of the keypair you created
   export KEYNAME=soc-west-key
   #replace with the actual external network name in your OpenStack environment
   export EXTERNAL_NETWORK=floating

With this done, proceed to next section of the documentation,
:ref:`ose-targethosts`.
