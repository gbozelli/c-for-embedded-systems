# RFC: Refatoração do Seguidor de Linha e Publicação de Bibliotecas

**Autor:** Gabriel Bozelli (mentor) , equipe Ana Laura, Bianca e Fabieli
**Status:** Proposta / Em discussão
**Repositórios envolvidos:**
- Curso: `gbozelli/c-for-embedded-systems`
- Código antigo: `seguidor-de-linha/seguidor-principal.ino`
- Lib em andamento: `gbozelli/mux-sensors-lib`

---

## 1. Contexto

Depois de implementar o mux-sensors-lib, imagino que o natural seria implementar o resto. Sim, mux-sensors-lib não está 100% pronto, porque, apesar de termos validado o funcionamento do código, não colocamos testes unitários e validamos só com uma placa de sensores (temos mais sensores na salinha, pode ser uma tarefa pra próxima semana). Nem testamos o código modularizado (sim, é só colocar a pasta no Arduino IDE e rodar via um arquivo .ino, mas acho que seria uma experiência legal).

Vale aproveitar essa validação com a placa de sensores completa pra também conferir se o settling time entre canais (que já está documentado nos slides) se sustenta com os 8 canais reais, em teoria não muda, mas eu fico meio encafifado com esse tipo de coisa. 

## 2. Problemas concretos no código antigo

- Tudo em um `.ino` só: setup, leitura de sensores, parsing de Bluetooth, PID e controle
  de motor misturados em ~170 linhas, sem separação de responsabilidades.
- Estado do robô espalhado em variáveis globais soltas (`start`, `linhaPerdida`,
  `stop_count`, `instante_inicial`...) em vez de uma struct/máquina de estados.
- Números mágicos sem constantes nomeadas (`error = position - 3000`, `sensorValues[i] > 150`,
  `timer_parada = 60000`).
- Nenhum teste automatizado; validação é só na pista (precisamos ver uns testes unitários em C).
- PID, parsing de comandos BT e controle de motor não são reutilizáveis fora deste sketch.

Esses pontos dão o escopo natural para as próximas bibliotecas.

## 3. Decisão de arquitetura: multirepo, não monorepo

Recomendo **não** unificar tudo em um monorepo, por um motivo prático: o **Arduino
Library Manager exige um repositório GitHub por biblioteca**, com `library.properties`
na raiz. Um monorepo com várias libs em subpastas não é indexável pelo Library Manager
sem gambiarras (submódulos/mirrors por lib, que dão mais trabalho do que resolvem).

Proposta de organização:

| Repositório | Conteúdo | Tipo |
|---|---|---|
| `c-for-embedded-systems` | Curso, slides, material didático | Curso (fica como está) |
| `mux-sensors-lib` | Leitura multiplexada de sensores | Biblioteca Arduino publicável |
| `motor-control-lib` *(novo)* | PWM (`ledcAttach`), controle diferencial, PID | Biblioteca Arduino publicável |
| `line-follower-fsm` *(novo)* | Máquina de estados do robô (idle/rodando/parado) | Biblioteca Arduino publicável |
| `seguidor-de-linha-firmware` *(novo)* | `.ino` principal, que só orquestra as libs acima | Aplicação (não é lib) |

O `.ino` final fica pequeno: instancia as libs, faz `setup()`/`loop()` chamando as
funções de entrada/processamento/saída de cada uma. Isso também facilita a
capacitação de novos membros (item que já está nos planos futuros da apresentação):
cada pessoa pode trabalhar numa lib isolada sem precisar entender o sistema inteiro.

## 4. Bibliotecas propostas

### 4.1 `mux-sensors-lib` (já iniciada)
- Já tem a estrutura certa (`Multiplexer`/`Sensor` como structs, funções `_init`/`_begin`/`_read`).
- Próximos passos: fechar os 3 PRs abertos, adicionar testes automatizados de verdade
  (hoje `test.ino` parece ser só um sketch de diagnóstico manual, não testes que rodam
  sozinhos), e documentar o *settling time* entre troca de canal (mencionado nos slides).

### 4.2 `motor-control-lib` (a criar)
Extrai o que hoje está solto no `.ino`:
- `struct MotorConfig` com pinos de PWM e da ponte H.
- Função de inicialização (`ledcAttach` para os dois canais).
- Processamento dos sensores: lembro de a gente ter tido algum problema tratando os sensores
como um valor digital, por isso estamos usando de forma analógica. Porém, isso vai introduzir
uma implementação de processamento. Juro que já vi algo assim em PDS.
  - Se for isso mesmo (o problema de tratar como digital era provavelmente ruído ou limiar mal   configurado), duas saídas comuns de PDS que resolvem isso sem
    complicar muito: um filtro passa-baixa simples (tipo
    `valor_filtrado = alpha * novo + (1 - alpha) * anterior`) pra suavizar leitura ruidosa, ou
    binarização com histerese (sim, do Afonso), comdois limiares em vez de um só, pra não ficar oscilando. Vale a gente decidir qual das duas resolve o problema que vocês tiveram antes
    de implementar, porque atacam ruídos diferentes.
- PID isolado (`struct PIDController` com `kp`, `kd`, `previousError` — hoje é uma
  variável `static` dentro de uma função, o que impede ter mais de um controlador
  e dificulta testar o PID isoladamente).
- Função `pararMotores()` e função que converte `error` do sensor em velocidade de
  cada roda, com `constrain`.

### 4.3 `line-follower-fsm` (a criar)
- Estados sugeridos: `IDLE`, `CALIBRATING`, `RUNNING`, `LINE_LOST`, `STOPPED`.
- Substitui o `bool start` + `bool linhaPerdida` + `stop_count` por transições
  explícitas, e substitui o uso de `millis()` cru por uma API de timers não bloqueantes
  (o que os slides já chamam de "substituir o `delay()`").
- Recebe como entrada os dados de `mux-sensors-lib` e comandos do parser de Bluetooth,
  e decide o estado; o `motor-control-lib` só executa o que a FSM manda.

### 4.4 Parsing de comandos Bluetooth
Preciso pensar se vamos ter que mexer nisso. O Bluetooth inicialmente era uma forma de alterarmos
os parâmetros do seguidor sem ter que fazer upload do código, mas também era bem chato configurar isso. Talvez pesquisar se tem formas mais eficientes de fazer isso.
 
Isso também esbarra direto no ponto de compatibilidade lá embaixo, visto que o código antigo usa
`BluetoothSerial`, que é nativo do ESP32, mas o Arduino Micro **não tem Bluetooth embutido**.
Então, migrando pra Micro, ou a gente adiciona um módulo externo (tipo HC-05/HC-06) só pra manter
essa funcionalidade, ou aproveita a troca pra repensar mesmo: dá pra ajustar `KP`/`KD`/`VEL_MAX`
por Serial via USB só durante os testes de bancada, ou até persistir os
parâmetros calibrados em EEPROM (memória) depois de achados, sem precisar reconfigurar toda vez que liga.
Vale decidir isso antes de desenhar a `line-follower-fsm`, já que ela recebe os comandos.

## 5. Convenções de código e fluxo de trabalho

Aproveitando o que já está nos slides (branches, PRs, review):
- Uma branch por feature/lib, nome sugerido: `lib/<nome>-<o-que-muda>` (ex: `lib/motor-control-pid`).
- PR obrigatório com pelo menos 1 review antes do merge (já é a prática da equipe).
- Cada lib nova segue a mesma estrutura de `mux-sensors-lib`: `src/<modulo>/<modulo>.h`
  e `.c`/`.cpp`, `docs/`, `library.properties`, `README.md` com exemplo de uso.
- Nome de função consistente entre libs: `<modulo>_init`, `<modulo>_begin`,
  `<modulo>_read`/`_update`, já que é o padrão que `mux-sensors-lib` estabeleceu.

## 6. Plano de fases

1. **Fechar `mux-sensors-lib`**: mergear PRs pendentes, testes, documentação , pré-requisito
   para o resto, já que as outras libs vão consumi-la.
2. **Extrair `motor-control-lib`**: PID + PWM, com testes de unidade do cálculo do PID
   (não depende de hardware, dá pra testar isolado). Esse aqui vamos ter um pouco mais de trabalho.
3. **Extrair `line-follower-fsm`**: estados e transições, ainda sem hardware real. A abstração dele como arquitetura é mais complexa, porém a implementação vai ser super simples
4. **Montar `seguidor-de-linha-firmware`**: novo `.ino` orquestrando as 3 libs.
5. **Testar na pista** e comparar desempenho com o código antigo.
6. **Publicar as libs** no Arduino Library Manager (registro via `library.properties`
   + tag de release no GitHub). Essa etapa pode ocorrer antes tambémm (passo 1-2) porque imagino que demora
   — só vale lembrar que, se publicar cedo, a API pública da lib (nomes de struct/função) já fica
   meio congelada, porque quem instalar via Library Manager vai puxar por versão/tag. Então dá pra
   adiantar o registro, mas o ideal é já ter passado pela validação com a placa completa de sensores
   antes de tagar a primeira versão "estável".
7. **Gravar material didático** para novos membros, usando as libs já publicadas como
   estudo de caso (Capacitação de outros membros como plano futuro).

## 7. Riscos e perguntas em aberto

- **Ambiente de teste**: as libs de PID/FSM dão pra testar sem hardware; a de mux/motor
  não. Eu acho que, antes de tudo, sempre devemos validar o código escrito com componente
  + arduino.
- **Compatibilidade**: o código antigo usa ESP32, mas agora vamos usar Arduino Micro. Uma refatoração vai ser feita também para cobrir isso, mas imagino que não será tanto trabalho.
- **Quem publica**: publicar no Library Manager exige conta e processo de aprovação da
  Arduino, bom já indicar quem vai cuidar disso na equipe.

## 8. Próxima ação sugerida

Comigo, o passo mais imediato é decidir a interface pública de `motor-control-lib`
(que structs e funções ela expõe) antes de começar a escrever, já que é a próxima lib
na fila. Posso ajudar a esboçar esse header quando vocês quiserem, mas, de cabeça, consigo pensar em algo mais ou menos assim:

```c
typedef struct {
  int pin_pwm_esq;
  int pin_pwm_dir;
  int freq_hz;      
  int resolucao_bits;
  int vel_max;
} MotorConfig;
 
typedef struct {
  float kp;
  float kd;
  float previous_error;
} PIDController;
```
