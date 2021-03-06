********************
EHO quickstart guide
********************

1 Setup prerequisites
=====================

1.1 OpenStack environment (Folsom+ version) installed.

1.2 OpenStack compute has to have floating IP autoassigment. You can read more here: http://docs.openstack.org/trunk/openstack-compute/admin/content/associating-public-ip.html

1.3 Git should be installed on the machine where EHO_API will be deployed.

1.4 Your OpenStack should have flavors with 'm1.small' and 'm1.medium' names defined because these flavors are referenced by EHO's default Node Templates.
You can check which flavors you have by running

.. sourcecode:: bash

    nova flavor-list

2 Image setup
=============

2.1 Go to OpenStack management node or you can configure ENV at another machine:

.. sourcecode:: bash

    ssh user@hostname

2.2 Download tarball from the following URL:

.. sourcecode:: bash

    wget http://eho-files.mirantis.com/hdp-img-01.tar.gz

2.3 Unpack image and import it into Glance:

.. sourcecode:: bash

    tar -xzf hdp-img-01.tar.gz
    glance image-create --name=hdp.image --disk-format=qcow2 --container-format=bare < ./hdp.img

You should see the output similar to the following:

.. sourcecode:: bash

    +------------------+--------------------------------------+
    | Property         | Value                                |
    +------------------+--------------------------------------+
    | checksum         | e5c77ac14b916de552199f09548adc2a     |
    | container_format | bare                                 |
    | created_at       | 2013-03-11T14:52:09                  |
    | deleted          | False                                |
    | deleted_at       | None                                 |
    | disk_format      | qcow2                                |
    | id               | 7989fd9a-5e30-49af-affa-dea4d7b23b9f |
    | is_public        | False                                |
    | min_disk         | 0                                    |
    | min_ram          | 0                                    |
    | name             | hdp.image                            |
    | owner            | 6b26f08455ec449ea7a2d3da75339255     |
    | protected        | False                                |
    | size             | 1675296768                           |
    | status           | active                               |
    | updated_at       | 2013-03-11T14:53:05                  |
    +------------------+--------------------------------------+


3 EHO API SETUP
===============

3.1 Git clone repo from the https://github.com/Mirantis/eho

.. sourcecode:: bash

    git clone git://github.com/Mirantis/eho.git

3.2 Go to the cloned repo directory

.. sourcecode:: bash

    cd eho

3.3 Install python headers and virtualenv:

.. sourcecode:: bash

    apt-get install python-dev python-virtualenv

3.4 Prepare virtual environment:

.. sourcecode:: bash

    tools/install_venv

3.5 Create config file from default template local.cfg-sample:

.. sourcecode:: bash

    cp ./etc/local.cfg-sample ./etc/local.cfg

3.6 In local.cfg you should edit the following parameters:

.. sourcecode:: bash

    # OPENSTACK CONF
    OS_AUTH_PROTOCOL = 'http'
    OS_AUTH_HOST = 'hostname' <- OpenStack hostname
    OS_AUTH_PORT = '35357'
    OS_ADMIN_USER = 'admin'  <- OpenStack user/password/tenant that will be used to check auth tokens
    OS_ADMIN_PASSWORD = 'nova'
    OS_ADMIN_TENANT = 'admin'

    # NODE CONF
    NODE_USER = 'root'
    NODE_PASSWORD = 'swordfish'
    NODE_INTERNAL_NET = 'novanetwork' <- name of the OpenStack network from which IPs are assigned to VMs

3.7 To run EHO from created environment just call:

.. sourcecode:: bash

    .venv/bin/python bin/eho-api --reset-db --stub-data --allow-cluster-ops

**Note:** ``--reset-db`` and ``--stub-data`` parameters should be inserted only with the first EHO-API startup.
With these parameters supplied EHO will create sqlite db with predefined data in ``/tmp/eho-server.db``

Next times these parameters should be omited:

.. sourcecode:: bash

    .venv/bin/python/ bin/eho-api --allow-cluster-ops

Now EHO service is running. Further steps show how you can verify from console that EHO API works properly.

3.8 First install httpie program. It allows you to send http requests to EHO API service.

.. sourcecode:: bash

    sudo easy_install httpie

**Note:** sure you can use another HTTP client like curl to send requests to EHO service

3.9 Then you need to get authentification token from OpenStack Keystone service:

.. sourcecode:: bash

    tools/get_auth_token <username> <password> <tenant>

E.g.:

.. sourcecode:: bash

    tools/get_auth_token eho-user nova eho-dev

If authentication succeed, output will be as follows:

.. sourcecode:: bash

    Configuration has been loaded from 'etc/local.cfg'
    User: eho-user
    Password: nova
    Tenant: eho-dev
    Auth URL: http://172.18.79.139:35357/v2.0/
    Auth succeed: True
    Auth token: d61e47a1423d477f9c77ecb23c64d424
    Tenant [eho-dev] id: 0677a89acc834e38bf8bb41665912416

**Note:** Save the token because you have to supply it with every request to EHO in X-Auth-Token header.
You will also use tenant id in request URL

3.10 Send http request to the EHO service:

.. sourcecode:: bash

    http http://{eho_api_ip}:8080/v0.2/{tenant_id}/node-templates X-Auth-Token:{auth_token}

Where:

* eho_api_ip - hostname where EHO API service is running

* tenant_id - id of the tenant for which you got token in previous item

* auth_token - token obtained in previous item

For example:

.. sourcecode:: bash

    http http://10.0.0.2:8080/v0.2/0677a89acc834e38bf8bb41665912416/node-templates X-Auth-Token:d61e47a1423d477f9c77ecb23c64d424

Output of this command will look as follows:

.. sourcecode:: bash

    HTTP/1.1 200 OK
    Content-Length: 1936
    Content-Type: application/json
    Date: Mon, 11 Mar 2013 17:17:03 GMT

.. sourcecode:: json

    {
        "node_templates": [
            {
                //Non-empty list of Node Templates
            }
    }

4 Hadoop Cluster startup
========================

4.1 Send the POST request to EHO API to create Hadoop Cluster.

Create file with name ``cluster_create.json`` and fill it with the following content:

.. sourcecode:: json

    {
        "cluster": {
            "name": "hdp",
            "node_templates": {
                "jt_nn.small": 1,
                "tt_dn.small": 3
            },
            "base_image_id": "image id"
        }
    }

Where:

* "name" - name of the cluster being created
* "jt_nn.small": 1 and "tt_dn.small": 3 - names and numbers of Node Templates for Hadoop NameNodes and JobTracker; DataNodes and TaskTrackers.

You can list available node templates by sending the following request to EHO API:

.. sourcecode:: bash

    http http://{eho_api_ip}:8080/v0.2/{tenant-id}/node-templates X-Auth-Token:{auth_token}

* "base_image_id" - OpenStack image id of image which was downloaded in the Item 2.

You can see image id in the OpenStack UI or by calling the following command of the OS Glance service:

.. sourcecode:: bash

    glance image-list

After creating the file you can send the request:

.. sourcecode:: bash

    http http://{eho_api_ip}:8080/v0.2/{tenant-id}/clusters X-Auth-Token:{auth_token} < cluster_create.json

Response for this request will look like:

.. sourcecode:: json

    {
        "cluster": {
            "status": "Starting",
            "node_templates": {
                "jt_nn.small": 1,
                "tt_dn.small": 3
            },
            "service_urls": {},
            "name": "hdp",
            "tenant_id": "tenant-01",
            "nodes": [],
            "id": "254d8a8c483046ab9209d7993cad2da2",
            "base_image_id": "7989fd9a-5e30-49af-affa-dea4d7b23b9f"
        }
    }


4.2 If the response in the 3.1. was ``202 ACCEPTED`` then you can check status of new cluster:

.. sourcecode:: bash

    http http://{eho_api_ip}:8080/v0.2/{tenant-id}/clusters/{cluster_id} X-Auth-Token:{auth_token}

Where "cluster_id" - id of created cluster. In our example above it the id is "254d8a8c483046ab9209d7993cad2da2"

Initially the cluster will be in "Starting" state, but eventually (in several minutes) you should get response with status "Active", like the following:

.. sourcecode:: json

    {
        "cluster": {
            "status": "Active",
            "node_templates": {
                "jt_nn.small": 1,
                "tt_dn.small": 3
            },
            "service_urls": {
                "namenode": "http://172.18.79.196:50070",
                "jobtracker": "http://172.18.79.196:50030"
            },
            "name": "hdp",
            "tenant_id": "tenant-01",
            "nodes": [
                {
                    "node_template": {
                        "id": "d19264649a5e47f98d1fcecccefbf748",
                        "name": "tt_dn.small"
                    },
                    "vm_id": "2a145a8b-0414-4d88-8335-9f3722d41724"
                },
                {
                    "node_template": {
                        "id": "d19264649a5e47f98d1fcecccefbf748",
                        "name": "tt_dn.small"
                    },
                    "vm_id": "c968c5d5-5825-4521-82b5-1c730ab8b1e4"
                },
                {
                    "node_template": {
                        "id": "d19264649a5e47f98d1fcecccefbf748",
                        "name": "tt_dn.small"
                    },
                    "vm_id": "6be15767-ff4e-4e49-9ff7-fb4b65a868d6"
                },
                {
                    "node_template": {
                        "id": "e675e9720f1e47dea5027ed7c13cc665",
                        "name": "jt_nn.small"
                    },
                    "vm_id": "11d120b2-f501-435f-a2f6-515fbacd86cf"
                }
            ],
            "id": "254d8a8c483046ab9209d7993cad2da2",
            "base_image_id": "7989fd9a-5e30-49af-affa-dea4d7b23b9f"
        }
    }

4.3 So you recieved NameNode's and JobTracker's URLs like this:

.. sourcecode:: json

    "service_urls": {
        "namenode": "http://NameNode_IP:50070",
        "jobtracker": "http://JobTracker_IP:50030"
    }
    
and you actually could access them via browser

4.4 To check that your Hadoop installation works correctly:

* Go to NameNode via ssh:

.. sourcecode:: bash

    ssh root@NameNode_IP
    using 'swordfish' as password

* Switch to hadoop user:

.. sourcecode:: bash

    su hadoop

* Go to hadoop home directory and run the simpliest MapReduce example:

.. sourcecode:: bash

    cd ~
    ./run_simple_MR_job.sh

* You can check status of MR job running by browsing JobTracker url:

.. sourcecode:: bash

    "jobtracker": "http://JobTracker_IP:50030"

Congratulations! Now you have Hadoop cluster ready on the OpenStack cloud!
