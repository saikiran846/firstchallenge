---
- hosts: all
  become: yes
  vars:
    nagios_version: '4.4.5'
    nagios_plugin_version: '2.2.1'
    packages_centos:
      - gcc 
      - glibc 
      - glibc-common
      - make
      - wget
      - unzip
      - httpd
      - php
      - php-cli
      - postfix
      - gd
      - gd-dev
      - gettext 
      - automake 
      - autoconf
      - openssl-devel
      - net-snmp 
      - net-snmp-utils 
      - epel-release
      - perl-Net-SNMP
    nagios_code_location: /tmp/nagioscore.tar.gz
    nagios_plugin_location: /tmp/nagios-plugins.tar.gz
    tasks:
    - name: fail for unsupported os
      fail:
        msg: This playbook runs only on centos
      when: ansible_os_family != 'Redhat'
    - name: install packages for centos
      yum:
        name: "{{ item }}"
        update_cache: yes
        state: present
      loop: "{{ packages_centos }}"
      tags:
        - nagios
    - name: Download the source to temp folder # Downloading zip file
      get_url:
        dest: "{{ nagios_code_location }}"
        url: "https://github.com/NagiosEnterprises/nagioscore/archive/nagios-{{ nagios_version }}.tar.gz"
      tags:
        - nagios
    - name: untar the code
      unarchive:
        src: "{{ nagios_code_location }}"
        dest: "/tmp/"
    - name: Compile Code
      shell:
        cmd: sudo ./configure --with-httpd-conf=/etc/apache2/sites-enabled > configure.log && sudo make all > makeoutput.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Create User and Group
      shell:
        cmd: sudo make install-groups-users > installgroups.log && sudo usermod -a -G nagios apache > usermod.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Binaries
      shell:
        cmd: sudo make install > makeinstall.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Service / Daemon
      shell:
        cmd: sudo make install-daemoninit > daemoninit.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Command Mode
      shell:
        cmd: sudo make install-commandmode > commandmode.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Configuration Files
      shell:
        cmd: sudo make install-config > config.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Apache Config Files
      shell:
        cmd: sudo make install-webconf > webconf.log && sudo a2enmod rewrite > a2enmodrewrite.log && sudo a2enmod cgi > cgi.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install latest passlib with pip
      pip: name=passlib
      tags:
        - nagios
    - name: Create nagiosadmin user account
      htpasswd:
        path: /usr/local/nagios/etc/htpasswd.users 
        name: nagiosadmin 
        password: nagiosadmin 
      tags:
        - nagios
      notify:
       - restarting httpd server  
       - restarting nagios 
    - name: Download the plugin to temp folder
      get_url:
        dest: "{{ nagios_plugin_location }}"
        url: "https://github.com/nagios-plugins/nagios-plugins/archive/release-{{ nagios_plugin_version }}.tar.gz"
      tags:
        - plugin
    - name: untar the plugin code
      unarchive:
        src: "{{ nagios_plugin_location }}"
        dest: "/tmp/"
      tags:
        - plugin
    - name: Compile + Install
      shell:
        cmd: sudo ./tools/setup > setup.log && sudo ./configure > configue.log && sudo make > make.log && sudo make install
      args:
        chdir: "/tmp/nagios-plugins-release-{{ nagios_plugin_version }}/"
      tags:
        - plugin
      notify:
        - restarted nagios  
        - restarted httpd server 
    - name: restart nagios  
       service:
         name: nagios
         state: restarted
      tags:
        - plugin
  handlers:
    - name: restart apache 
      service :
        name: httpd
        enabled: yes
        state: restarted
    - name: restart nagios 
      service:
        name: nagios
        enabled: yes
        state: restarted
