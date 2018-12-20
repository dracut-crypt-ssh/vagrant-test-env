Vagrant.configure('2') do |config|
  nodename = 'dracut-crypt-ssh-test'
  config.vm.box = 'bento/centos-7.5'
  config.vm.hostname = nodename

  config.vm.provider :virtualbox do |v|
    v.gui = true

    # add the second disk, which will be encrypted with luks
    vb_folder = `VBoxManage list systemproperties | grep "Default machine folder"`.split(':')[1].strip
    second_disk = File.join(vb_folder, nodename, "disk2-#{nodename}.vdi")
    unless File.exist?(second_disk)
      v.customize ['createhd', '--filename', second_disk, '--format', 'VDI',
                   '--size', 10 * 1024] # 10 GB
    end
    v.customize ['storageattach', :id, '--storagectl', 'SATA Controller',
                 '--port', 1, '--device', 0, '--type', 'hdd', '--medium',
                 second_disk]
  end

  # prepare the second disk: create encryption, filesystem and some test content
  config.vm.provision 'shell', inline: <<~SCRIPT
    yum -y install cryptsetup
    parted /dev/sdb mklabel msdos --script
    parted /dev/sdb mkpart primary 0% 100% --script
    echo "1234" | cryptsetup luksFormat -c aes-xts-plain64 -s 512 -q --force-password /dev/sdb1
    UUID=$(blkid -o value -s UUID /dev/sdb1)
    cat <<EOF > /etc/crypttab
      luks-${UUID} UUID=${UUID} none
    EOF
    mkdir -p /mnt/crypt
    echo "/dev/mapper/luks-${UUID} /mnt/crypt ext4 defaults 0 0" >> /etc/fstab

    echo "1234" | cryptsetup luksOpen /dev/sdb1 luks-${UUID}
    mkfs.ext4 /dev/mapper/luks-${UUID}
    mount /mnt/crypt
    touch /mnt/crypt/testfile

    echo 'add_dracutmodules+="crypt"' >> /etc/dracut.conf.d/crypt.conf
    dracut -f
  SCRIPT
end
