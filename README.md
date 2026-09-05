<p align="center">
  <img src="https://img.shields.io/badge/MCP-AutoCAD-blue?style=for-the-badge&logo=autodesk&logoColor=white" alt="AutoCAD MCP">
  <img src="https://img.shields.io/badge/python-3.10+-5586A4?style=for-the-badge&logo=python&logoColor=white" alt="Python 3.10+">
  <img src="https://img.shields.io/badge/license-MIT-green?style=for-the-badge&logo=opensourceinitiative&logoColor=white" alt="MIT License">
</p>

# 🏗️ AutoCAD MCP Server

> **Automatize o AutoCAD (ou gere DWG/DXF headless) por linguagem natural,**
> através do [Model Context Protocol](https://modelcontextprotocol.io). Uma API,
> dois backends: **File IPC** (AutoCAD real) e **ezdxf** (headless, sem AutoCAD).

<p align="center">
  <b>Claude, opencode, Cursor, ou qualquer cliente MCP</b> → desenha, edita,
  anota e exporta plantas direto no AutoCAD — ou gera arquivos DXF offline.
</p>

---

## ✨ O que este projeto faz

- **8 ferramentas consolidadas** (`drawing`, `entity`, `layer`, `block`,
  `annotation`, `pid`, `view`, `system`) sobre uma única API.
- **Backend File IPC**: controla o **AutoCAD LT 2024+ real** via
  `mcp_dispatch.lsp` + JSON em diretório temporário — sem roubar o foco da
  janela (`PostMessageW(WM_CHAR)`).
- **Backend ezdxf**: geração de DXF **totalmente headless** (Linux, macOS, WSL),
  com render por matplotlib.
- **`system(operation="execute_lisp")`**: executa **AutoLISP arbitrário** — vira
  uma plataforma de automação extensível (plugins, fiberQ, FTTH, P&ID...).
- **Screenshots** do desenho (Win32 `PrintWindow` no AutoCAD; render no headless).

---

## 🚀 Como funciona

```
MCP Client (Claude / opencode / Cursor)
     │
     │  stdio · JSON-RPC
     ▼
Python MCP Server (autocad_mcp — 8 tools)
     │
     ├──► File IPC ──► C:/temp/*.json ──► mcp_dispatch.lsp (AutoCAD LT)
     │      PostMessageW(WM_CHAR) → sem roubo de foco
     │
     └──► ezdxf ──► documento DXF em memória (headless)
            render de screenshot via matplotlib
```

| Backend | Runtime | Precisa de AutoCAD? | Screenshot |
|---------|---------|---------------------|------------|
| **File IPC** | Windows + Python | ✅ Sim — AutoCAD LT 2024+ | Win32 `PrintWindow` |
| **ezdxf** | Qualquer plataforma | ❌ Não (headless) | matplotlib |

> **`AUTOCAD_MCP_BACKEND=auto`** (padrão) tenta o File IPC e cai para `ezdxf`
> se não encontrar a janela do AutoCAD.

---

## 📦 Instalação

### Pré-requisitos (backends File IPC)

- **Windows 10/11** (usa APIs Win32 para envio de mensagens sem foco)
- **AutoCAD LT 2024 ou mais novo** — AutoLISP foi adicionado ao LT em 2024
  (Windows). AutoCAD LT para **Mac não** suporta AutoLISP.
- **Python 3.10+** nativo do Windows (não é o Python do WSL)
- Gerenciador de pacotes **uv** ([guia de instalação](https://docs.astral.sh/uv/getting-started/installation/))

> O backend headless `ezdxf` roda em **qualquer** plataforma (Linux, macOS, WSL)
> sem AutoCAD instalado, para geração offline de DXF.

### 1. Clone e instale

```powershell
git clone https://github.com/puran-water/autocad-mcp.git
cd autocad-mcp
uv sync
```
### 2. Carregue o dispatcher LISP no AutoCAD LT

Abra o AutoCAD LT e carregue `lisp-code/mcp_dispatch.lsp` com **APPLOAD**:

1. Digite `APPLOAD` na linha de comando do AutoCAD.
2. Navegue até `<repo>/lisp-code/mcp_dispatch.lsp`.
3. Clique em **Load**.
4. Você deve ver: `=== MCP Dispatch v3.1 loaded ===` e
   `Ready for commands via (c:mcp-dispatch)`.

> **Dica:** adicione o arquivo ao **Startup Suite** (na caixa APPLOAD) para carregar
> automaticamente em todo desenho.

### 3. Configure seu cliente MCP

Adicione à configuração do seu cliente MCP (ex.: Claude Desktop
`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "autocad-mcp": {
      "command": "C:\\path\\to\\autocad-mcp\\.venv\\Scripts\\python.exe",
      "args": ["-m", "autocad_mcp"],
      "env": { "AUTOCAD_MCP_BACKEND": "auto" }
    }
  }
}
```

**Pontos-chave:**
- `command` deve apontar para o **Python do Windows** dentro do venv do projeto
  (não o Python do WSL).
- `AUTOCAD_MCP_BACKEND` pode ser `auto` (padrão — tenta File IPC, cai para ezdxf),
  `file_ipc` (exige AutoCAD) ou `ezdxf` (somente headless).

#### Rodando a partir do WSL

Se seu cliente MCP roda no WSL (ex.: Claude Code), lance o servidor via `cmd.exe`
para que execute como processo nativo do Windows:

```json
{
  "mcpServers": {
    "autocad-mcp": {
      "type": "stdio",
      "command": "cmd.exe",
      "args": ["/d", "/s", "/c", "cd /d C:\\path\\to\\autocad-mcp && .venv\\Scripts\\python.exe -m autocad_mcp"],
      "env": { "AUTOCAD_MCP_BACKEND": "auto" }
    }
  }
}
```

### 4. Verifique

Do seu cliente MCP, chame:

```
system(operation="status")
```

Você deve ver `backend: "file_ipc"` se o AutoCAD estiver rodando, ou
`backend: "ezdxf"` em modo headless.

---

## 🧰 Ferramentas

O servidor expõe **8 ferramentas** sobre transporte stdio MCP:

### `drawing` — Gestão de arquivos/desenhos

`create`, `open`, `info`, `save`, `save_as_dxf`, `plot_pdf`, `purge`,
`get_variables`, `undo`, `redo`

| Operação | Descrição | File IPC | ezdxf |
|----------|-----------|----------|-------|
| `create` | Cria novo desenho vazio | ✅ | ✅ |
| `open` | Abre um `.dwg` existente (FILEDIA suprimido) | ✅ | ❌ |
| `info` | Extents, contagem de ent., layers, blocks | ✅ | ✅ |
| `save` | Salva (QSAVE, ou `path` com SAVEAS) | ✅ | ✅ |
| `save_as_dxf` | Exporta como DXF | ✅ | ✅ |
| `plot_pdf` | Plota para PDF | ✅ | ❌ |
| `purge` | Limpa objetos não usados | ✅ | ❌ |
| `get_variables` | Obtém variáveis de sistema | ✅ | ❌ |
| `undo` / `redo` | Desfazer / refazer | ✅ | ❌ |

> **`create`** agora zera e limpa o desenho atual (apaga tudo + purge) em vez de
> `_.NEW`, preservando o namespace do dispatcher LISP.

### `entity` — CRUD + modificação

**Criar:** `create_line`, `create_circle`, `create_polyline`, `create_rectangle`,
`create_arc`, `create_ellipse`, `create_mtext`, `create_hatch`
**Ler:** `list`, `count`, `get`
**Modificar:** `copy`, `move`, `rotate`, `scale`, `mirror`, `offset`, `array`,
`fillet`, `chamfer`, `erase`
### `layer` — Camadas

`list`, `create`, `set_current`, `set_properties`, `freeze`, `thaw`, `lock`,
`unlock`

### `block` — Blocos e atributos

`list`, `insert`, `insert_with_attributes`, `get_attributes`, `update_attribute`,
`define`

### `annotation` — Texto, cotas e líderes

`create_text`, `create_dimension_linear`, `create_dimension_aligned`,
`create_dimension_angular`, `create_dimension_radius`, `create_leader`

### `pid` — Diagrama P&ID (biblioteca CTO)

`setup_layers`, `insert_symbol`, `list_symbols`, `draw_process_line`,
`connect_equipment`, `add_flow_arrow`, `add_equipment_tag`, `add_line_number`,
`insert_valve`, `insert_instrument`, `insert_pump`, `insert_tank`

> A biblioteca **CTO** (`src/autocad_mcp/pid/cto_library.py`) cataloga **600+
> símbolos ISA 5.1-2009** em `.dwg` por categoria (válvulas, bombas, tanques,
> instrumentos, etc.). No backend headless os símbolos passam por conversão
> DWG→DXF via `ezdxf.addons.odafc`.

### `view` — Viewport e screenshot

`zoom_extents`, `zoom_window`, `get_screenshot`

> `get_screenshot` captura a view atual do AutoCAD como PNG via `PrintWindow`
> (Win32) — funciona **mesmo com o AutoCAD minimizado**. No backend ezdxf,
> o render é via matplotlib.

### `system` — Gestão do servidor

`status`, `health`, `get_backend`, `runtime`, `init`, `execute_lisp`

---

## 🧪 `execute_lisp` — plataforma de automação ilimitada

Além das ferramentas prontas, o servidor pode rodar **qualquer AutoLISP**:

```
system(operation="execute_lisp", data={"code": "(+ 1 2)"})
```

Isso transforma o servidor de um conjunto fixo de comandos em uma plataforma
extensível — por exemplo, para acessar plugins LISP de terceiros (fiberQ/FTTH/G-PON):

```
system(operation="execute_lisp", data={"code": "(c:FQ)"})
system(operation="execute_lisp", data={"code": "(command \"_FQKABL\")"})
```

> `execute_lisp` funciona somente no backend **File IPC**.

---

## ⚙️ Variáveis de ambiente

| Variável | Padrão | Descrição |
|----------|--------|-----------|
| `AUTOCAD_MCP_BACKEND` | `auto` | Seleção: `auto`, `file_ipc`, `ezdxf` |
| `AUTOCAD_MCP_IPC_DIR` | `C:/temp` | Diretório dos arquivos IPC (deve coincidir nos lados Python e LISP) |
| `AUTOCAD_MCP_IPC_TIMEOUT` | `10.0` | Timeout do IPC em segundos (1–300) |
| `AUTOCAD_MCP_ONLY_TEXT` | `false` | Desabilita screenshots (só texto) |
| `CTO_LIBRARY_PATH` | `C:/PIDv4-CTO` | Raiz da biblioteca CTO de símbolos P&ID |

> **Importante:** se você mudar `AUTOCAD_MCP_IPC_DIR`, precisa atualizar a
> variável `*mcp-ipc-dir*` no `mcp_dispatch.lsp` para corresponder.
---

## 💻 Compatibilidade com AutoLISP no LT

AutoLISP foi adicionado ao AutoCAD LT no lançamento de **2024 (Windows)**.
AutoCAD LT para Mac não suporta AutoLISP.

| Suportado (LT 2024+ Windows) | Não suportado |
|-------------------------------|---------------|
| `.lsp` / `.fas` / `.vlx` / `.dcl` | VLIDE (Visual LISP IDE) |
| Todas as funções `vl-*` | `vlax-*` (ActiveX/COM) |
| I/O de arquivo (`open`, `read-line`, ...) | Express Tools |
| Acesso a entidades (`entget`, `entmod`, ...) | Operações 3D |
| Selection sets | AutoLISP no Mac |

O dispatcher `mcp_dispatch.lsp` é totalmente compatível com LT 2024+.

---

## 🧪 Desenvolvimento

```powershell
uv sync
uv run pytest tests/ -v
```

---

## 🆕 O que há de novo (v3.1)

- **`execute_lisp`** — executa AutoLISP arbitrário via arquivo temporário. Vira
  uma plataforma de automação extensível.
- **Undo / Redo** — passo único via ferramenta `drawing`.
- **Abrir desenho** — abre `.dwg` existentes programaticamente (FILEDIA suprimido).
- **Criar desenho** — zera e limpa o desenho atual (apaga tudo + purge), preservando
  o namespace do dispatcher.
- **Salvar com caminho** — `save` com `path` usa SAVEAS; sem path usa QSAVE.
- **Correção `get_variables`** — respeita o parâmetro `names`.
- **Correção polyline/leader** — arrays de pontos codificados em formato separado
  por ponto e vírgula.
- **Prefixo ESC** — envia 2x ESC antes de cada dispatch para cancelar comandos
  pendentes de timeouts anteriores.
- **Fallback UTF-8/cp1252** — lida com caracteres não-ASCII nos arquivos de
  resultado LISP (AutoCAD grava Windows-1252).
- **Timeout IPC configurável** — `AUTOCAD_MCP_IPC_TIMEOUT` (1–300s, padrão 10).
- **Init thread-safe** — `asyncio.Lock` evita corridas de inicialização paralela.

---

## 🙏 Agradecimentos

Este projeto é uma adaptação e expansão do servidor de código aberto
[**puran-water/autocad-mcp**](https://github.com/puran-water/autocad-mcp) (MIT).
Agradecemos aos contribuidores originais por criar a base.

## 📄 Licença

Licenciado sob a **MIT License**. Veja o arquivo [`LICENSE`](LICENSE).