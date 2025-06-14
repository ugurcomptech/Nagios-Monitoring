# Nagios Core Kurulum ve Yapılandırma Kılavuzu

![image](https://github.com/user-attachments/assets/303769bb-419d-423d-9511-a56bf20f95f9)


Bu depo, **Nagios Core**'un Ubuntu tabanlı bir sistemde kurulumu ve yapılandırılması için adım adım bir rehber sunar. Ayrıca, özel SSH izleme betiklerinin nasıl oluşturulacağı ve yapılandırılacağı da açıklanmıştır. Komutlar, gerçek bir kurulum sürecine dayanır ve Nagios eklentileri ile özel izleme betiklerini içerir.

## Ön Koşullar

Başlamadan önce aşağıdaki gereksinimlere sahip olduğunuzdan emin olun:

- Root veya sudo ayrıcalıklarına sahip bir Ubuntu tabanlı sistem.
- Gerekli paketleri ve Nagios dosyalarını indirmek için internet erişimi.
- Linux komut satırı işlemleri hakkında temel bilgi.

## Kurulum Adımları

### 1. Sistemi Güncelleyin ve Bağımlılıkları Yükleyin

Sisteminizi güncelleyin ve Nagios, Apache, PHP ve diğer bağımlılıklar için gerekli paketleri yükleyin.

```bash
sudo apt update && apt upgrade -y
sudo apt install -y wget unzip apache2 php libapache2-mod-php php-gd build-essential libgd-dev openssl libssl-dev daemon unzip
```

### 2. Nagios Kullanıcısı ve Grubu Oluşturun

Nagios için özel bir kullanıcı ve grup oluşturun, ardından Apache kullanıcısını Nagios komut grubuna ekleyin.

```bash
sudo useradd nagios
sudo groupadd nagcmd
sudo usermod -a -G nagcmd nagios
sudo usermod -a -G nagcmd www-data
```

### 3. Nagios Core'u İndirin ve Kurun

Nagios Core'u (bu örnekte 4.5.1 sürümü) indirin ve derleyin.

```bash
cd /tmp
wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.5.1.tar.gz
tar -xzf nagios-*.tar.gz
cd nagios-4.5.1
./configure --with-command-group=nagcmd
make all
sudo make install
sudo make install-init
sudo make install-commandmode
sudo make install-config
sudo make install-webconf
```

### 4. Apache ve Nagios Web Arayüzünü Yapılandırın

Web arayüzünü kurun ve `nagiosadmin` kullanıcısı için bir şifre belirleyerek güvenliğini sağlayın.

```bash
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
sudo a2enmod cgi
sudo a2enconf nagios
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
sudo systemctl restart apache2
```

### 5. Nagios Servisini Etkinleştirin ve Başlatın

Nagios'un sistem açılışında otomatik olarak başlamasını sağlayın ve servisi başlatın.

```bash
sudo systemctl enable nagios
sudo systemctl start nagios
systemctl status nagios
```

### 6. Nagios'u İzleme için Yapılandırın

Sunucu-specific yapılandırmalar için bir dizin oluşturun ve ana Nagios yapılandırma dosyasını bu dizini içerecek şekilde güncelleyin.

```bash
sudo mkdir /usr/local/nagios/etc/servers
sudo nano /usr/local/nagios/etc/nagios.cfg
```

`/usr/local/nagios/etc/nagios.cfg` dosyasına şu satırı ekleyin:

```bash
cfg_dir=/usr/local/nagios/etc/servers
```

### 7. Sunucu Yapılandırması Ekleyin

Belirli servisleri izlemek için bir sunucu yapılandırma dosyası (örneğin, `ugur-sunucu.cfg`) oluşturun.

```bash
sudo nano /usr/local/nagios/etc/servers/ugur-sunucu.cfg
```

`ugur-sunucu.cfg` için örnek içerik:

```bash
define host {
    use                     linux-server
    host_name               ugur-sunucu
    alias                   Ugur Sunucu
    address                 <sunucu_ip_adresi>
    max_check_attempts      5
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}

define service {
    use                     generic-service
    host_name               ugur-sunucu
    service_description     SSH
    check_command           check_ssh
    max_check_attempts      3
    notification_interval   10
    notification_period     24x7
}
```

Değişiklikleri uygulamak için Nagios'u yeniden başlatın:

```bash
sudo systemctl restart nagios
```

### 8. Nagios Eklentilerini Kurun

Ek izleme yetenekleri için Nagios eklentilerini indirin ve kurun.

```bash
cd /tmp
wget https://nagios-plugins.org/download/nagios-plugins-2.3.3.tar.gz
tar -xzf nagios-plugins-2.3.3.tar.gz
cd nagios-plugins-2.3.3
./configure --with-nagios-user=nagios --with-nagios-group=nagios
make
sudo make install
```

## Özel SSH İzleme Betiği

SSH servislerini özel mantıkla izlemek için `check_ssh_stats.sh` gibi bir betik oluşturabilirsiniz. Aşağıda, böyle bir betiğin nasıl oluşturulacağı ve yapılandırılacağı açıklanmıştır.

### 9. Şifre Tabanlı SSH için `sshpass` Kurulumu

Otomatik SSH girişleri için `sshpass` yükleyin.

```bash
sudo apt-get install sshpass
```

### 10. Özel SSH Betiği Oluşturun

SSH bağlantısını veya diğer metrikleri kontrol etmek için `check_ssh_stats.sh` adında bir betik oluşturun.

```bash
nano /usr/local/nagios/libexec/check_ssh_stats.sh
```

`check_ssh_stats.sh` için örnek içerik:

```bash
#!/bin/bash

# Yapılandırma
HOST=$1
USER="kullanici_adi"
PASSWORD="sifre"

# SSH bağlantısını kontrol et
sshpass -p "$PASSWORD" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 $USER@$HOST exit
STATUS=$?

if [ $STATUS -eq 0 ]; then
    echo "TAMAM: $HOST adresine SSH bağlantısı başarılı"
    exit 0
else
    echo "KRİTİK: $HOST adresine SSH bağlantısı başarısız"
    exit 2
fi
```

Betiği çalıştırılabilir yapın:

```bash
chmod +x /usr/local/nagios/libexec/check_ssh_stats.sh
```

### 11. Nagios'ta Özel Komutu Tanımlayın

Yeni betiği kullanmak için Nagios komut yapılandırmasını düzenleyin.

```bash
sudo nano /usr/local/nagios/etc/objects/commands.cfg
```

Aşağıdaki komut tanımını ekleyin:

```bash
define command {
    command_name    check_ssh_stats
    command_line    $USER1$/check_ssh_stats.sh $HOSTADDRESS$
}
```

### 12. Özel Betiği Kullanan Bir Servis Yapılandırın

Özel SSH kontrolünü kullanmak için bir sunucu yapılandırma dosyası (örneğin, `nginx-secondary-server.cfg`) oluşturun veya düzenleyin.

```bash
sudo nano /usr/local/nagios/etc/servers/nginx-secondary-server.cfg
```

Örnek içerik:

```bash
define host {
    use                     linux-server
    host_name               nginx-secondary
    alias                   Nginx İkincil Sunucu
    address                 <sunucu_ip_adresi>
    max_check_attempts      5
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}

define service {
    use                     generic-service
    host_name               nginx-secondary
    service_description     Özel SSH Kontrolü
    check_command           check_ssh_stats
    max_check_attempts      3
    notification_interval   10
    notification_period     24x7
}
```

Değişiklikleri uygulamak için Nagios'u yeniden başlatın:

```bash
sudo systemctl restart nagios
```

## Nagios Web Arayüzüne Erişim

- Tarayıcınızı açın ve `http://<sunucu_ip>/nagios` adresine gidin.
- `nagiosadmin` kullanıcı adı ve `htpasswd` adımında belirlediğiniz şifre ile oturum açın.

## Notlar

- `<sunucu_ip_adresi>` yerine izlediğiniz sunucunun gerçek IP adresini yazın.
- `nagios` ve `nagcmd` gruplarının `/usr/local/nagios/etc/` ve `/usr/local/nagios/libexec/` dizinlerinde uygun izinlere sahip olduğundan emin olun.
- `check_ssh_stats.sh` betiğindeki `PASSWORD` bilgisinin üretim ortamında güvenli bir şekilde yönetildiğinden emin olun. Şifre yerine SSH anahtarlarını kullanmayı düşünün.
- `check_ssh_stats.sh` betiği temel bir örnektir. Gerektiğinde CPU, bellek veya disk kullanımı gibi ek kontroller ekleyebilirsiniz.

## Sorun Giderme

- Nagios servisi başlatılamazsa, logları kontrol edin: `journalctl -u nagios`.
- `/usr/local/nagios/etc/` ve `/usr/local/nagios/libexec/` dizinlerindeki dosya izinlerini doğrulayın.
- Apache'nin doğru yapılandırıldığından ve değişikliklerden sonra yeniden başlatıldığından emin olun.

## Katkıda Bulunma

Bu depoya katkıda bulunmak için pull request gönderebilir veya sorunları bildirebilirsiniz.

## Lisans

Bu proje MIT Lisansı altında lisanslanmıştır.
