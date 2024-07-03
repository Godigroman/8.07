# Домашнее задание к занятию "`Отказоустойчивость в облаке`" - `Клименко Олег`


### Инструкция по выполнению домашнего задания

   1. Сделайте `fork` данного репозитория к себе в Github и переименуйте его по названию или номеру занятия, например, https://github.com/имя-вашего-репозитория/git-hw или  https://github.com/имя-вашего-репозитория/7-1-ansible-hw).
   2. Выполните клонирование данного репозитория к себе на ПК с помощью команды `git clone`.
   3. Выполните домашнее задание и заполните у себя локально этот файл README.md:
      - впишите вверху название занятия и вашу фамилию и имя
      - в каждом задании добавьте решение в требуемом виде (текст/код/скриншоты/ссылка)
      - для корректного добавления скриншотов воспользуйтесь [инструкцией "Как вставить скриншот в шаблон с решением](https://github.com/netology-code/sys-pattern-homework/blob/main/screen-instruction.md)
      - при оформлении используйте возможности языка разметки md (коротко об этом можно посмотреть в [инструкции  по MarkDown](https://github.com/netology-code/sys-pattern-homework/blob/main/md-instruction.md))
   4. После завершения работы над домашним заданием сделайте коммит (`git commit -m "comment"`) и отправьте его на Github (`git push origin`);
   5. Для проверки домашнего задания преподавателем в личном кабинете прикрепите и отправьте ссылку на решение в виде md-файла в вашем Github.
   6. Любые вопросы по выполнению заданий спрашивайте в чате учебной группы и/или в разделе “Вопросы по заданию” в личном кабинете.
   
Желаем успехов в выполнении домашнего задания!
   
### Дополнительные материалы, которые могут быть полезны для выполнения задания

1. [Руководство по оформлению Markdown файлов](https://gist.github.com/Jekins/2bf2d0638163f1294637#Code)

---

### Задание 1

- Возьмите за основу [решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке»](https://github.com/netology-code/sdvps-homeworks/blob/main/7-03.md#задание-1)

1. `Теперь вместо одной виртуальной машины сделайте terraform playbook, который:`
- создаст 2 идентичные виртуальные машины. Используйте аргумент [count](https://developer.hashicorp.com/terraform/language/meta-arguments/count)
- создаст [таргет-группу](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_target_group). Поместите в неё созданные на шаге 1 виртуальные машины;
- создаст [сетевой балансировщик нагрузки](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_network_load_balancer), который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.
Рекомендуем изучить [документацию сетевого балансировщика нагрузки](https://yandex.cloud/ru/docs/network-load-balancer/quickstart?utm_referrer=https%3A%2F%2Fgithub.com%2Fnetology-code%2Fsflt-homeworks%2Fblob%2Fmain%2F4.md) для того, чтобы было понятно, что вы сделали.
---
2. `Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.`

---
   
3. `Перейдите в веб-консоль Yandex Cloud и убедитесь, что:`
- созданный балансировщик находится в статусе Active,
- обе виртуальные машины в целевой группе находятся в состоянии healthy.

---

4. `Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.`
  
*В качестве результата пришлите:*
1. Terraform Playbook.
2. Скриншот статуса балансировщика и целевой группы.
3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.

---

1. Terraform Playbook. main.tf:

---

```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  token = "y0_AgAAAABEezXuAATuwQAAAADXV8wjHipDbw-1ScaUnNdqKClO2Z3Ykoo"
  cloud_id = "b1gm7bp2grqbho73ke1l"
  folder_id = "b1g7l8ost873t9a9r3g3"
  zone = "ru-central1-b"
}
resource "yandex_compute_instance" "vm" {
  count = 2
  name = "vm${count.index}"


  resources {
    core_fraction = 20
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8a67rb91j689dqp60h"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }
  
  metadata = {
    user-data = "${file("./meta.yaml")}"
  }

}
resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network-1.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_lb_target_group" "target-1" {
  name      = "target-1"

  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[0].network_interface.0.ip_address
  }

  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[1].network_interface.0.ip_address
  }

}

resource "yandex_lb_network_load_balancer" "lb-1" {
  name = "lb1"
  listener {
    name = "listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }
  attached_target_group {
    target_group_id = yandex_lb_target_group.target-1.id
    healthcheck {
      name = "http"
        http_options {
          port = 80
          path = "/"
        }
    }
  }
}

output "internal_ip_address_vm-0" {
  value = yandex_compute_instance.vm[0].network_interface.0.ip_address
}
output "external_ip_address_vm-0" {
  value = yandex_compute_instance.vm[0].network_interface.0.nat_ip_address
}
output "internal_ip_address_vm-1" {
  value = yandex_compute_instance.vm[1].network_interface.0.ip_address
}
output "external_ip_address_vm-1" {
  value = yandex_compute_instance.vm[1].network_interface.0.nat_ip_address
}
```

---

meta.yaml:

---

```
#cloud-config
 disable_root: true
 timezone: Europe/Moscow
 repo_update: true
 apt:
   preserve_sources_list: true
 packages:
  - nginx
 runcmd:
  - [ systemctl, nginx-reload ]
  - [ systemctl, enable, nginx.service ]
  - [ systemctl, start, --no-block, nginx.service ]
 users:
  - name: golpa
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC08CZafmwhOejaF/aDJBhWPT6KJeZ1vLVd+rryu7oMl3v7yLAgtMJca5tDI61cv+KA3V4fGi9YeTRUiWeXvvU47xbADb9GXslaA4aM1jj9SjBWXfcUyQFxCVlv/IE/u7o9c0spSzH3QFHMYBrTU912p9LQz8J23+mj0Xsloxz911AqMZVsjyhNIEqJPUWkmOjzjMzcqP5HThmEOEVkoveDmOgVU/20jZEHQsgOAgg66tWzH15AvwIWfjzLihABXQBSGsWnSxfgeGpQkfOBREWbwpsDXHEmg6vGdDhX1r34CPC33YqA9WMeJo+a+mafKEdUBT0gcjehj1lpN4BZC39CaFFLOeithukg1fXtrgAmPwEMxkV+hfb3JBc8kXD688Rw/hyOMRHbqChAerjPjSQnUdB/ulHLIVxnCeb9LSrUcpPxUD8GF8jHF9TRx5Yu9McY7RoBT3V834fCOD1ggOZ1OVnID1Hk3v3DYOialWJf7ZwoUnZBzd051oE5vIPMx70= golpa@#cloud-config
 users:
  - name: golpa
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC08CZafmwhOejaF/aDJBhWPT6KJeZ1vLVd+rryu7oMl3v7yLAgtMJca5tDI61cv+KA3V4fGi9YeTRUiWeXvvU47xbADb9GXslaA4aM1jj9SjBWXfcUyQFxCVlv/IE/u7o9c0spSzH3QFHMYBrTU912p9LQz8J23+mj0Xsloxz911AqMZVsjyhNIEqJPUWkmOjzjMzcqP5HThmEOEVkoveDmOgVU/20jZEHQsgOAgg66tWzH15AvwIWfjzLihABXQBSGsWnSxfgeGpQkfOBREWbwpsDXHEmg6vGdDhX1r34CPC33YqA9WMeJo+a+mafKEdUBT0gcjehj1lpN4BZC39CaFFLOeithukg1fXtrgAmPwEMxkV+hfb3JBc8kXD688Rw/hyOMRHbqChAerjPjSQnUdB/ulHLIVxnCeb9LSrUcpPxUD8GF8jHF9TRx5Yu9McY7RoBT3V834fCOD1ggOZ1OVnID1Hk3v3DYOialWJf7ZwoUnZBzd051oE5vIPMx70= golpa@DESKTOP-U7UHP4K

```

---

2. Скриншот статуса балансировщика и целевой группы.

![](https://cdn.discordapp.com/attachments/1258025226546184202/1258025634018496633/image.png?ex=66868b2c&is=668539ac&hm=53532fbf133a4d19607620441ae2d4fe0212911cc98459e79c7b36019213c67c&)

---

![](https://cdn.discordapp.com/attachments/1258025226546184202/1258025734832914472/image.png?ex=66868b44&is=668539c4&hm=454517a647e7b4dd048cb2a75f77ba0fd544d1cb63a1efcd18c0c04b27adbd9d&)

---

![](https://cdn.discordapp.com/attachments/1258025226546184202/1258025845872656474/image.png?ex=66868b5e&is=668539de&hm=362505a6314b6c89cadbdf04720242896d78405505151f3ba878f5db7d612a37&)

---

3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.

![](https://cdn.discordapp.com/attachments/1258025226546184202/1258026036021559409/image.png?ex=66868b8b&is=66853a0b&hm=c74c4ac06204c6539c886a5244e36940fa8235ac4d6bf86d5ab8b56a3e66c9a7&)

---
---
