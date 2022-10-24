# Декларативное программирование.
***
## Создание экземпляров в Google Cloud Platform посредством Terraform и VMware.
***
### Первым делом создадим и скачаем на VMware машину ключ управления сервисами

Для начала зарегестрируйтесь на *`GCP(GoogleCloudPlatform)`* и перейдя по ссылке, где будет инструкция по созданию ключа:
+ https://cloud.google.com/iam/docs/creating-managing-service-account-keys#iam-service-accounts-upload-rest

После того как скачали ключ, который должен выглядеть так:
```json
{
  "type": "service_account",
  "project_id": "terraform-18524",
  "private_key_id": "ПРИВАТНЫЙ КЛЮЧ ID",
  "private_key": "-----BEGIN PRIVATE KEY----- ***ВАШ ПРИВАТНЫЙ КЛЮЧ***\n-----END PRIVATE KEY-----\n",
  "client_email": "terraform-sa@terraform-18524.iam.gserviceaccount.com",
  "client_id": "159856327415789565132",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/terraform-sa@terraform-18524.iam.gserviceaccount.com"
 }
```
***Никому не передавайте и не показывайте этот ключ, храните только у себя!!!***

Переименуем ключ в *`Credential.json`* (вам как удобно)
```bash
mv Private_key.json Credential.json 
#Переместим в новую папку `Terraform`
mkdir Terraform
mv Credential.json ~/Terraform
```
:white_check_mark: ***Начальный файл и директория готовы*** 

***
### Далее скачаем и установим сам `Terraform`

Используем следующие команды:
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```
После этого пробуем команду: 
```bash
terraform init
```
Должен выдавать:  *`Terraform has benn successfully initialized`*

Создадим ключевую пару, открытый ключ которого мы добавим в файл `main.tf`, чтобы у нас был доступ ко всему проекту: 
```bash 
ssh-keygen -t rsa  #дадим такое же название как у $USER или можете оставить по умолчанию
#в итоге у нас есть
"$USER".pub #открытый ключ или по умолчанию id_rsa.pub
"$USER" #закрытый ключ или по умолчанию id_rsa
```

Теперь можем приступить к созданию конфигурационного файла `main.tf`
```bash
sudo vim ~/Terraform/main.tf
```
Будем создавать 4 экземпляра(серверов), поэтому вот что находится у меня в файле:
```bash 
provider "google" {    #имя провайдера 
  project = "terraform-18524"  #имя проекта 
  credentials = "${file("credentials.json")}"  #указание переменной на ваш закрытый ключ
  region = "europe-west1"    #стандартные настройки
  zone = "europe-west1-b"    #выставляете зону
}

resource "google_compute_instance" "my_server1_instance" {  #задаете название ресурса с которого terraform будет получать информацию
  name = "server1"   #имя сервера
  machine_type = "n1-standard-1"   #тип машины
  zone = "europe-west3-b"   #зона 
  allow_stopping_for_update = true   

  boot_disk {    #тип ОС
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  
  network_interface {   #настройки сети
    network = "default"   
    access_config {    # обязательное поле должно быть 

    }
  }
  
  metadata_startup_script = "sudo apt-get update; sudo apt-get upgrade"  # начальный скрипт выполнения
}   # На этом заканчивается создание 1 ресурса 

resource "google_compute_instance" "my_server2_instance" {   # Создание следующего ресурса 
  name = "server2"
  machine_type = "n1-standard-1"
  zone = "europe-west3-b"
  allow_stopping_for_update = true

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  
  network_interface {
    network = "default"
    access_config {

    }
  }
    
  metadata_startup_script = "sudo apt-get update; sudo apt-get upgrade"
}

resource "google_compute_instance" "my_server3_instance" {
  name = "server3"
  machine_type = "n1-standard-1"
  zone = "europe-west3-c"
  allow_stopping_for_update = true

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  
  network_interface {
    network = "default"
    access_config {

    }
  }

  metadata_startup_script = "sudo apt-get update; sudo apt-get upgrade"
}

resource "google_compute_instance" "my_server4_instance" {
  name = "server4"
  machine_type = "n1-standard-1"
  zone = "us-central1-a"
  allow_stopping_for_update = true

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  
  network_interface {
    network = "default"
    access_config {

    }
  }

  metadata_startup_script = "sudo apt-get update; sudo apt-get upgrade"
}

resource "google_compute_project_metadata" "my_ssh_key" {  #добавление ключа SSH, который позволит управлять проектом полностью
  metadata = {
    ssh-keys = "$ИМЯ КЛЮЧА:ssh-rsa "ВАШ ОТКРЫТЫЙ КЛЮЧ" "$USER"@debian"  #Добавляете ваш открытый ключ, а имя ключу старайтесь давать схожий с названием имени вашего аккаунта GCP
  }
}
```
Создадим файл `output.tf` для получения нужных выходных данных
```bash
sudo vim ~/Terraform/output.tf
```
в файл прописываем:
```bash
output "Internal_IP_server1" {  #Внутренний адрес 
value = google_compute_instance.my_server1_instance.network_interface.0.network_ip
}

output "External_IP_server1" {   #Внешний адрес
value = google_compute_instance.my_server1_instance.network_interface.0.access_config.0.nat_ip
}

output "Internal_IP_server2" {
value = google_compute_instance.my_server2_instance.network_interface.0.network_ip
}

output "External_IP_server2" {
value = google_compute_instance.my_server2_instance.network_interface.0.access_config.0.nat_ip
}

output "Internal_IP_server3" {
value = google_compute_instance.my_server3_instance.network_interface.0.network_ip
}

output "External_IP_server3" {
value = google_compute_instance.my_server3_instance.network_interface.0.access_config.0.nat_ip
}

output "Internal_IP_server4" {
value = google_compute_instance.my_server4_instance.network_interface.0.network_ip
}

output "External_IP_server4" {
value = google_compute_instance.my_server4_instance.network_interface.0.access_config.0.nat_ip
}
```

Зпускаем `Terraform` с директории `/Terraform` командой:
```bash
terraform plan #Предварительный просмотр того что изменится и в каком количестве, со всеми данными
```
Если никаких ошибок не выдает, то вводим команду:
```bash
terraform apply #Применить все изменения
```
В выводе команды он показывает все изменения и данные, которые мы задавали в файле `output.tf`

К примеру может вывести: `Apply complete! Resources: 5 added, 0 changed, 0 destroyed.`*(было добавлено 5 ресурсов)*.

А вот вывод файла `output.tf` выглядит следующим образом:
```bash
Outputs:

External_IP_server1 = "0.1.2.3"
External_IP_server2 = "4.5.6.7"
External_IP_server3 = "8.9.10.11"
External_IP_server4 = "12.13.14.15"
Internal_IP_server1 = "16.17.18.19"
Internal_IP_server2 = "20.21.22.23"
Internal_IP_server3 = "24.25.26.27"
Internal_IP_server4 = "28.29.30.31"
```
***Готово*** 

Для большего изучения используйте: https://registry.terraform.io/providers/hashicorp/google/latest/docs

:white_check_mark: ***С помощью небольшого кода запустили 4 сервера***
***

### Теперь можем подключаться посредством SSH к любому из серверов.

Вводим команду:
```bash
ssh -i id_rsa $ИМЯ_КЛЮЧА@EXTERNAL_IP #Где id_rsa - это закрытый ключ, который мы добавили в мета-данные нашего сервис аккаунта GCP
```
После этого он выведет нам фразу *`permanently added to the list of known hosts`* и покажет *`Last login`* с датой

Остается только использовать сервер по своему усмотрению, например для настройки и поднятия сервера ВПН.

К примеру:
+ https://github.com/AsylbekKamalov/Easy-Rsa-for-OpenVPN-with-Prometheus





