# VMware Cloud Provider Interface (CPI) Addon для Talos Kubernetes

- [VMware Cloud Provider Interface (CPI) Addon для Talos Kubernetes](#vmware-cloud-provider-interface-cpi-addon-для-talos-kubernetes)
  - [Зачем нужен этот аддон](#зачем-нужен-этот-аддон)
  - [Требования](#требования)
  - [Конфигурация](#конфигурация)
    - [Ключевые переменные (в `ansible/<cluster-name>.yml`)](#ключевые-переменные-в-ansiblecluster-nameyml)
  - [Особенности реализации](#особенности-реализации)
  - [Валидация установки](#валидация-установки)
  - [Производственные рекомендации](#производственные-рекомендации)

## Зачем нужен этот аддон

VMware CPI обеспечивает **нативную интеграцию Kubernetes с VMware vSphere**, предоставляя кластеру возможность:

- Автоматического обнаружения и регистрации нод в облаке
- Управления жизненным циклом виртуальных машин (включение/выключение)
- Предоставления топологической информации (зоны, регионы) для планировщика
- Интеграции с VMware CSI для управления томами
- Корректного отображения состояния нод при проблемах с инфраструктурой

**Без этого аддона:**  
Кластер не сможет корректно определять состояние нод при сбоях vCenter, работать с зонами отказоустойчивости и интегрироваться с другими VMware-компонентами.

## Требования

| Компонент            | Версия/Требование                                       |
| -------------------- | ------------------------------------------------------- |
| VMware vCenter       | 7.0 U3+                                                 |
| Пользователь vCenter | С правами минимально необходимыми для заданных операций |

## Конфигурация

### Ключевые переменные (в `ansible/<cluster-name>.yml`)

```yaml
vcenter_hostname: "vcenter-01.infra.corp"  # FQDN vCenter
vcenter_cpi_user: "svc-k8s-vcenter"         # Специальный сервисный аккаунт
vcenter_cpi_password: !vault | ...          # Обязательно в Vault!
vcenter_datacenter: "Datacenter"            # Имя Datacenter в vCenter
vcenter_thumbprint: "70:B8:80:D4:82:..."    # SHA1 отпечаток сертификата vCenter
```

## Особенности реализации

- **Безопасность:**  
  Только `insecure: false` только для тестовых сред. Доверенные CA через `vcenter_thumbprint`.
- **Размещение:**  
  DaemonSet запускается **только на control-plane нодах** через node affinity:

  ```yaml
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: Exists
  ```

- **Критические tolerations:**  
  Обеспечивают работу на нодах во время инициализации:
  ```yaml
  tolerations:
    - key: node.cloudprovider.kubernetes.io/uninitialized
      value: "true"
      effect: NoSchedule
  ```

## Валидация установки

```bash
# Проверка состояния CPI
kubectl get ds -n kube-system vsphere-cloud-controller-manager

# Проверка удаления taint с нод
kubectl get nodes -o json | jq '.items[].spec.taints[] | select(.key=="node.cloudprovider.kubernetes.io/uninitialized")'

# Логи компонента
kubectl logs -n kube-system -l name=vsphere-cloud-controller-manager
```

## Производственные рекомендации

1. **Не используйте `insecure: true`** в production — настройте доверенные сертификаты
2. Для отказоустойчивости убедитесь, что **минимум 3 control-plane ноды** участвуют в запуске CPI
3. **Синхронизируйте версии** CPI и CSI (vmware-csi) для избежания конфликтов

> **Важно:** Этот аддон должен разворачиваться **до** vmware-csi, так как CSI зависит от топологической информации, предоставляемой CPI. И **после** cilium или любого другого CNI, так как зависит от наличия сети в кластере.
