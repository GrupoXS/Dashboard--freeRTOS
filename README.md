# Projeto 2 — Automotive Dashboard (Cluster Digital EV) com FreeRTOS e ESP32

Protótipo de painel de instrumentos automotivo (*Instrument Cluster / ICM*) para veículo elétrico, controlado em tempo real por um microcontrolador **ESP32** executando **FreeRTOS**, integrado a uma interface gráfica web (**VOLT**) via comunicação **Serial (JSON)**.

## Integrantes do Grupo
- Daniel Pini
- João Guilherme
- Leonardo Amador
- Luiz Brito
- Pedro Sanchez

---

## 1. Objetivo e Decisão de Arquitetura

Desenvolver um protótipo funcional de cluster digital automotivo que demonstre os conceitos fundamentais de um **Sistema Operacional de Tempo Real (RTOS)** — como escalonamento de tarefas (*scheduler*), execução periódica, isolamento de rotinas e gerenciamento de memória de pilha (*stack*).

### Evolução da Arquitetura (Trampoline POSIX → FreeRTOS em ESP32)
Na concepção inicial do projeto, previa-se o uso do Trampoline RTOS em ambiente simulado (POSIX/Linux) sem hardware físico. Durante o desenvolvimento, o grupo optou por migrar para o **FreeRTOS embarcado no microcontrolador ESP32**. Essa decisão permitiu:
- Validar o comportamento determinístico em um **hardware real** de baixo custo e alta capacidade de processamento;
- Manter todos os requisitos de tempo real (criação e escalonamento de *tasks* independentes);
- Comprovar em bancada o controle físico do painel pela placa embarcada.

---

## 2. Arquitetura do Sistema

```text
+---------------------------------------+
|  ESP32 (Firmware em C + FreeRTOS)     |
|  - Struct de telemetria EV            |
|  - Task periódica (1 s) no Scheduler  |
|  - Serialização em formato JSON       |
+-------------------+-------------------+
                    |
        Comunicação Serial (USB/UART)
                 (JSON)
                    |
                    v
+---------------------------------------+
|  Servidor Local (Node.js - cluster/)  |
|  - Leitura/validação dos dados JSON   |
|  - Distribuição via SSE (/api/events) |
+-------------------+-------------------+
                    |
         Server-Sent Events (SSE)
                    |
                    v
+---------------------------------------+
|  Interface Web VOLT (HTML/CSS/JS)     |
|  - Velocidade, Bateria, Autonomia,    |
|    Potência e Marcha em tempo real    |
+---------------------------------------+

```

### Por que Serial + JSON?

* **Formato JSON:** Adotado por ser o padrão amplamente utilizado na comunicação entre APIs e sistemas web, facilitando o *parsing* e a manutenção.
* **Porta Serial (em vez de Wi-Fi):** Inicialmente foi testada a atualização via Wi-Fi, mas a migração para a porta Serial simplificou a arquitetura, eliminou camadas desnecessárias de rede/segurança para o escopo de bancada e tornou a transmissão mais direta e estável.

---

## 3. Estrutura do Repositório

```text
Dashboard--freeRTOS/
├── cluster/                     # Servidor Node.js e interface gráfica VOLT
├── firmware_esp32/
│   └── cluster_firm/            # Código-fonte em C do ESP32 com FreeRTOS
├── AGENTS.md                    # Instruções auxiliares do repositório
└── README.md                    # Documentação principal do projeto

```

---

## 4. Funcionamento do Firmware (ESP32 + FreeRTOS)

O firmware em `firmware_esp32/cluster_firm/` implementa os seguintes conceitos:

* **Estrutura de Dados (`struct`):** Define um tipo personalizado em C para agrupar os sinais do veículo (`speed`, `battery`, `range`, `power` e `gear`). A estrutura é facilmente expansível para novos indicadores (como setas esquerda/direita).
* **Inicialização Segura:** Na partida do sistema, a instância da `struct` é preenchida com valores iniciais conhecidos, evitando lixo de memória ou exibições inválidas na tela ao ligar.
* **Tarefa Periódica no RTOS (*Task*):** Uma tarefa dedicada é registrada no *scheduler* do FreeRTOS para rodar periodicamente a cada **1 segundo**, atualizando os sinais de telemetria, montando a string JSON e enviando-a pela porta Serial.
* **Simulação de Sensores:** Em um veículo real, cada campo seria atualizado por leituras físicas (por exemplo, `analogRead` em pinos conectados a sensores de tensão da bateria). Para viabilizar a demonstração em bancada sem a necessidade de múltiplos potenciômetros e chaves físicas, os valores são atualizados com dados simulados/randômicos.
* **Gerenciamento de Memória (*Stack*):** Na criação da *task* durante o `setup()`, são alocados **4096 bytes** de memória. Dimensionar adequadamente essa pilha é essencial no ESP32 para evitar estouro de memória e reinicializações inesperadas (*resets*).
* **Vantagem sobre o `void loop()` tradicional:** Em uma programação sequencial comum dentro do `void loop()`, qualquer leitura ou rotina lenta trava todo o restante do código e gera atraso (*delay*) na tela. Com o FreeRTOS, rotinas mais lentas podem ser isoladas em *tasks* separadas sem impactar a atualização das demais variáveis.

---

## 5. Sinais de Telemetria Demonstrados

| Campo JSON | Descrição | Uso no Painel VOLT |
| --- | --- | --- |
| `speed` | Velocidade do veículo (km/h) | Velocímetro |
| `battery` | Nível de carga da bateria (%) | Indicador de bateria |
| `range` | Autonomia estimada (km) | Indicador de alcance |
| `power` | Potência / regeneração (kW) | Barra de potência |
| `gear` | Marcha selecionada (`P`, `R`, `N`, `D`, `B`) | Indicador de marcha |

---

## 6. Como Executar o Projeto

### Pré-requisitos

* Placa de desenvolvimento **ESP32** e cabo USB de dados;
* Ambiente de gravação compatível (ex.: **Arduino IDE** ou **PlatformIO**);
* **Node.js 20** ou superior instalado no computador.

### Passo 1 — Gravar o firmware no ESP32

1. Abra o projeto localizado em `firmware_esp32/cluster_firm/` na sua IDE.
2. Selecione a placa **ESP32** e a porta Serial (COM / `/dev/ttyUSB*`) correspondente.
3. Compile e faça o upload do código para a placa.

### Passo 2 — Iniciar a interface do cluster (VOLT)

1. Com o ESP32 conectado na porta USB, abra um terminal na pasta `cluster/`:
```bash
cd cluster
npm start

```


*(Verifique em `cluster/README.md` se é necessário informar a porta Serial nas variáveis de ambiente do seu sistema.)*
2. Abra o navegador em `http://localhost:3000` para visualizar o painel atualizando a cada 1 segundo com os dados enviados pelo hardware.

---

## 7. Validação em Bancada (Teste de Hardware)

Para comprovar que o painel está sendo efetivamente controlado pelo microcontrolador físico — e não por uma simulação local no navegador — foi realizado o **teste de interrupção física pelo botão `ENABLE` (Reset) do ESP32**:

1. **Operação normal:** Com o ESP32 ligado, os valores do painel VOLT atualizam continuamente a cada 1 segundo.
2. **Pressionando `ENABLE`:** O microcontrolador é mantido em estado de reset e a transmissão Serial é interrompida imediatamente.
3. **Liberando `ENABLE`:** O ESP32 reinicia o FreeRTOS, restabelece a porta Serial e volta a enviar os pacotes JSON automaticamente, retomando a atualização da interface.

```
