Review BIG-IP configuration
===========================

The BIG-IP we are working on has been licensed, and only these following commands below has been issued in the CLI so we have a very new/basic BIG-IP configured.

::

  License BIG-IP

  tmsh create net vlan internal interfaces add {1.2}

  tmsh create net self 10.10.199.98/24 vlan internal

  tmsh create net vlan external interfaces add {1.1}

  tmsh create net self 10.10.201.98/24 vlan external

  tmsh create auth partition kubernetes

  tmsh create net tunnel vxlan ose-vxlan {app-service none flooding-type multipoint}

  tmsh create net tunnel tunnel ose-tunnel {key 0 local-address 10.10.199.98 profile ose-vxlan}

  tmsh save sys config

**NOTE typically the command below is entered after running the ''oc create -f f5-hostsubnet.yaml'' command coming up in the next section (This is the range the self ip should come from, to make this lab quicker we have already done this tmsh command)**

::

  tmsh create net self <IP>/subnet vlan <tunnel>

  tmsh create net self <IP>10.131.0.98/14 vlan ose-tunnel

Let's validate your BIG-IP is just configured with VLANs, Self-IPs.  No no Virtual Servers and no Pools

Connect to your BIG-IP on https://10.10.200.98 and familiarize yourself with the the current VLAN's.  Proceed to Network -> VLAN.

.. image:: /_static/class4/F5-BIG-IP-NETWORK-VLAN.png
   :align: center
   :scale: 60%


Go to Local Traffic -> Network -> Self-IP.  You should have an internal and external SELF-IPs

.. image:: /_static/class4/F5-BIG-IP-NETWORK-SELFIP.png
   :align: center
   :scale: 60%

Jump to Local Traffic -> Network -> Tunnel.  You should see something similar to this:

.. image:: /_static/class4/F5-BIG-IP-NETWORK-TUNNEL.png
   :align: center
   :scale: 60%

Lastly, validate there are no Virtual Servers and no Pools.  Go to Local Traffic -> Virtual Servers and then Pools.

Last example we can see that there is no pool members defined.

.. image:: /_static/class4/F5-BIG-IP-LOCAL_TRAFFIC-POOL.png
   :align: center
   :scale: 60%

Great let's jump to the next section to work on the OpenShift CLI