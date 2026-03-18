# Hands-on Ansible-04: Directory Layout, Error Handling ve Execution Strategies

Bu hands-on eğitimin amacı, öğrencilere Ansible playbook'larında en iyi pratikleri öğretmektir.

## Öğrenim Hedefleri

Bu hands-on eğitimin sonunda öğrenciler:

- Ansible'da directory layout oluşturmayı açıklayabilecek
- Ansible'da hataları nasıl yöneteceğini açıklayabilecek
- Ansible'da playbook çalıştırma stratejilerini açıklayabilecek

## İçerik

- Part 1 - Infrastructure Kurulumu (3 Ubuntu 22.04 EC2 Instance)
- Part 2 - Controller Node'a Ansible Kurulumu
- Part 3 - Target Node'lara Ping Testi
- Part 4 - MySQL Kurulumu ve Phonebook Uygulamasını Çalıştırma
- Part 5 - Dosya Ayrıştırma (File Separation)
- Part 6 - Error Handling ve Execution Strategies

---

## Part 1 - Infrastructure Kurulumu

> **Bu bölümde ne yapıyoruz?**
> Gerçek bir senaryo gibi düşünün: bir veritabanı sunucusu ve bir web sunucusu kuracaksınız. Her sunucu farklı portlara ihtiyaç duyar.

AWS Console'da `Ubuntu 22.04` AMI ile 3 adet EC2 instance oluşturun:

| Instance | Rol | Açık Portlar |
|----------|-----|--------------|
| Controller Node | Ansible yönetim merkezi | 22 (SSH) |
| Target Node1 (db_server) | MySQL veritabanı sunucusu | 22 (SSH), 3306 (MySQL) |
| Target Node2 (web_server) | Python web uygulaması | 22 (SSH), 80 (HTTP) |

---

## Part 2 - Controller Node'a Ansible Kurulumu

> **Bu bölümde ne yapıyoruz?**
> Terraform ile altyapıyı otomatik ayağa kaldırıp Ansible'ın kurulu geldiğini doğruluyoruz.

```bash
# GitHub repo'sundaki Terraform dosyalarını çalıştırın.
# Terraform; EC2 instance'ları, security group'ları ve
# ansible.cfg + inventory.ini dosyalarını otomatik oluşturur.
```

Controller node'a bağlandıktan sonra kurulumu doğrulayın:

```bash
# Kurulu Ansible sürümünü ve yapılandırma bilgilerini gösterir.
ansible --version
```

> Terraform'un oluşturduğu `ansible.cfg` ve `inventory.ini` dosyalarını inceleyin.
> Önceki derslerde bu dosyaları elle oluşturdunuz; burada Terraform bu işi sizin için yapıyor.

---

## Part 3 - Target Node'lara Ping Testi

> **Bu bölümde ne yapıyoruz?**
> Ansible'ın tüm node'lara ulaşabildiğini doğruluyoruz. Bir sonraki adımlarda kurulum yapacağız; önce bağlantının sağlam olduğundan emin olmak gerekiyor.

```bash
# ansible-lesson adında bir klasör oluşturur ve içine girer.
mkdir ansible-lesson
cd ansible-lesson
```

Phonebook uygulama dosyalarını (`phonebook-app.py`, `requirements.txt`, `init.sql`, `templates`) GitHub reposundan control node'a kopyalayın.

> **Önemli:** `phonebook-app.py` içindeki veritabanı bağlantı satırını güncelleyin:
> ```python
> app.config['MYSQL_DATABASE_HOST'] = "<db_server_private_ip>"
> ```

```bash
# Boş bir playbook dosyası oluşturur.
touch ping-playbook.yml
```

```yaml
---
- name: Ping them all
  hosts: all        # inventory'deki tüm node'ları hedef al
  tasks:
    - name: Pinging
      ansible.builtin.ping:   # SSH + Python bağlantısını test eder (ICMP değil)
```

```bash
# Playbook'u çalıştırır.
# Her node'dan "pong" yanıtı geliyorsa bağlantı başarılı demektir.
ansible-playbook ping-playbook.yml
```

---

## Part 4 - MySQL Kurulumu ve Phonebook Uygulamasını Çalıştırma

> **Bu bölümde ne yapıyoruz?**
> İki ayrı playbook ile sistemi kuracağız:
> - `db_config.yml` → db_server'a MySQL kurar, veritabanını ve kullanıcıyı oluşturur.
> - `web_config.yml` → web_server'a Python uygulamasını deploy eder.
>
> Bu, gerçek dünya senaryosuna en yakın kullanım biçimidir.

### db_config.yml — Veritabanı Sunucusu Yapılandırması

```bash
vim db_config.yml
```

```yaml
---
- name: DB configuration
  become: true          # Tüm task'lar için root yetkisi kullan
  hosts: db_server      # Yalnızca db_server node'unu hedef al
  vars:
    hostname: my_db_server
    db_name: phonebook_db
    db_table: phonebook
    db_user: remoteUser
    db_password: passwd1234

  tasks:
    - name: Set hostname
      ansible.builtin.shell: "hostnamectl set-hostname {{ hostname }}"
      # hostnamectl: Sunucunun hostname'ini kalıcı olarak değiştirir.

    - name: Install MySQL and dependencies
      ansible.builtin.package:
        name: "{{ item }}"    # loop ile her paketi sırayla kurar
        state: present
        update_cache: yes     # apt paket listesini kurulumdan önce günceller
      loop:
        - mysql-server        # MySQL sunucusu
        - mysql-client        # MySQL komut satırı istemcisi
        - python3-mysqldb     # Ansible'ın MySQL modüllerini kullanması için gerekli Python kütüphanesi
        - libmysqlclient-dev  # MySQL C kütüphanesi geliştirme dosyaları

    - name: Start and enable MySQL service
      ansible.builtin.service:
        name: mysql
        state: started    # Servisi hemen başlatır
        enabled: yes      # Sistem yeniden başlatıldığında otomatik başlamasını sağlar

    - name: Create MySQL user
      community.mysql.mysql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        priv: '*.*:ALL'   # Tüm veritabanlarında tüm yetkiler (üretimde kısıtlanmalıdır)
        host: '%'         # '%' = her IP adresinden bağlantıya izin ver (remote access)
        state: present

    - name: Copy the SQL script
      ansible.builtin.copy:
        src: /home/ubuntu/phonebook/init.sql   # Control node'daki kaynak
        dest: ~/                               # db_server'daki home dizinine kopyalar

    - name: Create phonebook_db database
      community.mysql.mysql_db:
        name: "{{ db_name }}"
        state: present    # Veritabanı yoksa oluşturur, varsa dokunmaz (idempotent)

    - name: Check if the database has the table
      ansible.builtin.shell: |
        echo "USE {{ db_name }}; SHOW TABLES LIKE '{{ db_table }}';" | mysql
      register: resultOfShowTables
      # register: Komutun çıktısını 'resultOfShowTables' değişkenine kaydeder.
      # Bu değişken sonraki task'larda koşul kontrolü için kullanılır.

    - name: DEBUG — show query result
      ansible.builtin.debug:
        var: resultOfShowTables
      # debug: resultOfShowTables'ın içeriğini ekrana basar.
      # Tablo varsa stdout dolu, yoksa boş gelir.

    - name: Import database table
      community.mysql.mysql_db:
        name: "{{ db_name }}"
        state: import     # SQL dosyasını içe aktarır (NOT: import idempotent değildir)
        target: ~/init.sql
      when: resultOfShowTables.stdout == ""
      # when: Tablo henüz yoksa (stdout boşsa) bu task çalışır.
      # Tablo zaten varsa SKIPPED olarak geçilir.

    - name: Enable remote login to MySQL
      ansible.builtin.lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^bind-address'       # Bu regex ile eşleşen satırı bul
        line: 'bind-address = 0.0.0.0'  # Satırı bu içerikle değiştir (tüm IP'lerden bağlantı)
        backup: yes                   # Değişiklik öncesi yedeğini alır
      notify:
        - Restart mysql
      # notify: Bu task değişiklik yaparsa (changed) handler'ı tetikler.
      # Değişiklik yoksa (idempotent) handler çalışmaz.

  handlers:
    - name: Restart mysql
      ansible.builtin.service:
        name: mysql
        state: restarted
      # Handler: yalnızca notify ile çağrıldığında ve play sonunda çalışır.
      # bind-address değişikliğinin etkili olması için MySQL'in yeniden başlaması gerekir.
```

```bash
ansible-playbook db_config.yml

# MySQL'in kurulduğunu doğrulayın (ad-hoc ile).
# db_server'a SSH açmadan direkt control node'dan sorgulayabilirsiniz.
ansible db_server -m shell -a "mysql --version"
```

---

### web_config.yml — Web Sunucusu Yapılandırması

```bash
vim web_config.yml
```

```yaml
---
- name: Web server configuration
  hosts: web_server
  vars:
    hostname: my_web_server
  tasks:
    - name: Set hostname
      become: yes
      ansible.builtin.shell: "hostnamectl set-hostname {{ hostname }}"

    - name: Install Python and pip
      become: yes
      ansible.builtin.package:
        name:
          - python3      # Python 3 yorumlayıcısı
          - python3-pip  # Python paket yöneticisi
        state: present
        update_cache: yes

    - name: Copy the app file to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/phonebook/phonebook-app.py
        dest: ~/

    - name: Copy the requirements file to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/phonebook/requirements.txt
        dest: ~/

    - name: Copy the templates folder to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/phonebook/templates   # Klasörün tamamını kopyalar
        dest: ~/

    - name: Install dependencies from requirements file
      become: yes
      ansible.builtin.pip:
        requirements: /home/ubuntu/requirements.txt
        # pip modülü: requirements.txt içindeki Python paketlerini kurar.

    - name: Run the app
      become: yes
      ansible.builtin.shell: "nohup python3 phonebook-app.py &"
      # nohup: Terminal kapansa bile uygulamanın çalışmaya devam etmesini sağlar.
      # &   : Uygulamayı arka planda (background) başlatır; playbook bloklanmaz.
```

```bash
ansible-playbook web_config.yml
```

> Tarayıcınızda `http://<web_server_ip>` adresini açın → Phonebook uygulamasını görün.

---

## Part 5 - Dosya Ayrıştırma (File Separation)

> **Bu bölümde ne yapıyoruz?**
> Part 4'te her şeyi tek bir büyük playbook'a yazdık. Gerçek projelerde bu yaklaşım yönetimi zorlaştırır.
> Bu bölümde Ansible'ın en iyi pratik dizin yapısını öğreniyoruz:
> - **group_vars** → grup bazlı değişkenler
> - **host_vars** → host bazlı değişkenler
> - **tasks/** → task'ları ayrı dosyalara bölme
> - **include_tasks** → ana playbook'tan task dosyalarını dahil etme

### group_vars ve host_vars

`group_vars`: Bir grup içindeki tüm node'lara uygulanacak değişkenleri içerir. Dosya adı, `inventory`'deki grup adıyla **birebir aynı** olmalıdır.

`host_vars`: Belirli bir node'a özgü değişkenleri içerir. Dosya adı, `inventory`'deki host adıyla **birebir aynı** olmalıdır.

> Her iki dizin de varsayılan olarak oluşturulmaz; elle oluşturmanız gerekir.

```bash
# ansible-lesson dizinine dön ve klasörleri oluştur.
mkdir group_vars host_vars

# group_vars altında servers.yml oluştur (tüm sunucular için ortak değişkenler).
cd group_vars && touch servers.yml

# host_vars altında her sunucu için ayrı dosya oluştur.
cd ../host_vars && touch db_server.yml web_server.yml
```

`group_vars/servers.yml` — tüm sunuculara uygulanır:

```yaml
# Bu değişkenler 'servers' grubundaki tüm node'lara otomatik uygulanır.
db_name: phonebook_db
db_table: phonebook
db_user: remoteUser
db_password: passwd1234
```

`host_vars/db_server.yml` — yalnızca db_server'a uygulanır:

```yaml
hostname: my_db_server
```

`host_vars/web_server.yml` — yalnızca web_server'a uygulanır:

```yaml
hostname: my_web_server
```

---

### include_tasks ile Task Dosyalarını Ayırma

`include_tasks` modülü, bir task listesini ayrı bir YAML dosyasından dahil eder. Büyük playbook'ları küçük, yönetilebilir parçalara bölmek için kullanılır.

```bash
cd ..
mkdir tasks && cd tasks && touch db_tasks.yml web_tasks.yml
```

`tasks/db_tasks.yml`:

```yaml
    - name: Set hostname
      ansible.builtin.shell: "sudo hostnamectl set-hostname {{ hostname }}"

    - name: Install MySQL and dependencies
      ansible.builtin.package:
        name: "{{ item }}"
        state: present
        update_cache: yes
      loop:
        - mysql-server
        - mysql-client
        - python3-mysqldb
        - libmysqlclient-dev

    - name: Start and enable MySQL service
      ansible.builtin.service:
        name: mysql
        state: started
        enabled: yes

    - name: Create MySQL user
      community.mysql.mysql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        priv: '*.*:ALL'
        host: '%'
        state: present

    - name: Copy the SQL script
      ansible.builtin.copy:
        src: /home/ubuntu/ansible-lesson/phonebook/init.sql
        dest: ~/

    - name: Create phonebook_db database
      community.mysql.mysql_db:
        name: "{{ db_name }}"
        state: present

    - name: Check if the database has the table
      ansible.builtin.shell: |
        echo "USE {{ db_name }}; SHOW TABLES LIKE '{{ db_table }}';" | mysql
      register: resultOfShowTables

    - name: DEBUG — show query result
      ansible.builtin.debug:
        var: resultOfShowTables

    - name: Import database table
      community.mysql.mysql_db:
        name: "{{ db_name }}"
        state: import
        target: ~/init.sql
      when: resultOfShowTables.stdout == ""

    - name: Enable remote login to MySQL
      ansible.builtin.lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^bind-address'
        line: 'bind-address = 0.0.0.0'
        backup: yes
      notify:
        - Restart mysql
```

`tasks/web_tasks.yml`:

```yaml
    - name: Set hostname
      ansible.builtin.shell: "sudo hostnamectl set-hostname {{ hostname }}"

    - name: Install Python and pip
      become: yes
      ansible.builtin.package:
        name:
          - python3
          - python3-pip
        state: present
        update_cache: yes

    - name: Copy the app file to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/ansible-lesson/phonebook/phonebook-app.py
        dest: ~/

    - name: Copy the requirements file to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/ansible-lesson/phonebook/requirements.txt
        dest: ~/

    - name: Copy the templates folder to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/ansible-lesson/phonebook/templates
        dest: ~/

    - name: Install dependencies from requirements file
      become: yes
      ansible.builtin.pip:
        requirements: /home/ubuntu/requirements.txt

    - name: Run the app
      become: yes
      ansible.builtin.shell: "nohup python3 phonebook-app.py &"
```

Ana playbook `playbook.yml`:

```yaml
---
- name: Run the DB server
  hosts: db_server
  become: true
  tasks:
    - ansible.builtin.include_tasks: ./tasks/db_tasks.yml
    # include_tasks: tasks/db_tasks.yml dosyasındaki task'ları buraya dahil eder.
    # group_vars ve host_vars'tan gelen değişkenler otomatik olarak kullanılabilir.

  handlers:
    - name: Restart mysql
      ansible.builtin.service:
        name: mysql
        state: restarted

- name: Run the web server
  hosts: web_server
  tasks:
    - ansible.builtin.include_tasks: ./tasks/web_tasks.yml
```

```bash
# ansible-lesson dizinine dön ve playbook'u çalıştır.
cd ..
ansible-playbook playbook.yml
```

> Tarayıcınızda `http://<web_server_ip>` adresini açın → Uygulama çalışıyor olmalı.

---

## Part 6 - Error Handling ve Execution Strategies

> **Bu bölümde ne yapıyoruz?**
> Gerçek ortamlarda playbook'lar her zaman sorunsuz çalışmaz. Bu bölümde:
> - Hata olduğunda playbook nasıl davranır?
> - Hataları nasıl görmezden geliriz?
> - Task'ları hangi sırayla ve kaç sunucuda aynı anda çalıştırırız?
>
> sorularını cevaplıyoruz.

Önce inventory'e yeni bir node ekleyin (Amazon Linux 2023, t3.small):

```bash
# node3'ün erişilebilir olduğunu doğrulayın.
# node3: Amazon Linux 2023 — apt modülü bu sistemde çalışmaz!
# Bu kasıtlı bir hata senaryosudur.
ansible node3 -m ping
```

`playbook2.yml` dosyasını oluşturun:

```yaml
---
- hosts: servers
  # any_errors_fatal: true   # Herhangi bir node'da hata olursa tüm play durur
  # ignore_errors: true       # Tüm task'lardaki hataları görmezden gel
  # ignore_unreachable: true  # Ulaşılamayan node'ları görmezden gel
  # strategy: free            # Her node bağımsız ilerler; birbirini beklemez
  # serial: 2                 # Her seferinde 2 node'da çalış, bitince diğer 2'ye geç
  tasks:
    - ansible.builtin.debug:
        msg: "task 1"

    - ansible.builtin.debug:
        msg: "task 2"

    - name: Task 3 — install git (node3'te hata verecek!)
      become: true
      ansible.builtin.apt:
        name: git
        state: present
      # apt modülü yalnızca Debian/Ubuntu tabanlı sistemlerde çalışır.
      # node3 Amazon Linux 2023 olduğu için bu task HATA verir.
      # ignore_errors: true  # Yalnızca bu task için hatayı görmezden gel

    - ansible.builtin.debug:
        msg: "task 4"

    - ansible.builtin.debug:
        msg: "task 5"
```

---

### Adım Adım Senaryolar

**1. Varsayılan davranış — hatada ne olur?**

```bash
ansible-playbook playbook2.yml
```

> node3 task 3'te hata verir ve o node için play durur. Diğer node'lar tamamlanır.

---

**2. `any_errors_fatal: true` — herhangi bir hata tüm play'i durdurur**

`any_errors_fatal: true` satırının başındaki `#` işaretini kaldırın, sonra çalıştırın:

```bash
ansible-playbook playbook2.yml
```

> node3 hata verdiği anda **tüm node'lar** için play durur. Diğer node'lar task 4 ve 5'e geçemez.

---

**3. `ignore_errors: true` — hatalı task atlanır, devam edilir**

Task 3'ün altına `ignore_errors: true` ekleyin:

```yaml
    - name: Task 3
      become: true
      ansible.builtin.apt:
        name: git
        state: present
      ignore_errors: true   # Bu task hata verse bile bir sonraki task'a geç
```

```bash
ansible-playbook playbook2.yml
```

> node3 task 3'te hata alır ama task 4 ve 5'e devam eder.

---

**4. `strategy: free` — her node bağımsız ilerler**

> **Not:** `any_errors_fatal: true` ile `strategy: free` birlikte kullanılamaz. `any_errors_fatal`'ı yorum satırına alın.

`strategy: free` satırının `#` işaretini kaldırın:

```bash
ansible-playbook playbook2.yml
```

> **Linear (varsayılan):** Tüm node'lar task 1'i bitirir → hepsi task 2'ye geçer → hepsi task 3'e geçer.
> **Free:** Her node kendi hızında ilerler; yavaş bir node diğerlerini bekletmez.

---

**5. `serial: 2` — batch (toplu) çalıştırma**

`strategy: free`'yi yorum satırına alın, `serial: 2` satırının `#` işaretini kaldırın:

```bash
ansible-playbook playbook2.yml
```

> Ansible önce ilk 2 node'da tüm task'ları çalıştırır, bitince diğer 2 node'a geçer.
> Büyük ortamlarda (50+ sunucu) aynı anda hepsine dokunmak yerine kademeli rollout yapmak için kullanılır.

---

## Özet: Execution Strategies Karşılaştırması

| Parametre | Davranış | Ne Zaman Kullanılır? |
|-----------|----------|----------------------|
| `linear` (varsayılan) | Her task tüm node'larda biter, sonra sıradaki task başlar | Genel amaçlı kullanım |
| `strategy: free` | Her node bağımsız ilerler, birbirini beklemez | Node'lar arası bağımlılık yoksa |
| `serial: N` | Her seferinde N node'da çalış, bitince N tane daha al | Kademeli (rolling) deployment |
| `any_errors_fatal: true` | Herhangi bir hata tüm play'i durdurur | Kritik sistemlerde güvenli durdurma |
| `ignore_errors: true` | Hatalı task atlanır, play devam eder | Opsiyonel task'larda |

---

## Referans Linkler

- [Ansible Playbook Blocks](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_blocks.html)
- [Ansible Error Handling](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_error_handling.html)
- [Ansible Builtin Modules](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/index.html)
