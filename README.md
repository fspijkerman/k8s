Ansible Recipes to Install Kubernetes on CloudStack
=====================================

Basic recipes using the ansible cloudstack module to create ssh keys, sec group etc and deploy [Kubernetes](http://kubernetes.io) on [CoreOS](http://coreos.com).

Prerequisites
-------------

You will need python 2.7 or higher with [virtualenv](https://pypi.python.org/pypi/virtualenv)

    $ sudo pip install virtualenv
    
Setup cs
--------

Create a `~/.cloudstack.ini` file with your creds and cloudstack endpoint:

    [cloudstack]
    endpoint = <cloudstackapiendpoint>
    key = <apiaccesskey> 
    secret = <apisecretkey> 
    method = post

We need to use the http POST method to pass the userdata to the coreOS instances.

We can also use variables:

    CLOUDSTACK_ENDPOINT=<cloudstackapiendpoint>
    CLOUDSTACK_KEY=<apiaccesskey>
    CLOUDSTACK_SECRET=<apisecretkey>
    CLOUDSTACK_METHOD=post

Clone the repository and setup environment
---------------

This will install [cs](https://github.com/exoscale/cs) and Ansible

    $ git clone https://github.com/fspijkerman/k8s
    $ cd k8s
    $ virtualenv .venv
    (if you are using python 3 use this instead: $ python -mvenv .venv)
    $ source .venv/bin/activate
    $ pip install -r requirements.txt

Configure Ansible
-----------------

Copy and edit config.yml

    $ cp config.yml-example config.yml

    Edit config.yml and add your Cloudstack API Key's
    

Create a Kubernetes cluster
---------------------------

    $ ansible-playbook --extra-vars @config.yml k8s.yml

Some variables can be edited in the `k8s.yml` file.
This will start a Kubernetes master node and a number of compute nodes.
This is all setup via coreOS instances and passing userdata.

Check the tasks and templates in `roles/k8s`

If you retrieve an error during the ssh key copy:

    "msg": "file (/root/.ssh/id_rsa_k8s) is absent, cannot continue",

Please run the Playbook a second time [(related issue)](https://github.com/apachecloudstack/k8s/issues/5)

Install and configure kubectl
-----------------------------

Now you should have a working cluster.

Install kubectl using the following instructions: https://kubernetes.io/docs/tasks/tools/install-kubectl/

Configure your credentials:

    kubectl config set-cluster betanl2 --certificate-authority=certificates/ca.crt --embed-certs=true --server=https://<your public ip>:6443
    kubectl config set-credentials $USER --client-certificate=certificates/client.crt --client-key=certificates/client.key --embed-certs=true --token=$(cat certificates/token.txt)
    kubectl config set-context default/betanl2/$USER --cluster=betanl2 --namespace=default --user=$USER
    kubectl config use-context default/betanl2/$USER


Test if you can reach k8s
-------------------------

    $ kubectl cluster-info
    Kubernetes master is running at https://yourip:6443

    $ kubectl get nodes 
    NAME         STATUS    AGE       VERSION
    k8s-node-1   Ready     17h       v1.8.2
    k8s-node-2   Ready     17h       v1.8.2

Deploy your first resources
---------------------------

Dashboard:

    # This will create the certificate
    $ kubectl create secret generic kubernetes-dashboard-certs --from-file=certificates/dashboard -n kube-system
    
    # Deploy the dashboard
    $ kubectl apply -f resources/kubernetes-dashboard.yaml
    
How to reach the dashboard:

    Directly (only works if you have the client.pfx imported in your cert trust store).
    
    $ open https://<your ip>:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

    Via proxy:
    
    $ kubectl proxy 
    $ open http://127.0.0.1:8001/ui/
    
Note: due of a [bug](https://github.com/kubernetes/dashboard/issues/2465) in Kubernetes /ui doesnt redirect correctly when https is used. This will be fixed in a future release.
    
CoreDNS:
    
    $ kubectl apply -f resources/coredns.yaml

Heapster:

    $ kubectl apply -f resources/heapster
