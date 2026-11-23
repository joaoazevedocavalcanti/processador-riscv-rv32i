# RV32I Pipelined Base Project

Processador RISC-V de 32 bits com pipeline de 5 estágios implementado em SystemVerilog, baseado nas seções 4.6 a 4.10 de *Computer Organization and Design: RISC-V Edition* (Patterson & Hennessy). O projeto tem como plataforma alvo a placa **DE2-115** (Intel Cyclone IV E) e é estruturado para servir de base para extensões do conjunto de instruções pelos alunos.

**Status:** as 37 instruções do RV32I base estão implementadas e validadas na placa (LEDs + dump serial) — Etapas 01, 02 e 03.

---

## Instruções suportadas

| #  | Instrução | Tipo | Opcode  | Etapa | Status |
|----|-----------|------|---------|:-----:|:------:|
| 1  | `ADD`     | R    | 0110011 | base | ✅ |
| 2  | `SUB`     | R    | 0110011 | base | ✅ |
| 3  | `OR`      | R    | 0110011 | base | ✅ |
| 4  | `AND`     | R    | 0110011 | base | ✅ |
| 5  | `SLT`     | R    | 0110011 | base | ✅ |
| 6  | `LW`      | I    | 0000011 | base | ✅ |
| 7  | `SW`      | S    | 0100011 | base | ✅ |
| 8  | `BEQ`     | B    | 1100011 | base | ✅ |
| 9  | `XOR`     | R    | 0110011 | 01 | ✅ |
| 10 | `SLL`     | R    | 0110011 | 01 | ✅ |
| 11 | `SRL`     | R    | 0110011 | 01 | ✅ |
| 12 | `SRA`     | R    | 0110011 | 01 | ✅ |
| 13 | `SLTU`    | R    | 0110011 | 01 | ✅ |
| 14 | `ADDI`    | I    | 0010011 | 02 | ✅ |
| 15 | `ANDI`    | I    | 0010011 | 02 | ✅ |
| 16 | `ORI`     | I    | 0010011 | 02 | ✅ |
| 17 | `XORI`    | I    | 0010011 | 02 | ✅ |
| 18 | `SLTI`    | I    | 0010011 | 02 | ✅ |
| 19 | `SLTIU`   | I    | 0010011 | 02 | ✅ |
| 20 | `SLLI`    | I    | 0010011 | 02 | ✅ |
| 21 | `SRLI`    | I    | 0010011 | 02 | ✅ |
| 22 | `SRAI`    | I    | 0010011 | 02 | ✅ |
| 23 | `LUI`     | U    | 0110111 | 02 | ✅ |
| 24 | `AUIPC`   | U    | 0010111 | 02 | ✅ |
| 25 | `LB`      | I    | 0000011 | 03 | ✅ |
| 26 | `LH`      | I    | 0000011 | 03 | ✅ |
| 27 | `LBU`     | I    | 0000011 | 03 | ✅ |
| 28 | `LHU`     | I    | 0000011 | 03 | ✅ |
| 29 | `SB`      | S    | 0100011 | 03 | ✅ |
| 30 | `SH`      | S    | 0100011 | 03 | ✅ |
| 31 | `BNE`     | B    | 1100011 | 03 | ✅ |
| 32 | `BLT`     | B    | 1100011 | 03 | ✅ |
| 33 | `BGE`     | B    | 1100011 | 03 | ✅ |
| 34 | `BLTU`    | B    | 1100011 | 03 | ✅ |
| 35 | `BGEU`    | B    | 1100011 | 03 | ✅ |
| 36 | `JAL`     | J    | 1101111 | 03 | ✅ |
| 37 | `JALR`    | I    | 1100111 | 03 | ✅ |

### Resumo de cobertura do ISA RV32I

| Categoria          | Total ISA | Implementadas | Faltando |
|--------------------|:---------:|:-------------:|:--------:|
| R-type             | 10        | 10            | 0        |
| I-type aritmético  | 9         | 9             | 0        |
| I-type load        | 5         | 5             | 0        |
| S-type             | 3         | 3             | 0        |
| B-type             | 6         | 6             | 0        |
| U-type             | 2         | 2             | 0        |
| J-type             | 2         | 2             | 0        |
| **Total**          | **37**    | **37**        | **0**    |

### Etapas do projeto

| Etapa | Escopo | Status |
|:-----:|--------|:------:|
| 01 | R-type: `XOR`, `SLL`, `SRL`, `SRA`, `SLTU` | ✅ |
| 02 | I-type aritmético (`ADDI`, `ANDI`, `ORI`, `XORI`, `SLTI`, `SLLI`, `SRLI`, `SRAI`) + `LUI`, `AUIPC` | ✅ |
| 03 | Loads (`LB`, `LH`, `LBU`, `LHU`), stores (`SB`, `SH`), branches (`BNE`, `BLT`, `BGE`, `BLTU`, `BGEU`) e jumps (`JAL`, `JALR`) | ✅ |

---

## Arquitetura

### Pipeline de 5 estágios

```
+------+     +------+     +------+     +------+     +------+
|  IF  | --> |  ID  | --> |  EX  | --> | MEM  | --> |  WB  |
+------+     +------+     +------+     +------+     +------+
  Busca      Decode       Execução     Memória     Write-back
  instrução  Regs         ALU          dmem/MMIO   Regfile
             Sign-ext     Branch/Jump  Load/Store
```

**Registradores de pipeline:** `IF/ID`, `ID/EX`, `EX/MEM`, `MEM/WB` (definidos em `pl_pipe_pkg.sv`). O `ID/EX` carrega, além dos sinais originais, `alu_src_a`, `jump` e `jalr`.

### Tratamento de hazards

| Hazard | Mecanismo | Detalhe |
|--------|-----------|---------|
| RAW (dado) | Forwarding | EX/MEM → EX e MEM/WB → EX (`pl_forward.sv`) |
| Load-use | Stall | 1 bolha inserida em ID/EX (`pl_hazard.sv`) |
| Branch taken / Jump | Flush | 2 NOPs (IF e ID) na resolução em EX (`pl_datapath.sv`) |

**Resolução de branch e jump no estágio EX:** quando um branch é tomado (ou em qualquer `JAL`/`JALR`), o PC retorna ao alvo e os dois estágios anteriores são esvaziados. O custo é de 2 ciclos de penalidade.

**Banco de registradores com escrita em negedge:** a escrita ocorre na borda de descida do clock, garantindo que o estágio ID leia o valor atualizado no próximo posedge sem necessidade de forwarding WB→ID explícito.

### Decisões de projeto das Etapas 02 e 03

| Recurso | Como foi implementado |
|---------|-----------------------|
| `LUI` / `AUIPC` | Reaproveitam a ALU em modo `ADD`. Um mux na entrada A da ALU (`ALUSrcA`) escolhe `rs1` (`00`), `PC` (`01`, AUIPC) ou zero (`10`, LUI). O imediato U é `{Instr[31:12], 12'b0}` (`pl_sign_ext.sv`). |
| Decisão de branch | Módulo `pl_branch_unit`: decide por `funct3` (`000` ==, `001` !=, `100` < com sinal, `101` >= com sinal, `110` < sem sinal, `111` >= sem sinal). `pc_src = (branch && taken) \|\| jump`. |
| `JAL` / `JALR` | Sinais `Jump` e `Jalr` vindos do `pl_control`. Alvo do `JAL` = `PC + imm_J`; alvo do `JALR` = `(rs1 + imm_I)` com bit 0 zerado. O resultado escrito em `rd` é `PC + 4` (substitui o resultado da ALU em `ex_mem.alu_result`). |
| Loads `LB/LH/LBU/LHU` | Módulo `pl_load_unit`: seleciona o byte/halfword pelos bits baixos do endereço e faz extensão de sinal ou zero. `LW` passa direto. |
| Stores `SB/SH` | Módulo `pl_store_unit`: *read-modify-write* sobre a palavra lida de `pl_dmem` (leitura assíncrona): `NewWord = (OldWord & ~mask) \| (data_rep & mask)`. |

A divisão em módulos pequenos (`pl_branch_unit`, `pl_load_unit`, `pl_store_unit`) mantém `pl_control` responsável apenas por decodificar sinais de controle no estágio ID, enquanto a lógica que depende de dados e de endereços em tempo de execução (comparações, alinhamento, máscaras) fica nos estágios EX/MEM, onde esses valores existem.

### Mapa de memória

| Endereço (byte) | Tamanho | Acesso | Descrição |
|-----------------|---------|--------|-----------|
| `0x000 – 0x3FC` | 256 × 32 b | R/W | Memória de dados (`pl_dmem`) — word, halfword e byte |
| `0x400` | 32 b | R | SW\[17:0\] — chaves deslizantes |
| `0x404` | 32 b | R | KEY\[3:0\] — botões push |
| `0x408` | 32 b | W | LEDR\[17:0\] — LEDs vermelhos |
| `0x40C` | 32 b | W | LEDG\[8:0\] — LEDs verdes |
| `0x410` | 32 b | R/W | UART RS-232 (8N1, 9600 baud) |
| `0x414` | 32 b | R | Contador de ciclos de clock (32 bits, reset em 0) |

Seleção: `alu_result[10] = 1` redireciona o acesso ao controlador MMIO (`pl_mmio`).

**Leitura da UART (`lw` em 0x410):** `{22'b0, tx_busy[9], rx_ready[8], rx_data[7:0]}`

**Escrita na UART (`sw` em 0x410):** o hardware (`pl_mmio`) divide automaticamente a palavra de 32 bits em 4 bytes e os envia em sequência little-endian (byte 0 primeiro). `tx_busy` permanece alto até o último byte ser aceito pelo shift register. O software deve aguardar `tx_busy = 0` antes de escrever nova palavra.

### ALU — codificação de operação

| Operation | Instruções |
|-----------|------------|
| `4'd01` ADD | `LW`/`SW`/`LB`/`LH`/`LBU`/`LHU`/`SB`/`SH` (cálculo de endereço), `ADD`, `ADDI`, `LUI`, `AUIPC`, `JALR` (cálculo do alvo) |
| `4'd02` SUB | `SUB` e comparação de branches |
| `4'd04` OR  | `OR`, `ORI` |
| `4'd05` AND | `AND`, `ANDI` |
| `4'd11` SLT | `SLT`, `SLTI` |
| demais      | `XOR`/`XORI`, `SLL`/`SLLI`, `SRL`/`SRLI`, `SRA`/`SRAI`, `SLTU`/`SLTIU` — códigos em `pl_alu.sv` / `pl_alu_ctrl.sv` |

No `pl_alu_ctrl`, `ALUOp = 2'b11` indica instrução I-type aritmética: a operação é decodificada por `funct3`, e `funct7[5]` só é considerado para diferenciar `SRLI` de `SRAI`.

---

## Estrutura do repositório

```
rv32i_pipelined_base_project/
│
├── src/                        Código-fonte SystemVerilog
│   ├── pl_pipe_pkg.sv          Structs dos registradores de pipeline
│   ├── pl_top.sv               Top-level com PLL (50→10 MHz) + CPU
│   ├── pl_top_no_pll.sv        Top-level alternativo: CPU direto no CLOCK_50
│   ├── pl_cpu.sv               Wrapper: Control + ALU_Ctrl + Datapath
│   ├── pl_control.sv           Unidade de controle (estágio ID)
│   ├── pl_alu_ctrl.sv          Controle da ALU (estágio EX)
│   ├── pl_alu.sv               ALU 32 bits
│   ├── pl_branch_unit.sv       Decisão de branch por funct3
│   ├── pl_load_unit.sv         Seleção e extensão de sinal de loads (LB/LH/LBU/LHU)
│   ├── pl_store_unit.sv        Merge de byte/halfword para stores (SB/SH)
│   ├── pl_regfile.sv           Banco de registradores 32×32 b
│   ├── pl_sign_ext.sv          Extensão de imediatos I/S/B/U/J
│   ├── pl_imem.sv              Memória de instruções 256×32 b (MIF)
│   ├── pl_dmem.sv              Memória de dados 256×32 b (MIF)
│   ├── pl_mmio.sv              Controlador MMIO (SW/KEY/LED/UART/ciclos)
│   ├── pl_uart.sv              UART RS-232 8N1 (9600 baud)
│   ├── pl_hazard.sv            Detecção de hazard load-use
│   ├── pl_forward.sv           Unidade de forwarding
│   ├── pl_datapath.sv          Datapath 5 estágios
│   ├── pl_cpu_tb.sv            Testbench de verificação
│   └── pll_10mhz.v             PLL Altera (50 MHz → 10 MHz)
│
├── mifs/                       Arquivos de inicialização de memória
│   ├── instruction.mif         Programa de teste padrão
│   ├── data.mif                Dados iniciais do programa de teste
│   ├── mmio_test_program.mif   Programa de teste MMIO + UART
│   ├── mmio_test_data.mif      Dados para o teste MMIO
│   └── test/                   MIFs dos programas de validação das Etapas 01–03
│
├── assembler/
│   ├── assembler.py            Montador Python: assembly → .mif / dump
│   ├── hello.asm               Programa de entrada padrão
│   ├── instruction.mif         MIF gerado (saída do assembler)
│   ├── data.mif                MIF de dados gerado (modo --dump)
│   ├── program.hex             Hex gerado (modo normal)
│   └── test/                   Programas .asm de validação (ver "Validação")
│
├── dump/
│   ├── serial_dump.py          Captura e decodifica dump serial do FPGA
│   └── test/                   Dumps salvos de cada teste validado
│
├── modelsim/                   Projeto ModelSim para simulação
│   ├── *.mpf                   Arquivo de projeto ModelSim
│   ├── program.hex             Imagem da instrução (.hex para $readmemh)
│   ├── data.hex                Imagem dos dados (.hex para $readmemh)
│   ├── golden.txt              Saída esperada pelo testbench
│   └── output.txt              Saída gerada na última simulação
│
└── quartus/                    Projeto Quartus Prime para síntese
    ├── *.qpf / *.qsf           Arquivos de projeto Quartus
    ├── instruction.mif         MIF da instrução (cópia de trabalho)
    └── data.mif                MIF dos dados (cópia de trabalho)
```

> Os arquivos `.sv` ficam apenas em `src/`; o projeto Quartus referencia essa pasta (evite manter cópias duplicadas em `quartus/`).

---

## Validação

Cada instrução foi testada na DE2-115 com um programa `.asm` próprio, conferindo o resultado nos **LEDs** e no **dump serial** (registradores x1–x8, memória de dados e contador de ciclos). Todos os resultados abaixo bateram com o valor esperado, calculado por um simulador de referência do ISA.

| Teste | Instruções cobertas | Ciclos (dump) |
|-------|---------------------|:-------------:|
| `xor_test` | `XOR` | 39 |
| `sll_test` | `SLL` | 45 |
| `srl_sra_test` | `SRL`, `SRA` | 58 |
| `sltu_test` | `SLTU` | 46 |
| `addi` / `andi` / `ori` / `slti` / `slli` / `srli` / `srai` / `xori` | um teste por instrução I-type | 36 cada |
| `lui_test` | `LUI` | 38 |
| `auipc_test` | `AUIPC` | 34 |
| `regressao_test` | todas as instruções das Etapas 01–02 (64 instruções); LEDR = `0x14CD1` | 87 |
| `branch_test` | `BNE`, `BLT`, `BGE`, `BLTU`, `BGEU` (tomado e não tomado); LEDR = `0x7FF` | 105 |
| `mem_test` | `LB`, `LH`, `LBU`, `LHU`, `SB`, `SH`; LEDR = `0x392F0` | 70 |
| `jump_test` | `JAL`, `JALR`; LEDR = `0x7F` | 92 |
| `professor_test` | programa de verificação fornecido pelo professor; `dmem[0..12]` conferem com a tabela esperada | 76 |

Além da placa, o processador foi verificado em simulação RTL (Icarus Verilog) contra um simulador de referência do ISA, com testbench de unidades e testes de mutação.

### Como rodar um teste

```bash
cd assembler
python3 assembler.py --dump test/<programa>.asm     # gera instruction.mif + data.mif
cp instruction.mif data.mif ../quartus/             # copia para o projeto Quartus
# compilar e gravar na FPGA via Quartus; apertar KEY[0] para resetar
cd ../dump
python3 serial_dump.py COM3                         # captura o resultado (salva dump/dump.txt)
```

> O `data.mif` dos testes das Etapas 02–03 contém apenas as constantes de infraestrutura do dump (`0x080–0x086`); os dados de cada teste, quando existem, ficam no `.mif` de dados correspondente.

---

## Como simular (ModelSim)

1. Abra o ModelSim e carregue o projeto `modelsim/rv32i_pipelined_base_project.mpf`.
2. Compile todos os arquivos em `src/` — compilar `pl_pipe_pkg.sv` primeiro.
3. Inicie a simulação com o top `pl_cpu_tb`.
4. Execute `run -all`.

O testbench detecta o halt automaticamente (repetição periódica do PC causada pelo `beq x0, x0, 0`), imprime o estado final dos registradores e da memória, e compara a saída com `modelsim/golden.txt`.

**Programa de teste padrão (`mifs/instruction.mif`):**

```
000: lw  x1,  0(x0)   x1 = 10
001: lw  x2,  4(x0)   x2 = 20
002: add x3, x1, x2   x3 = 30   -- forwarding MEM/WB→EX
003: and x4, x3, x1   x4 = 10   -- forwarding EX/MEM→EX
004: lw  x5,  0(x0)   x5 = 10   -- causa load-use stall
005: add x6, x5, x2   x6 = 30   -- executado após 1 ciclo de stall
006: sw  x3,  8(x0)   dmem[2] = 30
007: beq x1, x2, +8   NÃO tomado (10 ≠ 20)
008: add x7, x1, x2   x7 = 30
009: beq x1, x5, +8   TOMADO (10 = 10) → flush de 00A
00A: add x8, x0, x0   DESCARTADO
00B: add x8, x1, x4   x8 = 20
00C: beq x0, x0,  0   halt
```

Estado final esperado: `x1=10  x2=20  x3=30  x4=10  x5=10  x6=30  x7=30  x8=20  dmem[2]=30`

---

## Como sintetizar (Quartus)

1. Abra o Quartus Prime (17.1) e carregue `quartus/rv32i_pipelined_base_project.qpf`.
2. Verifique que os arquivos `instruction.mif` e `data.mif` estão na pasta `quartus/`.
3. Escolha o top-level (`pl_top` — padrão — ou `pl_top_no_pll`).
4. Execute **Processing → Start Compilation**.
5. Grave o bitstream na DE2-115 via JTAG.

**Dispositivo alvo:** Intel Cyclone IV E — EP4CE115F29C7  
**Reset:** KEY\[0\] ativo-baixo.

### Top-levels e clock

| Top-level | Clock da CPU | `CLK_HZ` | Observação |
|-----------|:------------:|:--------:|------------|
| `pl_top` | 10 MHz (PLL 50→10 MHz) | `10_000_000` | Configuração validada na placa; reset liberado após a PLL travar |
| `pl_top_no_pll` | 50 MHz (CLOCK_50 direto) | `50_000_000` | Fmax reportado pelo Quartus: 57,97 MHz. **Ainda não validado na placa** (nos testes até agora os LEDs permaneceram apagados) |

O parâmetro `CLK_HZ` é definido no top-level e propagado por `pl_cpu` → `pl_datapath` → `pl_mmio` → `pl_uart`, onde define o divisor da UART (`CLK_HZ / BAUD`). **Ele precisa corresponder ao clock real da CPU**: com `CLK_HZ` errado o baud efetivo muda e o dump chega corrompido (por exemplo, bytes `F0`/`00` quando a UART acredita estar a 50 MHz e a CPU roda a 10 MHz).

---

## Assembler

O script `assembler/assembler.py` converte assembly textual para o formato `.mif`.

### Uso

```bash
cd assembler

# modo normal — gera instruction.mif + program.hex
python3 assembler.py programa.asm

# sem argumento — usa instructions.txt como padrão
python3 assembler.py

# modo dump — gera instruction.mif + data.mif com dump serial embutido
python3 assembler.py --dump programa.asm
```

Todos os arquivos de saída são gerados na pasta `assembler/`.

### Modo normal

| Arquivo | Formato | Uso |
|---------|---------|-----|
| `instruction.mif` | Quartus MIF | Síntese (Quartus) e simulação (ModelSim via `ram_init_file`) |
| `program.hex` | Um word hex por linha | Simulação via `$readmemh` |

### Modo `--dump`

Anexa automaticamente o código de dump serial ao programa do usuário, sem que o usuário precise escrever nenhuma instrução extra.

| Arquivo | Conteúdo |
|---------|----------|
| `instruction.mif` | Código do usuário + `beq` de desvio + dump serial a partir de `0x080` |
| `data.mif` | Zeros + constantes de infraestrutura do dump em `0x080–0x086` |

**Layout da memória de instruções com `--dump`:**

```
0x000..N-1  Código do usuário (máx. 127 instruções)
0xN         beq x0,x0,→0x80   (desvio incondicional para o dump)
0xN+1..07F  NOP (addi x0,x0,0)
0x080..0x0AD  Dump serial — Parts 2–8 (46 instruções)
```

**Layout da memória de dados com `--dump`:**

```
0x000..0x01F  Área do usuário  (byte 0x000–0x07C) — transmitida no dump
0x078..0x07F  Saves de x1–x8  (byte 0x1E0–0x1FC) — escritos em runtime
0x080..0x086  Constantes de infraestrutura         — inicializadas no MIF
```

**O que o dump transmite via UART (164 bytes = 41 palavras):**

```
bytes  0–  3   Contador de ciclos (MMIO 0x414)
bytes  4– 35   Registradores x1–x8
bytes 36–163   dmem[0x000–0x07C] (32 palavras do usuário)
```

O programa do usuário pode ter no máximo **127 instruções**; o assembler valida e reporta erro se excedido.

### Formato de entrada aceito

```
add x3,x1,x2
lw x1,0(x0)
sw x3,8(x0)
beq x1,x2,-4
addi x5,x1,-7
lui x6,74565
```

Uma instrução por linha, sem rótulos, sem comentários, **sem espaços após as vírgulas** e com **um único espaço** entre mnemônico e operandos.

### Limitações conhecidas do assembler

Descobertas durante a validação. O assembler é fornecido pela disciplina e **não deve ser alterado**; os programas de teste foram escritos para contorná-las.

| Limitação | Efeito | Contorno nos testes |
|-----------|--------|---------------------|
| Imediatos apenas em decimal | `0x10` não é aceito | Converter para decimal |
| Sem espaços após vírgulas e sem comentários | `add x3, x1, x2` é rejeitado | Escrever `add x3,x1,x2` |
| `-2048` rejeitado em instruções I-type (`Immediate value out of range`) | Não é possível usar o menor imediato de 12 bits | Usar `-2047` |
| `slli`/`srli`/`srai` com shamt de 16 a 31 são codificados incorretamente (`SRAI` sai como `SRLI`) | Resultado errado em `srai` com shamt ≥ 16 | Manter `srai` com shamt ≤ 15 (alternativa: corrigir manualmente o primeiro dígito hex do `instruction.mif`, `8` → `4`) |
| `lui` / `auipc` recebem o imediato de 20 bits como decimal com sinal | `lui x6,74565` = `0x12345` | Converter o valor desejado para decimal |

---

## Detalhes de implementação

### Banco de registradores — escrita em negedge

A escrita no banco de registradores ocorre na **borda de descida** do clock (`negedge clk`). Isso resolve o conflito de leitura/escrita simultânea que ocorre quando, após um load-use stall, o estágio WB escreve um registrador no mesmo ciclo em que o estágio ID o lê para captura no registrador ID/EX. Com a escrita em negedge, o valor é garantidamente visível antes do próximo posedge. Esta é a solução descrita na Fig. 4.63 do P&H.

### Detecção de halt no testbench

O halt é implementado como `beq x0, x0, 0`. Com resolução de branch no estágio EX, o PC oscila com período 3 (H, H+4, H+8, H, …). O testbench detecta halt quando `PC_atual == PC_de_3_ciclos_atrás` por 9 ciclos consecutivos (3 períodos completos).

### UART (9600 8N1)

Implementada em `pl_uart.sv`, com parâmetros `CLK_HZ` (vindo do top-level; 10 MHz em `pl_top`, 50 MHz em `pl_top_no_pll`) e `BAUD = 9600`. O divisor é `CLK_HZ / BAUD` (1041 clocks/bit a 10 MHz; 5208 a 50 MHz), com erro de baud < 0,1%. O sequenciador de 4 bytes em `pl_mmio.sv` divide automaticamente cada `sw` em 0x410 em 4 transmissões consecutivas, liberando o software de enviar byte a byte.

### Dump serial

O script `dump/serial_dump.py` captura e decodifica o dump transmitido pelo FPGA. Pré-requisito: `pip install pyserial`.

```bash
python3 dump/serial_dump.py COM3            # Windows
python3 dump/serial_dump.py /dev/ttyUSB0   # Linux
python3 dump/serial_dump.py COM3 --out resultado.txt
```

Aguarda **164 bytes** (41 palavras × ~4 ms a 9600 baud) e exibe ciclos, x1–x8 e dmem[0x000–0x07C]. Para usar, compile com `python3 assembler.py --dump` e grave o bitstream no FPGA. Se nenhum byte chegar (timeout), confira se o programa foi montado com `--dump`, se os `.mif` foram copiados para `quartus/`, se o projeto foi recompilado e se KEY\[0\] foi pressionado.

**Fluxo completo:**

```bash
cd assembler
python3 assembler.py --dump meu_programa.asm  # gera instruction.mif + data.mif
cp instruction.mif data.mif ../quartus/        # copia para o projeto Quartus
# compilar e gravar na FPGA via Quartus...
cd ../dump
python3 serial_dump.py COM3                    # captura o resultado
```

---

## Git

Pastas geradas pelo Quartus não devem ser versionadas (causam erros como `.flock: Permission denied` no `git add`). Sugestão de `.gitignore`:

```
quartus/db/
quartus/incremental_db/
quartus/.qsys_edit/
quartus/output_files/
quartus/simulation/
modelsim/work/
```

Observação: `git add .` adiciona apenas a pasta atual e suas subpastas; para adicionar todo o repositório, rode `git add -A` a partir da raiz.

---

## Recursos

- [Manual RISC-V ISA v2.2](https://riscv.org/wp-content/uploads/2017/05/riscv-spec-v2.2.pdf)
- [RISC-V ISA Reference (msyksphinz)](https://msyksphinz-self.github.io/riscv-isadoc/html/rvi.html)
- [RISC-V Interpreter — Cornell University](https://www.cs.cornell.edu/courses/cs3410/2019sp/riscv/interpreter/)
- Patterson & Hennessy, *Computer Organization and Design: RISC-V Edition*, seções 4.6 – 4.10
