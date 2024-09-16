


# Дипломный проект: Автоматизация CI/CD и Безопасности

---

Этот проект настраивает CI/CD пайплайн с интегрированными проверками безопасности для приложения [Django-DefectDojo](https://github.com/DefectDojo/django-DefectDojo). Пайплайн структурирован в несколько этапов, что обеспечивает безопасность кода и эффективную доставку. Следующие этапы подробно описывают настройки и требования для достижения поставленных целей.

Ознакомиться с дипломным проектом так же можно по ссылке: 
[GitLab](http://2261005-cg36175.twc1.net/S_Komkov/diplom) 

---



## Этапы проектирования

### Этап 1. CI/CD



**Описание:**
На первом этапе мы создаем непрерывный процесс интеграции и доставки (CI/CD), который включает в себя сборку и развертывание программного обеспечения. Для этого будем использовать GitLab CI/CD и облачную виртуальную машину (VM) для развертывания приложения.

**Конфигурация пайплайна в `.gitlab-ci.yml`:**

```yaml
stages:
  - build
  - test
  - dast
  - securitychecks
  - deploy
  
variables:
  CISERVERURL: "http://2261005-cg36175.twc1.net/"
   
beforescript:
    - apk update && apk add --no-cache openssh
    - mkdir -p ~/.ssh
    - echo "$CLOUDVMIP" > ~/.ssh/knownhosts
    - chmod 644 ~/.ssh/knownhosts
    - echo "$SSHPRIVATEKEY" > ~/.ssh/ided25519
    - chmod 600 ~/.ssh/ided25519
    - eval $(ssh-agent -s)
    - ssh-add ~/.ssh/ided25519
    - echo "$CLOUDVMIP"

build:
  stage: build
  tags:
    - diplom
  script:
    - ssh -o StrictHostKeyChecking=no $SSHUSERNAME@$CLOUDVMIP "git clone https://github.com/DefectDojo/django-DefectDojo || true"
    - ssh -o StrictHostKeyChecking=no $SSHUSERNAME@$CLOUDVMIP "cd django-DefectDojo && git pull"
    - ssh -o StrictHostKeyChecking=no $SSHUSERNAME@$CLOUDVMIP "cd django-DefectDojo && ./dc-build.sh"

```


### Этап 2. SAST



**Описание:**
На втором этапе мы добавим статический анализ безопасности кода (SAST) с использованием Semgrep и Trivy. Это позволит автоматически проверять код на наличие уязвимостей и проблем безопасности.

**Конфигурация пайплайна для Semgrep и Trivy:**
```yaml
semgrepscan:
  stage: test
  image: returntocorp/semgrep
  tags:
    - diplom
  script:
    - semgrep --config auto .

trivyscan:
  stage: test
  image:
    name: aquasec/trivy:latest
    entrypoint: ""
  tags:
    - diplom
  script:
    - apk --no-cache add openssh-client
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE || echo "Vulnerabilities found, check the report"
  allowfailure: true    
```

---

### Этап 3. DAST


**Описание:**
На третьем этапе мы добавим динамический анализ безопасности (DAST) с использованием OWASP ZAP для сканирования веб-приложений на наличие уязвимостей. Это поможет обнаружить и исправить уязвимости, которые могут быть использованы злоумышленниками.

**Конфигурация пайплайна для DAST:**
```yaml
dastscan:
  stage: dast
  image: owasp/dependency-check-action
  tags:
    - diplom
  script:
    - apk --no-cache add curl
    - zap-baseline.py -t http://$CLOUDVMIP -r zapreport.html
    - curl --upload-file zapreport.html $CISERVERURL/zapreports/
```

---

### Этап 4. Security Checks



**Описание:**
На четвертом этапе мы добавим проверку безопасности конфигураций и репозиториев на наличие секретов с использованием `detect-secrets`. Это поможет предотвратить случайное раскрытие конфиденциальных данных.

**Конфигурация пайплайна для Security Checks:**
```yaml
securitychecks:
  stage: securitychecks
  image: python:3.9
  tags: 
    - diplom
  script:
    - pip install detect-secrets
    - detect-secrets scan > secrets.baseline
  artifacts:
    paths:
      - secrets.baseline
  allowfailure: true
```
---


### Этап 5. Security Gateway



**Описание:**
На пятом этапе мы добавим триггеры для остановки релиза при обнаружении уязвимостей и автоматическую генерацию рекомендаций по их исправлению. Это обеспечит дополнительную защиту и контроль качества на каждом этапе выпуска ПО.

**Конфигурация пайплайна для деплоя:**

```yaml
deploy:
  stage: deploy
  tags:
    - diplom
  script:
    - ssh -o StrictHostKeyChecking=no $SSH_USERNAME@$CLOUD_VM_IP "cd django-DefectDojo && ./dc-up-d.sh postgres-redis"
    - |
      until ssh -o StrictHostKeyChecking=no $SSH_USERNAME@$CLOUD_VM_IP "cd django-DefectDojo && docker compose logs initializer | grep 'Admin password:'"
      do
        sleep 10
        echo "Waiting for the initializer to complete..."
      done
    - echo "Deploying to instance at $CLOUD_VM_IP"
    - ssh -o StrictHostKeyChecking=no $SSH_USERNAME@$CLOUD_VM_IP "cd django-DefectDojo && docker compose up -d"
```

---

## Заключение

Этот проект нацелен на создание надежного, безопасного и автоматизированного процесса CI/CD с использованием современных практик DevSecOps. Каждый этап нацелен на обеспечение безопасности и качества всего жизненного цикла разработки ПО.

**Обратите внимание:**
- Не забудьте настроить переменные окружения в вашей системе CI/CD для корректной работы пайплайна.
- Периодически просматривайте результаты анализов и следуйте рекомендациям для обеспечения безопасности вашего приложения.

**Автор:**
- Сергей Комков




