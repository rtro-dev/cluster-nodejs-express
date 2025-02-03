# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "practica" do |p|
      p.vm.box = "debian/bookworm64"
      p.vm.hostname = "practica"
      p.vm.network "forwarded_port", guest: 8080, host: 8080
      p.vm.network "private_network", ip: "192.168.12.12"
  end # practica
end # cofig