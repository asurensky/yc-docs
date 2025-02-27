# Миграция хостов кластера {{ KF }} в другую зону доступности

Хосты кластера {{ mkf-name }} располагаются в зонах доступности {{ yandex-cloud }}. Хосты можно перенести из одной зоны в другую. Процесс переноса различается для кластеров с одним и несколькими хостами.

{% include [zone-d-restrictions](../../_includes/mdb/ru-central1-d-restrictions.md) %}

Если кластер {{ mkf-name }} является эндпоинтом в сервисе {{ data-transfer-full-name }}, перезапустите трансфер для его корректной работы. Подробнее о том, какие трансферы нужно перезапускать и как это сделать, см. в разделе [Особенности миграции в сервисе {{ data-transfer-full-name }}](#data-transfer).

## Перенести кластер с одним хостом {#one-host}

1. [Создайте подсеть](../../vpc/operations/subnet-create.md) в зоне доступности, в которую вы переносите кластер.
1. Если группа безопасности кластера настроена для подсети в зоне доступности, из которой переносится кластер, [перенастройте группу](../../vpc/operations/security-group-add-rule.md) для новой подсети.
1. [Создайте кластер](cluster-create.md) {{ mkf-name }}, конфигурация которого отличается от конфигурации исходного кластера только подсетью и группой безопасности.
1. Перенесите данные из первоначального кластера в новый с помощью одного из двух инструментов:

   * [MirrorMaker](../tutorials/kafka-connectors.md) — подойдет как встроенный в {{ mkf-name }} MirrorMaker-коннектор, так и утилита MirrorMaker.
   * [{{ data-transfer-full-name }}](../../data-transfer/tutorials/mkf-to-mkf.md).

1. Чтобы успешно выполнять подключение к топикам после миграции, укажите FQDN брокера нового кластера в вашем бэкенде или клиенте (например, в коде или графической IDE). Удалите FQDN брокера прежнего кластера в первоначальной зоне.

   Чтобы узнать FQDN, получите список хостов в кластере:

   ```bash
   {{ yc-mdb-kf }} cluster list-hosts <имя_или_идентификатор_кластера>
   ```

   FQDN указан в выводе команды, в столбце `NAME`.

1. [Удалите первоначальный кластер](cluster-delete.md) {{ mkf-name }}.

## Перенести кластер с несколькими хостами {#multiple-hosts}

1. [Создайте подсеть](../../vpc/operations/subnet-create.md) в целевой зоне доступности, куда вы переносите кластер.
1. Если группа безопасности кластера настроена для подсети в исходной зоне доступности, перенастройте группу для новой подсети. Для этого в правилах группы безопасности замените CIDR исходной подсети на CIDR новой подсети.
1. Измените зону доступности у кластера.

   {% note warning %}

   Если в целевой зоне доступности находится несколько подсетей, измените зону и укажите нужную подсеть с помощью {{ TF }} или API. Используйте консоль управления и CLI, только если в целевой зоне доступности находится одна подсеть.

   {% endnote %}

   {% list tabs group=instructions %}

   * Консоль управления {#console}

      1. Перейдите на [страницу каталога]({{ link-console-main }}) и выберите сервис **{{ ui-key.yacloud.iam.folder.dashboard.label_managed-kafka }}**.
      1. В строке с нужным кластером нажмите на значок ![image](../../_assets/console-icons/ellipsis.svg), затем выберите ![image](../../_assets/console-icons/pencil.svg) **{{ ui-key.yacloud.mdb.cluster.overview.button_action-edit }}**.
      1. В разделе **{{ ui-key.yacloud.mdb.forms.section_network-settings }}** укажите новый набор зон доступности. Их количество не должно уменьшиться.
      1. Нажмите кнопку **{{ ui-key.yacloud.common.save }}**.

   * CLI {#cli}

      {% include [cli-install](../../_includes/cli-install.md) %}

      {% include [default-catalogue](../../_includes/default-catalogue.md) %}

      Чтобы изменить набор зон доступности у кластера, выполните команду:

      ```bash
      {{ yc-mdb-kf }} cluster update <имя_или_идентификатор_кластера> \
         --zone-ids <зоны_доступности>
      ```

      В параметре `--zone-ids` перечислите зоны доступности через запятую. Их количество не должно уменьшиться.

   * {{ TF }} {#tf}

      1. Откройте актуальный конфигурационный файл {{ TF }} с планом инфраструктуры.

         О том, как создать такой файл, см. в разделе [{#T}](cluster-create.md).

      1. Измените в описании кластера {{ mkf-name }} значения параметров `subnet_ids` и `zones`:

         ```hcl
         resource "yandex_mdb_kafka_cluster" "<имя_кластера>" {
           ...
           subnet_ids = ["<подсети>"]
           config {
             ...
             zones = ["<зоны_доступности>"]
           }
         }
         ```

         В параметре `subnet_ids` укажите новый набор подсетей. Убедитесь, что он содержит подсети для зон доступности `{{ region-id }}-a`, `{{ region-id }}-b` и `{{ region-id }}-d`. Эти зоны должны быть указаны, даже если хосты {{ KF }} размещаются в меньшем количестве зон. Все три зоны доступности нужны для хостов {{ ZK }}, которые автоматически добавляются в кластер с несколькими хостами {{ KF }}.

         В параметре `zones` количество зон доступности не должно уменьшиться.

      1. Проверьте корректность настроек.

         {% include [terraform-validate](../../_includes/mdb/terraform/validate.md) %}

      1. Подтвердите изменение ресурсов.

         {% include [terraform-apply](../../_includes/mdb/terraform/apply.md) %}

      Подробнее см. в [документации провайдера {{ TF }}]({{ tf-provider-resources-link }}/mdb_kafka_cluster).

      {% include [Terraform timeouts](../../_includes/mdb/mkf/terraform/cluster-timeouts.md) %}

   * API {#api}

      Чтобы изменить набор зон доступности у кластера, воспользуйтесь методом REST API [update](../api-ref/Cluster/update.md) для ресурса [Cluster](../api-ref/Cluster/index.md) или вызовом gRPC API [ClusterService/Update](../api-ref/grpc/cluster_service.md#Update) и передайте в запросе:

      * Идентификатор кластера в параметре `clusterId`. Чтобы узнать идентификатор, [получите список кластеров в каталоге](cluster-list.md#list-clusters).
      * Новый набор зон доступности в параметре `configSpec.zoneId`. Их количество не должно уменьшиться.
      * Новый набор подсетей в параметре `subnetIds`.
      * Список настроек, которые необходимо изменить, в параметре `updateMask`.

      {% include [Note API updateMask](../../_includes/note-api-updatemask.md) %}

   {% endlist %}

1. Чтобы успешно выполнять подключение к топикам после миграции, укажите FQDN брокера нового кластера в вашем бэкенде или клиенте (например, в коде или графической IDE). Удалите FQDN брокера прежнего кластера в первоначальной зоне.

   Чтобы узнать FQDN, получите список хостов в кластере:

   ```bash
   {{ yc-mdb-kf }} cluster list-hosts <имя_или_идентификатор_кластера>
   ```

   FQDN указан в выводе команды, в столбце `NAME`.

{% include [migration-in-data-transfer](../../_includes/data-transfer/migration-in-data-transfer.md) %}
