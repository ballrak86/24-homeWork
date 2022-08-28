#24-homeWork. Сетевые пакеты. VLAN'ы. LACP  
## Описание файлов в директории
logFileFull.log - полный лог выполнения  

Vagrant_folder - все что понадобится для поднятия ВМ и краткое описание файлов в ней  
Vagrantfile - вагрант файл  
provisioning - директория с файлами провижинга  
```
provisioning/
├── library
│   └── iptables_raw.py
├── playbook.yml
└── roles
    ├── centralRouter
    ├── inetRouter
    ├── office1Router
    └── testClientServer
```

## Описание как запустить виртуальную машину (кратко)
```
vagrant up
vagrant ssh vagrant ssh testServer2
[vagrant@testServer2 ~]$ ping 10.10.10.254
[vagrant@testServer2 ~]$ ping 10.10.10.2
[vagrant@testServer2 ~]$ ping e1.ru

vagrant ssh inetRouter
[vagrant@inetRouter ~]$ ifdown eth1
[vagrant@inetRouter ~]$ ping 192.168.255.2
[vagrant@inetRouter ~]$ cat /proc/net/bonding/bond0
```

## кратко о playbook
### inetRouter и centralRouter
Настраиваем бондинг с резервацией. Ранее подготовленные файлы копируем в нужную директорию и перезапускаем сеть.
```
- name: tamplate bonding
  copy:
    src: "{{ item }}"
    dest: /etc/sysconfig/network-scripts/
    owner: root
    group: root
    mode: 0644
  with_fileglob:
    - ifcfg*
    - route-bond0
```
bond из двух интерфейсов на машине centralRouter и inetRouter, параметры, которые использованы в конфигурационном файле интерфейса bond0:  
* mode=1 - определяет политику поведения объединенных интерфейсов. 1 - это политика активный-резервный.  
* miimon=100 - Устанавливает периодичность MII мониторинга в миллисекундах. Фактически монитринг будет осуществляться 10 раз в секунду (1с = 1000 мс).  
* fail_over_mac=1 - Определяет как будут прописываться MAC адреса на объединенных интерфейсах в режиме active-backup при переключении интерфесов.  
```
cat /etc/sysconfig/network-scripts/ifcfg-bond0
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
IPADDR=192.168.255.1
PREFIX=30
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
NAME=bond0
DEVICE=bond0
ONBOOT=yes
TYPE=Bond
BONDING_OPTS="mode=1 miimon=100 fail_over_mac=1"
```

### office1Router1 и office1Router2
Для того чтобы обойти проблему с одинаковыми адресами в сети, для каждого vlan поднят отдельный office1Router, с настроеным маскарадингом.
```
- name: template vlan
  template:
    src: ifcfg-eth2.vlan.j2
    dest : /etc/sysconfig/network-scripts/ifcfg-eth2.{{ vlan }}
    owner: root
    group: root
    mode: '0644'
  notify:
    - up interface_vlan

- name: MASQUERADE
  iptables_raw:
    name: MASQUERADE
    table: nat
    rules: |
      -A POSTROUTING -s 10.10.10.0/24 -o eth1 -j MASQUERADE
```
### testClientServer
Переносим шаблоны на ВМ testClient и testServer, для настройки vlan c тегами 10 и 20
```
- name: template vlan
  template:
    src: ifcfg-eth1.vlan.j2
    dest : /etc/sysconfig/network-scripts/ifcfg-eth1.{{ vlan }}
    owner: root
    group: root
    mode: '0644'
  notify:
    - up interface_vlan
```

Для переиспользования ролей задал переменные в playbook
```
- hosts: office1Router1
  become: true
  vars:
    vlan: 10
    ip_addr: 10.10.10.2
  roles:
    - office1Router

- hosts: office1Router2
  become: true
  vars:
    vlan: 20
    ip_addr: 10.10.10.2
  roles:
    - office1Router

- hosts: testClient1
  become: true
  vars:
    vlan: 10
    ip_addr: 10.10.10.254
  roles:
    - testClientServer
...
```


📚Домашнее задание/проектная работа разработано(-на) для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)