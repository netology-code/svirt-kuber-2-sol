# MyApp Helm Chart

Helm chart для развертывания приложения в Kubernetes.

## Установка

```bash
helm install myapp ./helm-chart
```

## Установка с кастомными значениями

```bash
helm install myapp ./helm-chart -f custom-values.yaml
```

## Обновление

```bash
helm upgrade myapp ./helm-chart
```

## Rollback

```bash
helm rollback myapp 1
```

## Удаление

```bash
helm uninstall myapp
```

## Параметры

Основные параметры описаны в `values.yaml`. Можно переопределить через `--set` или файл значений.

