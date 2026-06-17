ISP:
    Baza:
        nano /etc/network/interfaces 
            auto ens192\224\256
            iface ens___ inet dhcp
            
            auto ens___
            iface ens___ inet static
            address ___.___.___.___
        
        systemctl restart networking
        nano /etc/sysctl.conf
            net.ipv4.ip_forward=1

        nano /etc/apt/sources.list
            deb\deb-src http://mirror.yandex.ru/debian bookworm main contrib
        
    iptables:
        apt install iptables-persistent -y 
        iptables -t nat -A POSTROUTING -o ens192 -j MASQUERADE 
        iptables-save >> /etc/iptables/rules.v4

EcoRouter (HQ-RTR):
    Baza:
        en
        conf t
        port ge0 {к ISP}
        service-instance ge0 {к ISP}
        encapsuletion untagged
        int ge0
        ip add ___.___.___.___/___
        exit
        ip route 0.0.0.0/0 ___.___.___.___ {ip ISP}
        connect port ge0 service-instance ge0
        service-instance vl100
        encapsuletion dot1q 100
        rewrite pop 1
        exit
        service-instance vl200
        encapsuletion dot1q 200
        rewrite pop 1
        exit
        service-instance vl999
        encapsulation dot1q 999
        rewrite pop 1
        exit
        int vl100\200\999
        ip add ___.___.___.___/___
        connect port ge1 service-instance vl100
        do wr

    Tunnel:
        en
        conf t
        int tunnel.0
        ip add ___.___.___.___/___
        ip mtu 1400
        ip tunnel ___.___.___.___ 'ip HQ-RTR' ___.___.___.___ 'BR-RTR' mode gre
    
    OSPF:
        router ospf 1
        network ___.___.___.___/___ area 0
        int tunnel.0
        ip ospf network point-to-point
        ip ospf mtu-ignore
        do sh ip ro

    NAT:
        en
        conf t
        int vl100
        ip nat inside
        int vl200
        ip nat inside
        int vl999
        ip nat inside
        int ge0
        ip nat outside
        exit
        ip nat pool VL100 ___.___.___.___-___.___.___.___
        ip nat pool VL200 ___.___.___.___-___.___.___.___
        ip nat pool VL999 ___.___.___.___-___.___.___.___
        ip nat source dynamic inside pool VL100\200\999 overload interface ge0
    
    DHCP:
        ip pool V200 ___.___.___.___-___.___.___.___
        dhcp-server 200
        pool V200 10
        mask ___ {/28}
        gateway ___.___.___.___
        dns ___.___.___.___
        domain-name _________ {au-team.irpo}
        domain-search _________ {au-team.irpo}
        exit
        exit
        int vl200
        dhcp-server 200

    SSH:
        username net_admin
        password P@ssw0rd
        role admin
    
BR-RTR:
    Baza:
        nano /etc/network/interfaces 
            auto ens___ 
            iface ens___ inet dhcp\static
            address ___.___.___.___
            gateway ___.___.___.___ (ip ISP)

        systemctl restart networking
        nano /etc/apt/sources.list 
            deb\deb-src http://mirror.yandex.ru/debian bookworm main contrib

    iptables:
        apt update
        apt install iptablas-persistent -y
        iptables -t nat -A POSTROUTING -o ens192 -j MASQUERADE
        iptables-save >> /etc/iptables/rules.v4

    tunnel:
        touch gre.up
        chmod +x gre.up
        nano gre.up 
            #!bin/bash
            ip tunnel add gre1 mode gre remote ___.___.___.___ {ip HQ-RTR} local ___.___.___.___ {ip BR-RTR} ttl 255
            ip link set gre1 up
            ip addr add ___.___.___.___/___ dev gre1 {ip tunnel}
            sysctl -p 
            {потом: systemctl restart frr}

        nano /etc/crontab 
            @reboot root /root/gre.up

        nano /etc/sysctl.list 
            net.ipv4.ip_forward=1

        gre.up
        ping {ip rotera}
    
    iptables:
        apt install iptables-persistent -y
        iptables -t nat -A POSTROUTING -o ens___ -j MASQUERADE
        iptables-save >> /etc/iptables/rules.v4

    SSH:
        apt install sudo
        adduser net_admin
        nano /etc/sudoes 
            sshuser ALL =NOPASSWD: ALL
        gpasswd -a sshuser wheel

        apt install openssh-server -y
        nano /etc/ssh/sshd_config 
            Port ___
            Allowusers      net_admin
            PasswordAuthentication yes

        systemctl restart sshd

    OSPF:
        apt install frr -y
        nano /etc/frr/daemons 
            ospfd=yes

        systemctl restart frr
        vtysh
        conf t
        router ospf
        network ___.___.___.___/___ area 0.0.0.0
        network ___.___.___.___/___ area 0.0.0.0
        exit
        int gre1
        ip ospf network point-to-point
        exit
        wr
        exit
        systemctl restart frr
        nano /gre.up 
        systemctl restart frr

HQ-SRV:
    nmcli connection del 'Проводное соеденение'
    nmcli connection add con-name "LAN" type ethernet ifname ens___ connection.autoconnect yes ipv4.method manual ipv4.adress ___.___.___.___/___ ipv4.gateway ___.___.___.___
    reboot
    nmcli connection up LAN
    nmcli connectoin show

    OSPF:
        nano /etc/resolv/conf
            nameserver 8.8.8.8\77.88.8.8

        apt install frr -y
        nano /etc/daemons
            ospfd=yes

        systemctl restart frr
        vtysh
        conf t
        router ospf
        network ___.___.___.___/___ area 0.0.0.0
        exit
        ip ospf never writes vtysh.conf
    
    DNS:
        apt-get install bind bind-utils -y
        apt-get install nano
        nano /etc/bind/options.conf 
            listen-on { 192.168.100.1;}; 
            forwardes {8.8.8.8 };
        
        nano /etc/bind/rcf1912.conf
            zone "au-team.irpo" { 
            type master;
            file "au-team.irpo";
            allow-update { none; };
            };

            zone "100.168.192.in-addr.arpa" {
            type master;
            file "rev1";
            allow-update { none; };
            };
                
            zone "200.168.192.in-addr.arpa" {
            type master;
            file "rev2";
            allow-update { none; };
            };

        cp /etc/bind/zone/localhost /rtc/bind/zone/au-team.irpo
        cp /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/rev1
        cp /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/rev2
        nano /etc/bind/zone/au-team.ipro 
            @       IN  SOA au-team.irpo. root.au-team.irpo. (
                                    2024092400  ; seriaal
                                    12H         ; refresh
                                    1H          ; retry
                                    1W          ; expire
                                    1H          ; ncache
                            )
                    IN    NS  hq-srv.au-team.irpo.
            hq-srv  IN  A   ___.___.___.___
            hq-cli  IN  A   ___.___.___.___
            hq-rtr  IN  A   ___.___.___.___
            br-rtr  IN  A   ___.___.___.___
            br-srv  IN  A   ___.___.___.___
        
        nano /etc/bind/zone/rev1
            @   IN  SOA 100.168.192.in-addr.arpa. root.au-team.irpo. (
                                   2024092400  ; seriaal
                                    12H         ; refresh
                                    1H          ; retry
                                    1W          ; expire
                                    1H          ; ncache
                            )
                IN  NS  hq-srv.au-team.irpo.
            ___ IN  PTR hq-srv.au-team.irpo.
            ___ IN  PTR hq-rtr.au-team.irpo.
        
        nano /etc/bind/zone/rev2
            @   IN  SOA 200.168.192.in-addr.arpa. root.au-team.irpo. (
                                   2024092400  ; seriaal
                                    12H         ; refresh
                                    1H          ; retry
                                    1W          ; expire
                                    1H          ; ncache
                            )
                IN  NS  hq-srv.au-team.irpo.
            ___ IN  PTR hq-cli.au-team.irpo.
        chown root:named /etc/bind/zone/au-team.irpo
        chown root:named /etc/bind/zone/rev1
        chown root:named /etc/bind/zone/rev2
        systemctl restart bind

    SSH:
        useradd -i 1010 sshuser
        passwd sshuser
        visudo /etc/sudoes (можно через nano)
            sshuser ALL=NOPASSWD: ALL
        gpasswd -a sshuser wheel

        apt-get install openssh-server -y
        systemctl daemon-reload
        systemctl enable -pnow sshd
        nano /etc/openssh/sshd_config 
            Port 2026 
            MaxAuthTries 2
            PasswordAuthentication yes
            AllowUsers     sshuser
            Banner /etc/openssh/banner

        nano /etc/openssh/banner 
            Authorized access only!!!

        systemctl restart sshd

    RAID-0
        ls /dev/ ("sdb/sdc/sdd")
        perter
        select /dev/sd_
        using /dev/sd_
        mklabel gpt
        unit s
        mkpart primary ext4 0% 100%
        select /dev/sd_
        using /dev/sd_
        mklabel gpt
        unit s
        mkpart primary ext4 0% 100%

        mdadm --create --verbose --level=0 --metadata=1.2 --chunk=256 -raid-devices=2 /dev/md/md0 /dev/sd_1 /dev/sd_1
        mdadm --detail --scan >> /etc/mdadm.conf
        adadmassemble --scan
        cat /etc/mdadm.conf
        mkfs.ext4 /dev/nd/md0
        blkid >> /etc/fstab (убираем всё с /dev кроме /dev/sr0 и /dev/md)
            proc            /proc               proc    nosuid,noexec,gid=proc          0   0
            devpts          /dev/pts            devpts  nosuid,noexec,gid=tty,mode=620  0   0
            tmpfs           /tmp                tmpfs   nosuid                          0   0
            /dev/sr0        /media/ALTLinux udf,iso9660 ro,noauto,user,utf8,nofail,comment=x-gvfs-show 0 0
            [убрать: /dev/md127:] UUID=".........." BLOCK_SIZE="4096" TYPE="ext4" [заменить на:/raid0 ext4 defaults 0 0]
        
        mkdir /raid0
        mount -all
        ls /raid0

    Docker:
        apt-get install -y lamp-server
        mount /dev/sr0 /mnt/
        ls /mnt/
        ls /mnt/web/
        cp /mnt/web/index.php /var/www/html
        cp /mnt/web/logo.png /var/www/html
        nano /var/www/html/index.php
            username = "webc";
            password = "P@ssw0rd";
            dbname = "webdb";

        systemctl enable --now mariadb
        mariadb - u root
        CREATE DATABASE webdb;
        CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';
        GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost' WITH GRANT OPTION;
        EXIT;
        mariadb -u webc -p -D webdb < /mnt/web/dump.sql
        systemctl enable --now httpd2
        [Подключаемся с клиента по ___.___.___.___]
        
BR-SRV:
    nmcli connection del 'Проводное соеденение'
    nmcli connection add con-name "LAN" type ethernet ifname ens___ connection.autoconnect yes ipv4.method manual ipv4.adress ___.___.___.___/___ ipv4.gateway ___.___.___.___
    reboot
    nmcli connection up LAN
    nmcli connectoin show

    SSH:
        useradd -i 1010 sshuser
        passwd sshuser
        visudo /etc/sudoes (можно через nano)
            sshuser ALL=NOPASSWD: ALL

        gpasswd -a sshuser wheel

        apt-get install openssh-server -y
        systemctl daemon-reload
        systemctl enable -pnow sshd
        nano /etc/openssh/sshd_config 
            Port 2026 
            MaxAuthTries 2 
            PasswordAuthentication yes 
            AllowUsers     sshuser
            Banner /etc/openssh/banner

        nano /etc/openssh/banner 
            Authorized access only!!!

        systemctl restart sshd

    ansible:
        Version1:
            apt-get install ansible -y
            ip li set mtu 1200 dev ens192 (Возможно!!!)
            nano /etc/ansible/ansible.cfg 
                host_key_checking = False

            nano /etc/ansible/hosts 
                [all:vars]
                ansible_user=sshuser
                ansible_password=P@ssw0rd
                ansible_port=2026 
                [local]
                hq-srvv ansible_host=___.___.___.___

            ansible local -m ping

        apt-get install python python-module-yaml python-module-jinja2 python-modules-json python-modules-distutils -y
        security none

        Vercion2:
            nano /ansible/hosts
                [all:vars]
                ansible_python_interpreter=/usr/bind/python3
                [hq]
                hq-srv ansible_host=___.___.___.___
                hq-srv ansible_host=___.___.___.___
                [hq:vars]
                ansible_user=sshuser
                ansible_password=P@ssw0rd
                ansible_port___
                [ecorouter]
                hq-rtr ansible_host=___.___.___.___
                [ecorouter:vars]
                ansible_network_os=iso
                ansible_user=net_admin
                ansible_password=P@ssw0rd
                ansible_connection=networl_cli

            ansible all -m ping

        Version3:

    Docker:
        apt-get install -y docker-engine docker-compose-v2
        systemctl enable --now docker.service
        mount /dev/sr0 /mnt/
        ls /mnt/
        ls /mnt/docker/
        docker load < /mnt/docker/site_latest.tar
        docker load < /mnt/docker/mariadb_latest.tar
        docker image ls
        nano compose.yaml
            services:
                database:
                    container_name: db
                    image: mariadb: 10.11
                    restart: always
                    ports:
                        - "3306:3306"
                    environment:
                        MARIADB_DATABASE: "testdb"
                        MARIADB_USER: "testc"
                        MARIADB_PASSWORD: "P@ssw0rd"
                        MARIADB_ROOT_PASSWORD: "toor"
                
                app:
                    container_name: testapp
                    image: site:latest
                    restart: always
                    ports:
                        - "8080:8000"
                    environment:
                        DB_TYPE: "maria"
                        DB_HOST: "___.___.___.___"
                        DB_PORT: "3306"
                        DB_NAME: "testdb"
                        DB_USER: "testc"
                        DB_PASS: "P@ssw0rd"
                    depends_on:
                        - database
        
        docker compose up -d
        docker compose ps
        [подключаемся с клиента по адресу ___.___.___.___:8080]

HQ-CLI:
    apt-get install -y yandex-browser-stable
