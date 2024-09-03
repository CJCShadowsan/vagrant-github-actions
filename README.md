# vagrant-github-actions
[![Build Status](https://github.com/cjcshadowsan/vagrant-github-actions/workflows/vagrant-up/badge.svg)](https://github.com/jonashackt/vagrant-github-actions/actions)

Example project showing how to run a Vagrant box on GitHub Actions - Originally forked from [jonashackt](https://github.com/jonashackt/vagrant-github-actions)

Note that since this project originally was created, [there's Vagrant activated (incl. nested virtualization) on GitHub Action MacOS environments and on Linux environments for larger runners](https://github.com/actions/virtual-environments/issues/433) ([see here for Linux](https://github.com/actions/virtual-environments/issues/183)).

### How to run Vagrant on GitHub Actions

So let's first add a [Vagrantfile](Vagrantfile) like this:

```
Vagrant.configure("2") do |config|
    config.vm.box = "generic-x64/ubuntu2310"

    config.vm.define 'ubuntu'

    # Prevent SharedFoldersEnableSymlinksCreate errors
    config.vm.synced_folder ".", "/vagrant", disabled: true
end
```

Now add a MacOS environment powered GitHub Actions `vagrant-up.yml`:

```yaml
name: vagrant-up

on: [push]

jobs:
  vagrant-up:
    runs-on: macos-latest

    steps:
      - uses: actions/checkout@v2

      - name: Cache Vagrant boxes
        uses: actions/cache@v2
        with:
          path: ~/.vagrant.d/boxes
          key: ${{ runner.os }}-vagrant-${{ hashFiles('Vagrantfile') }}
          restore-keys: |
            ${{ runner.os }}-vagrant-

      - name: Show Vagrant version
        run: vagrant --version

      - name: Run vagrant up
        run: vagrant up

      - name: ssh into box after boot
        run: vagrant ssh -c "echo 'hello world!'"

```

Or a linux environment:

```yaml
name: vagrant-up

on: [push]

jobs:
  vagrant-up:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Cache Vagrant boxes
        uses: actions/cache@v2
        with:
          path: ~/.vagrant.d/boxes
          key: ${{ runner.os }}-vagrant-${{ hashFiles('Vagrantfile') }}
          restore-keys: |
            ${{ runner.os }}-vagrant-

      - name: setup vagrant repo GPG key
        run: wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

      - name: setup hashicorp repo
        run: echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

      - name: install vagrant
        run: sudo apt update && sudo apt install vagrant

      - name: setup VirtualBox repo GPG key
        run: wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor --yes --output /usr/share/keyrings/oracle-virtualbox-2016.gpg

      - name: setup VirtualBox repo
        run: echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] http://download.virtualbox.org/virtualbox/debian $(. /etc/os-release && echo "$VERSION_CODENAME") contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list
        
      - name: install VirtualBox
        run: sudo apt update && sudo apt install virtualbox-6.1

      - name: Show Vagrant version
        run: vagrant --version

      - name: Enable KVM group perms
        run: |
            echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
            sudo cat /etc/udev/rules.d/99-kvm4all.rules
            sudo udevadm control --reload-rules
            sudo udevadm trigger --name-match=kvm
            sudo ls /dev/kvm
      
      - name: Run vagrant up
        run: vagrant up --provider=virtualbox

      - name: ssh into box after boot
        run: vagrant ssh -c "echo 'hello world!'"
```

Check here for the available [environments on GitHub Actions](https://github.com/actions/virtual-environments) for more details.

We also use the https://github.com/actions/cache action here to cache our Vagrant boxes for further runs. Using `hashFiles('Vagrantfile')` will make sure we only create a new cache, if our `Vagrantfile` changed (and thus assume, that the box name is also different).

