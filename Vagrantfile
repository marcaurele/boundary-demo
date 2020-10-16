# -*- mode: ruby -*-
# vi: set ft=ruby :

#ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'
VMS = 3

Vagrant.configure("2") do |config|
  (0..VMS).each do |n|
    ip_addr = "10.100.10.#{100+n}"
    if n == 0
      name = "boundary-controller"
    elsif n == 1
      name = "boundary-worker"
    else
      name = "boundary-%.2d" % n
    end
    config.vm.define name do |subconfig|
      subconfig.vm.box = "generic/debian10"
      subconfig.vm.hostname = name
      subconfig.vm.box_check_update = false
      subconfig.vm.network :private_network, ip: "#{ip_addr}"
      # Don't sync the repo into the VM
      subconfig.vm.synced_folder "../data", "/vagrant_data", type: 'rsync', disabled: true

      subconfig.vm.provider "libvirt" do |v|
        v.memory = 1024
        v.cpus = 2
      end

      # Only load extra SSH key if present
      if ENV.has_key?('SSH_KEY')
        SSH_KEY = ENV['SSH_KEY']
        puts("Inserting SSH key #{Dir.home}/.ssh/#{SSH_KEY}.pub into vagrant box")
        subconfig.vm.provision "shell" do |s|
          s.privileged = false
          ssh_pub_key = File.readlines("#{Dir.home}/.ssh/#{SSH_KEY}.pub").first.strip
          s.inline = <<-EOF
echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
EOF
        end
      end

      subconfig.vm.provision "shell",
                             privileged: false,
                             inline: <<-EOF
set -xe
sudo apt-get update
sudo apt-get install -qy \
  apt-transport-https \
  ca-certificates \
  curl \
  unzip

# Download boundary
curl --silent -o /tmp/boundary.zip https://releases.hashicorp.com/boundary/0.1.0/boundary_0.1.0_linux_amd64.zip
sudo mkdir -p /usr/local/bin/
sudo unzip /tmp/boundary.zip -d /usr/local/bin/
rm -f /tmp/boundary.zip

# Define Systemd service as controller or worker
if [ "boundary-controller" = "#{name}" ]
then
  sudo apt-get install -qy postgresql-all
  sudo -u postgres bash -c "psql -c \\\"CREATE USER vagrant WITH PASSWORD 'vagrant'; ALTER USER vagrant WITH SUPERUSER;\\\""
  sudo -u postgres bash -c "createdb -O vagrant boundary"
  sudo groupadd -r boundary || true
  sudo useradd -Mr -g boundary boundary || true
  sudo mkdir /etc/boundary/
  cat <<EOH | sudo tee /etc/boundary/controller.hcl
disable_mlock = true
controller {
  name = "vagrant-controller"
  description = "An example controller on vagrant"
  database {
    url = "postgresql://vagrant:vagrant@localhost:5432/boundary"
  }
}

listener "tcp" {
  address = "0.0.0.0"
  purpose = "api"
  tls_disable = true
}

listener "tcp" {
  address = "10.100.10.100"
  purpose = "cluster"
  tls_disable = true
}

kms "aead" {
  purpose = "root"
  aead_type = "aes-gcm"
  key = "sP1fnF5Xz85RrXyELHFeZg9Ad2qt4Z4bgNHVGtD6ung="
  key_id = "global_root"
}

kms "aead" {
  purpose = "worker-auth"
  aead_type = "aes-gcm"
  key = "8fZBjCUfN0TzjEGLQldGY4+iE9AkOvCfjh7+p0GtRBQ="
  key_id = "global_worker-auth"
}
EOH
  sudo chown boundary: -R /etc/boundary
  sudo /usr/local/bin/boundary database init -config /etc/boundary/controller.hcl || true
  cat <<EOH | sudo tee /lib/systemd/system/boundary-controller.service
[Unit]
Description=Boundary Controller

[Service]
ExecStart=/usr/local/bin/boundary server -config /etc/boundary/controller.hcl
User=boundary
Group=boundary
LimitMEMLOCK=infinity
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK

[Install]
WantedBy=multi-user.target
EOH
  sudo chmod 664 /lib/systemd/system/boundary-controller.service
  sudo systemctl daemon-reload
  sudo systemctl enable boundary-controller.service
  sudo systemctl start boundary-controller.service

elif [ "boundary-worker" = "#{name}" ]
then
  sudo groupadd -r boundary || true
  sudo useradd -Mr -g boundary boundary || true
  sudo mkdir /etc/boundary
  cat <<EOH | sudo tee /etc/boundary/worker.hcl
disable_mlock = true
worker {
  name = "vagrant-worker"
  description = "An example worker on vagrant"
  public_addr = "#{ip_addr}"

  controllers = [
    "10.100.10.100"
  ]
}

listener "tcp" {
  purpose = "proxy"
  tls_disable = true
}

kms "aead" {
  purpose = "worker-auth"
  aead_type = "aes-gcm"
  key = "8fZBjCUfN0TzjEGLQldGY4+iE9AkOvCfjh7+p0GtRBQ="
  key_id = "global_worker-auth"
}
EOH
  sudo chown boundary: -R /etc/boundary/
  cat <<EOH | sudo tee /lib/systemd/system/boundary-worker.service
[Unit]
Description=Boundary Worker

[Service]
ExecStart=/usr/local/bin/boundary server -config /etc/boundary/worker.hcl
User=boundary
Group=boundary
LimitMEMLOCK=infinity
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK

[Install]
WantedBy=multi-user.target
EOH
  sudo chmod 664 /lib/systemd/system/boundary-worker.service
  sudo systemctl daemon-reload
  sudo systemctl enable boundary-worker.service
  sudo systemctl start boundary-worker.service
fi
EOF
    end
  end
end
