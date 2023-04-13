---
title: "CRD 简介"
date: 2022-10-11T23:50:34+08:00
draft: false
tags: 
  - Kubernetes
  - crd
keywords:
  - Kubernetes
  - CRD
weight: 1
---



CRD 字段校验配置

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: scalings.control.poneding.com
spec:
  group: control.poneding.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: scalings
    singular: scaling
    kind: Scaling
  validation:
    openAPIV3Schema:
      properties:
        spec:
          required:
          - targetDeployment
          - minReplicas
          - maxReplicas
          - metricType
          - step
          - scaleUp
          - scaleDown
          properties:
            targetDeployment:
              type: string
            minReplicas:
              type: integer
              minimum: 0
            maxReplicas:
              type: integer
              minimum: 0
            metricType:
              type: string
              enum:
              - CPU
              - MEMORY
              - REQUESTS
            step:
              type: integer
              minimum: 1
            scaleUp:
              type: integer
            scaleDown:
              type: integer
              minimum: 0
```

>   -   是否必须
>   -   参数类型
>   -   枚举范围
>   -   数值最大最小
