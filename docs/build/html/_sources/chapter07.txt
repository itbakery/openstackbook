บทที่ 7 หลักการทำงานของ Puppet
############################

ใช้หลักการทำงานของ client/server หรือในบางครั้งอาจจะเรียกว่า agent/server ในการใช้งานจะมีเครื่อง
หนึ่งที่ทำหน้าเป็น puppet master และเครื่องอื่นจะเป็นเครื่อง puppet agent เครื่อง master ทำหน้าที่ติดตั้ง
package, configuration บนเครื่อง client

การทำงาน

.. image:: /_images/puppet-agentmaster.png

สร้าง project folder

.. code-block:: bash
    :linenos:

    mkdir ~/Vagrant/puppet
    cd ~/Vagrant/puppet
    touch Vagrantfile


Vagrantfile

.. code-block:: bash
    :linenos:

    # -*- mode: ruby -*-
    # vi: set ft=ruby :

    MGN_NETWORK="10.0.0"
    Vagrant.configure(2) do |config|
        config.vm.box = "centos/7"
        config.vm.define :master do |c|
            c.vm.hostname =  "master.example.com"
            c.vm.provider :libvirt do |d|
                d.memory = 4096
                d.cpus = 2
                d.nested = true
           end
          # provision network
          c.vm.network :private_network, ip: "#{MGN_NETWORK}.10", netmask: "255.255.255.0"
          c.vm.provision "shell", inline: <<-EOF
          echo root:stack | chpasswd
          #only on master
          curl --fail -sSLo /etc/yum.repos.d/passenger.repo https://oss-binaries.phusionpassenger.com/yum/definitions/el-passenger.repo
          #install repo
          rpm -ivh https://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm
          cat  << HOST  >> /etc/hosts
    10.0.0.10  master.example.com master
    10.0.0.11  agent.example.com agent
    HOST
    EOF
        end

        config.vm.define :agent do |c|
          c.vm.hostname =  "agent.example.com"
          c.vm.provider :libvirt do |d|
            d.memory = 4096
            d.cpus = 2
            d.nested = true
          end
          # provision network
          c.vm.network :private_network, ip: "#{MGN_NETWORK}.11", netmask: "255.255.255.0"
          c.vm.provision "shell", inline: <<-EOF
          echo root:stack | chpasswd
          #install repo
          rpm -ivh https://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm
          cat  << HOST  >> /etc/hosts
    10.0.0.10  master.example.com master
    10.0.0.11  agent.example.com agent
    HOST
    EOF
       end
    end


ติดตั้ง puppet master
******************

- OS         : Centos 7 Minimal
- IP Address : 192.168.10.10
- Hostname   : master.example.com


.. code-block:: bash
    :linenos:

    [root@master ~]# yum -y install puppet-server
    [root@master ~]# vim /etc/puppet/puppet.conf

    [main]
    dns_alt_names = master,master.example.com
    certname = master.example.com

    # สร้าง puppet master certificate
    [root@master ~]# puppet master --verbose --no-daemonize

    # เมื่อ มีการแสดงผล ``Notice: Starting Puppet master version <VERSION>``
    # ให้การหยุดทำงาน ด้วยการกด Ctrl + c

    [root@master ~]# systemctl start puppetmaster
    [root@master ~]# systemctl enable puppetmaster

.. image:: /_images/puppet-cert.png


ติดตั้ง puppet agent (เปิดอีก terminal)
******************

.. code-block:: bash
    :linenos:

    # ติดตั้ง puppet และระบุ ว่า puppet master อยู่ที่ไหนใน config
    [root@agent ~]# yum install puppet
    [root@agent ~]# vi /etc/puppet/puppet.conf

    [agent]
    server = master.example.com

    [root@agent ~]# systemctl start puppet
    [root@agent ~]# systemctl enable puppet


ตรวจสอบ log message `` Could not request certificate: Connection refused - connect``
แสดงว่า puppetmaster ยังไม่ start ที่ master เมื่อ start puppetmaster เรียบร้อยก็ให้ทำการ restart
puppet ของ  client อีกครั้ง กลับไปยัง terminal ของ puppet master  จะพบว่า มีการ request
มาจาก puppet agent


.. code-block:: bash
    :linenos:

    # show certificate request
    [root@master ~]# puppet cert list
    "agent.example.com" (SHA256) C1:67:53:DB:69:D4:85:4D:4A:ED:28:E9:AF:DF:A0:4B:3D:27:BB:1E:D1:7B:7B:3D:AD:7D:5D:EE:8F:31:9F:A9

    # puppet master ทำการ sign

    [root@master ~]# puppet cert --allow-dns-alt-names sign agent.example.com
    (ผลที่ได้)
    Notice: Signed certificate request for agent.example.com
    Notice: Removing file Puppet::SSL::CertificateRequest agent.example.com at '/var/lib/puppet/ssl/ca/requests/agent.example.com.pem'


เมื่อ puppet master ทำการ sign ให้กับ certificate puppet client เรียบร้อยแล้ว ก็แสดงว่าต่อจากนี้ไป
puppet master สามารถสื่อสารกับ puppet client ได้แล้ว โดย default แล้ว puppet agent จะทำการ
เชื่อมต่อ มายัง manifests ที่อยู่ในฝั่ง server ``/etc/puppet/manifests/site.pp``


.. code-block:: bash
    :linenos:

    node "agent.example.com" {
      group { 'testgroup':
        ensure => present,
        gid    => 2000,}

      file { '/root/example_file.txt':
        ensure => "file",
        owner  => "root",
        group  => "root",
        mode   => "700",
        content => "Congratulations! Puppet has created this file.",}
    }

เพื่อความรวดเร็วก็ให้ restart puppet ในฝั่ง agent และลองดู log จะได้ว่า

.. code-block:: bash
    :linenos:

    systemctl restart puppet
    tail -f /var/log/messages

    Feb 21 15:07:10 localhost puppet-agent[21425]: (/Stage[main]/Main/Node[agent.example.com]/File[/root/example_file.txt]/ensure) defined content as '{md5}3f14d602a50294a915e8f933a49c8356'
    Feb 21 15:07:11 localhost puppet-agent[21425]: (/Stage[main]/Main/Node[agent.example.com]/Group[testgroup]/ensure) created
    Feb 21 15:07:12 localhost puppet-agent[21425]: Finished catalog run in 1.74 seconds

    [root@agent ~]# ls /root/
    anaconda-ks.cfg  example_file.txt
    [root@agent ~]# cat example_file.txt
    Congratulations! Puppet has created this file.


    [root@agent ~]# cat /etc/group | grep test
    testgroup:x:2000:
