# Simulador de Bomba de Jogos💣
 
![Arduino](https://img.shields.io/badge/Arduino-UNO-00979D?style=flat-square&logo=arduino&logoColor=white)
![Language](https://img.shields.io/badge/C++-00599C?style=flat-square&logo=c%2B%2B&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)
![TinkerCAD](https://img.shields.io/badge/TinkerCAD-Disponível-orange?style=flat-square)
 
Simulação física de uma bomba de jogos FPS, estilo **Valorant** e **CS:GO** , desenvolvida com Arduino UNO. O sistema recria a dinâmica completa de armar, desarmar e explodir, com display LCD 16x2, LEDs, buzzer e placar por rodada.
 
A lógica do projeto é baseada em uma **máquina de estados (FSM)**, o que deixa o código bem organizado e fácil de expandir.
 
---
 
## Materiais🛠️
 
<img width="683" height="800" alt="download" src="https://github.com/user-attachments/assets/b7f2fdb3-414f-40fb-bef6-ff9bef6eae6b" />


**1.** Arduino UNO R3
 
**2.** Display LCD 16x2 (sem módulo I2C)
 
**3.** Potenciômetro 10kΩ (contraste do LCD)
 
**4.** 2x Botão de pressão
 
**5.** Buzzer passivo
 
**6.** 3x LED (Verde, Vermelho e Amarelo)
 
**7.** 3x Resistor 220Ω
 
OBS: Também é necessário cabos jumper e uma protoboard.
 
---
 
## Como Jogar🎮
 
| Estado | O que fazer | Tempo |
|---|---|---|
| Aguardando | Segure o botão **ARMAR** para ativar a bomba | 4 segundos |
| Bomba ativa | A bomba explode automaticamente se não for desarmada | 45 segundos |
| Desarmando | Segure o botão **DESARMAR** sem soltar | 10 segundos (normal) |
| Desarme rápido | Se interromper o desarme após 5s, próxima tentativa é mais rápida | 5 segundos |
| Reset de rodada | Segure **REINICIAR** por 3 segundos | — |
| Zerar placar | Segure **REINICIAR** por 10 segundos | — |
 
### Indicadores visuais 💡
 
| LED | Situação |
|---|---|
| 🟡 Amarelo | Armando ou desarmando |
| 🟢 Verde (piscando) | Bomba ativa |
| 🔴 Vermelho | Explosão — Atacantes vencem |
| 🟢 Verde (fixo) | Bomba desarmada — Defensores vencem |
 
---
 
## Montagem⚙️
 
<img width="927" height="524" alt="image" src="https://github.com/user-attachments/assets/433f8b7e-a8d6-4e42-84db-9faa93cecdaf" />

* Conecte o pino RS do LCD no pino 12 do Arduino

* Conecte o pino E (Enable) do LCD no pino 11 do Arduino

* Conecte os pinos D4–D7 do LCD nos pinos 5, 6, 7 e 8 do Arduino (respectivamente)

* Conecte VSS e RW do LCD ao GND, e VDD ao 5V

* Coloque um potenciômetro 10kΩ: pino do meio → VO (contraste), extremos → 5V e GND

* Conecte o backlight do LCD: A (LED+) → 5V, K (LED–) → GND

---
 
## Conexões⚡
 
**LCD 16x2**
 
| Pino do LCD | Pino Arduino |
| ------------ | ------------- |
| RS           | D12           |
| E (Enable)   | D11           |
| D4           | D5            |
| D5           | D6            |
| D6           | D7            |
| D7           | D8            |
| VSS, RW      | GND           |
| VDD          | 5V            |
| VO           | Meio do potenciômetro |
| A (LED+)     | 5V            |
| K (LED–)     | GND           |
 
**Botões**
 
| Componente | Pino Arduino |
| ----------- | ------------ |
| Botão ARMAR | D2 (`INPUT_PULLUP`) |
| Botão DESARMAR / REINICIAR | D3 (`INPUT_PULLUP`) |
 
**LEDs e Buzzer**
 
| Componente | Pino Arduino |
| ----------- | ------------ |
| LED Verde    | D10 (com resistor 220Ω) |
| LED Vermelho | D13 (com resistor 220Ω) |
| LED Amarelo  | D9  (com resistor 220Ω) |
| Buzzer       | D4  |
 
---
 
## Código💻
 
```cpp
#include <LiquidCrystal.h>
 
// Definição dos pinos do LCD
LiquidCrystal lcd(12, 11, 5, 6, 7, 8);
 
// Definição dos pinos dos componentes
const int LED_VERMELHO = 13;
const int LED_AMARELO = 9;
const int LED_VERDE = 10;
const int BUZZER = 4;
const int BOTAO_ARMAR = 2;
const int BOTAO_REINICIAR = 3;
 
// Estados do jogo
enum Estado {
  AGUARDANDO,
  ARMANDO,
  ATIVA,
  DESARMANDO,
  EXPLODIDA,
  DESARMADA
};
 
Estado estadoAtual = AGUARDANDO;
 
// Variáveis de tempo
unsigned long tempoArmarInicio = 0;
unsigned long tempoDesarmarInicio = 0;
unsigned long tempoBombaInicio = 0;
unsigned long tempoPiscarInicio = 0;
unsigned long tempoUltimoBipe = 0;
unsigned long tempoAtualizacaoLCD = 0;
unsigned long tempoMostrarPlacar = 0;
unsigned long tempoBotaoReiniciar = 0;
 
// Constantes de tempo (em ms) — ajuste aqui para customizar
const unsigned long TEMPO_ARMAR           = 4000;
const unsigned long TEMPO_EXPLOSAO        = 45000;
const unsigned long TEMPO_DESARMAR        = 10000;
const unsigned long TEMPO_DESARMAR_RAPIDO = 5000;
const unsigned long TEMPO_REINICIAR_CURTO = 3000;
const unsigned long TEMPO_REINICIAR_LONGO = 10000;
const unsigned long INTERVALO_LCD         = 500;
const unsigned long TEMPO_MOSTRAR_PLACAR  = 5000;
 
// Frequências do buzzer
const int FREQ_DESARMANDO = 1000;
const int FREQ_EXPLOSAO   = 500;
const int FREQ_BIPE       = 1500;
const int FREQ_ARMAMENTO  = 2000;
 
// Variáveis de controle
bool ledVerdeEstado = false;
const unsigned long INTERVALO_BIPE = 500;
 
bool desarmeRapidoAtivo = false;
bool bipeArmamentoExecutado = false;
 
int placarDefensores = 0;
int placarAtacantes = 0;
bool mostrandoPlacar = false;
 
bool botaoReiniciarPressionado = false;
bool mensagemEncerramentoMostrada = false;
 
void setup() {
  pinMode(LED_VERMELHO, OUTPUT);
  pinMode(LED_AMARELO, OUTPUT);
  pinMode(LED_VERDE, OUTPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(BOTAO_ARMAR, INPUT_PULLUP);
  pinMode(BOTAO_REINICIAR, INPUT_PULLUP);
 
  lcd.begin(16, 2);
  lcd.print("Sistema Bomba");
  lcd.setCursor(0, 1);
  lcd.print("Valorant - Ready");
 
  digitalWrite(LED_VERMELHO, LOW);
  digitalWrite(LED_AMARELO, LOW);
  digitalWrite(LED_VERDE, LOW);
 
  Serial.begin(9600);
  delay(2000);
  limparLCD();
}
 
void loop() {
  unsigned long tempoAtual = millis();
 
  verificarBotaoReiniciar(tempoAtual);
 
  if (mostrandoPlacar) {
    if (tempoAtual - tempoMostrarPlacar >= TEMPO_MOSTRAR_PLACAR) {
      mostrandoPlacar = false;
      estadoAtual = AGUARDANDO;
      limparLCD();
    } else {
      mostrarPlacar();
      delay(50);
      return;
    }
  }
 
  if (tempoAtual - tempoAtualizacaoLCD >= INTERVALO_LCD) {
    atualizarLCD();
    tempoAtualizacaoLCD = tempoAtual;
  }
 
  switch (estadoAtual) {
 
    case AGUARDANDO:
      digitalWrite(LED_VERMELHO, LOW);
      digitalWrite(LED_AMARELO, LOW);
      digitalWrite(LED_VERDE, LOW);
      noTone(BUZZER);
      bipeArmamentoExecutado = false;
      desarmeRapidoAtivo = false;
 
      if (digitalRead(BOTAO_ARMAR) == LOW) {
        tempoArmarInicio = tempoAtual;
        estadoAtual = ARMANDO;
        digitalWrite(LED_AMARELO, HIGH);
        tone(BUZZER, FREQ_ARMAMENTO, 300);
        bipeArmamentoExecutado = true;
      }
      break;
 
    case ARMANDO:
      if (digitalRead(BOTAO_ARMAR) == HIGH) {
        estadoAtual = AGUARDANDO;
        digitalWrite(LED_AMARELO, LOW);
        limparLCD();
      } else if (tempoAtual - tempoArmarInicio >= TEMPO_ARMAR) {
        estadoAtual = ATIVA;
        tempoBombaInicio = tempoAtual;
        tempoPiscarInicio = tempoAtual;
        ledVerdeEstado = true;
        digitalWrite(LED_VERDE, ledVerdeEstado);
        digitalWrite(LED_AMARELO, LOW);
        tone(BUZZER, FREQ_BIPE, 200);
      }
      break;
 
    case ATIVA:
      if (tempoAtual - tempoPiscarInicio >= 500) {
        ledVerdeEstado = !ledVerdeEstado;
        digitalWrite(LED_VERDE, ledVerdeEstado);
        tempoPiscarInicio = tempoAtual;
      }
 
      if (tempoAtual - tempoUltimoBipe >= INTERVALO_BIPE) {
        tone(BUZZER, FREQ_BIPE, 100);
        tempoUltimoBipe = tempoAtual;
      }
 
      if (tempoAtual - tempoBombaInicio >= TEMPO_EXPLOSAO) {
        estadoAtual = EXPLODIDA;
      }
 
      if (digitalRead(BOTAO_REINICIAR) == LOW) {
        estadoAtual = DESARMANDO;
        tempoDesarmarInicio = tempoAtual;
        digitalWrite(LED_AMARELO, HIGH);
        digitalWrite(LED_VERDE, LOW);
      }
      break;
 
    case DESARMANDO:
      digitalWrite(LED_AMARELO, HIGH);
      digitalWrite(LED_VERDE, LOW);
 
      if (tempoAtual - tempoBombaInicio >= TEMPO_EXPLOSAO) {
        estadoAtual = EXPLODIDA;
        break;
      }
 
      unsigned long tempoDesarmeNecessario;
      tempoDesarmeNecessario = desarmeRapidoAtivo ? TEMPO_DESARMAR_RAPIDO : TEMPO_DESARMAR;
 
      if (desarmeRapidoAtivo) {
        if (tempoAtual - tempoUltimoBipe >= 600) {
          tone(BUZZER, FREQ_DESARMANDO, 300);
          tempoUltimoBipe = tempoAtual;
        }
      } else {
        if (tempoAtual - tempoUltimoBipe >= 1000) {
          tone(BUZZER, FREQ_BIPE, 200);
          tempoUltimoBipe = tempoAtual;
        }
      }
 
      if (digitalRead(BOTAO_REINICIAR) == HIGH) {
        unsigned long tempoSegurou = tempoAtual - tempoDesarmarInicio;
        if (!desarmeRapidoAtivo && tempoSegurou >= 5000) desarmeRapidoAtivo = true;
 
        estadoAtual = ATIVA;
        digitalWrite(LED_AMARELO, LOW);
        tempoPiscarInicio = tempoAtual;
        noTone(BUZZER);
 
      } else if (tempoAtual - tempoDesarmarInicio >= tempoDesarmeNecessario) {
        placarDefensores++;
        estadoAtual = DESARMADA;
        digitalWrite(LED_AMARELO, LOW);
        digitalWrite(LED_VERDE, HIGH);
        noTone(BUZZER);
        tone(BUZZER, FREQ_BIPE, 1000);
        desarmeRapidoAtivo = false;
        mostrandoPlacar = true;
        tempoMostrarPlacar = millis();
      }
      break;
 
    case EXPLODIDA:
      placarAtacantes++;
      digitalWrite(LED_VERMELHO, HIGH);
      digitalWrite(LED_VERDE, LOW);
      digitalWrite(LED_AMARELO, LOW);
      tone(BUZZER, FREQ_EXPLOSAO);
      desarmeRapidoAtivo = false;
      mostrandoPlacar = true;
      tempoMostrarPlacar = millis();
      break;
 
    case DESARMADA:
      digitalWrite(LED_VERDE, HIGH);
      mostrandoPlacar = true;
      tempoMostrarPlacar = millis();
      break;
  }
 
  delay(50);
}
 
void verificarBotaoReiniciar(unsigned long tempoAtual) {
  if (digitalRead(BOTAO_REINICIAR) == LOW) {
    if (!botaoReiniciarPressionado) {
      botaoReiniciarPressionado = true;
      tempoBotaoReiniciar = tempoAtual;
      mensagemEncerramentoMostrada = false;
    } else {
      unsigned long tempoPressionado = tempoAtual - tempoBotaoReiniciar;
 
      if (tempoPressionado >= 3000 && !mensagemEncerramentoMostrada) {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Encerrando");
        lcd.setCursor(0, 1);
        lcd.print("partida...");
        mensagemEncerramentoMostrada = true;
      }
 
      if (tempoPressionado >= TEMPO_REINICIAR_LONGO) {
        placarDefensores = 0;
        placarAtacantes = 0;
        estadoAtual = AGUARDANDO;
        mostrandoPlacar = false;
        digitalWrite(LED_VERMELHO, LOW);
        digitalWrite(LED_AMARELO, LOW);
        digitalWrite(LED_VERDE, LOW);
        noTone(BUZZER);
        limparLCD();
        botaoReiniciarPressionado = false;
 
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Placar Zerado!");
        lcd.setCursor(0, 1);
        lcd.print("Partida Nova");
        delay(2000);
        limparLCD();
        return;
      }
    }
  } else {
    if (botaoReiniciarPressionado) {
      unsigned long tempoPressionado = tempoAtual - tempoBotaoReiniciar;
      if (tempoPressionado >= TEMPO_REINICIAR_CURTO && tempoPressionado < TEMPO_REINICIAR_LONGO) {
        estadoAtual = AGUARDANDO;
        mostrandoPlacar = false;
        digitalWrite(LED_VERMELHO, LOW);
        digitalWrite(LED_AMARELO, LOW);
        digitalWrite(LED_VERDE, LOW);
        noTone(BUZZER);
        limparLCD();
      }
      botaoReiniciarPressionado = false;
    }
  }
}
 
void atualizarLCD() {
  lcd.clear();
 
  switch (estadoAtual) {
    case AGUARDANDO:
      lcd.setCursor(0, 0); lcd.print("Aguardando...");
      lcd.setCursor(0, 1); lcd.print("Pressione ARMAR");
      break;
 
    case ARMANDO:
      lcd.setCursor(0, 0); lcd.print("ARMANDO BOMBA");
      lcd.setCursor(0, 1); lcd.print("Segure: ");
      lcd.print(4 - (millis() - tempoArmarInicio) / 1000);
      lcd.print("s");
      break;
 
    case ATIVA:
      lcd.setCursor(0, 0); lcd.print("BOMBA ATIVA!");
      lcd.setCursor(0, 1); lcd.print("Explode em: ");
      lcd.print((TEMPO_EXPLOSAO - (millis() - tempoBombaInicio)) / 1000);
      lcd.print("s");
      break;
 
    case DESARMANDO:
      lcd.setCursor(0, 0);
      lcd.print(desarmeRapidoAtivo ? "DESARME [RAPIDO]" : "DESARME [NORMAL]");
      lcd.setCursor(0, 1);
      {
        unsigned long tempoRestanteDesarme = (desarmeRapidoAtivo ? TEMPO_DESARMAR_RAPIDO : TEMPO_DESARMAR)
                                            - (millis() - tempoDesarmarInicio);
        lcd.print("Faltam: ");
        lcd.print(tempoRestanteDesarme / 1000);
        lcd.print("s");
      }
      break;
 
    case EXPLODIDA:
      lcd.setCursor(0, 0); lcd.print("!!! EXPLODIU !!!");
      lcd.setCursor(0, 1); lcd.print("  ATTACK WIN");
      break;
 
    case DESARMADA:
      lcd.setCursor(0, 0); lcd.print("BOMBA DESARMADA!");
      lcd.setCursor(0, 1); lcd.print("  DEFENSE WIN !");
      break;
  }
}
 
void mostrarPlacar() {
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print("DEFENSE X ATTACK");
  lcd.setCursor(0, 1);
  lcd.print("    ");
  lcd.print(placarDefensores);
  lcd.print(" X ");
  lcd.print(placarAtacantes);
}
 
void limparLCD() {
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print("Aguardando...");
  lcd.setCursor(0, 1); lcd.print("Pressione ARMAR");
}
```
 
---
 
## Projeto no TinkerCAD🖥️
 
Teste o circuito no navegador antes de montar:
 
🔗 https://www.tinkercad.com/things/c0tn1UnBdFd-simulador-de-bomba
 
---
 
## Resultado Final❗
 
https://github.com/user-attachments/assets/f94ba12f-550b-4eb4-abf0-e6f4b362c188
 
