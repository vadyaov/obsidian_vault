Сегодня научился подключаться к кластеру k8s в nova-impala-chart.

Что сделал:
1. Склонировал на локальную машину репозиторий [nova-impala-chart]().
2. Скопировал туда config-файл [nova-sand.kubeconfig](https://drive.google.com/file/d/1RGhm-vXplBtuRTP9aoiBCTZNIr3uFFa5/view?usp=sharing). Как я понял этот файл нужен для корректного подключения к кластеру и для работы [Lens](https://wiki.glowbyteconsulting.com/pages/viewpage.action?spaceKey=DEV&title=Lens).
3. Сделал `export KUBECONFIG=nova-sand.kubeconfig`
4. В файле values/nova-sand-override.yaml создал структуру:
```yaml
containers:
	versions:
		impala:
			catalog: "cr.yandex/crpcrur29rbavh39f8bh/nova-impala-catalog:20240911-4.4.1-release"
```
Это задает образ импалы, который соберется в пайплайне.
Если нужно протестить какую-то версию импалы, после `impala-catalog:` нужно указать гит-ветку с нужной версией.
5. Выполнил команды:
```bash
kubectl create namespace myakishv # создал неймспэйс
helm -n myakishv install psql charts/postgres
helm dependency update charts/impala
helm -n myakishv install impala charts/impala -f values/nova-sand-override.yaml
```
6. Подключился к hue в k8s:
	- kubectl -n myakishv get ingress - узнал адрес hue
	- открыл в браузере - попал в hue с работающей impala

## Как развернуть образ импалы в nova-impala-chart

Нужно перезаписать (override) параметры container-a в файле values/nova-sand-override.yaml.

Важно соблюдать корректность шаблона yaml. Сам шаблон можно найти в файле impala/templates/imapa/impala.yaml (ищем там блок `container:`  и просто забираем в файл values/nova-sand-override.yaml, удаляя лишнее)

```yaml
containers:
  versions:
    impala:
      catalog: "cr.yandex/crpcrur29rbavh39f8bh/nova-impala-catalog:4.3.0-release-Task27281-release-apache-hive"
      statestore: "cr.yandex/crpcrur29rbavh39f8bh/nova-impala-statestore:4.3.0-release-Task27281-release-apache-hive"
      statefulset: "cr.yandex/crpcrur29rbavh39f8bh/nova-impala-coordexec:4.3.0-release-Task27281-release-apache-hive"
```
В итоге такой вариант сработал.
Catalog, statestore и statefulset - компоненты импалы. Важно перезаписать не только catalog, но и остальные.

В контексте Impala и её работы в кластере Kubernetes, вот что означают термины `catalog`, `statestore` и `statefulset`:

1. **Catalog** (catalogd):
   - Компонент каталога Impala, который управляет метаданными о таблицах и схемах. Он синхронизируется с Hive Metastore, чтобы обновлять информацию о таблицах, партициях, и других метаданных. Когда запросы SQL выполняются в Impala, `catalogd` отвечает за передачу этих метаданных на исполнители (executors), чтобы они могли корректно обрабатывать запросы.
   
2. **Statestore** (statestored):
   - Statestore — это служба координации в кластере Impala, которая отвечает за распространение информации о состоянии между всеми узлами (например, о том, какие узлы активны). Statestore помогает узлам координировать работу и обнаруживать сбои или изменения в кластере.

3. **Statefulset (coordexec)**:
   - StatefulSet в Kubernetes — это объект, который управляет состоянием подов. В случае с Impala, `coordexec` — это компоненты координации и выполнения запросов. Эти узлы принимают SQL-запросы, распределяют их между исполнительными узлами и обрабатывают результаты. StatefulSet гарантирует, что каждый под в кластере имеет уникальный идентификатор и состояние сохраняется даже при перезапуске подов.

Эти три компонента работают совместно, обеспечивая корректное выполнение запросов, управление метаданными и координацию работы в кластере Impala.

Как правильно указать образ для сборки?
После двоеточия в описании каждого кластера: `statefulset: "cr.yandex/crpcrur29rbavh39f8bh/nova-impala-coordexec:4.3.0-release-Task27281-release-apache-hive"` нужно указать то название тэга, который запушился в реестр cr.yandex. В данном варианте это 4.3.0-release-Task27281-release-apache-hive. Правильное название тэга можно посмотреть в усмешно выполненных пайплайнах. 

