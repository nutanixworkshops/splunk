.. Adding labels to the beginning of your lab is helpful for linking to the lab from other pages
.. _splunkobjectslab:

------------------
Splunk Objects Lab
------------------

Overview
++++++++

Now that Nutanix is Splunk Smart Store certified, we thought it would be a good time to introduce you to the power of running Splunk on top of Nutanix Objects. In the following lab, you'll walk through the steps of configuring Splunk to write data via SmartStore to Nutanix Objects.

Create Nutanix Objects AMI User Keys
++++++++++++++++++++++++++++++++++++

In order for Splunk to communicate with Nutanix Objects, you'll need to create a set of API Keys.

Open \https://<*NUTANIX-PC-IP*>:9440 in your browser to access Prism. Log in as a user with administrative priveleges.

.. figure:: images/1.png

Click **☰ Menu > Services > Objects**.

.. figure:: images/2.png

Click on **Access Keys > Add People > Add People not in a directory service**.

Enter in an email address that is unique (it does not need to be able to receive email).

.. figure:: images/3.png

Click on **Download Keys**. Depending on your browser, it will either open a new tab or download a text file.

**NOTE:** It is important you save the **Access Key** and **Secret Access Key** as it will only be shown once.

.. figure:: images/5.png

.. figure:: images/4.png


Install Splunk
++++++++++++++

Now let's set up a Splunk virtual machine to connect to Objects.

Click **☰ Menu > Virtual Infrastructure > VMs**.

Click **Create VM**.

  - VM Name
  - 2 vCPUs
  - 8GB Memory

Click **Add a New Disk** and configure like the image below.

.. figure:: images/6.png

Click **Add New NIC** and choose **Primary**.

.. figure:: images/7.png

Click **Save** to create the VM.

Find your VM in the VM list, then choose it.

.. figure:: images/8.png

Click **Power On**.

.. figure:: images/9.png

Make a note of the **IP Address** of the VM.

.. figure:: images/10.png

SSH into the Splunk VM (Putty on Windows, Terminal on Mac)

  - User: root
  - Pass: nutanix/4u

.. code-block:: bash

  ssh root@10.38.19.50

Now let's download the tar files for Splunk and get Splunk installed.

.. code-block:: bash

  mkdir /opt/splunk
  cd /tmp
  curl http://10.42.194.11/workshop_staging/splunk-8.0.1.tar -o splunk-8.0.1.tar
  tar -xvf splunk-8.0.1.tar
  echo '[user_info]' > /tmp/user-seed.conf
  echo 'USERNAME = admin' >> /tmp/user-seed.conf
  echo 'PASSWORD = nutanix/4u' >> /tmp/user-seed.conf
  export SPLUNK_HOME=/opt/splunk
  export PATH=$SPLUNK_HOME/bin:$PATH
  cp -rp splunk/* /opt/splunk/splunk
  mv /tmp/user-seed.conf $SPLUNK_HOME/etc/system/local
  echo '[clustering]' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'mode = master' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'replication_factor = 1' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'search_factor = 1' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'pass4SymmKey = nutanix/4u' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'cluster_label = cluster1' >> $SPLUNK_HOME/etc/system/local/server.conf
  splunk start --answer-yes --no-prompt --accept-license

At this point Splunk should be installed and running, but we need to make a small firewall change in order to connect to it.

.. code-block:: bash

  firewall-cmd --permanent --add-port=8000/tcp
  firewall-cmd --reload

Open your web browser and go to **http://<SPLUNK_IP>:8000**.

The username and password should be as you set them above:

  - admin
  - nutanix/4u


Install Log Generator App
+++++++++++++++++++++++++

Now let's install the log generator app, so we can give Splunk something to consume.

SSH into the Splunk VM (Putty on Windows, Terminal on Mac)

  - User: root
  - Pass: nutanix/4u

.. code-block:: bash

  ssh root@10.38.19.50

Copy down the GoGen files, modified for Nutanix/Splunk.

.. code-block:: bash

  cd /tmp
  curl 

Configure SmartStore
++++++++++++++++++++

Now let's configure Splunk SmartStore.

SSH into the Splunk VM (Putty on Windows, Terminal on Mac)

  - User: root
  - Pass: nutanix/4u

.. code-block:: bash

  ssh root@10.38.19.50

Use **vi** or **nano** to edit the following file:

.. code-block:: bash

  vi /opt/splunk/etc/system/local/indexes.conf
  OR
  nano /opt/splunk/etc/system/local/indexes.conf

The file contents should look like the below. Ensure to replace any **ALL CAPS** sections with your relevant details.

.. code-block:: bash

  [default]
  remotePath = volume:remote_store/$_index_name

  [volume:remote_store]
  storageType = remote
  path = s3://MYAWESOMEBUCKETHERE/
  remote.s3.access_key = MYOBJECTSACCESSKEY
  remote.s3.secret_key = MYOBJECTSSECRETKEY
  remote.s3.endpoint = https://OBJECTSCLIENTIP
  remote.s3.auth_region = us-east-1

  [main]
  hotTimePeriodInSecs=60

Save the file (Nano: CTRL+O, CTRL+X, or VI: ESC, :wq ENTER ).

Restart **Splunk** so the new configuration comes into effect.

.. code-block:: bash

  /opt/splunk/bin/splunk restart



Takeaways
+++++++++

- Here is where we summarize any key takeaways from the module
- Such as how a Nutanix feature used in the lab delivers value
- Or highlighting a differentiator
