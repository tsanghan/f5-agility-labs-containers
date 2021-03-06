Lab 1.1 - F5 Container Connector Setup
======================================

The BIG-IP Controller for OpenShift installs as a
`Deployment object <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>`_

.. seealso:: The official CC documentation is here:
   `Install the BIG-IP Controller: Openshift <https://clouddocs.f5.com/containers/v2/openshift/kctlr-openshift-app-install.html>`_

BIG-IP Setup
------------

To use F5 Container connector, you'll need a BIG-IP up and running first.

Through the Jumpbox, you should have a BIG-IP available at the following
URL: https://10.1.1.245

.. warning:: 
   - Connect to your BIG-IP and check it is active and licensed. Its
     login and password are: **admin/admin**

   - If your BIG-IP has no license or its license expired, renew the license.
     You just need a LTM VE license for this lab. No specific add-ons are
     required (ask a lab instructor for eval licenses if your license has
     expired)

   - Be sure to be in the ``Common`` partition before creating the following
     objects.

     .. image:: images/f5-check-partition.png
        :align: center

#. You need to setup a partition that will be used by F5 Container Connector.

   .. code-block:: bash

      # From the CLI:
      tmsh create auth partition ose

      # From the UI:
      GoTo System --> Users --> Partition List
      - Create a new partition called "ose" (use default settings)
      - Click Finished

   .. image:: images/f5-container-connector-bigip-partition-setup.png
      :align: center

#. Create a vxlan tunnel profile

   .. code-block:: bash

      # From the CLI:
      tmsh create net tunnel vxlan ose-vxlan {app-service none flooding-type multipoint}

      # From the UI:
      GoTo Network --> Tunnels --> Profiles --> VXLAN
      - Create a new profile called "ose-vxlan"
      - Set the Flooding Type = Multipoint
      - Click Finished

   .. image:: images/create-ose-vxlan-profile.png
      :align: center

#. Create a vxlan tunnel

   .. code-block:: bash

      # From the CLI:
      tmsh create net tunnel tunnel ose-tunnel {key 0 local-address 10.10.199.60 profile ose-vxlan}

      # From the UI:
      GoTo Network --> Tunnels --> Tunnel List
      - Create a new tunnel called "ose-tunnel"
      - Set the Local Address to 10.10.199.60
      - Set the Profile to the one previously created called "ose-vxlan"
      - Click Finished

   .. image:: images/create-ose-vxlan-tunnel.png
      :align: center

Container Connector Deployment
------------------------------

.. seealso:: For a more thorough explanation of all the settings and options see
   `F5 Container Connector - Openshift <https://clouddocs.f5.com/containers/v2/openshift/>`_

Now that BIG-IP is licensed and prepped with the "ose" partition, we need to
define a `Kubernetes deployment <https://kubernetes.io/docs/user-guide/deployments/>`_
and create a `Kubernetes secret <https://kubernetes.io/docs/user-guide/secrets/>`_
to hide our bigip credentials.

#. From the jumpbox open **mRemoteNG** and start a session with ose-master.

   .. note:: As a reminder we're utilizing a wrapper called **MRemoteNG** for
      Putty and other services. MRNG hold credentials and allows for multiple
      protocols(i.e. SSH, RDP, etc.), makes jumping in and out of SSH
      connections easier.

   On your desktop select **MRemoteNG**, once launched you'll see a few tabs
   similar to the example below.  Open up the OpenShift Enterprise /
   OSE-Cluster folder and double click ose-master.

   .. image:: images/MRemoteNG-ose.png
      :align: center

#. "git" the demo files

   .. note:: These files should be here by default, if **NOT** run the
      following commands.

   .. code-block:: bash

      git clone https://github.com/f5devcentral/f5-agility-labs-containers.git ~/agilitydocs

      cd ~/agilitydocs/openshift

#. Log in with an Openshift Client.

   .. note:: Here we're using a prebuilt user "demouser" and prompted for a
      password, which is: demouser

   .. code-block:: bash

      oc login -u demouser -n default

   .. image:: images/OC-DEMOuser-Login.png
      :align: center

   .. important:: Upon logging in you'll notice access to several projects. In
      our lab well be working from the default "default".

#. Create bigip login secret

   .. code-block:: bash

      oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=admin

   You should see something similar to this:

   .. image:: images/f5-container-connector-bigip-secret.png
      :align: center

#. Create kubernetes service account for bigip controller

   .. code-block:: bash

      oc create serviceaccount k8s-bigip-ctlr -n kube-system

   You should see something similar to this:

   .. image:: images/f5-container-connector-bigip-serviceaccount.png
      :align: center


#. Create cluster role for bigip service account (admin rights, but can be
   modified for your environment)

   .. code-block:: bash

      oc create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr

   You should see something similar to this:

   .. image:: images/f5-container-connector-bigip-clusterrolebinding.png
      :align: center

#. Next let's explore the f5-hostsubnet.yaml file

   .. code-block:: bash

      cd /root/agilitydocs/openshift

      cat f5-bigip-hostsubnet.yaml

   You'll see a config file similar to this:

   .. literalinclude:: ../../../openshift/f5-bigip-hostsubnet.yaml
      :language: yaml
      :linenos:
      :emphasize-lines: 2,9,10

   .. attention:: This YAML file creates an OpenShift Node and the Host is the
      BIG-IP with an assigned /23 subnet of IP 10.131.0.0 (3 imagas down).

#. Next let's look at the current cluster,  you should see 3 members
   (1 master, 2 nodes)

   .. code-block:: bash

      oc get hostsubnet

   .. image:: images/F5-OC-HOSTSUBNET1.png
      :align: center

#. Now create the connector to the BIG-IP device, then look before and after
   at the attached devices

   .. code-block:: bash

      oc create -f f5-bigip-hostsubnet.yaml

   You should see a successful creation of a new OpenShift Node.

   .. image:: images/F5-OS-NODE.png
      :align: center

#. At this point nothing has been done to the BIG-IP, this only was done in
   the OpenShift environment.

   .. code-block:: bash

      oc get hostsubnet

   You should now see OpenShift configured to communicate with the BIG-IP

   .. image:: images/F5-OC-HOSTSUBNET2.png
      :align: center

   .. important:: The Subnet assignment, in this case is 10.131.0.0/23, was
      assigned by the **subnet: "10.131.0.0/23"** line in "HostSubnet" yaml
      file.

   .. note:: In this lab we're manually assigning a subnet. We have the option
      to let openshift auto assign ths by removing **subnet: "10.131.0.0/23"**
      line at the end of the "hostsubnet" yaml file and setting the
      **assign-subnet: "true"**. It would look like this:

      .. code-block:: yaml
         :emphasize-lines: 7

         apiVersion: v1
         kind: HostSubnet
         metadata:
            name: openshift-f5-node
            annotations:
               pod.network.openshift.io/fixed-vnid-host: "0"
               pod.network.openshift.io/assign-subnet: "true"
         host: openshift-f5-node
         hostIP: 10.10.199.60

#. Create the vxlan tunnel self-ip

   .. tip:: For your SELF-IP subnet, remember it is a /14 and not a /23 -
      Why? The Self-IP has to be able to understand those other /23 subnets are
      local in the namespace in the example above for Master, Node1, Node2,
      etc. Many students accidently use /23, but then the self-ip will be only
      to communicate to one subnet on the openshift-f5-node. When trying to
      ping across to services on other /23 subnets from the BIG-IP for instance,
      communication will fail as your self-ip doesn't have the proper subnet
      mask to know those other subnets are local.
      
   .. code-block:: bash
      
      # From the CLI:
      tmsh create net self ose-vxlan-selfip address 10.131.0.1/14 vlan ose-tunnel

      # From the UI:
      GoTo Network --> Self IP List
      - Create a new Self-IP called "ose-vxlan-selfip"
      - Set the IP Address to "10.131.0.1". (An IP from the subnet assigned in the previous step.)
      - Set the Netmask to "255.252.0.0"
      - Set the VLAN / Tunnel to "ose-tunnel" (Created earlier)
      - Set Port Lockdown to "Allow All"
      - Click Finished

   .. image:: images/create-ose-vxlan-selfip.png
      :align: center

#. Now we'll create an Openshift F5 Container Connector to do the API calls
   to/from the F5 device. First we need the "deployment" file.

   .. code-block:: bash

      cd /root/agilitydocs/openshift

      cat f5-cluster-deployment.yaml

   You'll see a config file similar to this:

   .. literalinclude:: ../../../openshift/f5-cluster-deployment.yaml
      :language: yaml
      :linenos:
      :emphasize-lines: 2,5,17,34-38

#. Create the container connector deployment with the following command

   .. code-block:: bash

      oc create -f f5-cluster-deployment.yaml

#. Check for successful creation:

   .. code-block:: bash

      oc get pods -n kube-system -o wide

   .. image:: images/F5-CTRL-RUNNING.png
      :align: center

#. If the tunnel is up and running big-ip should be able to ping the cluster
   nodes. SSH to big-ip and run one or all of the following ping tests:

   .. code-block:: bash

      # ping ose-master
      ping 10.128.0.1

      # ping ose-node1
      ping 10.129.0.1

      # ping ose-node2
      ping 10.130.0.1
