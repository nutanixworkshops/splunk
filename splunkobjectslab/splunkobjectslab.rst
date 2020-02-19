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

Create Bucket Using AMI User
++++++++++++++++++++++++++++

Since Object Storage uses API keys to grant access to various buckets, we'll want to create a bucket using the API key we just created above.

Open Cyberduck (Specifically v6.9.4, as newer versions have some issues) - https://cyberduck.io/changelog/

Open Connection.

.. figure:: images/18.png

Choose **Amazon S3** as the Connection Type, then fill in your details and click Connect.

.. figure:: images/19.png

Click **Continue** when the certificate error pops up.

.. figure:: images/20.png

Right Click and choose **New Folder**, the folder name will be the name of the bucket.

.. figure:: images/21.png

.. figure:: images/22.png

If you check in the Objects console, you'll see that a new bucket has been created.

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
  curl http://10.42.194.11/workshop_staging/Splunk/splunk-8.0.1.tar -o splunk-8.0.1.tar
  tar -xvf splunk-8.0.1.tar
  echo '[user_info]' > /tmp/user-seed.conf
  echo 'USERNAME = admin' >> /tmp/user-seed.conf
  echo 'PASSWORD = nutanix/4u' >> /tmp/user-seed.conf
  export SPLUNK_HOME=/opt/splunk
  export PATH=$SPLUNK_HOME/bin:$PATH
  cp -rp splunk/* /opt/splunk/
  mv /tmp/user-seed.conf $SPLUNK_HOME/etc/system/local
  echo '[clustering]' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'mode = master' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'replication_factor = 1' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'search_factor = 1' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'pass4SymmKey = nutanix/4u' >> $SPLUNK_HOME/etc/system/local/server.conf
  echo 'cluster_label = cluster1' >> $SPLUNK_HOME/etc/system/local/server.conf
  splunk start --answer-yes --no-prompt --accept-license

.. figure:: images/11.png

At this point Splunk should be installed and running, but we need to make a small firewall change in order to connect to it.

.. code-block:: bash

  firewall-cmd --permanent --add-port=8000/tcp
  firewall-cmd --reload

Open your web browser and go to **http://<SPLUNK_IP>:8000**.

The username and password should be as you set them above:

  - admin
  - nutanix/4u

.. figure:: images/12.png

There's not a lot going on right now, but before we give Splunk something to do, we need to connect it to Nutanix Objects.

.. figure:: images/13.png

Configure SmartStore
++++++++++++++++++++

Gather the required information:

  - MYOBJECTSACCESSKEY: You should have this from the AMI Key section above
  - MYOBJECTSSECRETKEY: You should have this from the AMI Key section above
  - OBJECTSCLIENTIP: You can get this from **☰ Menu > Services > Objects**

.. figure:: images/17.png

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

We'll restart Splunk in the next section after installing the Log Generator App.


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
  curl -LJO https://github.com/livearchivist/splunk/raw/master/assets/TA-Nutanix.zip -o TA-Nutanix.zip
  yum install unzip -y
  unzip TA-Nutanix.zip
  cp -r gogen-master/splunk_app_gogen /opt/splunk/etc/apps/

Restart **Splunk** so the new application shows up.

.. code-block:: bash

  /opt/splunk/bin/splunk restart

Now log back into the Splunk web interface, you'll see that **GoGen** is now showing up in the application list.

.. figure:: images/14.png

Click on **Settings > Data Inputs**.

.. figure:: images/15.png

Click on **GoGen**.

**Disable** retail_transaction temporarily. Click on the stanza name: **retail_transaction**.

Fill in the fields to look like the below image, click save:

.. figure:: images/23.png

Re-enable **retail_transaction**.

.. figure:: images/24.png

Data in Objects
+++++++++++++++

After a little bit of time, you should be able to head over to Objects in PC and see that your bucket is being populated with data. If after a period of time, you're not seeing this, you can try running the following script from the Splunk server:

.. code-block:: bash

  splunk _internal call /data/indexes/main/roll-hot-buckets -auth admin:nutanix/4u

You can see in the performance information for my bucket that there have been some Puts and Gets, although the timeline is short for the purposes of this demo, these patterns would continue.

.. figure:: images/25.png

Takeaways
+++++++++

- SmartStore is simple to configure with Nutanix Objects
- You can easily generate test data for your POCs using the GoGen data generator
- Nutanix Objects makes it easy for your customers to migrate to SmartStore, giving them the flexibility to scale incrementally as their Splunk environment grows.
