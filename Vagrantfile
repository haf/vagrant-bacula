# -*- mode: ruby -*-
# vi: set ft=ruby :

$maria_pass = '12kdakj32j9d'

Vagrant.configure(2) do |config|
  config.vm.box = "chef/centos-7.1"
  config.vm.box_check_update = true
  # config.vm.network "forwarded_port", guest: 80, host: 8080
  # config.vm.network "private_network", ip: "192.168.33.10"
  # config.vm.synced_folder "../data", "/vagrant_data"
  config.vm.provision "shell", inline: <<-SHELL
    cd /root/

    yum update -y && yum upgrade -y
    yum install -y curl unzip epel-release yum-utils nginx vim ca-certificates jq
    yum install -y bacula-director bacula-storage bacula-console bacula-client mariadb-server
    systemctl start mariadb
    /usr/libexec/bacula/grant_mysql_privileges
    /usr/libexec/bacula/create_mysql_database -u root
    /usr/libexec/bacula/make_mysql_tables -u bacula

    # Make sure that NOBODY can access the server without a password
    mysql -e "UPDATE mysql.user SET Password = PASSWORD('#{$maria_pass}') WHERE User = 'root'"
    # Kill the anonymous users
    mysql -e "DROP USER ''@'localhost'"
    # Because our hostname varies we'll use some Bash magic here.
    mysql -e "DROP USER ''@'$(hostname)'"
    # Kill off the demo database
    mysql -e "DROP DATABASE test"
    # Make our changes take effect
    mysql -e "FLUSH PRIVILEGES"

    systemctl enable mariadb

    chmod 755 /etc/bacula/scripts/delete_catalog_backup
    mkdir -p /bacula/backup /bacula/restore
    chown -R bacula:bacula /bacula
    chmod -R 700 /bacula

  SHELL

  config.vm.define 'bacula-server', primary: true do |node|
    node.vm.hostname = 'bacula-server'
  end

  config.vm.define 'bacula-client' do |node|
    node.vm.hostname = 'bacula-client'
  end
end
