Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.define "selinux"
  config.vm.hostname = "selinux"
  config.vm.network "private_network", ip: "192.168.56.60"
end
