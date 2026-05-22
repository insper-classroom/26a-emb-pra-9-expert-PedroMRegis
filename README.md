# Mouse IMU FreeRTOS - Medições

Sistema de mouse por IMU em Raspberry Pi Pico (RP2040) com FreeRTOS. Quatro tasks no pipeline `mpu -> fusion -> {pwm, uart}`, instrumentadas por GPIO e medidas com Saleae Logic 2.

## Setup

Cada task levanta um GPIO no início do trabalho e baixa no fim. As métricas são extraídas por uma extensão `DigitalMeasurement` do Logic 2 que processa as transições de cada canal.

| Task | GPIO instrumentação |
|---|---|
| mpu | 15 |
| fusion | 11 |
| pwm | 12 |
| uart | 13 |

- WCET: maior duração de pulso alto. Inclui preempção (tempo de parede, não CPU pura).
- Jitter: maior desvio absoluto do período entre ativações (rise->rise) em relação ao período médio.
- Deadline miss rate: fração de períodos acima do deadline de 10 ms.
- Stack usage: `1 - HighWaterMark / StackSize`, lido via `uxTaskGetSystemState`.

## Parte 1 - Single Core

| Métrica | mpu | uart | pwm | fusion |
|---|---|---|---|---|
| WCET | 428,5 µs | 17 µs | 300 ns | 14,5 µs |
| Jitter | 1,81 µs | 3,689 µs | 3,589 µs | 2,11 µs |
| Deadline miss rate | 44% | 55% | 55% | 33,33% |
| Stack usage | 0,44% | 2,64% | 3,52% | 1,78% |

## Parte 2 - SMP

Afinidade: mpu no Core 0, fusion + pwm + uart no Core 1. A mpu é isolada por ser a única task pesada e bloqueante (I2C, ~380 µs por leitura). Prioridades mantidas (mpu 3, fusion 3, uart 2, pwm 1).

| Métrica | mpu | uart | pwm | fusion |
|---|---|---|---|---|
| WCET | 427,1 µs | 31,3 µs | 300 ns | 35,6 µs |
| Jitter | 66,67 ns | 5,144 µs | 6,144 µs | 1,633 µs |
| Deadline miss rate | 11,11% | 44,44% | 55,55% | 22,22% |
| Stack usage | 0,59% | 2,64% | 3,52% | 1,78% |

## Stack - regra dos 80%

Uso real medido no SMP e redimensionamento. O stack alvo mantém o uso abaixo de 80% sem desperdiçar RAM. O stack ajustado é dimensionado por `uso_real / 0,8`, arredondado para cima com folga e respeitando um piso prático de 128 palavras.

### Antes do ajuste

| Task | Stack alocado | Uso real | Stack usage |
|---|---|---|---|
| mpu | 8192 | 48 palavras | 0,59% |
| fusion | 8192 | 146 palavras | 1,78% |
| uart | 2048 | 54 palavras | 2,64% |
| pwm | 1024 | 36 palavras | 3,52% |

### Depois do ajuste

| Task | Stack alocado | Uso real | Stack usage |
|---|---|---|---|
| mpu | 256 | 48 palavras | 18,8% |
| fusion | 384 | 146 palavras | 38,0% |
| uart | 128 | 54 palavras | 42,2% |
| pwm | 128 | 36 palavras | 28,1% |

Antes do ajuste o uso ficava entre 0,59% e 3,52%, muito abaixo de 80%. O sistema era seguro, mas reservava cerca de 18 mil palavras de RAM (72 KB) para tasks que usavam menos de 600 B no total. Depois do ajuste todas as tasks ficam dentro de uma faixa saudável (18,8% a 42,2%), ainda confortavelmente abaixo do limite de 80%, liberando a RAM ociosa para o heap. A fusion recebeu folga maior (384 palavras, 38% de uso) por ser a task mais complexa: o caminho do clique exercita ramos de código que a captura pode não ter registrado por completo.

## Análise comparativa Single Core vs SMP

Variação de cada métrica do single core para o SMP:

| Métrica | Task | Single Core | SMP | Variação |
|---|---|---|---|---|
| WCET | mpu | 428,5 µs | 427,1 µs | praticamente igual |
| WCET | uart | 17 µs | 31,3 µs | subiu |
| WCET | pwm | 300 ns | 300 ns | igual |
| WCET | fusion | 14,5 µs | 35,6 µs | subiu |
| Jitter | mpu | 1,81 µs | 66,67 ns | caiu ~27x |
| Jitter | uart | 3,689 µs | 5,144 µs | subiu |
| Jitter | pwm | 3,589 µs | 6,144 µs | subiu |
| Jitter | fusion | 2,11 µs | 1,633 µs | caiu |
| Deadline miss | mpu | 44% | 11,11% | caiu |
| Deadline miss | uart | 55% | 44,44% | caiu |
| Deadline miss | pwm | 55% | 55,55% | igual |
| Deadline miss | fusion | 33,33% | 22,22% | caiu |

O SMP melhorou de forma clara a mpu, a task que foi isolada no Core 0. O jitter dela caiu cerca de 27 vezes, de 1,81 µs para 66,67 ns, porque sem nenhuma task concorrente no mesmo núcleo o release periódico fica quase perfeito. O deadline miss da mpu também caiu de 44% para 11,11%. A fusion, primeira beneficiária do pipeline, teve o jitter reduzido por não disputar mais com a mpu.

O custo dessa escolha aparece em uart e pwm. As duas ficaram concentradas com a fusion no Core 1, então sofrem mais preempção dela. Isso elevou o jitter de ambas e é a mesma causa do aumento do WCET de parede da uart (17 µs para 31,3 µs) e da fusion (14,5 µs para 35,6 µs): o pulso medido passa a incluir a preempção das tasks vizinhas do mesmo núcleo. Não houve aumento de carga computacional, apenas de tempo de parede.

O fator dominante foi a afinidade, não as prioridades. A afinidade definiu quais tasks executam em paralelo de fato e quais dividem núcleo; foi ela que isolou a fonte de interferência. A prioridade, sob SMP, só ordena tasks dentro de um mesmo núcleo, e como a mpu ficou sozinha no Core 0 a prioridade dela tornou-se irrelevante.

Conclusão: a divisão 1+3 privilegia o produtor do pipeline (mpu e, em seguida, fusion) e aceita uma degradação leve nas tasks folha (uart e pwm). Como todas operam com folga enorme dentro do orçamento de 10 ms, a degradação é irrelevante na prática.

