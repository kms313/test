KINX-loadbalancer-agent
========================

What is KINX loadbalancer agent?
---------------------------------

HAproxy agent is agent which help to make HAProxy configuration and support neighboring with L3 switch to support N+1 Loadbalancer using ECMP.
HAproxy agent serve REST API request using Flask & Gunicorn.

KINX Loadbalancer Agent Architecture
-------------------------------------

.. image:: images/klb_agent.png

Work Flow
---------

* Create **Loadbalancer**

   * API: ``/api/haproxy/loadbalancers``
   * Request

     .. code-block:: JSON
        {
            "id": "lb_id",
            "name": "lb_name",
            "description": "lb_description",
            "enabled": "lb_admin_state_up",
            "project_id": "lb.tenant_id"
        }

Installation
------------

.. note::  You should use Ubuntu 16.04 image
   Because of auto network interface setting

#. Install Package::

    $ apt-get update -y
    $ apt-get install python-minimal -y
    $ apt-get install python2.7-dev git -y
    $ add-apt-repository ppa:vbernat/haproxy-1.6 #version 1.6.11
    $ apt-get update -y
    $ apt-get install haproxy -y
    $ apt-get install python-setuptools build-essential libssl-dev -y
    $ easy_install pip
    $ pip install flask==1.0.2
    $ pip install Flask-HTTPAuth==3.2.4
    $ pip install psutil==5.0.1
    $ pip install parse==1.8.4
    $ pip install python-crontab==2.3.4
    $ pip install gunicorn==19.9.0
    $ pip install webob==1.8.2
    $ pip install netaddr==0.7.19
    $ apt-get install supervisor -y
    $ apt-get install quagga
    $ apt install socat

#. Make ``/opt/kinx_loadbalancer_agent`` directory for File Storage(Member, Peer)::

    $ mkdir /opt/kinx_loadbalancer_agent

#. Make ``/etc/haproxy/conf.d`` directory for piece of haproxy file::

    $ mkdir /etc/haproxy/conf.d

#. Download kinx loadbalancer github & copy whole file to local dist-packages::

    $ git clone https://github.com/kinxnet/kinx-loadbalancer.git
    $ cp kinx-loadbalancer/bin/kinx_loadbalancer_agent /usr/local/bin/
    $ sudo chmod +x /usr/local/bin/kinx_loadbalancer_agent
    $ cp -rf kinx-loadbalancer/kinx_loadbalancer_agent /usr/local/lib/python2.7/dist-packages/

#. Supervisor Setting (Retrieve from sample file in ``samples/kinx_haproxy_agent.conf``)::

    $ cp kinx-loadbalancer/etc/kinx_loadbalancer_agent.conf /etc/supervisor/conf.d/
    $ vim kinx_haproxy_agent.conf

    [program:kinx_loadbalancer_agent]  ;
    command=/usr/local/bin/kinx_loadbalancer_agent ;
    autostart=true ;
    autorestart=true ;
    user=root ;
    redirect_stderr=true  ;
    stdout_logfile=/var/log/supervisor/haproxy_agent.log  ;

#. Supervisor reload::

    $ supervisorctl reload

#. Supervisor status check::

    $ supervisorctl status

#. Change logrotate settings::

    # /etc/logrotate.d/haproxy (ex: rotate 52 -> rotate 7)
    /var/log/haproxy.log {
        daily
        rotate 7
        missingok
        notifempty
        compress
        delaycompress
        postrotate
            invoke-rc.d rsyslog rotate >/dev/null 2>&1 || true
        endscript
    }

#. Quagga configuration::

    $ vim /etc/quagga/debian.conf
    vtysh_enable=yes
    zebra_options="  --daemon -A 0.0.0.0"
    bgpd_options="   --daemon -A 0.0.0.0"

    $ vim /etc/quagga/daemons
    zebra=yes
    bgpd=yes

    $ service quagga restart

#. Create Glace Image from VM::

    $ nova --debug image-create --show {vm-uuid} {image-name} # raw file
    $ glance image-download --file {file-name] --progress {image-id}
    $ qemu-img convert -f raw -O qcow2 {raw-image} {qcow2-image}
    $ glance image-create --disk-format qcow2 --container-format bare --visibility public --progress --name {image-name} --file {qcow2-image}
