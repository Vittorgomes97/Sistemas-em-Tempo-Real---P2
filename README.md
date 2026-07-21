# Hospital Automatizado com ESP32 e FreeRTOS

## Visão geral

Este projeto simula o funcionamento de um hospital em um sistema embarcado de tempo real. A implementação utiliza **ESP32**, **FreeRTOS** e o simulador online **Wokwi**.

O modelo possui:

- 3 consultórios;
- 3 maqueiros;
- 4 leitos de recuperação;
- triagem por prioridade;
- modos automático, manual e teste;
- sinalização por LEDs;
- sincronização do cheat com uma tarefa periódica.

## Simulador

A montagem foi desenvolvida no **Wokwi** com uma placa **ESP32 DevKit V1**.

O arquivo `diagram.json` define:

- switches de seleção de modo;
- botões de paciente, alta e cheat;
- LEDs dos modos e consultórios;
- LED de recuperação cheia;
- resistores;
- analisador lógico.

## Implementação

O sistema foi estruturado sem um mutex global controlando todo o hospital. A comunicação entre as tarefas ocorre por:

- **filas:** pacientes, trabalhos, recursos livres e logs;
- **semáforo contador:** controle dos quatro leitos;
- **Event Group:** estados dos consultórios e pedido de cheat;
- **notificações:** ativação do despachante e de outras tarefas;
- **`vTaskDelayUntil()`:** execução periódica da amostragem, entradas e sinalização.

Fluxo principal:

```text
Triagem
   ↓
Filas por prioridade
   ↓
Despachante
   ↓
3 maqueiros
   ↓
3 consultórios
   ↓
Recuperação com 4 leitos
   ↓
Alta
```

Também foram implementados:

- fila individual para cada maqueiro;
- distribuição equilibrada dos transportes;
- debounce dos botões;
- envelhecimento de prioridade para evitar espera indefinida;
- logging assíncrono;
- monitor simples do estado do hospital.

## Modos de operação

| Modo 1 | Modo 0 | Funcionamento |
|---:|---:|---|
| 0 | 0 | Automático |
| 0 | 1 | Manual |
| 1 | 0 | Teste/Cheat |
| 1 | 1 | Reservado |

### Automático

Pacientes e altas são gerados em intervalos aleatórios.

### Manual

Os botões `PACIENTE` e `ALTA` controlam as entradas e saídas da recuperação.

### Teste/Cheat

O botão `CHEAT` solicita um paciente vermelho prioritário. A solicitação é executada apenas no próximo pulso de amostragem.

## Triagem

| Cor | Prioridade |
|---|---:|
| Vermelho | 1 |
| Laranja | 2 |
| Azul | 3 |

Para impedir starvation:

- azul passa a laranja após 15 s;
- azul passa a vermelho após 30 s;
- laranja passa a vermelho após 20 s.

## Como executar

1. Crie um projeto ESP32 no Wokwi.
2. Copie `sketch.ino` para o editor de código.
3. Copie `diagram.json` para o editor do circuito.
4. Inicie a simulação.
5. Selecione o modo pelos switches.
6. Observe os LEDs, o monitor serial e o analisador lógico.

## Estrutura do repositório

```text
hospital-freertos/
├── README.md
├── sketch.ino
├── diagram.json
└── docs/
    └── relatorio.pdf
```

## Vídeo de Demonstração



## Limitação

O Wokwi valida a lógica do sistema e a comunicação entre tarefas, mas não substitui totalmente testes em um ESP32 físico.
