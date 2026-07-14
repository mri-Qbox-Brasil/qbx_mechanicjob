# qbx_mechanicjob — Manual

Emprego de mecânico da Autocare: registra o desgaste de peças dos veículos conforme a quilometragem rodada e permite consertá-las peça a peça, com o carro preso a uma plataforma da oficina.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Sistema de peças e desgaste](#sistema-de-peças-e-desgaste)
5. [A oficina](#a-oficina)
6. [Comandos](#comandos)
7. [Integrações](#integrações)
8. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
9. [Localização](#localização)
10. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | Framework base, `GetPlayer`, `Notify`, `SetJob`, `qbx.spawnVehicle` |
| `ox_lib` | Sim | Callbacks, zonas, context menu, progress bar, comandos, locale |
| `oxmysql` | Sim | Lê e grava `status`, `mods` e `drivingdistance` em `player_vehicles` |
| `ox_inventory` | Sim | Baú da oficina (`RegisterStash`), consumo dos materiais de reparo |
| `scully_emotemenu` | Sim | Animação de reparo. O `client/main.lua` chama `exports.scully_emotemenu:playEmoteByCommand('mechanic')` sem checagem |
| `ox_target` | Não | Só quando `useTarget = true` |
| `qbx_hud` (ou outro que ouça `hud:client:UpdateDrivingMeters`) | Não | Exibe a quilometragem do veículo |
| Recurso de chaves compatível com `vehiclekeys:client:SetOwner` | Não | Recebe o evento ao spawnar veículo da lista |

---

## Instalação

1. Copie a pasta `qbx_mechanicjob` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_mechanicjob
   ```
3. Cadastre o job `mechanic` (tipo `mechanic`) no `qbx_core`. O baú e os pontos da oficina exigem `job.type == 'mechanic'` e o jogador em serviço.
4. Garanta que os itens de reparo existam no `ox_inventory`: `metalscrap`, `plastic`, `steel`, `aluminum`, `copper`, `iron` e `advancedrepairkit`.
5. Preencha `authorizedIds` em `config/server.lua` com os `citizenid` dos donos da oficina — sem isso, `/setmechanic` e `/firemechanic` não funcionam para ninguém.
6. **Conflitos** — o `fxmanifest.lua` declara `provide 'qb-mechanicjob'`. Não rode junto com o `qb-mechanicjob` ou o `qb-vehicletuning`: os eventos `qb-vehicletuning:*` e `vehiclemod:*` são os mesmos.

A tabela `player_vehicles` do `qbx_core` já traz as colunas usadas (`status`, `mods`, `drivingdistance`) — não há SQL próprio.

---

## Configuração

### `config/shared.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `maxStatusValues` | tabela | Sim | Valor máximo de cada peça. `engine` e `body` vão a `1000.0`; as demais a `100` |
| `repairCost` | tabela `peça = item` | Sim | Item exibido no relatório de status no chat (apenas rótulo) |
| `repairCostAmount` | tabela `peça = {item, costs}` | Sim | Item e quantidade **realmente consumidos** no reparo daquela peça |
| `plates` | array | Sim | Plataformas (elevadores) da oficina. Ver abaixo |
| `locations.exit` | `vec3` | Sim | Posição do blip da oficina |
| `locations.duty` | `vec3` | Sim | Ponto de entrar/sair de serviço |
| `locations.stash` | `vec3` | Sim | Ponto do baú da oficina |
| `locations.vehicle` | `vec4` | Sim | Ponto de spawn/guarda dos veículos de trabalho |

`repairCost` e `repairCostAmount` são tabelas independentes e, no config padrão, **divergem** (ex.: `radiator` mostra `plastic` no chat mas consome `steel`). Se ajustar uma, ajuste a outra.

#### Formato de uma plataforma (`plates`)

```lua
{
    coords = vec4(-340.95, -128.24, 39, 160.0),
    boxData = { heading = 340, length = 5, width = 2.5, debugPoly = false },
    AttachedVehicle = nil,
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `coords` | `vec4` | Posição e heading em que o veículo é fixado |
| `boxData.heading` / `.length` / `.width` | number | Dimensões da zona de interação |
| `boxData.debugPoly` | bool | Desenha a zona |
| `AttachedVehicle` | — | Estado de runtime. Mantenha `nil` |

### `config/client.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `useTarget` | bool | Sim | `true` usa `ox_target` no ponto de serviço e no baú; `false` usa zonas com tecla `E` |
| `debugPoly` | bool | Não | Desenha a zona da garagem |
| `targets` | tabela | Sim | Estado de runtime das zonas. Deixe vazio |
| `partLabels` | tabela | Sim | Rótulos das peças (vindos do locale) |
| `vehicles` | tabela `modelo = 'Label'` | Sim | Veículos de trabalho spawnáveis na garagem da oficina |
| `minimalMetersForDamage` | array | Sim | Faixas de quilometragem (em km) e o multiplicador de dano correspondente. Ver abaixo |
| `damageableParts` | array de string | Sim | Peças que sofrem desgaste por quilometragem (`radiator`, `axle`, `brakes`, `clutch`, `fuel`) |

#### Formato de `minimalMetersForDamage`

```lua
{ min = 8000, max = 12000, multiplier = { min = 1, max = 8 } }
```

Se a quilometragem (km) do veículo estiver entre `min` e `max`, o desgaste por tick é `math.random(multiplier.min, multiplier.max) / 100` em cada peça de `damageableParts`. Acima da última faixa, o último multiplicador é usado.

### `config/server.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `authorizedIds` | array de string | Sim | `citizenid` autorizados a usar `/setmechanic` e `/firemechanic` |

---

## Sistema de peças e desgaste

Cada placa tem um estado de 7 peças: `engine`, `body`, `radiator`, `axle`, `brakes`, `clutch`, `fuel`. `engine` e `body` acompanham a saúde nativa do veículo; as outras cinco são gerenciadas pelo recurso e persistidas na coluna `status` de `player_vehicles`.

**Desgaste por quilometragem** (`client/drivingdistance.lua`): enquanto o jogador dirige, a distância percorrida é acumulada e enviada ao servidor a cada ~2 segundos. Se a quilometragem cair numa das faixas de `minimalMetersForDamage`, há 1 chance em 3, por tick, de desgastar as peças de `damageableParts`. Veículos que não estão no banco começam com uma quilometragem aleatória entre 111.111 e 999.999.

**Efeitos do dano** (`client/damage-effects.lua`): a cada 10–15 ticks dirigindo, um efeito é sorteado entre as peças abaixo de 80% de vida. Quanto pior a peça, mais severo o efeito. Veículos das classes 13, 14, 15, 16 e 21 (bicicletas, aviões, helicópteros, barcos, trens) são ignorados.

| Peça | Efeito |
|---|---|
| `radiator` | Perde saúde do motor (de 10–15 até 40–50 por tick) |
| `axle` | A direção fica descalibrada (`SetVehicleSteeringScale`) |
| `brakes` | O freio de mão trava sozinho, de 1 a 9 segundos |
| `clutch` | O motor morre e o carro fica indirigível por instantes |
| `fuel` | Vaza combustível (2 a 10 unidades por tick) |

---

## A oficina

Com o job `mechanic` e em serviço:

- **Ponto de serviço** — entra/sai de serviço (`QBCore:ToggleDuty`).
- **Baú** — abre a stash `mechanicstash` (500 slots, 4.000.000 de peso), restrita ao grupo `mechanic`.
- **Garagem** — na zona de veículos, abre a lista de veículos de trabalho (`Flatbed`, `Towtruck`, `Minivan`, `Blista`). Se já estiver dentro de um veículo, a mesma tecla o guarda. O veículo spawnado recebe placa `MECH####` e tanque cheio.
- **Plataforma** — dirigindo até uma das plataformas, `E` prende o veículo ao elevador. Com o veículo preso, `E` abre o menu:
  - **Desconectar veículo** — solta o veículo e coloca o mecânico ao volante.
  - **Ver status** — envia ao chat o relatório com o valor de cada peça e o item de reparo correspondente.
  - **Peças do veículo** — lista as peças, e reparar uma consome o item de `repairCostAmount` e roda um progresso de 5 a 10 segundos.

O estado das plataformas (`AttachedVehicle`) é sincronizado no servidor: todos os clientes veem o mesmo veículo preso.

---

## Comandos

| Comando | Permissão | Descrição |
|---|---|---|
| `/setmechanic <id>` | `citizenid` em `authorizedIds` | Contrata o jogador como mecânico (define o job `mechanic`) |
| `/firemechanic <id>` | `citizenid` em `authorizedIds` | Demite o mecânico (define o job `unemployed`) |
| `/setvehiclestatus <peça> <valor>` | `group.god` | Define o valor de uma peça do veículo em que o jogador está |

`/setmechanic` e `/firemechanic` não são restritos por ACE: qualquer um pode digitar, mas só executam se o `citizenid` do autor estiver em `config/server.lua > authorizedIds`.

---

## Integrações

### scully_emotemenu

O reparo de peças toca a emote `mechanic` durante a barra de progresso, e a cancela ao terminar ou desistir.

### ox_inventory

O baú `mechanicstash` é registrado na inicialização do servidor com o grupo `mechanic`. Os materiais de reparo são consumidos no servidor via `RemoveItem`, após checagem de quantidade.

### HUD (`hud:client:UpdateDrivingMeters`)

A quilometragem calculada (em km) é enviada por esse evento a cada atualização, para exibição no HUD.

---

## Entrypoints para outros recursos

### Exports de cliente

```lua
-- Estado completo das peças de uma placa
local status = exports.qbx_mechanicjob:GetVehicleStatusList(plate)

-- Valor de uma peça específica
local engine = exports.qbx_mechanicjob:GetVehicleStatus(plate, 'engine')

-- Define o valor de uma peça (propaga ao servidor)
exports.qbx_mechanicjob:SetVehicleStatus(plate, 'radiator', 100)
```

### Eventos de servidor

```lua
-- Inicializa o status de uma placa (usa o status do banco, se houver)
TriggerServerEvent('vehiclemod:server:setupVehicleStatus', plate, engineHealth, bodyHealth)

-- Atualiza uma peça (com clamp no valor máximo)
TriggerServerEvent('vehiclemod:server:updatePart', plate, part, level)

-- Define uma peça sem clamp
TriggerServerEvent('qb-vehicletuning:server:SetPartLevel', plate, part, level)

-- Restaura todas as peças ao máximo
TriggerServerEvent('vehiclemod:server:fixEverything', plate)

-- Persiste o status atual em player_vehicles
TriggerServerEvent('vehiclemod:server:saveStatus', plate)

-- Salva os mods do veículo em player_vehicles
TriggerServerEvent('qb-vehicletuning:server:SaveVehicleProps', vehicleProps)

-- Atualiza a quilometragem
TriggerServerEvent('qb-vehicletuning:server:UpdateDrivingDistance', amount, plate)
```

### Eventos de cliente

```lua
-- Repara tudo no veículo em que o jogador está (precisa ser o motorista)
TriggerClientEvent('vehiclemod:client:fixEverything', source)

-- Define o valor de uma peça no veículo em que o jogador está
TriggerClientEvent('vehiclemod:client:setPartLevel', source, part, level)

-- Sincroniza o status de uma placa
TriggerClientEvent('vehiclemod:client:setVehicleStatus', -1, plate, status)
```

### Callbacks (`lib.callback`)

- `qb-vehicletuning:server:GetDrivingDistances()` — quilometragem de todas as placas conhecidas
- `qb-vehicletuning:server:IsVehicleOwned(plate)` — se a placa existe em `player_vehicles`
- `qb-vehicletuning:server:GetAttachedVehicle()` — estado das plataformas

---

## Localização

Strings via `ox_lib` locale, em `locales/`:

`cs`, `de`, `en`, `es`, `pt`, `pt-br`, `ro`

```
setr ox:locale "pt-br"
```

---

## Estrutura de arquivos

```
qbx_mechanicjob/
├── client/
│   ├── main.lua              — zonas da oficina, plataformas, menus, reparo de peças, blip, exports
│   ├── damage-effects.lua    — efeitos do desgaste (radiador, direção, freios, embreagem, vazamento)
│   └── drivingdistance.lua   — cálculo da quilometragem e desgaste progressivo das peças
├── server/
│   └── main.lua              — status das peças, persistência em player_vehicles, stash, spawn de veículos, comandos
├── config/
│   ├── client.lua            — target, veículos de trabalho, faixas de desgaste, peças danificáveis
│   ├── server.lua            — citizenids autorizados a contratar/demitir
│   └── shared.lua            — valores máximos, custos de reparo, plataformas e locais da oficina
├── locales/                  — cs, de, en, es, pt, pt-br, ro (.json)
└── fxmanifest.lua
```
