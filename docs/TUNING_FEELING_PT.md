# Tutorial — Ajustando o feeling do volante

<sub>🇬🇧 [Read in English](TUNING_FEELING.md)</sub>

Guia prático pra tunar o "feel" do volante usando as ferramentas integradas — do Performance Test, passando pelos filtros do motor (CF/Damper/Friction/Inertia) e terminando nos efeitos FFB que o jogo manda (Constant Force, Spring, Damper, Friction).

---

## Visão geral

Tunar feeling de volante é resolver **3 problemas em camadas**, do mais baixo nível ao mais alto:

| Camada | Problema | Ferramentas |
|---|---|---|
| **1. Hardware** | Quanto o motor entrega? | Performance Test |
| **2. Controle interno** | Filtros bem cortados? | Coastdown, FFB Filters, Frequency Sweep |
| **3. Efeitos do jogo** | Sensações certas chegando na mão? | Constant Force / Spring / Damper / Friction gains |

Tunar de cima pra baixo (começar pelos efeitos do jogo) é frustrante porque você fica brigando com problemas das camadas inferiores. Comece pela base: caracterize o hardware, estabilize o controle, configure os filtros — e quando você chegar nos efeitos, o ajuste é só "menos ou mais", não "por que está vibrando?".

> ⚠️ **Realidade prática dos jogos de corrida**
>
> A esmagadora maioria dos sims de corrida (iRacing, ACC, AMS2, BeamNG, rFactor 2, Le Mans Ultimate) **só envia Constant Force** pelo USB FFB. Não usam Spring, Damper, Friction nem Inertia — essas forças são simuladas dentro da CF.
>
> Implicações:
> - **Ajustar o filtro CF importa muito.** É o que pega 100% do sinal do jogo.
> - **Ajustar filtros de Damper/Friction/Inertia só faz sentido** se você habilitar esses efeitos **manualmente** na aba FFB Wheel (eles entram em paralelo com a CF do jogo).
>
> **Como saber o que o seu jogo está enviando:** aba **FFB Live** → painel **`Efeitos ativos`**. Mostra até 3 effect slots ativos com `type` (Constant / Spring / Damper / Periodic / Friction / Inertia), `state`, `magnitude` e `gain`. Em sim de corrida moderno você vai ver **apenas 1 slot ativo, com type = Constant Force** — confirmando que o jogo só envia CF.

---

## Pré-requisitos

Antes de começar este tutorial:

- ✅ Quick Start completo (motor calibrado, encoder OK, FFB configurado)
---

## Passo 1 — Performance Test (caracterizar o motor)

**Por que primeiro:** todos os outros passos usam os números que esse teste mede.

### O que faz

Aplica CF até saturação, deixa o motor acelerar livremente até o batente, captura tudo via HID a 1 kHz. Calcula:

| Métrica | O que significa | Pra que serve depois |
|---|---|---|
| `peakRPM` | RPM pico atingida | Limite real da sua chain mecânica |
| `peakAccel` | Aceleração máxima (RPM/s) | Sanity check de J |
| `J_kgm2` | Inércia equivalente | Entra em **toda** a math dos filtros |
| `breakawayPct` | % CF pra wheel sair do lugar | Fricção estática (stiction) |
| `iqMax` | Iq máximo medido | Confirma se motor saturou |
| `iqSat%` | % do tempo em saturação | Headroom de torque |

### Como rodar

1. Aba **Performance Test**
2. Clica `▶ Iniciar` (HID conecta sozinho na primeira vez via popup do browser)
3. Confirma o aviso de segurança
4. Motor centraliza → empurra até o batente → libera → rampeia → mede
5. Resultado aparece em ~10 segundos

### Como interpretar

- **J alto (> 0.005 kg·m²)** = wheel pesado/grande → use bandwidth de filtro mais conservador (menor exemplo 30 a 50Hz)
- **J baixo (< 0.001 kg·m²)** = wheel leve/pequeno → bandwidth pode ser mais agressivo (exemplo acima de 80Hz)
- **iqSat% > 30%** = motor saturando muito → considere current_lim maior, fxratio menor, ou motor mais potente
- **breakawayPct > 15%** = stiction alta → encoder com folga ou mancal duro; afeta capacidade do anti-cogging (futuro) e do feeling em movimentos lentos

### Quando refazer

- Mudou mecânica (volante, shaft, mancais)
- Trocou motor
- Mudou `current_lim` ou `maxtorque`

Resultado é salvo automaticamente no `motorCal` (localStorage).

---

## Passo 2 — Coastdown (medir atrito viscoso)

**Por que importa:** o atrito viscoso `b` (Nm·s/rad) é o que **freia naturalmente** o volante quando você solta. Junto com `J`, define o pólo mecânico do sistema:

```
f_c_mec = b / (2π · J)
```

Esse pólo é a base pra escolher cutoff dos filtros (regra prática: filtro acima de **10 × f_c_mec**).

### Como rodar

1. Aba **Performance Test**, role até a seção `🌀 Coastdown test`
2. Clica `▶ Iniciar`
3. Motor faz spin-up até velocidade alvo, mantém, depois **libera** — você vê a velocidade decair naturalmente
4. Resultado: `b_visc` (Nm·s/rad), `tau` (constante de tempo)

### Como interpretar

- **b alto** → wheel "freia sozinho" rápido → tem amortecimento natural alto
- **b baixo** → wheel coast forever → precisa de damper artificial no FFB chain
- **τ longo (> 5s)** → wheel "fluida", boa pra alto realism
- **τ curto (< 1s)** → wheel "pesada/atritada", responde mas perde inércia rápido

---

## Passo 3 — Filtros do FFB chain

Aqui é onde a maior parte do feeling é definida. O FFB chain processa o torque do jogo através de **4 filtros**:

```
Game registra Constant Force  ─→ [filtro CF] ──────┐
Game registra Damper          ─→ [filtro Damper] ──┤
Game registra Friction        ─→ [filtro Friction] ┼─→ SOMA → × gain × fxratio → Motor
Game registra Inertia         ─→ [filtro Inertia] ─┤
Game registra Spring          ─→ (sem biquad próprio)─┘
```

Cada filtro tem `Freq` (cutoff em Hz) e `Q` (fator de qualidade — quão "ressonante" o filtro é).

### Princípios

| Filtro | O que faz | Importa pra... |
|---|---|---|
| **CF (Constant Force)** ⭐ | Passa-baixa principal — limita ruído de alta freq do jogo | **TODO sim de corrida** — único filtro que sempre atua, porque CF é o único efeito que o jogo manda |
| **Damper** | Filtra o Damper effect | Só **efeitos Damper adicionados manualmente** na aba FFB Wheel ou jogos antigos que usam Damper effect |
| **Friction** | Filtra o Friction effect | Só **efeitos Friction adicionados manualmente** ou jogos que usam |
| **Inertia** | Filtra o Inertia effect | Raríssimo — quase nenhum jogo usa |

> Como sim de corrida moderno só manda CF, **na prática CF é o único filtro que importa pra ajustar**. Os outros filtros só viram relevantes se você habilitar Damper/Friction/Inertia manualmente na aba FFB Wheel (efeitos do firmware, em paralelo ao CF do jogo).

### Onde mexer

Aba **FFB Filters** mostra os 4 cards com sliders de Freq e Q. Cada um tem o gráfico de resposta em frequência atualizando ao vivo.

### Como escolher CF (o mais importante)

A aba Performance Test, no card **💡 Análise e sugestões**, usa J e b pra computar e sugerir um valor:

```
Pólo mecânico  f_c_mec = b / (2π · J)
Pólo elétrico  f_LR    = R_phase / (2π · L_phase)
CF sugerido    entre 10 × f_c_mec  e  0.8 × f_LR     (clamp 20–100 Hz)
```

A faixa **10×f_c_mec até 0.8×f_LR** é onde o motor responde bem (acima do pólo mecânico) mas o controle de corrente ainda dá conta (abaixo do pólo elétrico).

Caso típico de wheel sim racing:
- `J ≈ 0.002 kg·m²`, `b ≈ 0.0001 Nm·s/rad` → `f_c_mec ≈ 0.008 Hz`
- `R = 0.1 Ω`, `L = 0.0001 H` → `f_LR ≈ 159 Hz`
- Range: `~0.08 Hz a 127 Hz` → CF típico **~60 Hz** (clamp em 100)

### Como escolher Damper / Friction / Inertia

Em sim de corrida moderno: **deixa nos defaults**. Esses filtros só processam efeitos que **não estão sendo enviados**. Mexer neles é mexer em algo que não acontece.

Cenários em que ajustar **faz sentido**:

1. **Você habilita Damper manualmente na aba FFB Wheel** pra dar "peso" extra que o jogo não dá → aí o filtro Damper passa a importar. Recomendação: cutoff 30–50 Hz, Q 0.7.
2. **Você habilita Friction manualmente** pra simular volante "pesado" sem velocidade → filtro Friction importa. Mesmo range.
3. **Joga sim antigo ou flight sim** que usa Damper/Friction effects separados → todos os filtros importam.

A maior parte do feeling vem do **CF + gain do efeito CF no jogo**. Os outros são acessórios.

### Quick reference Q

| Q | Comportamento | Quando usar |
|---|---|---|
| 0.5 | Subdamped — corta sem overshoot | Conservador, recomendado |
| 0.707 | Butterworth — flat até cutoff | Default ideal |
| 1.0 | Underdamped — pequeno bump em cutoff | Realça frequências perto do cutoff |
| 2.0+ | Resonante — bump grande | Cuidado, pode oscilar |

---

## Passo 4 — Validar com Frequency Sweep

Configurou os filtros? Hora de testar se a chain real bate com a teoria.

### Como rodar

1. Aba **Performance Test**, role até `📊 Frequency Sweep test`
2. Escolhe modo:
   - **Full sweep** — com efeitos ativos (mede a chain inteira como vai funcionar no jogo)
   - **Natural response only** — zera os efeitos (mede só motor + mecânica)
3. Clica `▶ Rodar varredura`
4. Motor faz sine em ~8 frequências (0.5, 1, 2, 5, 10, 20, 50, 100 Hz tipicamente)
5. Recenter automático entre cada — leva ~75s no modo natural, ~75s no full

### Interpretar o resultado

Vai pra aba **FFB Filters**, gráfico **Frequency Response**:

- 🟢 **Curva verde sólida** = resposta TEÓRICA dos filtros configurados
- 🟢 **Pontos verdes** = MEDIDOS pelo sweep

**Se batem** → modelo validado, filtros funcionando como esperado.

**Se divergem:**
- **Pontos abaixo da curva** = atenuação real maior que teórica → tem perda extra que o modelo não pega (current loop, saturação)
- **Pontos acima da curva** = ressonância não modelada (típico em wheels com cog forte ou mancal solto)
- **Pontos divergindo em alta freq (> 50 Hz)** = current bandwidth limitando — considere aumentar `current_control_bandwidth`

### Bonus: torque medido

O Sweep usa o **torque medido** real (Iq × Kt do motor), não o commanded. Isso significa que a curva Bode reflete o que o motor **entrega**, não o que o controlador pediu — bem mais fiel à realidade.

---

## Passo 5 — Live FFT durante gameplay

Filtros validados em testes sintéticos? Hora de ver como eles se comportam com o que o **jogo real** está mandando.

### Como rodar

1. Aba **Overlay**, marca o checkbox **`Mostrar gráfico de espectro (FFT τ_cmd vs Iq @ 1 kHz)`**
2. Abre o PiP overlay
3. Joga normalmente OU mova o volante manualmente
4. Olha o painel inferior do PiP

### Modos

- **Modo "τ_cmd vs Iq"** (sobreposto): vê o que o jogo está pedindo (azul) vs o que o motor está entregando (laranja). Se laranja cai cedo demais comparado a azul → seus filtros estão cortando energia que o jogo queria entregar.
- **Modo "Iq / τ_cmd"** (Bode da chain): mostra a função de transferência da chain extraída do gameplay. Compare visualmente com a curva teórica da aba FFB Filters.

### Anotações no chart

3 linhas verticais te ajudam a localizar onde os pólos estão:
- ⚪ **f_c_mec** (pólo mecânico, do Coastdown + PT) — abaixo daqui você sente massa
- ⚪ **f_LR** (pólo elétrico, R/L do motor) — acima daqui controle vira ruído
- 🟡 **CF** (cutoff atual do filtro Constant Force) — aqui é onde você está cortando

### O que ajustar com base no FFT

| Observação | Diagnóstico | Ação |
|---|---|---|
| Energia do jogo até 100 Hz, CF cortando em 30 Hz | Filtro CF muito agressivo | Sobe CF pra 60–80 Hz |
| Energia do jogo só até 20 Hz, CF em 100 Hz | Filtro CF não precisa ser tão alto | Reduz CF — menos ruído de encoder no motor |
| Pico isolado em ~150 Hz no Iq sem nada em τ_cmd | Ruído de cogging ou encoder | Sobre o CF mais baixo OU rode anti-cogging |
| Iq travado em saturação várias vezes | Você está clipando | Reduz `fxratio` ou aumenta `current_lim` |

---

## Passo 6 — Tunando os efeitos do jogo

Filtros configurados, validados em sweep e gameplay. Agora vai pros efeitos finais — o que o jogo manda.

### Hierarquia de gain

```
Jogo manda força (-100% a +100%) — em sim de corrida = só Constant Force
       ↓
   × fxratio    (slider master FFB)
       ↓
   × maxtorque  (limite absoluto em Nm)
       ↓
   = torque entregue ao motor
```

| Param | Range típico | Onde mexer |
|---|---|---|
| `maxtorque` | 3–10 Nm | Aba FFB Wheel — limite físico do que motor pode dar (depende de `current_lim`) |
| `fxratio` | 50–100% | Aba FFB Wheel — atenuador global, use pra evitar clipping |
| `range_deg` | 540–1080° | Aba FFB Wheel — range total da rotação |

### Efeitos no jogo (sim de corrida)

**Em 90% dos casos, só importa um slider:**

- **Force Feedback Strength / Gain / Intensidade** — o gain do Constant Force. **Este é o único que faz diferença real.**

### Diagnóstico — descobrir o que o jogo realmente envia

Antes de gastar tempo ajustando sliders no jogo, **veja primeiro o que ele está mandando** pelo USB:

1. Abra a aba **FFB Live** durante o gameplay (ou com o jogo em pista, FFB ativo)
2. Olhe o painel **`Efeitos ativos`** — mostra até 3 slots concorrentes
3. Para cada slot ativo (`state ≠ idle`), você vê:
   - **`type`** — qual efeito foi registrado pelo jogo (Constant Force, Spring, Damper, Friction, Periodic, Inertia, etc.)
   - **`magnitude`** — intensidade atual do effect (-32768 a 32767)
   - **`gain`** — gain individual do effect (0–10000)
4. No mesmo painel, **`Effect 0 magnitude — análise de dinâmica`** mostra:
   - **`samples na janela`** — quantas atualizações de magnitude o jogo enviou desde o último Reset. Dividido pelo tempo de janela = **taxa de refresh do effect** (Hz que o jogo manda atualização)
   - **`range`** — variação dinâmica (range pequeno + média alta = sinal estático tipo mola; range grande = dinâmica chegando)
   - **`delta máximo`** — maior salto entre samples consecutivos (transientes tipo kerb/slip)

**Padrões típicos:**

| Observação no `Efeitos ativos` | Significado |
|---|---|
| 1 slot ativo, `type = Constant Force`, magnitude variando rápido | Sim de corrida típico — CF é tudo |
| 2+ slots ativos com types diferentes (CF + Spring, CF + Damper) | Jogo antigo ou sim que **realmente** envia effects separados — vale ajustar todos os filtros |
| Slot com `type = Spring/Damper/Friction` mas mag estático | Jogo registrou mas não atualiza — efeito de auto-centro fixo |
| Taxa de refresh < 60 Hz | Jogo lento, sinal vai chegar "stair-stepped" — CF filter ajuda a suavizar |
| Taxa de refresh > 250 Hz | Jogo de alta taxa (iRacing 360 Hz, etc.) — pode aproveitar CF mais agressivo |

**Teste empírico rápido:** mexa o slider **Damper** do jogo durante o gameplay. Se nenhum slot novo aparecer com `type = Damper`, o slider **não envia effect separado** — ele apenas modula a CF. Único caminho de ter Damper real é habilitando manualmente na aba **FFB Wheel**.

### Estratégia de tuning (sim de corrida)

2. **Sobe FFB Strength (gain de CF) até sentir os efeitos** — kerbs devem dar pulso forte
3. **Se Live FFT mostrar clipping (Iq saturando > 5%):** reduz `fxratio` na aba FFB Wheel ou reduz FFB Strength no jogo
4. **Se quiser mais peso/damping que o jogo não dá:** habilita manualmente Damper ou Friction na aba **FFB Wheel** com gain pequeno (10–30%). Esses efeitos rodam em paralelo no firmware OFFB, somam com o CF do jogo. Não recomendado, pelo bem da senseção real de pista.

### Quando os efeitos extras valem a pena

| Caso | Adicionar manualmente |
|---|---|
| Wheel coast forever, oscila parando | **Damper** (10–20%) — adiciona freio velocidade-dependente que o sim não tá dando |
| Wheel leve demais, sem "peso" parado | **Friction** (5–15%) — adiciona resistência constante |

### Sinais de clipping

Se durante gameplay você sentir:
- **Pulsos fortes saturando no mesmo nível** — `maxtorque` ou `current_lim` atingido
- **Granulosidade em alta velocidade** — current bandwidth insuficiente, ou CF cortando demais
- **Vibração em uma região específica do volante** — cogging residual, considere anti-cogging
- **Lag entre input e resposta** — `fxratio` muito baixo OU filtros muito agressivos

### Validação contínua

Mantém o **Live FFT no PiP overlay** aberto durante uma sessão de gameplay. Se Iq saturar > 5% do tempo, você tá clipando — recua `fxratio`. Se Iq seguir τ_cmd com fidelidade alta (curva Bode plana até CF), você tá no ponto certo.

---

## Troubleshooting

### "Wheel oscila quando paro o carro"
- **Solução:** habilita Damper manualmente na aba **FFB Wheel** com 15–30% — vai somar com a CF do jogo.

### "Wheel parece pesado demais em curva"
- CF cortando demais OU jogo entregando muito sinal
- Reduz FFB Strength no jogo OU sobe CF cutoff
- Se você habilitou Friction manual, reduz ou desabilita

### "Não sinto kerb"
- CF filter cortando demais OU `fxratio` baixo OU FFB Strength baixo no jogo
- Sobe CF cutoff pra 80 Hz, sobe `fxratio` pra 100%, sobe FFB Strength

### "Wheel `morde` ao soltar"
- Sim de corrida modernos: o "morder" é parte do CF de simulação, não Spring effect.
- **Solução:** reduz FFB Strength geral do jogo, ou habilita Damper manual na aba FFB Wheel pra suavizar o retorno

### "Vibração rápida em retas"
- Workaround: baixa CF cutoff (mas perde detalhe)

### "Volante `pula` em vibrações fortes"
- Saturação de current
- Reduz `fxratio`, ou aumenta `current_lim` se o motor aguenta

--

Bom tuning. 🏎️
