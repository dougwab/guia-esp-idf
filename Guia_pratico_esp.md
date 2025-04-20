# Guia Prático ESP-IDF 🤖

> Um repositório para desenvolvedores que querem dominar o desenvolvimento embarcado com ESP32 utilizando a **ESP-IDF**. Este guia vai evoluindo conforme novos tópicos forem sendo explorados.

---

## ✨ Introdução

Este repositório tem como objetivo apresentar de forma clara, acessível e direta os principais recursos da ESP-IDF, com exemplos comentados e boas práticas para quem está entrando no mundo do desenvolvimento embarcado com ESP32.

O foco é manter um padrão didático sem abrir mão dos aspectos técnicos. O guia será expandido ao longo do tempo com tópicos como:

- [x] Modularização de componentes
- [x] FreeRTOS e multitarefa
- [ ] GPIO e interrupções
- [ ] Conexão Wi-Fi
- [ ] Bluetooth Low Energy (BLE)
- [ ] Modo de baixo consumo (Low Power)
- [ ] OTA (Over-the-Air Updates)

---

## ⚙️ Comandos Básicos do `idf.py`

```bash
get_idf                             # Ativa as variáveis de ambiente da ESP-IDF
idf.py create-project <nome>       # Cria um novo projeto base
idf.py set-target esp32            # Define o chip alvo (ex: esp32)
idf.py build                       # Compila o projeto
idf.py flash                       # Grava o firmware na placa
idf.py monitor                     # Abre o monitor serial
idf.py flash monitor               # Combina flash + monitor
idf.py clean                       # Limpa os arquivos de build
idf.py fullclean                   # Limpa completamente (inclusive dependências)
```

---

## 📂 Criando um Projeto com a ESP-IDF

Após configurar um atalho para o ambiente da ESP-IDF:
```bash
alias get_idf='. $HOME/esp/esp-idf/export.sh'
```

Basta rodar `get_idf` no terminal, ir até a pasta desejada e executar:
```bash
idf.py create-project <nome_do_projeto>
```

Esse comando cria:
- Um diretório `main` com um arquivo `.c`
- Dois arquivos `CMakeLists.txt`: um na raiz e outro dentro de `main`

### Por que dois arquivos CMake?
- O da **raiz** é como um "gerente geral" do projeto, informando quais diretórios participam da build.
- O de **main** é o "chefe local", que lista os arquivos `.c/.h` daquele componente.

Esse esquema reflete o modelo **modular** da ESP-IDF, essencial para projetos escaláveis e organizados.

---

## 🛠️ Estruturando Componentes na ESP-IDF

Na ESP-IDF, é comum dividir o projeto em **componentes** para manter o código limpo e reutilizável. Em vez de concentrar tudo no `main.c`, você pode criar arquivos como `sensor.c`, `lcd.c`, `wifi.c`, cada um com sua responsabilidade.

Cada componente possui:
- Arquivos `.c` e `.h`
- Um `CMakeLists.txt` próprio (caso estejam em diretórios separados)

Essa organização facilita:
- Reutilização de código entre projetos
- Manutenção e testes modulares
- Trabalho em equipe e controle de versão

---

## ⏳ FreeRTOS no ESP32

O ESP32 roda **FreeRTOS**, um sistema operacional de tempo real que permite o uso de multitarefas com controle de prioridade, atrasos não bloqueantes e acesso organizado a recursos compartilhados.

### Bibliotecas:
```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
```

### Exemplo Básico de Task:
```c
void task1(void *params) {
    while (1) {
        printf("Leitura de sensores\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void task2(void *params) {
    while (1) {
        printf("Escrever no display\n");
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

void app_main(void) {
    xTaskCreate(&task1, "leitura", 2048, NULL, 1, NULL);
    xTaskCreate(&task2, "display", 2048, NULL, 1, NULL);
}
```

### Logs ao invés de `printf`

```c
#include "esp_log.h"
static const char *TAG = "leitura";
ESP_LOGI(TAG, "Leitura de sensores");
```

### Selecionando Núcleo:
```c
xTaskCreatePinnedToCore(&task2, "display", 2048, NULL, 1, NULL, 0); // 0 ou 1
```

---

## 🔒 Concorrência e Mutex

Quando duas tasks acessam o mesmo recurso (ex: barramento I2C), pode haver conflito.

**Solução:** use `mutex` para garantir acesso exclusivo:

```c
#include "freertos/semphr.h"
```

### Exemplo com sensor e LCD modularizados:

**sensor.c**
```c
// [...]
if (xSemaphoreTake(mutexI2C, 1000 / portTICK_PERIOD_MS)) {
    temperatura = acessa_i2c(1);
    ESP_LOGI(TAG_SENSOR, "Temperatura: %f", temperatura);
    xSemaphoreGive(mutexI2C);
}
```

**lcd.c**
```c
// [...]
if (xSemaphoreTake(mutexI2C, 1000 / portTICK_PERIOD_MS)) {
    acessa_i2c_display();
    xSemaphoreGive(mutexI2C);
}
```

**main.c**
```c
SemaphoreHandle_t mutexI2C;
mutexI2C = xSemaphoreCreateMutex();
```

---

## 📊 Estrutura de Projeto

**CMakeLists.txt (raiz):**
```cmake
cmake_minimum_required(VERSION 3.5)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(esp_modular_project)
```

**CMakeLists.txt (main):**
```cmake
idf_component_register(SRCS "main.c" "sensor.c" "lcd.c"
                    INCLUDE_DIRS ".")
```

---

## 🖋️ Em breve...

- Controle de GPIO com interrupções
- Wi-Fi Station/Access Point
- BLE para comunicação com smartphones
- Otimização de consumo com modos sleep
- Atualização remota com OTA

Fique ligado! ✨

---

Quer contribuir com exemplos, correções ou sugestões? Fique à vontade para abrir um PR ou uma issue 🚀

