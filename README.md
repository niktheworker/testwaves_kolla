# Testwaves

Testwaves - это фреймворк тестирования, который используется openstack проекты tempest и rally для автоматизации тестирования регионов openstack. Написан на ansible, имеет возможность независимого выполнения процессов (статическая нагрузка, функциональное или нагрузочное тестирование) и подготовлен для регионов с OVS или LXBR (без L3) агента. Имеет возможность кастомизации через переменные ansible под конкретный регион.   
Testwaves - это ansible - роль. 

### Структура

 - testwaves.yml - плейбук для запуска роли
 - user_testwaves_overrides.yml - файл оверрайд переменных 
 - roles/testwaves/defaults/main.yaml - дефолтные переменные, определяющие настройку региона и тестирования
 - roles/testwaves/defaults/skip-list.lxbr.yml - скип лист темпеста для lxbr региона
 - roles/testwaves/defaults/skip-list.ovs.yml - скип лист темпеста для ovs региона
 - roles/testwaves/files/tasks - сценарии для Rally
 - roles/testwaves/templates/task_args.j2 - темлпейт аргументов для сценариев ралли
 - roles/testwaves/templates/skip.j2 - темплейт для скип-листа
 - roles/testwaves/templates/tempest.lxbr.j2 - теплейт для темпест конфига lxbr региона
 - roles/testwaves/templates/tempest.ovs.j2 - теплейт для темпест конфига ovs региона
 - roles/testwaves/task/main.yaml - основная таска, из которых через include выполняются все таски 
 - roles/testwaves/task/depends.yml - выполняет зависимости
 - roles/testwaves/task/project.yml - первияная настройка региона
 - roles/testwaves/task/static-load.yml - создание статической нагрузки
 - roles/testwaves/task/static-instances.yml - цикл создания инстансов статической нагрузки
 - roles/testwaves/task/tempest.yml - интеграционное тестирование с помощью Tempest
 - roles/testwaves/task/rally-test.yml - нагрузочное тестирования с помощью Rally
 - roles/testwaves/task/clear.yml - удаление конфигурации региона
 - roles/testwaves/task/clear-instances.yml - удаление инстансов статической нагрузки


## Запуск и настройка роли


### Необходимые условия 

- OSA Queens
- Для авторизации запросов используется . openrc на utility контейнерах, должен быть доступен для пользователя от которого выполняется ansible  
- Для авторизации os_* модулей используется  .config/openstack/clouds.yaml на utility контейнерах, должен быть доступен для пользователя от которого выполняет ansible  
- Валидная инвенторий в /etc/openstack_deploy, содержащая целевые утилити контейнеры
- Все внешние пакеты (pip репозиторий, github репо и т.д.) должны быть доступны для скачивания
- /openstack/venvs/rally-17.1.17/bin/cinder --version = 3.5.0 ( бага: https://bugs.launchpad.net/rally/+bug/1785519). Установка описана в секции запуска, ниже.


### Переопределение переменных

Все перменные прокомментированы в defaults/main.yml

- testwaves_cloud: default  указывает на clound в .config/openstack/clouds.yaml Последний должен быть сгенерирован
- testwaves_endpoint_type: admin  Endpoint URL type 
- testwaves_project_user: admin  пользователь от которого будет создан проект
- testwaves_project_role: admin  роль этого пользователя
- testwaves_subnets  указание подсети для создания
- testwaves_cloud_images  указать какие образы загружать в glance, их источник и параметры
- testwaves_static  указать образ для машин со статикой
- testwaves_static_network  указать сети для машин со статикой
- testwaves_cpu_allocation_ratio  коэфициент переподписки для учета запуска машин со статикой
- testwaves_ubench_repo адрес источника архива с юниксбенчом
- testwaves_glance_image_location  адрес источника образа для использования в сценариях ралли
- testwaves_provider_networks: True  если true, то используются провайдерские внешние сети
- testwaves_smoke  если true, то все в smoke режиме
- testwaves_run для запуска тестов должна быть в true
- testwaves_region_type lxbr/ovs тип региона 
- testwaves_networks_flat: описание флэт сетей
- testwaves_tempest_fixed_network_name указание сети для использвани темпестом
- testwaves_tempest_build_timeout: указание времени ожидание для темпеста
- testwaves_tempest_dashboard_url адрес дашборд
- testwaves_service_list список сервисов для тестирования ралли


### В зависимости от параметров. которые ниже, будут вычислены times и concurency сценариев Rally
- testwaves_users_amount: 5 - количество пользователей региона
- testwaves_tenants_amount: 10 - количество тенантов
- testwaves_controllers_amount: 3 - количество котрол нод | основной параметр от которого зависит увеличение times/concurrency
- testwaves_compute_amount: 5 - количество компьют нод
- testwaves_storage_amount: 20 - количество сторедж нод у
- testwaves_network_amount: 1 - количество сетей
- testwaves_time_multiplier: 1 - увеличивает время выполнения тестов, путем увеличения количества тестов

### Отключение ненужных действий

Для отключения ненужных действий, которые при выпадении в ошибку могут прерывать выполнения роли, нужно закомментировать эти действия в тасках.
Так же нужно закомментировать в clear* тасках теже действия. 

### Clear task

Для дальнейшего использования, созданного окружения после проведения тестирования, нужно отключить удаление:

- testwaves_clear_dir: True 
- testwaves_clear_key: True
- testwaves_clear_nets: True
- testwaves_clear_subnets: True
- testwaves_clear_routers: True
- testwaves_clear_images: True
- testwaves_clear_secgroup: True
- testwaves_clear_flavors: True

### Роль выполняется с использованием тэгов

- testwaves-depends - выполнение depends.yml
- testwaves-project - выполнение project.yml
- testwaves-static-load - выполнение static-load.yml 
- testwaves-tempest-test - выполнение tempest-test.yml
- testwaves-rally-test - выполнение rally-test
- testwaves-clear - выполнение clear.yml

### Каталоги

При выполнении роли создаются временные каталоги, которые удалются таском clear.yml:

- /root/testwaves/
- /root/testwaves/scenarios - для сценариев rally
- /root/testwaves/reports - для отчетов rally и tempest
- /root/testwaves/images - для загрузки образов
- /root/testwaves/keypair - для ssh ключей



### Запуск и настройка

1. Клонирование репозитория 
```
git clone https://gitlab.itkey.com/SBP/testwaves
```

2. Копирование роли, плейбука и оверрайд файла в нужные директории
```
cp testwaves/user_testwaves_overrides.yml /etc/openstack_deploy/user_testwaves_overrides.yml
cp testwaves/testwaves.yml /opt/openstack-ansible/playbooks/testwaves.yml
cp testwaves/user_testwaves_overrides_lxbr.yml #скип лист для lxbr
cp testwaves/user_testwaves_overrides_ovs.yml #скип лист для ovs
cp -R testwaves/roles/testwaves /etc/ansible/roles/testwaves
```

3. Необходимо отредактировать овверрайд файл в соответствии с настройками региона
```
vim /etc/openstack_deploy/user_testwaves_overrides.yml
```

4. Запсук плебйука
```
openstack-ansible /opt/openstack-ansible/playbooks/testwaves.yml
```


## Отчеты

Отчеты сохраняются в /root/*имя утилити контейнера*/reports на деплой хосте в формате HTML, если не удаляется testvawes директория таской clear.
Так же отчеты возвращаются на деплой-хост в /root/_имя утилити контейнера_/reports


## Перезапуск failed тестов темпеста

```
cd /opt/openstack-ansible
ansible utility[0] -m shell -a "/openstack/venvs/rally-17.1.17/bin/rally verify rerun --failed"

```

## Очистка региона 

Удаление всех инстансов статической нагрузки, сетей и т.д.
```
openstack-ansible -i /opt/openstack-ansible/playbooks/testwaves.yml --tags testwaves-clear
```