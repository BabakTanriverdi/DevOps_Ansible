# Hands-on Ansible-03: Facts, Variables, Secrets ve Dynamic Inventory

Bu hands-on eğitimin amacı, öğrencilere Ansible fact toplama, gizli verilerle çalışma ve dynamic inventory kullanımını öğretmektir.

## Öğrenim Hedefleri

Bu hands-on eğitimin sonunda öğrenciler:

- Fact gathering'in ne olduğunu ve playbook'larda nasıl kullanıldığını açıklayabilecek
- Ansible Vault ile gizli verilerle nasıl çalışıldığını öğrenmiş olacak
- Dynamic inventory'nin ne olduğunu açıklayabilecek
- EC2 plugin ile dynamic inventory kullanımını öğrenmiş olacak

## İçerik

- Part 1 - Ansible Kurulumu
- Part 2 - Ansible Variables ve Facts
- Part 3 - Hassas Verilerle Çalışma
- Part 4 - Dynamic Inventory ile Çalışma

---

## Part 1 - Ansible Kurulumu

3 adet Amazon Linux 2023 EC2 instance oluşturun ve aşağıdaki gibi isimlendirin:

1. **control node**
2. **node1** — SSH PORT 22, HTTP PORT 80
3. **node2** — SSH PORT 22, HTTP PORT 80

Control node'a SSH ile bağlanın:

```bash
sudo dnf update -y
sudo dnf install ansible -y

# Kurulumu doğrula
ansible --version
```

`inventory.ini` dosyasını oluşturun:

```bash
vim inventory.ini
```

```ini
[webservers]
node1 ansible_host=<node1_ip> ansible_user=ec2-user

[dbservers]
node2 ansible_host=<node2_ip> ansible_user=ec2-user

[all:vars]
ansible_ssh_private_key_file=/home/ec2-user/<pem_dosyasi>
```

`ansible.cfg` dosyasını oluşturun:

```bash
vim ansible.cfg
```

```ini
[defaults]
host_key_checking    = False
inventory            = inventory.ini
deprecation_warnings = False
interpreter_python   = auto_silent
```

`.pem` dosyasını control node'a kopyalayın:

```bash
# Kendi bilgisayarınızdan çalıştırın.
scp -i <pem_dosyasi> <pem_dosyasi> ec2-user@<control_node_public_dns>:/home/ec2-user

# Kopyalanan dosyanın izinlerini sadece sahibi okuyabilir şekilde ayarlar.
# SSH, izinleri geniş olan .pem dosyalarını reddeder.
chmod 400 <pem_dosyasi>
```

Bağlantıyı doğrulayın:

```bash
# Tüm host'lara ping atarak SSH + Python bağlantısını test eder.
ansible all -m ping -o
```

---

## Part 2 - Ansible Variables ve Facts

### Variables (Değişkenler)

Ansible'da değişkenler, sistemler arasındaki farklılıkları yönetmek için kullanılır. Aynı playbook'u farklı değerlerle çalıştırmayı sağlar.

`myplaybook.yml` dosyasını oluşturun — önce sabit değerle:

```bash
vim myplaybook.yml
```

```yaml
---
- name: Copy ip address to node1
  hosts: node1
  tasks:
    - name: Copy ip address to the nodes
      ansible.builtin.copy:
        content: 'Private ip address of this node is 172.31.88.207'   # IP sabit yazılmış
        dest: /home/ec2-user/myfile

- name: Copy ip address to node2
  hosts: node2
  tasks:
    - name: Copy ip address to the nodes
      ansible.builtin.copy:
        content: 'Private ip address of this node is 172.31.81.197'   # Her node için ayrı play
        dest: /home/ec2-user/myfile
```

```bash
ansible-playbook myplaybook.yml
```

Şimdi aynı playbook'u `vars` kullanarak güncelleyin:

```yaml
---
- name: Copy ip address to node1
  hosts: node1
  vars:
    ip_address: 172.31.88.207   # Değişken tanımı
  tasks:
    - name: Copy ip address to the nodes
      ansible.builtin.copy:
        content: 'Private ip address of this node is {{ ip_address }}'
        # {{ }} : Jinja2 şablon sözdizimi; değişken değerini buraya yerleştirir
        dest: /home/ec2-user/myfile

- name: Copy ip address to node2
  hosts: node2
  vars:
    ip_address: 172.31.81.197
  tasks:
    - name: Copy ip address to the nodes
      ansible.builtin.copy:
        content: 'Private ip address of this node is {{ ip_address }}'
        dest: /home/ec2-user/myfile
```

```bash
# Çalıştırın — çıktının değişmediğine dikkat edin (idempotent davranış).
ansible-playbook myplaybook.yml

# File icerisini kontrol et 
ansible all -a "cat home/ec2-user/myfile"
```

---

### debug Modülü

`debug` modülü, playbook çalışırken değişken değerlerini ekrana basar. Hata ayıklamak için kullanışlıdır; playbook'u durdurmaz.

`myplaybook.yml` dosyasını güncelleyin:

```yaml
---
- name: Copy ip address to node1
  hosts: node1
  vars:
    ip_address: 172.31.88.207
  tasks:
    - name: Copy ip address to the nodes
      ansible.builtin.copy:
        content: 'Private ip address of this node is {{ ip_address }}'
        dest: /home/ec2-user/myfile

    - name: Using debug module
      ansible.builtin.debug:
        var: ip_address   # Bu değişkenin değerini ekrana basar
```

```bash
# Çalıştırın ve çıktıda ip_address değerini gözlemleyin.
ansible-playbook myplaybook.yml
```

---

### Ansible Facts

Ansible facts, managed node'lardan otomatik olarak toplanan sistem bilgileridir: işletim sistemi, IP adresleri, disk bilgisi, donanım gibi veriler. Playbook çalışmadan önce `gather_facts` adımıyla toplanır.

```bash
# setup modülü: node1'deki tüm fact'leri JSON formatında listeler.
ansible node1 -m setup
```

Örnek çıktı (kısaltılmış):
```
node1 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": ["172.31.20.246"],
        "ansible_architecture": "x86_64",
        "ansible_os_family": "RedHat",
        "ansible_distribution": "Amazon",
        ...
    }
}
```

Tüm fact'leri playbook ile görüntüleyin — `facts.yml`:


```bash
vim facts.yml
```

```yaml
---
- name: Show facts
  hosts: all
  tasks:
    - name: Print all facts
      ansible.builtin.debug:
        var: ansible_facts   # Toplanan tüm fact'leri ekrana basar
```

```bash
ansible-playbook facts.yml
```

Belirli bir fact'i kullanın — `ipaddress.yml`:

```bash
vim ipaddress.yml
```

```yaml
---
- hosts: all
  tasks:
    - name: Show IP address
      ansible.builtin.debug:
        msg: >
          This host uses IP address {{ ansible_facts.default_ipv4.address }}
          # Nokta notasyonu ile iç içe fact'e ulaşılır
          # Alternatif köşeli parantez notasyonu:
          # {{ ansible_facts['default_ipv4']['address'] }}
```

```bash
ansible-playbook ipaddress.yml
```

---

## Part 3 - Hassas Verilerle Çalışma

### vars_files ile Değişken Dosyası Kullanımı

Değişkenleri ayrı bir dosyada tutmak playbook'u daha temiz tutar. Ancak bu yöntemde dosya şifresizdir.

`myuser.yml` dosyasını oluşturun:

```yaml
username: john
password: 123qwe
```

`create-user.yml` dosyasını oluşturun:

```yaml
---
- name: Create a user
  hosts: all
  become: true
  vars_files:
    - myuser.yml   # Değişken dosyasını dahil eder; {{ username }} ve {{ password }} burada tanımlı
  tasks:
    - name: Creating user
      ansible.builtin.user:
        name: "{{ username }}"
        password: "{{ password }}"
```

```bash
ansible-playbook create-user.yml

# Kullanıcının oluşturulduğunu doğrulayın.
# /etc/shadow: şifrelenmiş parola bilgilerini içerir; yalnızca root okuyabilir.
ansible all -b -m command -a "grep john /etc/shadow"
```

---

### Ansible Vault ile Şifreleme

`vars_files` yöntemi dosyayı düz metin olarak saklar. Ansible Vault, değişken dosyasını şifreleyerek güvenli hale getirir.

```bash
# Şifreli bir değişken dosyası oluşturur.
# Komut çalışınca vault şifresi oluşturmanız istenir.
ansible-vault create secret.yml
```

```
New Vault password: xxxx
Confirm New Vault password: xxxx
```

Açılan editörde içeriği yazın:

```yaml
username: tyler
password: 99abcd
```

```bash
# Şifreli dosyanın içeriğini görüntüleyin — okunamaz hex formatındadır.
cat secret.yml
```

```
33663233353162643530353634323061613431366332373334373066353263353864...
```

`create-user.yml` dosyasını güncelleyin:

```yaml
---
- name: Create a user
  hosts: all
  become: true
  vars_files:
    - secret.yml   # Artık şifreli dosyayı kullanıyoruz
  tasks:
    - name: Creating user
      ansible.builtin.user:
        name: "{{ username }}"
        password: "{{ password }}"
```

```bash
# Vault şifresi olmadan çalıştırırsanız hata alırsınız:
ansible-playbook create-user.yml
# ERROR! Attempting to decrypt but no vault secrets found

# Vault şifresiyle çalıştırın.
# --ask-vault-pass: Çalıştırma sırasında vault şifresini sorar.
ansible-playbook --ask-vault-pass create-user.yml
```

```bash
# Kullanıcıyı doğrulayın.
ansible all -b -m command -a "grep tyler /etc/shadow"
```

---

### SHA512 ile Güvenli Parola Hash'leme

Önceki örnekte parola `/etc/shadow`'a düz metin yazıldı. Güvenli kullanım için parolayı hash'leyerek saklamalısınız.

```bash
ansible-vault create secret-1.yml
```

```yaml
username: Oliver
password: 14abcd
```

`create-user-1.yml` dosyasını oluşturun:

```yaml
---
- name: Create a user
  hosts: all
  become: true
  vars_files:
    - secret-1.yml
  tasks:
    - name: Creating user
      ansible.builtin.user:
        name: "{{ username }}"
        password: "{{ password | password_hash('sha512') }}"
        # | (pipe): Jinja2 filtresidir; değeri bir sonraki fonksiyona aktarır.
        # password_hash('sha512'): Parolayı SHA-512 algoritmasıyla hash'ler.
        # /etc/shadow'a hash'lenmiş değer yazılır; düz metin asla saklanmaz.
```

```bash
ansible-playbook --ask-vault-pass create-user-1.yml

# Doğrulayın — shadow'daki değerin hash'lenmiş olduğuna dikkat edin.
ansible all -b -m command -a "grep Oliver /etc/shadow"
```

---

## Part 4 - Dynamic Inventory ile Çalışma

### Statik Inventory ile Ping Testi

```bash
# Çalışma dizinini oluşturun.
mkdir dynamic-inventory
cd dynamic-inventory
```

```bash
nano inventory.ini
```

```ini
[servers]
db_server   ansible_host=<db_server_ip>   ansible_user=ec2-user  ansible_ssh_private_key_file=~/<pem_dosyasi>
web_server  ansible_host=<web_server_ip>  ansible_user=ec2-user  ansible_ssh_private_key_file=~/<pem_dosyasi>
```

```bash
nano ansible.cfg
```

```ini
[defaults]
host_key_checking = False
inventory         = /home/ec2-user/dynamic-inventory/inventory.ini
interpreter_python = auto_silent
private_key_file  = ~/<pem_dosyasi>
```

```bash
nano ping-playbook.yml
```

```yaml
---
- name: Ping them all
  hosts: all
  tasks:
    - name: Pinging
      ansible.builtin.ping:
```

```bash
ansible-playbook ping-playbook.yml
```

---

### Dynamic Inventory Kurulumu

Statik inventory'de her yeni EC2 instance için dosyayı elle güncellemeniz gerekir. Dynamic inventory, AWS API'yi sorgulayarak instance'ları otomatik olarak bulur.

**IAM Rolü Oluşturma:**

1. AWS Console → IAM → Roles → Create role
2. Use case: `EC2` → Policy: `AmazonEC2FullAccess`
3. EC2 Dashboard → control node → Actions → Security → Modify IAM role → Oluşturduğunuz rolü seçin

```bash
# boto3 ve botocore: AWS SDK for Python.
# Ansible'ın AWS API'ye erişmesi için gereklidir.
sudo dnf install pip -y
pip install --user boto3 botocore
```

`inventory_aws_ec2.yml` dosyasını oluşturun:

> **Not:** Dynamic inventory dosyası mutlaka `aws_ec2.yml` veya `aws_ec2.yaml` ile bitmelidir.

```bash
nano inventory_aws_ec2.yml
```

```yaml
plugin: amazon.aws.aws_ec2   # Ansible'ın AWS EC2 dinamik inventory plugin'i
regions:
  - "us-east-1"              # Hangi AWS bölgesinin taranacağını belirtir
keyed_groups:
  - key: tags.Name           # EC2 instance'larını Name tag'ına göre gruplar
compose:
  ansible_host: public_ip_address   # Bağlantı için public IP'yi kullan
```

```bash
# Dynamic inventory çıktısını ağaç yapısında görüntüler.
ansible-inventory -i inventory_aws_ec2.yml --graph
```

Beklenen çıktı:
```
@all:
  |--@aws_ec2:
  |  |--ec2-34-201-69-79.compute-1.amazonaws.com
  |  |--ec2-54-234-17-41.compute-1.amazonaws.com
  |--@ungrouped:
```

`ansible.cfg` dosyasındaki inventory satırını güncelleyin:

```ini
inventory = /home/ec2-user/dynamic-inventory/inventory_aws_ec2.yml
```

```bash
# Dynamic inventory ile tüm host'lara ping atın.
# --key-file: SSH bağlantısı için kullanılacak .pem dosyasını belirtir.
ansible all -m ping --key-file "~/<pem_dosyasi>"
```

---

### Dynamic Inventory ile Kullanıcı Oluşturma

`user.yml` dosyasını oluşturun:

```yaml
---
- name: Create a user using a variable
  hosts: all
  become: true
  vars:
    user: lisa
    ansible_ssh_private_key_file: "/home/ec2-user/<pem_dosyasi>"
    # vars içinde SSH key de tanımlanabilir; inventory'den bağımsız çalışır
  tasks:
    - name: Create user {{ user }}
      ansible.builtin.user:
        name: "{{ user }}"
```

```bash
# -i ile inventory dosyasını override ederek çalıştırın.
ansible-playbook user.yml -i inventory_aws_ec2.yml

# Kullanıcının oluşturulduğunu doğrulayın.
# tail -2: /etc/passwd dosyasının son 2 satırını gösterir (en son eklenen kullanıcılar).
ansible all -a "tail -2 /etc/passwd"
```

---

## Opsiyonel: AWS Parameter Store ile Vault Şifresi Yönetimi

Vault şifresini düz metin dosyada saklamak güvensizdir. AWS Parameter Store ile şifreyi bulutta güvenli şekilde saklayabilirsiniz.

**IAM Rolü:** EC2 instance'a `AmazonSSMManagedInstanceCore` policy'si eklenmiş bir rol atayın.

Önce düz metin şifreyle şifreli dosya oluşturun:

```bash
# vault_passwd.sh: Vault şifresini içeren dosya (başlangıçta düz metin).
echo "123456" > vault_passwd.sh

ansible-vault create secret-2.yml
```

```yaml
username: alex
password: qaz321
```

`create-user-2.yml` dosyasını oluşturun:

```yaml
---
- name: Create a user
  hosts: all
  become: true
  vars_files:
    - secret-2.yml
  tasks:
    - name: Creating user
      ansible.builtin.user:
        name: "{{ username }}"
        password: "{{ password | password_hash('sha512') }}"
```

```bash
# --vault-password-file: Şifreyi dosyadan okur, elle sormaz.
ansible-playbook create-user-2.yml --vault-password-file ./vault_passwd.sh

# Doğrulayın.
ansible all -b -m command -a "grep alex /etc/shadow"
```

Şimdi şifreyi AWS Parameter Store'a taşıyın:

1. AWS Console → Systems Manager → Parameter Store → Create parameter
2. Name: `serag-vault_passwd` → Value: `123456`

`vault_passwd.sh` dosyasını güncelleyin:

```bash
vim vault_passwd.sh
```

```bash
#!/bin/bash
# AWS CLI ile Parameter Store'dan şifreyi çeker.
# --region     : Parametrenin bulunduğu AWS bölgesi
# --names      : Parametre adı
# --query      : Yalnızca Value alanını döndürür
# --output text: JSON yerine düz metin çıktısı verir
aws --region=us-east-1 ssm get-parameters \
    --names "serag-vault_passwd" \
    --query "Parameters[*].{Value:Value}" \
    --output text
```

```bash
# Dosyayı çalıştırılabilir yapın.
chmod +x vault_passwd.sh

# Şifrelenmiş dosyayı Parameter Store şifresiyle görüntüleyin.
ansible-vault view secret-2.yml --vault-password-file ./vault_passwd.sh
```

> **Sonuç:** Vault şifresi artık dosyada değil, AWS Parameter Store'da güvenli şekilde saklanıyor. Script çalışırken şifreyi çekip Ansible'a iletir.
