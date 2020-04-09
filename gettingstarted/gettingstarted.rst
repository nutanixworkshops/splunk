.. _gettingstarted:

---------------
Getting Started
---------------

Lab Setup
+++++++++

This lab requires both the :ref:`windows_tools_vm` and :ref:`linux_tools_vm` VMs.

If you have not yet deployed these VMs, see the linked steps before proceeding with the lab.

Create Nutanix Objects IAM User Keys
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

Create Bucket Using IAM User
++++++++++++++++++++++++++++
Since Object Storage uses API keys to grant access to various buckets, we'll want to create a bucket using the API key we just created above.
A bucket is a sub-repository within an object store which can have policies applied to it, such as versioning, WORM, etc. By default a newly created bucket is a private resource to the creator. The creator of the bucket by default has read/write permissions, and can grant permissions to other users.

#. Click on your Object Store then click **Create Bucket**

   .. figure:: images/buckets-1.png

#. Name the bucket *INITIALS*-**bucket** > click **Create**

   .. note::

     Bucket names must be lower case and only contain letters, numbers, periods and hyphens.
     Additionally, all bucket names must be unique within a given Object Store. Note that if you try to create a folder with an existing bucket name (e.g. *your-name*-my-bucket), creation of the folder will not succeed.
     Creating a bucket in this fashion allows for self-service for entitled users, and is no different than a bucket created via the Prism Buckets UI.

   .. figure:: images/buckets-2.png

#. Click on the bucket you just created, then click **Edit User Access**

   .. figure:: images/buckets-3.png

   .. figure:: images/buckets-4.png

#. Find your user and give it **Read and Write** access

   .. figure:: images/buckets-5.png
