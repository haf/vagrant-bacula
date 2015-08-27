# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "chef/centos-7.1"
  config.vm.box_check_update = true
  #config.vm.network "forwarded_port", guest: 80, host: 8080
  #config.vm.synced_folder "../data", "/vagrant_data"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
  end

  config.vm.provision "shell", inline: <<-SHELL
    set -x
    set -e
    cp /vagrant/bareos.repo /etc/yum.repos.d/bareos.repo
    yum update -y
    yum upgrade -y
    yum install -y vim
  SHELL

  config.vm.define 'bareos-dir', primary: true do |s|
    s.vm.hostname = 'backup-dir'
    s.vm.network "private_network", ip: '192.168.111.10'
    s.vm.provision "shell", inline: <<-SHELL
      yum install -y bareos bareos-database-postgresql postgresql-server
      postgresql-setup initdb
      systemctl enable postgresql
      systemctl start postgresql

      if [[ ! -f /etc/bareos-inited ]]; then
        . /usr/lib/bareos/scripts/bareos-config-lib.sh

        db_name="${db_name:-`get_database_name bareos`}"
	db_user="${db_user:-`get_database_user bareos`}"
	dir_user=`get_user_dir`
	dir_group=`get_group_dir`
	default_db_type=`get_database_driver_default`
        working_dir=`get_working_dir`

        PSQLVERSION=`su - postgres -c "psql -d template1 -c 'SELECT version()' 2>/dev/null" | \
                     awk '/PostgreSQL/ { print $2 }' | \
                     cut -d '.' -f 1,2`

        if [ -z "${PSQLVERSION}" ]; then
          echo "Unable to determine PostgreSQL version."
          exit 1
        fi

        ENCODING="ENCODING 'SQL_ASCII' LC_COLLATE 'C' LC_CTYPE 'C'"

        su - postgres -c <<END
psql -f - -d template1 $* <<END2
\set ON_ERROR_STOP on
CREATE DATABASE ${db_name} $ENCODING TEMPLATE template0;
ALTER DATABASE ${db_name} SET datestyle TO 'ISO, YMD';
END2
END

        if su - postgres -c 'psql -l ${dbname}' | grep " ${db_name}.*SQL_ASCII" >/dev/null; then
          echo "Database encoding OK"
        else
          echo "Database encoding bad. Do not use this database"
        fi

        su - postgres -c /usr/bin/env db_name=bareos db_user=beros /usr/lib/bareos/scripts/make_bareos_tables postgresql
        /usr/lib/bareos/scripts/grant_bareos_privileges
        touch /etc/bareos-inited
      fi

      systemctl start bareos-dir
      systemctl start bareos-sd
      systemctl start bareos-fd

      # TODO: open ports 9101, 9102, 9103 in FW
      # tail -f /var/log/bareos/bareos.log
    SHELL
  end

  config.vm.define 'client-1' do |c|
    c.vm.hostname = 'client-1'
    c.vm.network "private_network", ip: "192.168.111.11"
    c.vm.provision "shell", inline: <<-SHELL
      if [[ $(getent passwd eventstore >/dev/null) -ne 0 ]]; then
        # Create eventstore
        mkdir -p /var/lib/eventstore && \
          groupadd -r eventstore && \
          useradd -d /var/lib/eventstore -r -g eventstore eventstore && \
          chown -R eventstore:eventstore /var/lib/eventstore && \
          chmod -R 755 /var/lib/eventstore
          touch /etc/sysconfig/eventstore
      fi
    SHELL
  end

  config.vm.define 'client-2' do |c|
    c.vm.hostname = 'client-2'
    c.vm.network "private_network", ip: "192.168.111.12"
    c.vm.provision "shell", inline: <<-SHELL
      # Create consul dir
      if [[ $(getent passwd consul >/dev/null) -ne 0 ]]; then
        mkdir -p /var/lib/consul && \
          groupadd -r consul && \
          useradd -d /var/lib/consul -r -g consul consul && \
          chown -R consul:consul /var/lib/consul && \
          chmod -R 755 /var/lib/consul
          touch /etc/sysconfig/consul
      fi
    SHELL
  end
end
