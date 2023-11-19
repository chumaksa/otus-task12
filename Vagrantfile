Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.define "selinux"
  config.vm.hostname = "selinux"
  config.vm.network "forwarded_port", guest: 4881, host: 4881
end
