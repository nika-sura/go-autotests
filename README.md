# go-autotests

Автотесты для курса «Go-разработчик».

## Локальный запуск

### Трек «Сервис сокращения URL»

1. Скомпилируйте ваш сервер в папке `cmd/shortener` командой `go build -o shortener *.go`.
2. Скачайте [бинарный файл с автотестами](https://github.com/Yandex-Practicum/go-autotests/releases/latest) для вашей ОС — например, `shortenertest-darwin-arm64` для MacOS на процессоре Apple Silicon.
3. Разместите бинарный файл так, чтобы он был доступен для запуска из командной строки, — пропишите путь в переменную `$PATH`.
4. Ознакомьтесь с параметрами запуска автотестов в файле `.github/workflows/shortenertest.yml` вашего репозитория. Автотесты для разных инкрементов требуют различных аргументов для запуска.

Пример запуска теста для первого инкремента:

```shell
shortenertest -test.v -test.run=^TestIteration1$ -binary-path=cmd/shortener/shortener
```

### Трек «Сервис сбора метрик и алертинга»

1. Скомпилируйте ваши сервер и агент в папках `cmd/server` и `cmd/agent` командами `go build -o server *.go` и `go build -o agent *.go` соответственно.
2. Скачайте [бинарный файл с автотестами](https://github.com/Yandex-Practicum/go-autotests/releases/latest) для вашей ОС — например, `devopstest-darwin-arm64` для MacOS на процессоре Apple Silicon.
3. Разместите бинарный файл так, чтобы он был доступен для запуска из командной строки, — пропишите путь в переменную `$PATH`.
4. Ознакомьтесь с параметрами запуска автотестов в файле `.github/workflows/devopstest.yml` вашего репозитория. Автотесты для разных инкрементов требуют различных аргументов для запуска.

Пример запуска теста для первого инкремента:

```shell
devopstest -test.v -test.run=^TestIteration1$ -agent-binary-path=cmd/agent/agent
```

## Автоматический запуск 

Для проверки того, насколько код инкремента удовлетворяет функциональным требованиям задания, мы используем автотесты. Они запускаются внутри CI/CD-инструмента под названием **GitHub Actions**. 

GitHub Actions (далее GA) позволяет с помощью собственного синтаксиса описывать некоторый набор операций (**workflow**), которые должны применяться к коду автоматически при изменениях или при нажатии кнопки. На практике с помощью CI/CD часто автоматизируют синтаксические анализаторы, автотесты, генерацию документации, выкатку на различные окружения и так далее. При этом все эти действия выполняются не в вакууме: в нашем случае GitHub предоставляет свои машины (runner'ы), где код и будет выполняться в соответствии с заготовленными конфигурациями.

Если вкраце, GA разбивает каждый `workflow` на несколько `job`, которые могут работать последовательно или параллельно. Джобы в свою очередь состоят из одного или нескольких `steps`, в которых можно указывать конкретные действия, которые должны быть выполнены. Также вы можете указать условия выполнения `workflow` (например, на каждый коммит в ветке `main`), что позволяет гибко настраивать все ваши самые извращённые CI/CD-фантазии.

<details>
  <summary>ПРИМЕР WORKFLOW-КОНФИГА, КОТОРЫЙ МЫ ИСПОЛЬЗУЕМ ДЛЯ АВТОМАТИЧЕСКОГО ПРИМЕНЕНИЯ go vet</summary>
  
```yaml
name: go vet test # название workflow
# если вы хотите настроить автоматическое применение workflow в зависимости от условий,
# то используйте ключевое слово on
on: 
  pull_request: # здесь говорите, что workflow должен запускаться для любого события внутри PR (пуш, тег и другие)
  push:
    branches: # здесь говорите, что хотите применять workflow и для пушей в main-ветку
      - main
# каждый workflow представляет собой набор джоб,
# которые выполняются последовательно или параллельно,
# для каждой джобы можно задать докер-образ, в котором будут выполняться шаги (steps),
# и ОС, в которой будет запущен контейнер
jobs:
  statictest: # описываете джобу statictest
    runs-on: ubuntu-latest # говорите, что джоба должна выполняться на машине с убунтой (предоставляется гитхабом)
    container: golang:1.18 # запускаете в ней контейнер докер-образа golang:1.18
    steps: # последовательно выполняемые шаги
      - name: Checkout code
        # GitHub Actions позволяет самим описывать команды линукса внутри шага
        # или использовать заготовленные шаги, как здесь
        uses: actions/checkout@v2 
      - name: Download statictest binary
        # или здесь
        uses: robinraju/release-downloader@v1
        with:
        # для заготовленных шагов иногда требуется указать параметры, как здесь
          repository: Yandex-Practicum/go-autotests
          latest: true
          fileName: statictest
          out-file-path: .tools
      - name: Setup autotest binary
        # тут описываете произвольный набор команд
        run: |
          chmod -R +x $GITHUB_WORKSPACE/.tools/statictest
          mv $GITHUB_WORKSPACE/.tools/statictest /usr/local/bin/statictest
      - name: Run statictest
        run: |
          go vet -vettool=$(which statictest) ./...
```
</details>

Работает это следующим образом:

1. Когда вы подтягиваете шаблон проекта, вместе с ним подтягивается директория `.github/workflows`. В ней находятся два `workflow`-файла: `statictest.yaml` и `shortenertest.yaml` или `devopstest.yaml` (в зависимости от того, какой шаблон вы стянули).

2. В следующий раз, когда вы выполните изменения в репозитории (например, запушите коммит), которые попадают под условия, описанные в `on` конфигов `workflow` в папке `.github`, GA выполнит все джобы в каждом файле (по умолчанию — параллельно).

3. Вы можете следить за ходом выполнения джоб в интерфейсе Pull Request или перейти во вкладку `Actions`.

![16](https://user-images.githubusercontent.com/85521342/208649160-1717b258-8738-4cfa-8de5-45a769efb9a6.png)

4. Слева во вкладке `Actions` есть список `workflow`. Например, если вы хотите посмотреть на ход выполнения автотестов, переходите в `autotests`.

![17](https://user-images.githubusercontent.com/85521342/208649323-4021f284-1300-4e4d-95c2-9d16c6a52bb7.png)

5. Перейдя в интересующий вас `workflow`, вы увидите страницу с историей выполнения только этого `workflow`. Чаще всего вас интересует последний, так как он, вероятно, был выполнен на самой актуальной версии кода. Кстати, не удивляйтесь тому, что `workflow` помечен красной иконкой, — она сменится на приятную глазу зелёную, только когда вы выполните все инкременты курса. 

![18](https://user-images.githubusercontent.com/85521342/208649489-849b6407-61f9-4817-9657-08920e1b42c4.png)

6. Если вы зайдёте в последний выполненный автотест `workflow`, то увидите несколько джоб.

![19](https://user-images.githubusercontent.com/85521342/208649596-293254f1-8830-4c40-87ad-7287dd30c314.png)

7. Возможно, с первого раза ваш код не пройдёт автотесты. Разберём провалившийся тест одного из инкрементов.

![20](https://user-images.githubusercontent.com/85521342/208649656-a47edd7c-3c20-4231-9aca-9aaeea0cbcbe.png)
![21](https://user-images.githubusercontent.com/85521342/208649661-6673cf56-1a26-4084-8ea6-9cb659d4aac6.png)
![22](https://user-images.githubusercontent.com/85521342/208649665-b36f7ead-6066-4a2b-a60d-9a11338ad80d.png)

8. После того, как у вас завалится какой-то из шагов джобы, джоба завершится, а все контейнеры и их содержимое удалятся. 
