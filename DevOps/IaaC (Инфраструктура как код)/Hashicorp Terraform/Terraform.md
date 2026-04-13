**Terraform** — это инструмент Infrastructure as Code (IaC) от HashiCorp, позволяющий описывать, создавать, изменять и версионировать инфраструктуру в **декларативном** стиле. Конфигурации пишутся на языке **HCL** (HashiCorp Configuration Language) и поддерживают сотни провайдеров: облачные платформы (AWS, GCP, Azure, Yandex Cloud), контейнерные оркестраторы, SaaS, базы данных, сети и многое другое.

**Ключевые концепции:**

| Концепция               | Описание                                                                    |
| ----------------------- | --------------------------------------------------------------------------- |
| **Провайдеры**          | Плагины, которые взаимодействуют с API целевых платформ                     |
| **Ресурсы**             | Объекты инфраструктуры (VM, сеть, БД, балансировщик и т.д.)                 |
| **Модули**              | Переиспользуемые блоки конфигурации для группировки ресурсов                |
| **Состояние (State)**   | JSON-файл, хранящий сопоставление ресурсов из конфига с реальными объектами |
| **Переменные и выходы** | Параметры конфигурации и экспортируемые значения после применения           |

**Типичный рабочий процесс:**

`Написать → Инициализировать → Спланировать → Применить → (при необходимости) Удалить`

### Состояние в terraform:

Terraform **не хранит информацию о реальной инфраструктуре внутри кода**. Конфигурация (`.tf`) описывает _желаемое_ состояние, а файл состояния (`.tfstate`) хранит _фактическое_ состояние на момент последнего успешного `apply`.

Без стейта Terraform не сможет:

- Определить, какие ресурсы уже созданы, а какие нужно добавить
- Отследить зависимости между ресурсами
- Сохранить ID, IP-адреса, DNS-имена и другие динамические атрибуты
- Выполнить корректное обновление или удаление

#### Структура файла состояния:

```json
{
  "version": 4,
  "terraform_version": "1.8.2",
  "serial": 12,
  "lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "modules": [
    {
      "resources": {
        "aws_instance.web": {
          "type": "aws_instance",
          "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
          "instances": [
            {
              "attributes": {
                "id": "i-0a1b2c3d4e5f6",
                "public_ip": "54.12.34.56",
                "ami": "ami-0c55b159cbfafe1f0"
              },
              "sensitive_attributes": ["private_key"]
            }
          ]
        }
      }
    }
  ]
}
```

- `serial` — счётчик изменений (инкрементируется при каждом `apply`)
- `lineage` — уникальный идентификатор стейта (используется для слияния и миграции)
- `sensitive_attributes` — поля, помеченные как чувствительные (TF 0.15+)

#### Где хранить файл состояния?

|Тип|Плюсы|Минусы|
|---|---|---|
|**Локальный** (`terraform.tfstate`)|Просто для локальных тестов|Нет блокировок, нет версионирования, риск потери, секреты на диске|
|**Удалённый бэкенд** (S3+DynamoDB, GCS, Azure Blob, Terraform Cloud, OpenTofu Registry)|Блокировка, версионирование, шифрование, совместная работа|Требует настройки IAM/ключей|

> 🔒 **Важно:** Никогда не коммитьте `.tfstate` в Git. Он содержит секреты, внутренние IP, ARN и другую чувствительную информацию. Используйте `.gitignore` и удалённые бэкенды.

### Основные команды CLI:

**Инициализация и проверка:**

|Команда|Назначение|
|---|---|
|`terraform init`|Инициализирует рабочий каталог: скачивает провайдеры и модули, настраивает бэкенд состояния. **Обязательна** перед первым запуском.|
|`terraform validate`|Проверяет синтаксис и внутреннюю согласованность конфигов (без обращения к облаку).|
|`terraform fmt [-recursive]`|Приводит код к стандартному стилю HCL. Удобно использовать в CI/CD и pre-commit хуках.|
|`terraform version`|Показывает версию Terraform и установленных провайдеров.|

**Планирование и применение:**

| Команда                          | Назначение                                                                                                                          |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `terraform plan [-out=planfile]` | Сравнивает текущее состояние с желаемой конфигурацией и показывает, что будет создано/изменено/удалено. Не вносит изменений.        |
| `terraform apply [planfile]`     | Применяет изменения. Если передать файл плана (`terraform apply tfplan`), изменения выполнятся без интерактивного подтверждения.    |
| `terraform destroy`              | Удаляет **всю** инфраструктуру, описанную в текущей конфигурации. Использует тот же план, что и `apply`, но в обратном направлении. |

**Управление состоянием (`terraform state`):**

Состояние — критически важный артефакт. Никогда не храните его в Git без шифрования, используйте удалённые бэкенды (S3, GCS, Terraform Cloud, Azure Blob) с поддержкой блокировок.

| Команда                                | Назначение                                                                                               |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `terraform state list`                 | Показывает все ресурсы, зафиксированные в текущем стейте.                                                |
| `terraform state show <адрес_ресурса>` | Выводит детальные атрибуты ресурса из стейта.                                                            |
| `terraform state rm <адрес>`           | Удаляет привязку ресурса из стейта **без удаления реального объекта**.                                   |
| `terraform state mv <старый> <новый>`  | Переименовывает/перемещает ресурс в стейте (полезно при рефакторинге).                                   |
| `terraform state pull` / `push`        | Ручное скачивание/загрузка стейта (обычно используется для резервного копирования или миграции бэкенда). |

**Импорт и замена:**

|Команда|Назначение|
|---|---|
|`terraform import <адрес_ресурса> <ID_в_облаке>`|Добавляет уже существующую инфраструктуру в стейт, чтобы Terraform начал ею управлять.|
|`terraform apply -replace="<адрес_ресурса>"`|Принудительно пересоздаёт ресурс. **Заменил устаревшие** `terraform taint` и `untaint`.|

**Workspace и окружения:**

Воркспейсы удобны для быстрых тестов, но в production чаще используют разделение по директориям (`environments/dev/`, `environments/prod/`) или отдельные репозитории.

| Команда                            | Назначение                                               |
| ---------------------------------- | -------------------------------------------------------- |
| `terraform workspace list`         | Показывает доступные воркспейсы.                         |
| `terraform workspace new <имя>`    | Создаёт изолированное окружение (отдельный файл стейта). |
| `terraform workspace select <имя>` | Переключается между окружениями.                         |

**Выводы и отладка:**

| Команда                                    | Назначение                                                                                                    |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `terraform output [-json]`                 | Показывает значения, объявленные в блоках `output`. Полезно для передачи данных между модулями или скриптами. |
| `terraform console`                        | Интерактивная REPL-консоль для тестирования функций HCL, выражений и обращений к переменным/стейту.           |
| `terraform providers`                      | Выводит дерево используемых провайдеров и их версии.                                                          |
| `terraform graph \| dot -Tpng > graph.png` | Генерирует граф зависимостей ресурсов (требует установленного `graphviz`).                                    |

### HCL базовые конструкции:

HCL — **декларативный** язык, оптимизированный для описания инфраструктуры. Вы указываете что должно существовать, а Terraform сам вычисляет _как_ этого добиться, учитывая зависимости и порядок операций.

#### Основные блоки конфигурации:

##### `terraform {}` - Настройки самого Terraform:

Управляет версиями, бэкендом и провайдерами.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "my-tf-state-bucket"
    key    = "prod/terraform.tfstate"
    region = "eu-west-1"
  }
}
```

##### `provider {}` - Настройки для провайдеров:

```hcl
provider "aws" {
  region = var.aws_region
  alias  = "west"          # Алиас для работы в нескольких регионах/аккаунтах
}

provider "aws" {
  region = "us-east-1"
  alias  = "east"
}
```

##### `variable {}` — Декларация переменных:

Декларация переменных которые могут быть использованы в конфигурациях, и которые могут быть переданы при применении конфигураций.

```hcl
variable "environment" {
  description = "Окружение: dev, staging, prod"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Допустимые значения: dev, staging, prod"
  }
}

variable "instance_count" {
  type        = number
  sensitive   = false
}
```

##### `locals {}` — Локальные вычисляемые переменные:

Не принимают значения снаружи, нужны для DRY и переиспользования логики внутри модуля.

```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = "web-app"
  }
  is_prod = var.environment == "prod" ? true : false
}
```

##### `data {}` — Чтение существующих ресурсов (только для чтения):

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# Использование: data.aws_ami.ubuntu.id
```

##### `resource {}` — Создание/управление инфраструктурой:

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-web-server"
  })

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [tags["LastModified"]]
  }
}
```

**Ключевые мета-аргументы ресурсов:**

|Аргумент|Назначение|
|---|---|
|`depends_on`|Явная зависимость (использовать только если неявные ссылки не сработали)|
|`count` / `for_each`|Генерация нескольких экземпляров|
|`provider`|Выбор конкретного провайдера (при наличии алиасов)|
|`lifecycle`|Управление поведением создания/обновления/удаления|
##### `output {}` — Экспорт значений после `apply`:

```hcl
output "instance_ip" {
  description = "Публичный IP веб-сервера"
  value       = aws_instance.web.public_ip
  sensitive   = false
}

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true  # Не выводится в консоль, только в стейт
}
```

##### `module {}` — Вызов переиспользуемого блока:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "${var.environment}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["eu-west-1a", "eu-west-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}
```

#### Выражения, ссылки и логика:

##### Ссылки на другие блоки:

Синтаксис: `<тип_блока>.<имя_блока>.<атрибут>`

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id          # ссылка на data
  subnet_id     = module.vpc.public_subnets[0]    # ссылка на module
  user_data     = file("scripts/init.sh")         # ссылка на файл
}
```

##### Условные выражения:

```hcl
instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

# Безопасный доступ к атрибутам (TF 0.13+)
region_name = try(var.region_override, data.aws_region.current.name)
```

##### Коллекции и доступ:

```hcl
# Списки
availability_zones = ["eu-west-1a", "eu-west-1b"]
first_az           = var.azs[0]

# Карты (мапы)
subnet_map = {
  prod = "subnet-111"
  dev  = "subnet-222"
}
target_subnet = var.subnet_map[var.environment]
```

#### Генерация ресурсов: `count`, `for_each`, `dynamic`:

| Конструкция          | Тип входных данных | Когда использовать                                                      |
| -------------------- | ------------------ | ----------------------------------------------------------------------- |
| `count = N`          | Число              | Одинаковые ресурсы, простые случаи                                      |
| `for_each = map/set` | Коллекция          | Уникальные ресурсы, изменение отдельных элементов без пересоздания всех |
| `dynamic {}`         | Любой iterable     | Повторяющиеся вложенные блоки внутри ресурса                            |

```hcl
# count
resource "aws_instance" "workers" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  tags          = { Name = "worker-${count.index}" }
}

# for_each (рекомендуется для production)
resource "aws_security_group_rule" "ingress" {
  for_each = {
    http  = 80
    https = 443
  }
  type              = "ingress"
  from_port         = each.value
  to_port           = each.value
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_sg.main.id
}

# dynamic (вложенные блоки)
resource "aws_instance" "with_volumes" {
  dynamic "ebs_block_device" {
    for_each = var.extra_volumes
    content {
      device_name = ebs_block_device.value.device
      volume_size = ebs_block_device.value.size
    }
  }
}
```

### Организация файлов (конвенции):

Terraform **не требует** строгого именования файлов. Он склеивает все `*.tf` и `*.tf.json` в директории в единую конфигурацию. Но сообщество выработало стандарт:

```Structure
project/
├── main.tf           # Ресурсы, data, locals
├── variables.tf      # Объявления переменных
├── outputs.tf        # Выходные значения
├── providers.tf      # Блоки terraform {} и provider {}
├── locals.tf         # Вычисляемые значения
├── versions.tf       # required_version, required_providers (опционально)
├── modules/          # Локальные модули
├── envs/
│   ├── dev.tfvars
│   └── prod.tfvars
└── .gitignore
```

> Не создавайте `main.tf` размером в 2000 строк. Разбивайте по логическим доменам: `vpc.tf`, `compute.tf`, `db.tf`, `iam.tf`.