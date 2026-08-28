# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Local Kubespray lab: 3x Ubuntu 24.04 (bento) nodes.
#   knode1 -> control-plane + etcd + worker (low-resource lab, no HA)
#   knode2 -> worker
#   knode3 -> worker
#
# Box: bento/ubuntu-24.04 -- traditional packer box with VirtualBox Guest
# Additions baked in, so the default /vagrant synced folder (this project dir
# -> /vagrant in each guest) works with no extra config.
#
# Each node gets TWO extra raw 10GB disks, attached to the box's existing
# "SATA Controller" (AHCI, 14 ports; the OS disk sits on port 0) at ports 1-2.
# They are attached only -- NOT partitioned or formatted -- and are reserved
# for a future MinIO + DirectPV playbook, which discovers/formats the raw
# devices itself. In the guest they show up as /dev/sdb and /dev/sdc.
#
# Provisioning: `vagrant up` (or `vagrant provision`) runs site.yml once against
# all three nodes, after the last VM boots. Ansible runs on the HOST, so the
# project venv (ansible-core 2.18 / Kubespray pin) must be on PATH:
#   source venv/bin/activate && vagrant provision
# The playbook still works standalone too:
#   ansible-playbook -i inventory/local_vagrant site.yml

require 'fileutils'

boxes = [
  { name: "knode1", hostname: "knode1.local", ip: "192.168.56.111", memory: 4096, cpus: 2 },
  { name: "knode2", hostname: "knode2.local", ip: "192.168.56.112", memory: 3072, cpus: 2 },
  { name: "knode3", hostname: "knode3.local", ip: "192.168.56.113", memory: 3072, cpus: 2 }
]

box_image  = "bento/ubuntu-24.04"
osd_path   = "/data/vboxdisks"    # where the extra VirtualBox data disks (.vdi) live
sata_ctl   = "SATA Controller"    # the box's existing OS-disk controller (AHCI, 14 ports)
disk_count = 2                    # extra raw disks per node
disk_mb    = 10240                # 10 GiB each
disk_port0 = 1                    # first free SATA port (OS disk is on port 0)

# Vagrant's insecure keypair -- `config.ssh.insert_key = false` keeps it
# authorized on every box. Ansible talks to the nodes over the private network
# using the project inventory, which doesn't hard-code a key path, so pass it
# explicitly.
insecure_key = [
  "~/.vagrant.d/insecure_private_keys/vagrant.key.rsa",
  "~/.vagrant.d/insecure_private_key"
].map { |p| File.expand_path(p) }.find { |p| File.exist?(p) }

Vagrant.require_version ">= 2.0.0"

Vagrant.configure("2") do |config|
  config.ssh.insert_key  = false
  config.vm.boot_timeout = 600

  FileUtils.mkdir_p(osd_path)

  boxes.each_with_index do |box, index|
    config.vm.define box[:name] do |node|
      node.vm.box      = box_image
      node.vm.hostname = box[:hostname]
      node.vm.network "private_network", ip: box[:ip]

      node.vm.provider "virtualbox" do |vb|
        vb.name   = box[:name]
        vb.memory = box[:memory]
        vb.cpus   = box[:cpus]

        # Create + attach the raw data disks once. Guarded so provider
        # customizations (replayed on every `vagrant up` / `reload`) don't
        # re-attach media that is already present.
        attached = system(
          "VBoxManage showvminfo #{box[:name]} --machinereadable 2>/dev/null " \
          "| grep -q '#{box[:name]}-data-disk#{disk_count}.vdi'"
        )

        unless attached
          (0...disk_count).each do |i|
            disk = File.join(osd_path, "#{box[:name]}-data-disk#{i + 1}.vdi")
            unless File.exist?(disk)
              vb.customize ['createhd', '--filename', disk,
                            '--size', disk_mb, '--variant', 'Standard']
            end
            vb.customize ['storageattach', :id,
                          '--storagectl', sata_ctl,
                          '--port', disk_port0 + i,
                          '--device', 0,
                          '--type', 'hdd',
                          '--medium', disk]
          end
        end
      end

      # Attach the Ansible run to the LAST node so it fires once, after every
      # VM is up, against the whole inventory. `vagrant provision knode1/knode2`
      # is therefore a no-op -- use `vagrant provision` (or `vagrant provision
      # knode3`) to (re-)run site.yml.
      next unless index == boxes.size - 1

      node.vm.provision "ansible" do |ansible|
        ansible.compatibility_mode = "2.0"
        ansible.playbook           = "site.yml"
        ansible.inventory_path     = "inventory/local_vagrant"
        ansible.limit              = "all"
        ansible.galaxy_role_file   = "roles/requirements.yml"
        ansible.galaxy_roles_path  = "roles"
        ansible.galaxy_command     = "ansible-galaxy install -r %{role_file} -p %{roles_path}"
        ansible.raw_arguments      = ["--private-key=#{insecure_key}"] if insecure_key
      end
    end
  end
end
