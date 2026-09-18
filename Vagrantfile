# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Local Kubespray lab: 4x Ubuntu 24.04 (bento) nodes.
#   cp1 -> control-plane + etcd only (low-resource lab, no HA)
#   kn1 -> worker
#   kn2 -> worker
#   kn3 -> worker
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
# all nodes, after the last VM boots. Ansible runs on the HOST, so the
# project venv (ansible-core 2.18 / Kubespray pin) must be on PATH:
#   source venv/bin/activate && vagrant provision
# The playbook still works standalone too:
#   ansible-playbook -i inventory/local_vagrant site.yml

require 'fileutils'

boxes = [
  { name: "cp1", hostname: "cp1.local", ip: "192.168.56.111", memory: 2048, cpus: 2 },
  { name: "kn1", hostname: "kn1.local", ip: "192.168.56.112", memory: 4096, cpus: 4 },
  { name: "kn2", hostname: "kn2.local", ip: "192.168.56.113", memory: 4096, cpus: 4 },
  { name: "kn3", hostname: "kn3.local", ip: "192.168.56.114", memory: 4096, cpus: 4 }
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
        # Faster NICs
        vb.customize ["modifyvm", :id, "--nictype1", "virtio"]
        vb.customize ["modifyvm", :id, "--nictype2", "virtio"]

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
      # VM is up, against the whole inventory. `vagrant provision cp1/kn1`
      # is therefore a no-op -- use `vagrant provision` (or `vagrant provision
      # kn3`) to (re-)run site.yml.
      next unless index == boxes.size - 1

      # Install the Galaxy role(s) via a host trigger, NOT via Vagrant's
      # ansible.galaxy_* options. Those options export ANSIBLE_ROLES_PATH,
      # which overrides `roles_path` in ansible.cfg and hides the vendored
      # Kubespray roles (the run then fails on 'dynamic_groups' et al.).
      # Letting ansible.cfg own roles_path keeps ./roles + ./kubespray/roles
      # both on the search path.
      node.trigger.before :provision do |t|
        t.name = "ansible-galaxy install"
        t.run  = { inline: "ansible-galaxy install -r roles/requirements.yml -p roles" }
      end

      node.vm.provision "ansible" do |ansible|
        ansible.compatibility_mode = "2.0"
        ansible.playbook           = "site.yml"
        ansible.inventory_path     = "inventory/local_vagrant"
        ansible.limit              = "all"
        ansible.raw_arguments      = ["--private-key=#{insecure_key}"] if insecure_key
      end
    end
  end
end
