Vagrant.configure("2") do |config|
    config.vm.box = "generic-x64/ubuntu2310"

    config.vm.define 'ubuntu'

    # Prevent SharedFoldersEnableSymlinksCreate errors
    config.vm.synced_folder ".", "/vagrant", disabled: true
end
