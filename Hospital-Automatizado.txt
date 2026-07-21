#include <Arduino.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include <stdarg.h>
#include <stdint.h>
#include <stdio.h>

/*
  HOSPITAL AUTOMATIZADO COM ESP32 E FREERTOS

  Estrutura do sistema:
  - 3 consultorios independentes;
  - 3 maqueiros executando em tarefas separadas;
  - 4 leitos de recuperacao;
  - triagem dividida em filas por prioridade;
  - despachante responsavel por distribuir trabalhos;
  - sem mutex global: a comunicacao ocorre principalmente por filas,
    notificacoes, Event Group e semaforo contador.

  Os comentarios abaixo destacam os pontos relevantes da arquitetura,
  da sincronizacao e do fluxo dos pacientes.
*/

// -------------------- PINOS --------------------
// LEDs que indicam o modo atual: automatico, manual ou teste.
constexpr uint8_t LED_MODO[3] = {2, 4, 5};

// Um LED para indicar se cada consultorio esta reservado ou ocupado.
constexpr uint8_t LED_CONSULTORIO[3] = {18, 19, 22};

// Acende quando nao existe nenhum leito disponivel para reserva.
constexpr uint8_t LED_REC_CHEIA = 21;

// Dois switches formam os bits usados para selecionar um dos modos.
constexpr uint8_t SW_MODO_0 = 32;
constexpr uint8_t SW_MODO_1 = 33;

// Botoes usados nos modos manual e teste.
constexpr uint8_t BTN_PACIENTE = 13;
constexpr uint8_t BTN_ALTA = 14;
constexpr uint8_t BTN_CHEAT = 27;

// Saidas observadas pelo analisador logico:
// amostragem periodica, execucao do cheat e instante do pedido.
constexpr uint8_t PULSO_AMOSTRA = 25;
constexpr uint8_t PULSO_CHEAT = 26;
constexpr uint8_t PULSO_PEDIDO = 23;

// -------------------- CONFIGURACAO --------------------
// N representa simultaneamente o numero de consultorios e maqueiros.
constexpr int N = 3;

// Quantidade total de vagas na recuperacao.
constexpr int MAX_LEITOS = 4;

// Capacidade individual das filas vermelha, laranja e azul.
constexpr int CAP_PRIORIDADE = 40;

constexpr uint32_t CHEGADA_MIN = 1500;
constexpr uint32_t CHEGADA_MAX = 3500;
constexpr uint32_t TRANSPORTE = 1000;
constexpr uint32_t ATEND_MIN = 2000;
constexpr uint32_t ATEND_MAX = 4000;
constexpr uint32_t ALTA_MIN = 4000;
constexpr uint32_t ALTA_MAX = 7000;

constexpr uint32_t T_AMOSTRA = 1000;
constexpr uint32_t T_PULSO = 80;
constexpr uint32_t T_ENTRADAS = 20;
constexpr uint32_t T_LEDS = 50;
constexpr uint32_t T_MONITOR = 5000;
constexpr uint32_t DEBOUNCE = 50;

/*
  Tempos de envelhecimento da prioridade.

  Essa regra evita starvation: um paciente de baixa prioridade nao pode
  permanecer indefinidamente sem atendimento quando continuam chegando
  pacientes mais graves.
*/
constexpr uint32_t AZUL_LARANJA = 15000;
constexpr uint32_t AZUL_VERMELHO = 30000;
constexpr uint32_t LARANJA_VERMELHO = 20000;

// Bit que registra um pedido de cheat pendente ate a proxima amostragem.
constexpr EventBits_t BIT_CHEAT = 1U << 0;

// -------------------- TIPOS --------------------
// Modos controlados pelos dois switches.
enum Modo : uint8_t { AUTOMATICO, MANUAL, TESTE, INVALIDO };

// Define se o paciente veio de uma chegada normal ou do cheat.
enum Admissao : uint8_t { NORMAL, CHEAT };

// Tipos de transporte que podem ser atribuídos a um maqueiro.
enum TipoTrabalho : uint8_t { PARA_CONSULTORIO, PARA_RECUPERACAO };

// Dados que acompanham o paciente durante todo o fluxo do hospital.
struct Paciente {
  int id;                 // Identificador unico.
  const char *cor;        // Classificacao de risco.
  uint8_t prioridade;     // 1=vermelho, 2=laranja, 3=azul.
  uint32_t chegada;       // Instante usado para ordenar e envelhecer a fila.
};

// Ordem de transporte enviada pelo despachante para um maqueiro.
struct Trabalho {
  TipoTrabalho tipo;
  Paciente paciente;
  int consultorio;
};

struct PedidoRecuperacao {
  Paciente paciente;
  int consultorio;
};

struct LogMsg {
  char texto[176];
};

// Estado necessario para eliminar oscilacoes mecanicas dos botoes.
struct Debounce {
  bool bruto;
  bool estavel;
  TickType_t mudouEm;
};

// -------------------- PROTOTIPOS --------------------
void registrar(const char *fmt, ...);
void acordarDespachante();
Modo lerModo();
void mudarModo(Modo modo);
const char *nomeModo(Modo modo);
EventBits_t bitConsultorio(int i);
uint32_t aleatorio(uint32_t min, uint32_t max);
int totalTriagem();
uint8_t prioridadeEfetiva(const Paciente &p, uint32_t agora);
bool enfileirarPaciente(const Paciente &p);
bool retirarPaciente(Paciente &p);
bool debounce(Debounce &d, bool leitura, TickType_t agora);
bool despacharRecuperacao();
bool despacharConsultorio();

void tLogger(void *);
void tEntradas(void *);
void tTriagem(void *);
void tDespachante(void *);
void tMaqueiro(void *);
void tConsultorio(void *);
void tAlta(void *);
void tAmostragem(void *);
void tCheat(void *);
void tSinalizacao(void *);
void tMonitor(void *);

void configurarPinos();
void falhar(const char *msg);
void criarTarefa(TaskFunction_t fn, const char *nome, uint32_t pilha,
                 void *arg, UBaseType_t prioridade, TaskHandle_t *handle = nullptr);

// -------------------- ESTADO E OBJETOS --------------------
// O modo e lido por varias tarefas; sistemaPronto impede inicio prematuro.
volatile Modo modoAtual = AUTOMATICO;
volatile bool sistemaPronto = false;

/*
  Filas FreeRTOS do sistema.

  Cada fila possui uma responsabilidade especifica. Essa separacao reduz
  a disputa entre tarefas e substitui um mutex global sobre todo o hospital.
*/
QueueHandle_t qLog;                       // Mensagens para a tarefa Logger.
QueueHandle_t qPrioridade[3];             // Filas vermelho, laranja e azul.
QueueHandle_t qPedidoTriagem;             // Pedidos manuais e pedidos do cheat.
QueueHandle_t qPedidoAlta;                // Pedidos manuais de alta.
QueueHandle_t qConsultoriosLivres;        // Indices dos consultorios disponiveis.
QueueHandle_t qMaqueirosLivres;           // Indices dos maqueiros disponiveis.
QueueHandle_t qPacienteConsultorio[N];    // Um paciente entregue a cada consultorio.
QueueHandle_t qAguardandoRecuperacao;     // Pacientes atendidos aguardando transporte.
QueueHandle_t qTrabalhoMaqueiro[N];       // Fila de trabalho individual de cada maqueiro.
QueueHandle_t qRecuperacao;               // IDs dos pacientes internados na recuperacao.

// Semaforo contador: cada token representa um leito ainda reservavel.
SemaphoreHandle_t leitosLivres;

// Event Group: guarda o pedido de cheat e os bits visuais dos consultorios.
EventGroupHandle_t eventos;

TaskHandle_t hTriagem;
TaskHandle_t hDespachante;
TaskHandle_t hAlta;
TaskHandle_t hCheat;

int ids[N] = {0, 1, 2};

// -------------------- FUNCOES AUXILIARES --------------------
// Todas as tarefas aguardam o setup concluir a criacao das filas e recursos.
void esperarInicio() {
  while (!sistemaPronto) vTaskDelay(pdMS_TO_TICKS(10));
}

/*
  Envia a mensagem para uma fila de log.

  A tarefa operacional nao espera a Serial terminar. Isso reduz atrasos
  provocados pela impressao e evita que o terminal controle o ritmo do hospital.
*/
void registrar(const char *fmt, ...) {
  if (!qLog) return;
  LogMsg m{};
  va_list ap;
  va_start(ap, fmt);
  vsnprintf(m.texto, sizeof(m.texto), fmt, ap);
  va_end(ap);
  xQueueSend(qLog, &m, 0); // log nunca bloqueia o hospital
}

uint32_t aleatorio(uint32_t min, uint32_t max) {
  return max <= min ? min : min + esp_random() % (max - min + 1);
}

// Decodifica os dois switches: 00=automatico, 01=manual, 10=teste.
Modo lerModo() {
  const bool b0 = digitalRead(SW_MODO_0);
  const bool b1 = digitalRead(SW_MODO_1);
  if (!b1 && !b0) return AUTOMATICO;
  if (!b1 &&  b0) return MANUAL;
  if ( b1 && !b0) return TESTE;
  return INVALIDO;
}

const char *nomeModo(Modo m) {
  const char *nomes[] = {"AUTOMATICO", "MANUAL", "TESTE/CHEAT", "INVALIDO"};
  return nomes[m];
}

/*
  Atualiza o modo e acorda as tarefas que estavam aguardando tempos automaticos.
  Assim, a mudanca de modo e percebida sem esperar o temporizador terminar.
*/
void mudarModo(Modo m) {
  modoAtual = m;
  if (m != TESTE) xEventGroupClearBits(eventos, BIT_CHEAT);
  if (hTriagem) xTaskNotifyGive(hTriagem);
  if (hAlta) xTaskNotifyGive(hAlta);
}

// Cada consultorio utiliza um bit diferente dentro do Event Group.
EventBits_t bitConsultorio(int i) {
  return 1U << (i + 1);
}

void acordarDespachante() {
  if (hDespachante) xTaskNotifyGive(hDespachante);
}

int totalTriagem() {
  int total = 0;
  for (int i = 0; i < 3; ++i)
    total += uxQueueMessagesWaiting(qPrioridade[i]);
  return total;
}

// Calcula a prioridade atual considerando o tempo de espera do paciente.
uint8_t prioridadeEfetiva(const Paciente &p, uint32_t agora) {
  const uint32_t espera = agora - p.chegada;
  if (p.prioridade == 3) {
    if (espera >= AZUL_VERMELHO) return 1;
    if (espera >= AZUL_LARANJA) return 2;
  }
  if (p.prioridade == 2 && espera >= LARANJA_VERMELHO) return 1;
  return p.prioridade;
}

bool enfileirarPaciente(const Paciente &p) {
  return xQueueSend(qPrioridade[p.prioridade - 1], &p, 0) == pdPASS;
}

/*
  Compara apenas o primeiro paciente de cada fila.

  A escolha considera:
  1. prioridade efetiva;
  2. em caso de empate, o maior tempo de espera.
*/
bool retirarPaciente(Paciente &saida) {
  Paciente frente[3]{};
  bool existe[3]{};
  uint8_t efetiva[3]{};
  const uint32_t agora = millis();
  int melhor = -1;

  for (int i = 0; i < 3; ++i) {
    existe[i] = xQueuePeek(qPrioridade[i], &frente[i], 0) == pdPASS;
    if (!existe[i]) continue;
    efetiva[i] = prioridadeEfetiva(frente[i], agora);
    if (melhor < 0 || efetiva[i] < efetiva[melhor] ||
        (efetiva[i] == efetiva[melhor] &&
         frente[i].chegada < frente[melhor].chegada))
      melhor = i;
  }

  if (melhor < 0 || xQueueReceive(qPrioridade[melhor], &saida, 0) != pdPASS)
    return false;

  if (efetiva[melhor] < saida.prioridade)
    registrar("[Triagem] Paciente %d subiu da prioridade %u para %u.\n",
         saida.id, saida.prioridade, efetiva[melhor]);
  return true;
}

// Confirma uma mudanca somente depois que o sinal permanece estavel.
bool debounce(Debounce &d, bool leitura, TickType_t agora) {
  if (leitura != d.bruto) {
    d.bruto = leitura;
    d.mudouEm = agora;
  }
  if (leitura != d.estavel &&
      agora - d.mudouEm >= pdMS_TO_TICKS(DEBOUNCE)) {
    d.estavel = leitura;
    return d.estavel == LOW;
  }
  return false;
}

/*
  Tenta montar um transporte consultorio -> recuperacao.

  Antes de retirar o paciente, o despachante precisa obter:
  - um maqueiro livre;
  - um token do semaforo de leitos.

  O token reserva a vaga antes do transporte e impede duas reservas iguais.
*/
bool despacharRecuperacao() {
  if (!uxQueueMessagesWaiting(qAguardandoRecuperacao)) return false;

  int m;
  if (xQueueReceive(qMaqueirosLivres, &m, 0) != pdPASS) return false;
  if (xSemaphoreTake(leitosLivres, 0) != pdPASS) {
    xQueueSend(qMaqueirosLivres, &m, 0);
    return false;
  }

  PedidoRecuperacao p{};
  if (xQueueReceive(qAguardandoRecuperacao, &p, 0) != pdPASS) {
    xSemaphoreGive(leitosLivres);
    xQueueSend(qMaqueirosLivres, &m, 0);
    return false;
  }

  Trabalho t{PARA_RECUPERACAO, p.paciente, p.consultorio};
  if (xQueueSend(qTrabalhoMaqueiro[m], &t, 0) != pdPASS) {
    xQueueSendToFront(qAguardandoRecuperacao, &p, 0);
    xSemaphoreGive(leitosLivres);
    xQueueSend(qMaqueirosLivres, &m, 0);
    return false;
  }

  registrar("[Despachante] M%d: P%d | C%d -> Recuperacao.\n",
       m + 1, p.paciente.id, p.consultorio + 1);
  return true;
}

/*
  Tenta montar um transporte triagem -> consultorio.

  O despacho so ocorre quando existem simultaneamente:
  - paciente aguardando;
  - maqueiro livre;
  - consultorio livre.
*/
bool despacharConsultorio() {
  if (!totalTriagem()) return false;

  int m, c;
  if (xQueueReceive(qMaqueirosLivres, &m, 0) != pdPASS) return false;
  if (xQueueReceive(qConsultoriosLivres, &c, 0) != pdPASS) {
    xQueueSend(qMaqueirosLivres, &m, 0);
    return false;
  }

  Paciente p{};
  if (!retirarPaciente(p)) {
    xQueueSend(qConsultoriosLivres, &c, 0);
    xQueueSend(qMaqueirosLivres, &m, 0);
    return false;
  }

  Trabalho t{PARA_CONSULTORIO, p, c};
  xEventGroupSetBits(eventos, bitConsultorio(c));

  if (xQueueSend(qTrabalhoMaqueiro[m], &t, 0) != pdPASS) {
    enfileirarPaciente(p);
    xEventGroupClearBits(eventos, bitConsultorio(c));
    xQueueSend(qConsultoriosLivres, &c, 0);
    xQueueSend(qMaqueirosLivres, &m, 0);
    return false;
  }

  registrar("[Despachante] M%d: P%d | Triagem -> C%d.\n", m + 1, p.id, c + 1);
  return true;
}

// -------------------- TAREFAS --------------------
// Unica tarefa que escreve fisicamente no monitor serial.
void tLogger(void *) {
  LogMsg m{};
  while (true)
    if (xQueueReceive(qLog, &m, portMAX_DELAY) == pdPASS)
      Serial.print(m.texto);
}

/*
  Le switches e botoes a cada 20 ms.

  A tarefa aplica debounce, altera os modos e transforma botoes em eventos
  enviados para as tarefas de triagem, alta e cheat.
*/
void tEntradas(void *) {
  esperarInicio();
  const TickType_t agora = xTaskGetTickCount();

  Debounce botoes[3] = {
    {digitalRead(BTN_PACIENTE) == HIGH, digitalRead(BTN_PACIENTE) == HIGH, agora},
    {digitalRead(BTN_ALTA) == HIGH, digitalRead(BTN_ALTA) == HIGH, agora},
    {digitalRead(BTN_CHEAT) == HIGH, digitalRead(BTN_CHEAT) == HIGH, agora}
  };

  Modo estavel = lerModo(), candidato = estavel;
  TickType_t mudouModo = agora, anterior = agora;
  mudarModo(estavel);
  registrar("[Entradas] Modo: %s.\n", nomeModo(estavel));

  while (true) {
    const TickType_t t = xTaskGetTickCount();
    const Modo lido = lerModo();

    if (lido != candidato) {
      candidato = lido;
      mudouModo = t;
    }
    // A mudanca do switch so e aceita depois de permanecer estavel.
    if (candidato != estavel &&
        t - mudouModo >= pdMS_TO_TICKS(DEBOUNCE)) {
      estavel = candidato;
      mudarModo(estavel);
      registrar("[Entradas] Modo: %s.\n", nomeModo(estavel));
    }

    const uint8_t pinos[3] = {BTN_PACIENTE, BTN_ALTA, BTN_CHEAT};
    bool pressionou[3]{};
    for (int i = 0; i < 3; ++i)
      pressionou[i] = debounce(botoes[i], digitalRead(pinos[i]), t);

    // Botao PACIENTE gera uma solicitacao somente nos modos manual e teste.
    if (pressionou[0]) {
      if (modoAtual == MANUAL || modoAtual == TESTE) {
        const Admissao a = NORMAL;
        if (xQueueSend(qPedidoTriagem, &a, 0) == pdPASS)
          xTaskNotifyGive(hTriagem);
      } else registrar("[Entradas] PACIENTE ignorado neste modo.\n");
    }

    // Botao ALTA tenta retirar o paciente mais antigo da recuperacao.
    if (pressionou[1]) {
      if (modoAtual == MANUAL || modoAtual == TESTE) {
        const uint8_t pedido = 1;
        if (xQueueSend(qPedidoAlta, &pedido, 0) == pdPASS)
          xTaskNotifyGive(hAlta);
      } else registrar("[Entradas] ALTA ignorada neste modo.\n");
    }

    // O cheat apenas registra o pedido; a execucao ocorrera na amostragem.
    if (pressionou[2]) {
      if (modoAtual == TESTE) {
        xEventGroupSetBits(eventos, BIT_CHEAT);
        digitalWrite(PULSO_PEDIDO, HIGH);
        vTaskDelay(pdMS_TO_TICKS(30));
        digitalWrite(PULSO_PEDIDO, LOW);
        registrar("[Cheat] Pedido aguardando a proxima amostragem.\n");
      } else registrar("[Cheat] Selecione o modo TESTE.\n");
    }

    vTaskDelayUntil(&anterior, pdMS_TO_TICKS(T_ENTRADAS));
  }
}

/*
  Cria pacientes.

  No modo automatico, usa um intervalo aleatorio.
  Nos modos manual e teste, aguarda pedidos na fila qPedidoTriagem.
*/
void tTriagem(void *) {
  esperarInicio();
  int proximoId = 1;

  while (true) {
    Admissao tipo{};
    bool admitir = xQueueReceive(qPedidoTriagem, &tipo, 0) == pdPASS;

    if (!admitir && modoAtual == AUTOMATICO) {
      const TickType_t espera = pdMS_TO_TICKS(aleatorio(CHEGADA_MIN, CHEGADA_MAX));
      if (ulTaskNotifyTake(pdTRUE, espera) == 0 && modoAtual == AUTOMATICO) {
        tipo = NORMAL;
        admitir = true;
      }
    } else if (!admitir) {
      ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(100));
    }

    if (!admitir) continue;

    Paciente p{};
    p.id = proximoId++;
    p.chegada = millis();

    // O cheat sempre cria um paciente vermelho de prioridade maxima.
    if (tipo == CHEAT) {
      p.cor = "Vermelho-CHEAT";
      p.prioridade = 1;
    } else {
      const uint32_t r = aleatorio(0, 99);
      p.cor = r < 30 ? "Vermelho" : (r < 60 ? "Laranja" : "Azul");
      p.prioridade = r < 30 ? 1 : (r < 60 ? 2 : 3);
    }

    // O paciente e colocado na fila correspondente a sua prioridade original.
    if (enfileirarPaciente(p)) {
      registrar("[Triagem] P%d | %s | Fila=%d.\n", p.id, p.cor, totalTriagem());
      acordarDespachante();
    } else {
      registrar("[Triagem] Fila cheia. P%d recusado.\n", p.id);
    }
  }
}

/*
  Centraliza a distribuicao dos trabalhos.

  A recuperacao e tentada primeiro, pois um consultorio com atendimento
  concluido deve ser liberado antes de receber outro paciente.
*/
void tDespachante(void *) {
  esperarInicio();
  while (true) {
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
    while (despacharRecuperacao() || despacharConsultorio()) {}
  }
}

/*
  Cada maqueiro possui sua propria tarefa e sua propria fila de trabalho.

  Isso permite que ate tres transportes avancem de forma concorrente,
  desde que existam pacientes e destinos disponiveis.
*/
void tMaqueiro(void *arg) {
  const int id = *static_cast<int *>(arg);
  esperarInicio();

  while (true) {
    Trabalho t{};
    xQueueReceive(qTrabalhoMaqueiro[id], &t, portMAX_DELAY);

    if (t.tipo == PARA_CONSULTORIO)
      registrar("[Maqueiro %d] INICIO: P%d -> Consultorio %d.\n",
                id + 1, t.paciente.id, t.consultorio + 1);
    else
      registrar("[Maqueiro %d] INICIO: P%d -> Recuperacao.\n",
                id + 1, t.paciente.id);

    vTaskDelay(pdMS_TO_TICKS(TRANSPORTE));

    if (t.tipo == PARA_CONSULTORIO) {
      // Entrega o paciente diretamente para a tarefa do consultorio escolhido.
      xQueueSend(qPacienteConsultorio[t.consultorio], &t.paciente, portMAX_DELAY);
    } else {
      const int paciente = t.paciente.id;

      // O leito ja foi reservado pelo despachante antes do transporte.
      xQueueSend(qRecuperacao, &paciente, portMAX_DELAY);
      xEventGroupClearBits(eventos, bitConsultorio(t.consultorio));
      xQueueSend(qConsultoriosLivres, &t.consultorio, portMAX_DELAY);
    }

    registrar("[Maqueiro %d] FIM: P%d entregue.\n", id + 1, t.paciente.id);
    // Ao concluir, o maqueiro retorna ao fim da fila de recursos livres.
    xQueueSend(qMaqueirosLivres, &id, portMAX_DELAY);
    acordarDespachante();
  }
}

/*
  Cada consultorio aguarda um paciente em sua fila exclusiva,
  realiza o atendimento e envia um pedido de transporte para a recuperacao.
*/
void tConsultorio(void *arg) {
  const int id = *static_cast<int *>(arg);
  esperarInicio();

  while (true) {
    Paciente p{};
    xQueueReceive(qPacienteConsultorio[id], &p, portMAX_DELAY);
    registrar("[Consultorio %d] Atendimento de P%d iniciado.\n", id + 1, p.id);

    vTaskDelay(pdMS_TO_TICKS(aleatorio(ATEND_MIN, ATEND_MAX)));

    PedidoRecuperacao pedido{p, id};
    xQueueSend(qAguardandoRecuperacao, &pedido, portMAX_DELAY);
    registrar("[Consultorio %d] P%d aguardando maqueiro.\n", id + 1, p.id);
    acordarDespachante();
  }
}

/*
  Libera leitos da recuperacao.

  No automatico, a alta ocorre depois de um tempo aleatorio.
  Nos demais modos, depende do botao ALTA.
*/
void tAlta(void *) {
  esperarInicio();

  while (true) {
    bool darAlta = false;

    if (modoAtual == AUTOMATICO) {
      const TickType_t espera = pdMS_TO_TICKS(aleatorio(ALTA_MIN, ALTA_MAX));
      darAlta = ulTaskNotifyTake(pdTRUE, espera) == 0 && modoAtual == AUTOMATICO;
    } else {
      uint8_t pedido;
      darAlta = xQueueReceive(qPedidoAlta, &pedido, 0) == pdPASS;
      if (!darAlta) ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(100));
    }

    if (!darAlta) continue;

    int paciente;
    if (xQueueReceive(qRecuperacao, &paciente, 0) == pdPASS) {
      // Devolve um token ao semaforo, tornando o leito reservavel novamente.
      xSemaphoreGive(leitosLivres);
      registrar("[Recuperacao] P%d recebeu ALTA. Leitos=%u/%d.\n",
           paciente, uxQueueMessagesWaiting(qRecuperacao), MAX_LEITOS);
      acordarDespachante();
    } else if (modoAtual != AUTOMATICO) {
      registrar("[Recuperacao] Nenhum paciente para alta.\n");
    }
  }
}

/*
  Gera uma base de tempo periodica de 1 segundo.

  O pedido de cheat permanece pendente ate esta tarefa encontrar o bit ativo.
  Isso permite demonstrar a sincronizacao no analisador logico.
*/
void tAmostragem(void *) {
  esperarInicio();
  TickType_t anterior = xTaskGetTickCount();

  while (true) {
    vTaskDelayUntil(&anterior, pdMS_TO_TICKS(T_AMOSTRA));
    digitalWrite(PULSO_AMOSTRA, HIGH);

    if (modoAtual == TESTE &&
        (xEventGroupGetBits(eventos) & BIT_CHEAT)) {
      xEventGroupClearBits(eventos, BIT_CHEAT);
      xTaskNotifyGive(hCheat);
    } else if (modoAtual != TESTE) {
      xEventGroupClearBits(eventos, BIT_CHEAT);
    }

    vTaskDelay(pdMS_TO_TICKS(T_PULSO));
    digitalWrite(PULSO_AMOSTRA, LOW);
  }
}

// Converte a liberacao da amostragem em uma solicitacao de paciente prioritario.
void tCheat(void *) {
  esperarInicio();
  while (true) {
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
    digitalWrite(PULSO_CHEAT, HIGH);

    const Admissao a = CHEAT;
    if (xQueueSend(qPedidoTriagem, &a, 0) == pdPASS) {
      xTaskNotifyGive(hTriagem);
      registrar("[Cheat] EXECUTADO: paciente vermelho solicitado.\n");
    }

    vTaskDelay(pdMS_TO_TICKS(T_PULSO));
    digitalWrite(PULSO_CHEAT, LOW);
  }
}

/*
  Atualiza os LEDs de modo, consultorios e recuperacao.

  Essa tarefa apenas le estados do sistema; ela nao altera o fluxo operacional.
*/
void tSinalizacao(void *) {
  esperarInicio();
  TickType_t anterior = xTaskGetTickCount();

  while (true) {
    const EventBits_t bits = xEventGroupGetBits(eventos);

    for (int i = 0; i < 3; ++i)
      digitalWrite(LED_MODO[i], modoAtual == i);

    for (int i = 0; i < N; ++i)
      digitalWrite(LED_CONSULTORIO[i], (bits & bitConsultorio(i)) != 0);

    digitalWrite(LED_REC_CHEIA, uxSemaphoreGetCount(leitosLivres) == 0);
    vTaskDelayUntil(&anterior, pdMS_TO_TICKS(T_LEDS));
  }
}

// Exibe um resumo simples do sistema sem imprimir informacoes de pilha.
void tMonitor(void *) {
  esperarInicio();
  TickType_t anterior = xTaskGetTickCount();

  while (true) {
    vTaskDelayUntil(&anterior, pdMS_TO_TICKS(T_MONITOR));
    registrar("[Monitor] Triagem=%d | Recuperacao=%u/%d | C livres=%u/%d | M livres=%u/%d.\n",
         totalTriagem(), uxQueueMessagesWaiting(qRecuperacao), MAX_LEITOS,
         uxQueueMessagesWaiting(qConsultoriosLivres), N,
         uxQueueMessagesWaiting(qMaqueirosLivres), N);
  }
}

// -------------------- INICIALIZACAO --------------------
void falhar(const char *msg) {
  Serial.println(msg);
  while (true) delay(1000);
}

// Configura as entradas e saidas utilizadas na montagem do Wokwi.
void configurarPinos() {
  for (uint8_t p : LED_MODO) pinMode(p, OUTPUT);
  for (uint8_t p : LED_CONSULTORIO) pinMode(p, OUTPUT);
  pinMode(LED_REC_CHEIA, OUTPUT);

  const uint8_t saidas[] = {PULSO_AMOSTRA, PULSO_CHEAT, PULSO_PEDIDO};
  for (uint8_t p : saidas) pinMode(p, OUTPUT);

  pinMode(SW_MODO_0, INPUT);
  pinMode(SW_MODO_1, INPUT);
  pinMode(BTN_PACIENTE, INPUT_PULLUP);
  pinMode(BTN_ALTA, INPUT_PULLUP);
  pinMode(BTN_CHEAT, INPUT_PULLUP);
}

void criarTarefa(TaskFunction_t fn, const char *nome, uint32_t pilha,
                 void *arg, UBaseType_t prioridade, TaskHandle_t *handle) {
  if (xTaskCreate(fn, nome, pilha, arg, prioridade, handle) != pdPASS)
    falhar("ERRO: nao foi possivel criar uma tarefa.");
}

/*
  Inicializacao geral.

  Ordem:
  1. configura GPIOs;
  2. cria filas, semaforo e Event Group;
  3. registra recursos inicialmente livres;
  4. cria todas as tarefas;
  5. libera o inicio simultaneo do sistema.
*/
void setup() {
  Serial.begin(115200);
  delay(400);
  configurarPinos();

  qLog = xQueueCreate(48, sizeof(LogMsg));
  for (int i = 0; i < 3; ++i)
    qPrioridade[i] = xQueueCreate(CAP_PRIORIDADE, sizeof(Paciente));

  qPedidoTriagem = xQueueCreate(8, sizeof(Admissao));
  qPedidoAlta = xQueueCreate(8, sizeof(uint8_t));
  qConsultoriosLivres = xQueueCreate(N, sizeof(int));
  qMaqueirosLivres = xQueueCreate(N, sizeof(int));
  qAguardandoRecuperacao = xQueueCreate(N, sizeof(PedidoRecuperacao));
  qRecuperacao = xQueueCreate(MAX_LEITOS, sizeof(int));

  for (int i = 0; i < N; ++i) {
    qPacienteConsultorio[i] = xQueueCreate(1, sizeof(Paciente));
    qTrabalhoMaqueiro[i] = xQueueCreate(1, sizeof(Trabalho));
  }

  // O semaforo inicia com quatro tokens: todos os leitos estao livres.
  leitosLivres = xSemaphoreCreateCounting(MAX_LEITOS, MAX_LEITOS);

  // Event Group para cheat pendente e LEDs dos consultorios.
  eventos = xEventGroupCreate();

  if (!qLog || !qPedidoTriagem || !qPedidoAlta || !qConsultoriosLivres ||
      !qMaqueirosLivres || !qAguardandoRecuperacao || !qRecuperacao ||
      !leitosLivres || !eventos)
    falhar("ERRO ao criar objetos FreeRTOS.");

  // Insere os tres consultorios e os tres maqueiros nas filas de livres.
  for (int i = 0; i < N; ++i) {
    if (!qPrioridade[i] || !qPacienteConsultorio[i] || !qTrabalhoMaqueiro[i])
      falhar("ERRO ao criar filas.");
    xQueueSend(qConsultoriosLivres, &ids[i], 0);
    xQueueSend(qMaqueirosLivres, &ids[i], 0);
  }

  /*
    Prioridades principais:
    - amostragem: 5;
    - entradas e despachante: 4;
    - maqueiros e cheat: 3;
    - triagem, alta e consultorios: 2;
    - logger, sinalizacao e monitor: 1.
  */
  criarTarefa(tLogger, "Logger", 3072, nullptr, 1);
  criarTarefa(tDespachante, "Despachante", 3072, nullptr, 4, &hDespachante);
  criarTarefa(tTriagem, "Triagem", 3072, nullptr, 2, &hTriagem);
  criarTarefa(tAlta, "Alta", 2560, nullptr, 2, &hAlta);
  criarTarefa(tCheat, "Cheat", 2560, nullptr, 3, &hCheat);

  for (int i = 0; i < N; ++i) {
    criarTarefa(tConsultorio, "Consultorio", 2816, &ids[i], 2);
    criarTarefa(tMaqueiro, "Maqueiro", 2816, &ids[i], 3);
  }

  criarTarefa(tAmostragem, "Amostragem", 2560, nullptr, 5);
  criarTarefa(tEntradas, "Entradas", 3072, nullptr, 4);
  criarTarefa(tSinalizacao, "Sinalizacao", 2560, nullptr, 1);
  criarTarefa(tMonitor, "Monitor", 2560, nullptr, 1);

  // Somente agora as tarefas recebem permissao para iniciar suas rotinas.
  modoAtual = lerModo();
  sistemaPronto = true;
  acordarDespachante();

  registrar("\nHOSPITAL FreeRTOS: 3 consultorios, 3 maqueiros, 4 leitos.\n");
  registrar("Arquitetura sem mutex global. Monitoramento de pilha removido.\n");
}

void loop() {
  vTaskDelay(pdMS_TO_TICKS(1000));
}
