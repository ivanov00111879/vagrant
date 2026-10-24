Vagrant.configure("2") do |config|
  # Ubuntu
  config.vm.define "ubuntu-26.04" do |srv|
    srv.vm.boot_timeout = 3600
    srv.vm.box = "bento/ubuntu-26.04"
    srv.vm.provider "virtualbox" do |vb|
      vb.memory = "4096"
      vb.cpus = 2
      vb.name = "ubuntu-26.04"
      vb.customize ["modifyvm", :id, "--audio", "none"]
      vb.customize ["modifyvm", :id, "--graphicscontroller", "vmsvga"]
      vb.customize ["modifyvm", :id, "--accelerate3d", "off"]
      vb.customize ["modifyvm", :id, "--vram", "32"]
      vb.customize ["modifyvm", :id, "--nested-hw-virt", "off"]
    end

    # Синхронизация папок
    srv.vm.synced_folder "./", "/vagrant"

    # Подключение дисков
    srv.vm.disk :disk, size: "1GB", name: "disk1"
    srv.vm.disk :disk, size: "1GB", name: "disk2"

    # Проброс порта для HTTP (будет доступен по localhost:8080)
    srv.vm.network "forwarded_port", guest: 80, host: 8080

    # Приватная сеть
    #srv.vm.network "private_network", ip: "192.168.56.150"

    # Автоматическая настройка системы при старте
    srv.vm.provision "shell", inline: <<-SHELL
      #!/bin/bash
      set -euo pipefail

      echo "Подготовка дисков"
      
      # Поиск дисков без точки монтирования размером 1ГБ
      DISKS=($(lsblk -dno NAME,SIZE,MOUNTPOINT | awk '$2=="1G" && $3=="" {print "/dev/"$1}' | sort))

      if [ ${#DISKS[@]} -eq 0 ]; then
          echo "Диски размером 1ГБ не найдены или уже примонтированы."
      else
          # Перебор найденных дисков
          for i in "${!DISKS[@]}"; do
              DISK="\${DISKS[\$i]}"
              NUMBER=\$((i + 1))
              MOUNT_DIR="/mnt/data\${NUMBER}"

              mkdir -p "\$MOUNT_DIR"

              # Форматируем, только если файловая система отсутствует
              if ! blkid "\$DISK" >/dev/null 2>&1; then
                  echo "Форматирование устройства \$DISK в ext4..."
                  mkfs.ext4 -F "\$DISK"
              fi

              # Получаем UUID диска
              UUID=\$(blkid -s UUID -o value "\$DISK")

              # Безопасное добавление в /etc/fstab
              if ! grep -q -E "UUID=\$UUID|\$MOUNT_DIR" /etc/fstab; then
                  echo "Добавление \$DISK (\$UUID) в /etc/fstab..."
                  echo "UUID=\$UUID \$MOUNT_DIR ext4 defaults 0 2" >> /etc/fstab
              fi
          done

          # Накатываем изменения fstab
          echo "Монтирование файловых систем..."
          mount -a
      fi

      echo "Установка и настройка Apache"
      
      # Обновление пакетов
      apt-get update
      
      # Установка Apache
      apt-get install -y apache2
      
      # Отключаем стандартную папку /var/www/html
      a2dissite 000-default.conf
      
      # Создаем конфигурацию для нашего сайта (экранируем \$ в *:80)
      echo "<VirtualHost *\:80>
          DocumentRoot /vagrant
          <Directory /vagrant>
              Options Indexes FollowSymLinks
              AllowOverride All
              Require all granted
          </Directory>
      </VirtualHost>" | tee /etc/apache2/sites-available/vagrant.conf
      
      # Включаем наш сайт
      a2ensite vagrant.conf
      
      # Включение mod_rewrite
      a2enmod rewrite
      
      # Перезапуск Apache
      systemctl restart apache2

      echo "Настройка успешно завершена!"
    SHELL
  end
end