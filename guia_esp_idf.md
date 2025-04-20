
# Guia Prático ESP-IDF 🚀

Este guia prático tem como objetivo auxiliar no desenvolvimento de projetos utilizando o **ESP-IDF** para o ESP32. Vamos passar por diversos conceitos importantes, desde a criação de projetos até a utilização de componentes como **FreeRTOS**, **I2C**, e a modularização do código. A ideia é proporcionar uma compreensão clara e acessível dos tópicos técnicos, mantendo a precisão para o contexto embarcado.

## Comandos Básicos `idf.py` 🧑‍💻

Aqui estão alguns dos comandos básicos utilizados com o **idf.py** para gerenciar o ambiente e o build do seu projeto:

- `get_idf` ✨  
  Ativa as variáveis de ambiente necessárias para usar o **ESP-IDF**.

- `idf.py create-project <nome-do-projeto>` 🔧  
  Cria um novo projeto base com a estrutura necessária.

- `idf.py set-target esp32` 🧠  
  Define o chip alvo para o seu projeto (exemplo: ESP32).

- `idf.py build` 🔨  
  Compila o projeto gerando os arquivos binários para o firmware.

- `idf.py flash` ⚡  
  Grava o firmware na placa conectada via USB.

- `idf.py monitor` 📊  
  Abre o monitor serial para visualizar a saída do microcontrolador.

- `idf.py flash monitor` 💻⚡  
  Combina os comandos de flash e monitor em uma única execução.

- `idf.py clean` 🧹  
  Limpa os arquivos gerados durante o build.

- `idf.py fullclean` 🧼  
  Limpa completamente o projeto, incluindo dependências.

## Criando um Arquivo 📁

Após configurar um atalho para as variáveis de ambiente do **ESP-IDF** (por exemplo, com o comando `alias get_idf='. $HOME/esp/esp-idf/export.sh'`), basta rodar o comando `get_idf` no seu terminal para ativar o ambiente. Depois, navegue até o diretório onde deseja criar o projeto e use:

```bash
idf.py create-project <nome-do-projeto>
```

Esse comando cria a estrutura básica do seu projeto, com um diretório **main** contendo um arquivo `.c`, além de dois arquivos `CMakeLists.txt`: um na raiz do projeto e outro dentro do diretório **main**.

**Por que dois arquivos CMakeLists.txt?** 🤔  
- O arquivo da **raiz** do projeto funciona como o "gerente", indicando quais partes do projeto fazem parte da compilação.
- O arquivo **CMakeLists.txt** dentro do diretório **main** atua como o "chefe local", controlando os arquivos **.c** e **.h** daquele módulo específico.

A abordagem modular da **ESP-IDF** permite que cada componente do projeto tenha seu próprio controle de build, o que facilita o gerenciamento à medida que o projeto cresce.

## Estruturando Componentes dentro da ESP-IDF 🏗️

### FreeRTOS no ESP32 🕹️

O **ESP32** possui suporte ao **FreeRTOS**, um sistema operacional em tempo real que abstrai o controle de tempo e facilita a modularização do código. Através das **tasks**, o FreeRTOS permite o uso eficiente de interrupções e o gerenciamento de prioridades, organizando melhor o fluxo de execução e tornando o sistema mais responsivo.

#### Bibliotecas utilizadas:

```c
#include "freertos/FreeRTOS.h"  
#include "freertos/task.h"
```

### Exemplo Básico de Leitura e Escrita de Sensor 📡

Neste exemplo, duas **tasks** são criadas: uma para fazer a leitura de um sensor e outra para exibir os dados em um display.

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

// Função que será executada como task
void task1(void *params)
{
    while (1) {
        printf("Leitura de sensores
");
        vTaskDelay(pdMS_TO_TICKS(1000)); // Aguarda 1000 ms (1 segundo)
    }
}

void task2(void *params)
{
    while (1) {
        printf("Escrever no display
");
        vTaskDelay(pdMS_TO_TICKS(5000)); // Aguarda 5000 ms (5 segundos)
    }
}

void app_main(void)
{
    // Criação das tasks
    xTaskCreate(&task1, "leitura", 2048, NULL, 1, NULL);
    xTaskCreate(&task2, "display", 2048, NULL, 1, NULL);
}
```

### Utilizando `LOG` ao invés de `printf()` 🖥️

Em projetos embarcados, é preferível usar a biblioteca de **logging** ao invés de `printf()`, pois ela permite um controle melhor sobre os níveis de log, além de ser mais eficiente em termos de uso de recursos.

```c
#include "esp_log.h"

static const char *TAG_1 = "leitura", *TAG_2 = "escrita";
ESP_LOGI(TAG_1, "Leitura de sensores");
```

Onde:
- `LOGI` = Informação ℹ️
- `LOGW` = Aviso ⚠️
- `LOGE` = Erro ❌

### Selecionando o Núcleo onde a Task Será Rodada 🧠

Em sistemas multi-core, o **ESP32** permite escolher em qual núcleo uma **task** será executada:

```c
xTaskCreatePinnedToCore(&task2, "display", 2048, NULL, 1, NULL, 0); // Escolhendo núcleo 0 ou 1
```

## Problema e Solução: Acesso Concorrente ⚔️

### Problema:

Quando duas ou mais tasks acessam o mesmo recurso compartilhado (como um barramento I2C, por exemplo), pode haver conflitos ou até corrupção de dados.

### Solução:

Usar um **mutex** (exclusive access) para garantir que apenas uma task acesse o recurso por vez. Isso evita problemas de integridade de dados e comportamentos inesperados.

#### Exemplo com Módulos de Leitura e Escrita de Sensor

##### `sensor.c`

```c
#include "sensor.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "esp_log.h"

extern SemaphoreHandle_t mutexI2C;
static const char *TAG_SENSOR = "Sensor";
float temperatura;

float acessa_i2c(int comando)
{
    if (comando == 1) {
        ESP_LOGI("I2C", "Leitura do sensor de Temperatura");
        return 20.0 * ((float) rand() / (float)(RAND_MAX / 10));
    }
    return 0;
}

void le_sensor(void *params)
{
    while (true) {
        if (xSemaphoreTake(mutexI2C, 1000 / portTICK_PERIOD_MS)) {
            temperatura = acessa_i2c(1);
            ESP_LOGI(TAG_SENSOR, "Temperatura: %f", temperatura);
            xSemaphoreGive(mutexI2C);
            vTaskDelay(2000 / portTICK_PERIOD_MS);
        } else {
            ESP_LOGI(TAG_SENSOR, "Não foi possível ler o sensor");
        }
    }
}
```

##### `sensor.h`

```c
#ifndef SENSOR_H
#define SENSOR_H

void le_sensor(void *params);

#endif
```

##### `lcd.c`

```c
#include "lcd.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "esp_log.h"
#include <stdio.h>

extern SemaphoreHandle_t mutexI2C;
extern float temperatura;
static const char *TAG_LCD = "LCD";

void acessa_i2c_display()
{
    ESP_LOGI("I2C", "Escrita do LCD");
    printf("Tela LCD - Temperatura = %f
", temperatura);
}

void lcd_display(void *params)
{
    while (true) {
        if (xSemaphoreTake(mutexI2C, 1000 / portTICK_PERIOD_MS)) {
            ESP_LOGI(TAG_LCD, "Escrevendo no LCD");
            acessa_i2c_display();
            xSemaphoreGive(mutexI2C);
            vTaskDelay(2000 / portTICK_PERIOD_MS);
        } else {
            ESP_LOGI(TAG_LCD, "Não foi possível escrever no display");
        }
    }
}
```

##### `lcd.h`

```c
#ifndef LCD_H
#define LCD_H

void lcd_display(void *params);

#endif
```

##### `main.c`

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "sensor.h"
#include "lcd.h"

SemaphoreHandle_t mutexI2C;

void app_main(void)
{
    mutexI2C = xSemaphoreCreateMutex();
    xTaskCreate(&le_sensor, "Leitura Sensor", 2048, NULL, 2, NULL);
    xTaskCreate(&lcd_display, "Display LCD", 2048, NULL, 2, NULL);
}
```

## Arquivos CMake 🛠️

O arquivo `CMakeLists.txt` na raiz do projeto deve ficar assim:

```cmake
cmake_minimum_required(VERSION 3.5)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(esp_modular_project)
```

Enquanto o arquivo `CMakeLists.txt` dentro do diretório **main** deve ser:

```cmake
idf_component_register(SRCS "main.c" "sensor.c" "lcd.c"
                    INCLUDE_DIRS ".")
```

