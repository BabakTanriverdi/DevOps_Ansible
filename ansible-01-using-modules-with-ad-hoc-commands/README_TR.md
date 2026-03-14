# Hands-on Ansible-01: Ansible Kurulumu ve Temel Operasyonlar

Bu hands-on eğitimin amacı, öğrencilere temel Ansible becerilerini kazandırmaktır.

## Öğrenim Hedefleri

Bu hands-on eğitimin sonunda öğrenciler:

- Ansible'ın ne işe yaradığını açıklayabilecek
- Temel Ad-hoc komutları öğrenmiş olacak

## İçerik

- Part 1 - Ansible Kurulumu
- Part 2 - Ansible Ad-hoc Komutlar

---

## Part 1 - Ansible Kurulumu

3 adet Amazon Linux 2023 EC2 instance oluşturun ve aşağıdaki gibi isimlendirin:

1. **control node** — Ansible'ın kurulu olacağı yönetim makinesi
2. **node1** — SSH PORT 22, HTTP PORT 80
3. **node2** — SSH PORT 22, HTTP PORT 80

Control node'a SSH ile bağlanın ve aşağıdaki komutları çalıştırın:

```bash
# Sistemdeki mevcut paketleri güncelleriz.
# dnf: Amazon Linux 2023'ün paket yöneticisidir (apt'nin Red Hat karşılığı).
# -y: Kurulum sırasında çıkan "emin misiniz?" sorularına otomatik "evet" der.
sudo dnf update -y

# Ansible paketini kurarız.
sudo dnf install ansible -y
```

### Kurulumu Doğrulama

Ansible'ın başarıyla kurulduğunu doğrulamak için aşağıdaki komutu çalıştırın:

```bash
# Kurulu Ansible sürümünü ve yapılandırma bilgilerini gösterir.
ansible --version
```

Beklenen çıktı:
```
ansible [core 2.15.3]           # 1. Kurulu Ansible sürümü
  config file = None            # 2. Henüz ansible.cfg dosyası yok, oluşturacağız
  configured module search path = ['/home/ec2-user/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
                                # 3. Ansible modüllerinin arandığı dizinler (sırayla kontrol edilir)
  ansible python module location = /usr/lib/python3.9/site-packages/ansible
                                # 4. Ansible'ın Python kütüphanelerinin bulunduğu yol
  executable location = /usr/bin/ansible
                                # 5. `ansible` komutunun gerçek dosya yolu
  python version = 3.9.16 (main, Sep  8 2023, 00:00:00) [GCC 11.4.1 20230605 (Red Hat 11.4.1-2)]
                                # 6. Ansible'ın kullandığı Python sürümü ve derleyici bilgisi
  jinja version = 3.1.2         # 7. Playbook'larda değişken kullanımı için Jinja2 şablon motoru
  libyaml = True                # 8. YAML işleme için C kütüphanesi mevcut; daha hızlı çalışır
```

### Control Node'u Yapılandırma

```bash
# ansible_lesson1 adında bir klasör oluşturur ve içine girer.
# && : soldaki komut başarılıysa sağdakini çalıştırır.
mkdir ansible_lesson1 && cd ansible_lesson1
```

```bash
# Boş bir inventory dosyası oluşturur.
# touch: Dosya yoksa oluşturur, varsa değiştirmez.
touch inventory.txt
```

`inventory.txt` dosyasının içeriğini aşağıdaki gibi doldurun:

```ini
[webservers]
node1 ansible_host=<node1_ip> ansible_user=ec2-user
node2 ansible_host=<node2_ip> ansible_user=ec2-user

[all:vars]
ansible_ssh_private_key_file=/home/ec2-user/<pem_dosyasi>
```

> **Parametre Açıklamaları:**
> - `[webservers]` : Grup adı. Bu gruba `ansible webservers ...` şeklinde komut gönderebilirsiniz.
> - `node1`, `node2` : Alias (takma ad). Gerçek IP yerine bu isimleri komutlarda kullanırsınız.
> - `ansible_host` : Sunucunun gerçek IP adresi veya DNS adı.
> - `ansible_user` : SSH bağlantısında kullanılacak kullanıcı adı. Amazon Linux için `ec2-user`, Ubuntu için `ubuntu`.
> - `[all:vars]` : `all` grubundaki (yani inventory'deki TÜM) sunuculara uygulanacak değişkenler.
> - `ansible_ssh_private_key_file` : SSH bağlantısı için kullanılacak `.pem` dosyasının tam yolu.

`.pem` dosyasını kendi bilgisayarınızdan control node'a kopyalamak için:

```bash
# scp: SSH üzerinden güvenli dosya kopyalama aracıdır.
# -i <pem>      : Bağlantı için kullanılacak kimlik doğrulama anahtarı.
# İlk <pem>     : Kopyalanacak kaynak dosya.
# ec2-user@..   : Hedef sunucu kullanıcısı ve IP adresi.
# :/home/ec2-user : Dosyanın kopyalanacağı hedef dizin.
scp -i <pem_dosyasi> <pem_dosyasi> ec2-user@<control_node_public_dns>:/home/ec2-user
```

Inventory dosyasını doğrulayın:

```bash
# Inventory dosyasını okuyarak tüm host ve grupları JSON formatında listeler.
# Dosya sözdiziminin doğru olup olmadığını kontrol etmek için kullanılır.
# -i inventory.txt : Kullanılacak inventory dosyasını belirtir.
ansible-inventory -i inventory.txt --list
```

`ansible.cfg` yapılandırma dosyası oluşturun:

```bash
touch ansible.cfg
```

```ini
[defaults]
host_key_checking  = False
inventory          = inventory.txt
interpreter_python = auto_silent
```

> **Parametre Açıklamaları:**
> - `host_key_checking = False` : SSH bağlantısında "bu sunucuya daha önce bağlanmadınız" uyarısını atlar.
> - `inventory = inventory.txt` : Her komutta `-i inventory.txt` yazmak zorunda kalmayız; varsayılan olarak bu dosyayı kullanır.
> - `interpreter_python = auto_silent` : Python yorumlayıcısını otomatik tespit eder; uyarı mesajlarını sessizce gizler. Bu satır olmadan terminalde deprecation (kullanımdan kalkma) uyarısı çıkar.

---

## Part 2 - Ansible Ad-hoc Komutlar

Ad-hoc komutlar, bir Playbook dosyası oluşturmadan tek satırda hızlıca çalıştırılan Ansible komutlarıdır.

```bash
# Aktif ansible.cfg dosyasının içeriğini ekrana basar.
# Hangi ayarların uygulandığını görmek için kullanışlıdır.
ansible-config view

# inventory'deki TÜM host'ları listeler. Komut ÇALIŞTIRMAZ, sadece listeler.
ansible all --list-hosts

# Yalnızca webservers grubundaki host'ları listeler.
ansible webservers --list-hosts
```

Tüm host'ların erişilebilir olduğunu doğrulamak için ping modülünü çalıştırın:

```bash
# Tüm host'lara SSH+Python testi yapar. ICMP ping DEĞİLDİR.
# Başarılıysa "pong" yanıtı döner.
ansible all -m ping

# Yalnızca webservers grubuna ping gönderir.
ansible webservers -m ping

# Yalnızca node1'e ping gönderir.
ansible node1 -m ping

# node1 ve node2'ye ping gönderir. Çıktıyı tek satırda özetler.
# -o (--one-line): Her host'un çıktısını tek satıra sıkıştırır; toplu görünüm için kullanışlıdır.
# ':' operatörü: node1 VEYA node2 anlamına gelir.
ansible node1:node2 -m ping -o
```

### Ad-hoc Komutlarla Devam Edelim

```bash
# Ping modülünün belgelerini gösterir: açıklama, parametreler ve playbook örnekleri.
# Belgeler içinde gezinmek için ok tuşları, çıkmak için 'q' kullanılır.
ansible-doc ping
```

```bash
# Tüm host'lara ping gönderir. Çıktıyı tek satırda gösterir.
ansible all -m ping -o

# Modülün tam (fully qualified) adıyla da kullanılabilir; sonuç aynıdır.
ansible all -m ansible.builtin.ping
```

```bash
# Tüm kullanılabilir seçenekleri listeler.
# Önemli seçenekler:
# -o            : One-line çıktı
# -a            : Modüle gönderilecek argümanlar
# -m            : Kullanılacak modül adı
# -i            : Kullanılacak inventory dosyası
# --list-hosts  : Eşleşen host listesini gösterir, komut çalıştırmaz
# --become-user : Hangi kullanıcıya geçileceğini belirtir
ansible --help
```

```bash
# webservers grubundaki tüm sunucularda 'uptime' komutunu çalıştırır.
# Varsayılan modül 'command'dır; -m belirtmezseniz command modülü kullanılır.
# command modülü: pipe (|), redirect (>) gibi shell operatörlerini DESTEKLEMEZ.
ansible webservers -a "uptime"
```

Örnek çıktı:
```
node1 | CHANGED | rc=0 >>
 13:00:59 up 42 min,  1 user,  load average: 0.08, 0.02, 0.01
```

> **Çıktı Açıklaması:**
> - `up 42 min` : Sunucu 42 dakikadır çalışıyor.
> - `1 user` : Şu an 1 kullanıcı bağlı.
> - `load average: 0.08, 0.02, 0.01` : Son 1 / 5 / 15 dakikanın ortalama CPU yükü.
>   - `0.08` → son 1 dakikada CPU %8 doluydu
>   - `0.02` → son 5 dakikada CPU %2 doluydu
>   - `0.01` → son 15 dakikada CPU %1 doluydu

Dosya kopyalamak için:

```bash
# Önce control node'da test dosyası oluşturalım.
touch testfile
echo "This is a test file." > testfile

# copy modülü: control node'daki bir dosyayı managed node'lara kopyalar.
# src=  : Control node'daki kaynak dosyanın tam yolu.
# dest= : Managed node'da hedef yol.
ansible webservers -m copy -a "src=/home/ec2-user/ansible_lesson1/testfile dest=/home/ec2-user/testfile"

# shell modülü: pipe, redirect gibi shell operatörlerini destekler.
# ';' ile birden fazla komutu sırayla çalıştırabilirsiniz.
# Bu komut: testfile2'ye yazar, ardından içeriğini ekrana basar.
ansible node1 -m shell -a "echo Hello Ansible > /home/ec2-user/testfile2 ; cat testfile2"
```

### Ubuntu ile Devam

Bir Ubuntu EC2 instance oluşturun ve `inventory.txt` dosyasına ekleyin:

```ini
[ubuntuserver]
node3 ansible_host=<node3_ip> ansible_user=ubuntu
```

```bash
# Güncellenmiş inventory'deki tüm host'ları listeleyelim.
ansible all --list-hosts

# Tüm host'lara ping atalım.
ansible all -m ping -o

# HATA VERECEK ÖRNEK: node1 ve node2'de /home/ubuntu dizini yoktur!
ansible all -m shell -a "echo Hello Ansible > /home/ubuntu/testfile3"
```

Beklenen hata çıktısı:
```
node1 | FAILED | rc=1 >>
/bin/sh: /home/ubuntu/testfile3: No such file or directory
```

Doğru kullanım — sunucuları ayrı hedefleyin:

```bash
# ':' ile birden fazla host ya da grup hedeflenebilir.
# node1 ve node2: Amazon Linux → home dizini /home/ec2-user
ansible node1:node2 -m shell -a "echo Hello Ansible > /home/ec2-user/testfile3"

# node3: Ubuntu → home dizini /home/ubuntu
ansible node3 -m shell -a "echo Hello Ansible > /home/ubuntu/testfile3"
```

### shell Modülü ile Nginx Kurulumu

```bash
# -b (--become): Managed node'da sudo (root) yetkisi kullanır.
# Üç komut sırayla çalışır:
#   1. dnf install -y nginx : nginx'i kurar
#   2. systemctl start nginx : servisi hemen başlatır
#   3. systemctl enable nginx : sistem açılışında otomatik başlamasını sağlar
ansible webservers -b -m shell -a "dnf install -y nginx ; systemctl start nginx ; systemctl enable nginx"
```

Ubuntu sunucusu için:

```bash
# Ubuntu'da paket yöneticisi apt'dir.
# apt update: Paket listesini günceller (yükleme yapmaz).
# apt-get install -y nginx: nginx'i kurar.
ansible node3 -b -m shell -a "apt update -y ; apt-get install -y nginx ; systemctl start nginx ; systemctl enable nginx"
```

Nginx'i kaldırmak için:

```bash
ansible webservers -b -m shell -a "dnf -y remove nginx"
```

### yum ve package Modülleri

```bash
# yum modülünün belgelerini inceleyelim.
# EXAMPLES bölümündeki örneklerin playbook dosyaları için yazıldığına dikkat edin.
ansible-doc yum
```

```bash
# node1'e nginx kurar.
# yum modülü: RHEL/CentOS/Amazon Linux 2 ve öncesi için kullanılır.
# name=nginx   : Kurulacak paketin adı.
# state=present: Paket yoksa kur, varsa dokunma (idempotent davranış).
ansible node1 -b -m yum -a "name=nginx state=present"

# node2'ye nginx kurar.
# dnf modülü: Amazon Linux 2023, RHEL 8+ ve Fedora için kullanılır.
ansible node2 -b -m dnf -a "name=nginx state=present"
```

> **Idempotency (Tekrar Çalıştırılabilirlik):**
> Aynı komutu **iki kez** çalıştırın ve çıktıyı karşılaştırın:
> - **İlk çalıştırma** (nginx yoktu): `CHANGED` — sarı renk → değişiklik yapıldı.
> - **İkinci çalıştırma** (nginx zaten var): `SUCCESS, changed: false` — yeşil renk → hiçbir şey yapılmadı.
>
> `shell` modülüyle `dnf install` yapılsaydı her seferinde tekrar kurulum denerdi. `yum`/`dnf`/`package` modülleri idempotent'tir.

```bash
# package modülü: İşletim sistemini otomatik algılar.
# Amazon Linux / RHEL → dnf/yum kullanır.
# Ubuntu / Debian    → apt kullanır.
# Böylece tek komutla farklı OS'lere aynı paketi kurabilirsiniz.
ansible -b -m package -a "name=nginx state=present" all

# Kurulumu doğrulamak için her sunucuda nginx sürümünü sorgulayın.
ansible all -b -a "nginx -v"

# Nginx'i tüm sunuculardan kaldır.
# state=absent: Paket varsa kaldır, yoksa dokunma.
ansible -b -m package -a "name=nginx state=absent" all
```

---

## Komut Özet Tablosu

| Komut | Açıklama |
|---|---|
| `ansible all -m ping` | Tüm host'lara SSH+Python testi yapar |
| `ansible webservers -m ping -o` | Gruba ping; çıktıyı tek satırda gösterir |
| `ansible all --list-hosts` | Eşleşen host listesini gösterir; komut çalıştırmaz |
| `ansible-config view` | Aktif ansible.cfg içeriğini gösterir |
| `ansible-doc <modül>` | Modülün belgelerini gösterir |
| `ansible all -a "uptime"` | Varsayılan (command) modülle komut çalıştırır |
| `ansible node1 -m shell -a "ls \| grep txt"` | Shell operatörlü komut çalıştırır |
| `ansible webservers -m copy -a "src=... dest=..."` | Dosyayı hedef sunuculara kopyalar |
| `ansible node1 -b -m yum -a "name=nginx state=present"` | nginx'i kurar (idempotent) |
| `ansible all -b -m package -a "name=nginx state=absent"` | Paketi tüm sunuculardan kaldırır (OS bağımsız) |
