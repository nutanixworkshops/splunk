.. Adding labels to the beginning of your lab is helpful for linking to the lab from other pages
.. _splunkobjectslab:

------------------
Splunk Objects Lab
------------------

Overview
++++++++

Now that Nutanix is Splunk Smart Store certified, we thought it would be a good time to introduce you to the power of running Splunk on top of Nutanix Objects. In the following lab, you'll walk through the steps of configuring Splunk to write data via SmartStore to Nutanix Objects.

Nutanix Objects is an S3-compatible object storage solution that leverages the underlying Nutanix storage fabric which allows it to benefit from features such as encryption, compression, and erasure coding (EC-X).

Objects allows users to store petabytes of unstructured data on the Nutanix platform, with support for features such as WORM (write once, read many) and object versioning that are required for regulatory compliance, and easy integration with 3rd party backup software and S3-compatible applications.

Lab Setup
+++++++++

This lab requires applications provisioned as part of the :ref:`windows_tools_vm` **and** :ref:`linux_tools_vm`.

If you have not yet deployed **both** of these VMs, see the linked steps before proceeding with the lab.

**Google Chrome is recommended for this lab.**

Create Nutanix Objects AMI User Keys
++++++++++++++++++++++++++++++++++++

In order for Splunk to communicate with Nutanix Objects, you'll need to create a set of API Keys.

#. In **Prism Central** > select :fa:`bars` **> Services > Objects**.

   .. figure:: images/2.png

#. Click on **Access Keys > Add People > Add People not in a directory service**.

   Enter in an email address that is unique (it does not need to be able to receive email).

   .. figure:: images/3.png

#. Click on **Download Keys**. Depending on your browser, it will either open a new tab or download a text file.

    .. note::

        It is important you save the **Access Key** and **Secret Access Key** as it will only be shown once.


    .. figure:: images/5.png

    .. figure:: images/4.png

Create Bucket Using AMI User
++++++++++++++++++++++++++++

Since Object Storage uses API keys to grant access to various buckets, we'll want to create a bucket using the API key we just created above.

A bucket is a sub-repository within an object store which can have policies applied to it, such as versioning, WORM, etc. By default a newly created bucket is a private resource to the creator. The creator of the bucket by default has read/write permissions, and can grant permissions to other users.

In this exercise you will use `Cyberduck <https://cyberduck.io/>`_ to create and use buckets in the object store using your generated access key. Cyberduck is a multi-platform GUI application that supports multiple protocols including FTP, SFTP, WebDAV, and S3.

.. note::

  Cyberduck is already installed on the Windows Tools VM you deployed earlier. If it is not already pinned to the taskbar, you can find it in the Windows menu.

#. Launch **Cyberduck** (Click the Window icon > Down Arrow > Cyberduck).

   If you are prompted to update Cyberduck, click **Skip This Version**.

#. Click on **Open Connection**.

   .. figure:: images/18.png

#. Select **Amazon S3** from the dropdown list.

#. Enter the following fields for the user created earlier, and click **Connect**:

   - **Server**  - *Objects Client Used IP*
   - **Port**  - 443
   - **Access Key ID**  - *Generated When User Created*
   - **Password (Secret Key)** - *Generated When User Created*

   .. figure:: images/19.png

#. Check the box **Always Trust** and then click **Continue** on the **The certificate is not valid** dialog box.

   .. figure:: images/20.png

#. Right Click and choose **New Folder**.

#. Enter the following name for your bucket, and click **Create**:

   - **Bucket Name** - *your-name*-bucket

   .. note::

     Bucket names must be lower case and only contain letters, numbers, periods and hyphens.

     Additionally, all bucket names must be unique within a given Object Store. Note that if you try to create a folder with an existing bucket name (e.g. *your-name*-my-bucket), creation of the folder will not succeed.

     Creating a bucket in this fashion allows for self-service for entitled users, and is no different than a bucket created via the Prism Buckets UI.


   .. figure:: images/21.png

   .. figure:: images/22.png

If you check in the Objects console, you'll see that a new bucket has been created.

Install Splunk
++++++++++++++

Now let's set up a Splunk virtual machine to connect to Objects.

#. In **Prism Central** > select :fa:`bars` **> Virtual Infrastructure > VMs**.

#. Fill out the following fields and click **Save**.

    Leave other settings at their default values.

    - **Name** - *Initials*-Splunk_VM
    - **Description** - (Optional) Description for your VM.
    - **vCPU(s)** - 2
    - **Number of Cores per vCPU** - 1
    - **Memory** - 8 GiB

    - Select **+ Add New Disk**
       - **Type** - DISK
       - **Operation** - Clone from Image Service
       - **Image** - CentOS7.qcow2
       - Select **Add**

   - Select **Add New NIC**
       - **VLAN Name** - Primary
       - Select **Add**

   .. figure:: images/6.png

   .. figure:: images/7.png

#. Click **Save** to create the VM.

#. Find your VM in the VM list, then choose it.

   .. figure:: images/8.png

#. Click **Power On**.

   .. figure:: images/9.png

   .. note::

      Make a note of the **IP Address** of the VM.

      .. figure:: images/10.png

#. Click **Update** in the Prism UI for the VM, then modify the **vDisk**.

      .. figure:: images/26.png

#. Change the **vDisk** size to **100GiB** and click save.

      .. figure:: images/27.png

#. SSH into the Splunk VM using the following credentials (Putty on Windows, Terminal on Mac):

    - **Username** - root
    - **Password** - nutanix/4u

   .. code-block:: bash

     ssh root@10.38.19.50

#. Modify the root partition to take advantage of the extra space.

   .. note::

     Please manually type this step, do not copy it in!

   .. code-block:: bash

    fdisk /dev/sda

    p # Print table
    d # Delete
    2 # Deletes second partition, since Partition 1 is /boot
    n # New Partition
    p # New Primary Partition
    2 # New Primary Partition - /dev/sda2
    Accept defaults for Start Block and End Block
    t # Partition Type
    2 # Partition 2
    8e # Change partition type to "Linux LVM"
    p # Print new partition table
    w # Write New Partition Table

#. Update Kernel Partition Table and Resize Volume

   .. code-block:: bash

    partx -u /dev/sda
    pvresize /dev/sda2
    lvextend -r centos_centos/root /dev/sda2


#. Now let's download the tar files for Splunk and get Splunk installed.

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

#. At this point Splunk should be installed and running, but we need to make a small firewall change in order to connect to it.

   .. code-block:: bash

     firewall-cmd --permanent --add-port=8000/tcp
     firewall-cmd --reload

#. Open your web browser and go to **http://<SPLUNK_IP>:8000**.

#. The username and password should be as you set them above:

   - **Username** - admin
   - **Password** - nutanix/4u

   .. figure:: images/12.png

#. There's not a lot going on right now, but before we give Splunk something to do, we need to connect it to Nutanix Objects.

   .. figure:: images/13.png

Configure SmartStore
++++++++++++++++++++

#. Gather the required information:

   - MYOBJECTSACCESSKEY: You should have this from the AMI Key section above
   - MYOBJECTSSECRETKEY: You should have this from the AMI Key section above
   - OBJECTSCLIENTIP: You can get this from **☰ Menu > Services > Objects**

   .. figure:: images/17.png

#. SSH into the Splunk VM (Putty on Windows, Terminal on Mac)

   - **Username** - root
   - **Password** - nutanix/4u

   .. code-block:: bash

     ssh root@10.38.19.50

#. Use **vi** or **nano** to edit the following file:

   .. code-block:: bash

     vi /opt/splunk/etc/system/local/indexes.conf
     OR
     nano /opt/splunk/etc/system/local/indexes.conf

   .. note::

     If you're using **vi**, ensure to type "**i**" to enter **INSERT** mode.

#. The file contents should look like the below. Ensure to replace any **ALL CAPS** sections with your relevant details.

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

#. Save the file (Nano: CTRL+O, CTRL+X, or VI: ESC, :wq ENTER ).

   .. note::

     We'll restart Splunk in the next section after installing the Log Generator App.


Install Log Generator App
+++++++++++++++++++++++++

Now let's install the log generator app, so we can give Splunk something to consume.

#. SSH into the Splunk VM (Putty on Windows, Terminal on Mac)

   - **Username** - root
   - **Password** - nutanix/4u

   .. code-block:: bash

     ssh root@10.38.19.50

#. Copy down the GoGen files, modified for Nutanix/Splunk.

   .. code-block:: bash

     cd /tmp
     curl -LJO https://github.com/livearchivist/splunk/raw/master/assets/TA-Nutanix.zip -o TA-Nutanix.zip
     yum install unzip -y
     unzip TA-Nutanix.zip
     cp -r gogen-master/splunk_app_gogen /opt/splunk/etc/apps/

#. Restart **Splunk** so the new application shows up.

   .. code-block:: bash

     /opt/splunk/bin/splunk restart

#. Log back into the Splunk web interface, you'll see that **GoGen** is now showing up in the application list.

   .. figure:: images/14.png

#. Click on **Settings > Data Inputs**.

   .. figure:: images/15.png

#. Click on **GoGen**.

#. Click on the stanza name: **retail_transaction**.

#. Fill in the fields to look like the below image, click save:

   .. figure:: images/23.png

#. Enable **retail_transaction**.

   .. figure:: images/24.png

#. Restart **Splunk** one more time.

   .. code-block:: bash

     /opt/splunk/bin/splunk restart


Data in Objects
+++++++++++++++

After a little bit of time, you should be able to head over to Objects in PC and see that your bucket is being populated with data.

.. note::

   If after 5 minutes, you're not seeing this, you can try running the following script from the Splunk server:

   .. code-block:: bash

     splunk _internal call /data/indexes/main/roll-hot-buckets -auth admin:nutanix/4u

#. You can see in the performance information for my bucket that there have been some Puts and Gets, although the timeline is short for the purposes of this demo, these patterns would continue.

   .. figure:: images/25.png

Takeaways
+++++++++

- SmartStore is simple to configure with Nutanix Objects
- You can easily generate test data for your POCs using the GoGen data generator
- Nutanix Objects makes it easy for your customers to migrate to SmartStore, giving them the flexibility to scale incrementally as their Splunk environment grows.
