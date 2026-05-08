# Manual de Instalação e Configuração — LiteLLM Gateway + LM Studio + Claude Code

**Escopo:** Gateway LLM compartilhado em rede local (sem exposição à internet), com modelos Anthropic, Gemini e modelos open source locais. Cliente: Claude Code CLI com suporte nativo a gateway via `ANTHROPIC_BASE_URL`.

---

## Sumário

1. [Visão geral da arquitetura](#1-visão-geral-da-arquitetura)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Parte 1 — Configuração do PC Servidor](#parte-1--configuração-do-pc-servidor-1015006)
   - 3.1 [Instalação e configuração do LM Studio](#31-instalação-e-configuração-do-lm-studio)
   - 3.2 [Download dos modelos open source](#32-download-dos-modelos-open-source)
   - 3.3 [Configuração do servidor local do LM Studio](#33-configuração-do-servidor-local-do-lm-studio)
   - 3.4 [Instalação do Python](#34-instalação-do-python)
   - 3.5 [Instalação do LiteLLM](#35-instalação-do-litellm)
   - 3.6 [Configuração de variáveis de ambiente no nível de sistema](#36-configuração-de-variáveis-de-ambiente-no-nível-de-sistema)
   - 3.7 [Geração da master key segura](#37-geração-da-master-key-segura)
   - 3.8 [Criação do arquivo de configuração `config.yaml`](#38-criação-do-arquivo-de-configuração-configyaml)
   - 3.9 [Teste manual antes de virar serviço](#39-teste-manual-antes-de-virar-serviço)
   - 3.10 [Configuração do Firewall do Windows](#310-configuração-do-firewall-do-windows)
   - 3.11 [Instalação como serviço Windows com NSSM](#311-instalação-como-serviço-windows-com-nssm)
   - 3.12 [Geração de virtual keys individuais para devs](#312-geração-de-virtual-keys-individuais-para-devs)
   - 3.13 [Acesso e uso do dashboard do LiteLLM](#313-acesso-e-uso-do-dashboard-do-litellm)
4. [Parte 2 — Configuração do Notebook do Dev](#parte-2--configuração-do-notebook-do-dev)
   - 4.1 [Instalação do Node.js](#41-instalação-do-nodejs)
   - 4.2 [Instalação do Claude Code CLI](#42-instalação-do-claude-code-cli)
   - 4.3 [Configuração das variáveis de ambiente do gateway](#43-configuração-das-variáveis-de-ambiente-do-gateway)
   - 4.4 [Configuração persistente no perfil do PowerShell](#44-configuração-persistente-no-perfil-do-powershell)
   - 4.5 [Instalação e configuração do OpenSpec](#45-instalação-e-configuração-do-openspec)
   - 4.6 [Teste de conexão](#46-teste-de-conexão)
   - 4.7 [Uso prático no dia a dia](#47-uso-prático-no-dia-a-dia)
5. [Troubleshooting](#5-troubleshooting)
6. [Comandos de manutenção](#6-comandos-de-manutenção)

---

## 1. Visão geral da arquitetura

```
┌─ Rede local 10.150.0.0/24 ─────────────────────────────────────────┐
│                                                                     │
│   Notebook Dev 1                    Notebook Dev 2                  │
│   Claude Code CLI + OpenSpec        Claude Code CLI + OpenSpec      │
│   ANTHROPIC_BASE_URL=:4000          ANTHROPIC_BASE_URL=:4000        │
│   ANTHROPIC_AUTH_TOKEN=sk-vkey-1    ANTHROPIC_AUTH_TOKEN=sk-vkey-2  │
│         │                                    │                      │
│         └──────────────┬─────────────────────┘                      │
│                        │ HTTP :4000                                 │
│                        ▼                                            │
│        ┌───────────────────────────────────┐                        │
│        │  PC Servidor — 10.150.0.69        │                        │
│        │                                   │                        │
│        │   ┌───────────────────────────┐   │                        │
│        │   │ LiteLLM Gateway :4000     │   │                        │
│        │   │ (ANTHROPIC_API_KEY,       │   │                        │
│        │   │  GEMINI_API_KEY internos) │   │                        │
│        │   └──────┬──────────┬─────────┘   │                        │
│        │          │          │             │                        │
│        │          │          ▼             │                        │
│        │          │  ┌────────────────┐    │                        │
│        │          │  │ LM Studio :1234│    │                        │
│        │          │  │ (127.0.0.1)    │    │                        │
│        │          │  └────────────────┘    │                        │
│        └──────────┼────────────────────────┘                        │
└──────────────────-┼─────────────────────────────────────────────────┘
                    │ HTTPS
                    ▼
       ┌──────────────────────────┐
       │ APIs externas            │
       │ • Anthropic              │
       │ • Google Gemini          │
       └──────────────────────────┘
```

**Stack completa por camada:**

| Camada | Componente | Onde roda |
|---|---|---|
| **Cliente** | Claude Code CLI + OpenSpec | Notebook do dev |
| **Gateway** | LiteLLM (NSSM service) | PC Servidor :4000 |
| **Runtime local** | LM Studio | PC Servidor :1234 (localhost only) |
| **APIs externas** | Anthropic, Gemini | Internet (HTTPS) |

**Princípios de segurança aplicados:**

- API keys reais (Anthropic, Gemini) ficam **apenas no PC Servidor**, nunca distribuídas
- Cada dev recebe uma **virtual key individual** com permissões e budget próprios
- Claude Code usa `ANTHROPIC_AUTH_TOKEN` (virtual key) — **nunca** a API key real da Anthropic
- Porta 4000 acessível **apenas na LAN** (firewall restringe ao range `10.150.0.0/24`)
- **Sem exposição à internet** — sem port forwarding no roteador
- LM Studio restrito a `127.0.0.1` — acesso local passa obrigatoriamente pelo LiteLLM

---

## 2. Pré-requisitos

**No PC Servidor (10.150.0.69):**

- Windows 11 Pro instalado e atualizado
- Acesso administrativo (instalação de programas e configuração de firewall)
- Conexão com a internet (para downloads e chamadas às APIs Anthropic/Gemini)
- Mínimo 16 GB de RAM, recomendado 24 GB+ (para carregar modelos open source)
- ~30 GB livres em disco (modelos quantizados ocupam de 6 a 13 GB cada)
- IP fixo `10.150.0.69` configurado na placa de rede

**Nos notebooks dos devs:**

- Windows, macOS ou Linux
- Acesso à mesma rede local (range `10.150.0.0/24`)
- Node.js 20.19.0 ou superior (requisito do Claude Code e do OpenSpec CLI)
- Permissão para ler/escrever em `~/.claude/`

---

# Parte 1 — Configuração do PC Servidor (10.150.0.69)

## 3.1 Instalação e configuração do LM Studio

1. Acesse https://lmstudio.ai/ e baixe a versão mais recente para Windows
2. Execute o instalador como administrador
3. Aceite as configurações padrão durante a instalação
4. Após a primeira execução, vá em **Settings → General**:
   - Marque **"Run LM Studio at startup"** (importante: garante que o servidor de modelos esteja disponível após reboot)
   - Marque **"Start minimized to system tray"**
5. Vá em **Settings → Developer**:
   - Marque **"Auto-start LLM server on app launch"**
   - Em **"Server port"** mantenha `1234`
   - Em **"Network"** mantenha **"Serve on local network: OFF"** (queremos que o servidor responda apenas no `127.0.0.1`, já que o LiteLLM rodará no mesmo PC)

> ⚠️ **Importante:** Não habilite "Serve on local network" no LM Studio. O acesso da rede local deve passar **somente** pelo LiteLLM, que controla autenticação. O LM Studio fica restrito ao localhost.

## 3.2 Download dos modelos open source

Na aba **Discover** (lupa) do LM Studio, baixe os três modelos quantizados:

| Modelo | Quantização | Tamanho | Identificador para busca |
|---|---|---|---|
| DeepSeek-Coder-V2-Lite | Q4_K_M | ~10 GB | `deepseek-coder-v2-lite-instruct GGUF` (versão lmstudio-community) |
| Qwen2.5-Coder 14B | Q4_K_M | ~9 GB | `qwen2.5-coder-14b-instruct GGUF` (versão lmstudio-community) |
| Qwen2.5-Coder 7B | Q6_K | ~6 GB | `qwen2.5-coder-7b-instruct GGUF` (versão lmstudio-community) |

Para cada modelo:

1. Pesquise pelo nome
2. Selecione o arquivo com a quantização correta (ex: `Q4_K_M` ou `Q6_K`)
3. Clique em **Download**
4. Anote o **identificador exato** que aparece no LM Studio após o download (você precisará dele no `config.yaml`). Em geral é algo como:
   - `deepseek-coder-v2-lite-instruct`
   - `qwen2.5-coder-14b-instruct`
   - `qwen2.5-coder-7b-instruct`

> 💡 **Dica:** Para descobrir o identificador exato, vá na aba **My Models** do LM Studio. O nome listado ali é o que o servidor expõe na API.

## 3.3 Configuração do servidor local do LM Studio

1. Vá na aba **Developer** (ícone `</>`)
2. Clique em **Start Server** (deve ficar verde, indicando "Running on port 1234")
3. Em **Server Options**:
   - **Server Port:** `1234`
   - **JIT (Just-In-Time) Loading:** habilitado (carrega o modelo sob demanda quando o LiteLLM faz uma requisição)
   - **Auto-Unload:** habilitado (descarrega após 10 minutos de inatividade — economiza RAM)
   - **CORS:** desabilitado (não precisamos)
4. Verifique no PowerShell:
   ```powershell
   curl http://localhost:1234/v1/models
   ```
   Deve retornar uma lista JSON com os três modelos baixados.

## 3.4 Instalação do Python

O LiteLLM é uma aplicação Python.

1. Acesse https://www.python.org/downloads/windows/ e baixe a **versão estável mais recente** do Python 3.12 ou 3.13 (instalador 64-bit)
2. Execute o instalador como administrador
3. **Marque obrigatoriamente** as opções:
   - ✅ **"Add python.exe to PATH"**
   - ✅ **"Install for all users"**
4. Clique em **Customize installation** e mantenha todas as opções padrão
5. Após a instalação, abra um **novo** PowerShell e verifique:
   ```powershell
   python --version
   pip --version
   ```
   Ambos devem retornar as versões instaladas.

## 3.5 Instalação do LiteLLM

Em PowerShell **como administrador**:

```powershell
pip install --upgrade pip
pip install "litellm[proxy]"
```

A flag `[proxy]` instala dependências adicionais (FastAPI, uvicorn, SQLAlchemy etc.) necessárias para o modo gateway.

Verificar instalação:

```powershell
litellm --version
```

Deve exibir a versão instalada (algo como `1.84.x` ou superior).

## 3.6 Configuração de variáveis de ambiente no nível de sistema

As API keys da Anthropic e Gemini precisam estar acessíveis ao **serviço Windows** que rodará o LiteLLM, então devem ser definidas como variáveis **de sistema**, não de usuário.

**Caminho gráfico:**

1. Tecle `Win + R`, digite `sysdm.cpl` e pressione Enter
2. Aba **Avançado** → botão **Variáveis de Ambiente**
3. Na seção **Variáveis de sistema** (parte de baixo, **NÃO** a de cima), clique em **Novo**
4. Crie cada variável:

| Nome da variável | Valor |
|---|---|
| `ANTHROPIC_API_KEY` | `sk-ant-api03-xxxxxxxx...` (sua chave Anthropic) |
| `GEMINI_API_KEY` | `AIzaSy...` (sua chave Google AI Studio) |

**Caminho via PowerShell (alternativa, como administrador):**

```powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-ant-api03-xxxxxxxx", "Machine")
[Environment]::SetEnvironmentVariable("GEMINI_API_KEY", "AIzaSy...", "Machine")
```

**Verificação (em um novo PowerShell):**

```powershell
[Environment]::GetEnvironmentVariable("ANTHROPIC_API_KEY", "Machine")
[Environment]::GetEnvironmentVariable("GEMINI_API_KEY", "Machine")
```

> ⚠️ **Importante:** Após criar as variáveis de sistema, **feche e reabra todos os PowerShells** para que elas sejam carregadas.

## 3.7 Geração da master key segura

A master key é a credencial **administrativa** do gateway. Ela permite gerar virtual keys, acessar o dashboard administrativo e fazer qualquer operação. **Nunca** distribua para os devs.

**Geração via Python (recomendado — usa CSPRNG do sistema):**

```powershell
python -c "import secrets; print('sk-' + secrets.token_urlsafe(48))"
```

Saída exemplo:

```
sk-aB3xZ9-Kp_qN7vR4tY2mFw8GhJlE6sCnD0bV5oXuI1zPyT-rQmHaSwLkUvN
```

**Geração nativa via PowerShell (alternativa):**

```powershell
$bytes = New-Object byte[] 48
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
"sk-" + [Convert]::ToBase64String($bytes).Replace("+","-").Replace("/","_").Replace("=","")
```

**Boas práticas:**

- Anote a master key em um gerenciador de senhas (Bitwarden, 1Password, KeePass)
- Nunca commite ela em repositórios Git
- Nunca envie por WhatsApp/email sem criptografia
- Substitua todos os locais no `config.yaml` antes de subir o serviço

Vamos chamá-la daqui em diante de `<MASTER_KEY>`.

Adicione também como variável de sistema (para o serviço lê-la):

```powershell
[Environment]::SetEnvironmentVariable("LITELLM_MASTER_KEY", "sk-suaMasterKeyAqui...", "Machine")
```

## 3.8 Criação do arquivo de configuração `config.yaml`

Crie a pasta de trabalho do LiteLLM:

```powershell
New-Item -ItemType Directory -Path "C:\litellm" -Force
cd C:\litellm
```

Crie o arquivo `C:\litellm\config.yaml` com o conteúdo abaixo (use Notepad++, VS Code, ou outro editor que respeite UTF-8 sem BOM):

```yaml
# ===== Modelos disponíveis =====
model_list:

  # --- Anthropic ---
  - model_name: claude-opus
    litellm_params:
      model: anthropic/claude-opus-4-7
      api_key: os.environ/ANTHROPIC_API_KEY

  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4-6
      api_key: os.environ/ANTHROPIC_API_KEY

  # --- Google Gemini ---
  - model_name: gemini-3-1-pro
    litellm_params:
      model: gemini/gemini-3.1-pro-preview
      api_key: os.environ/GEMINI_API_KEY

  - model_name: gemini-2-5-pro
    litellm_params:
      model: gemini/gemini-2.5-pro
      api_key: os.environ/GEMINI_API_KEY

  - model_name: gemini-2-5-flash
    litellm_params:
      model: gemini/gemini-2.5-flash
      api_key: os.environ/GEMINI_API_KEY

  # --- Modelos locais via LM Studio ---
  - model_name: deepseek-local
    litellm_params:
      model: lm_studio/deepseek-coder-v2-lite-instruct
      api_base: http://localhost:1234/v1
      api_key: "lm-studio"

  - model_name: qwen-14b-local
    litellm_params:
      model: lm_studio/qwen2.5-coder-14b-instruct
      api_base: http://localhost:1234/v1
      api_key: "lm-studio"

  - model_name: qwen-7b-local
    litellm_params:
      model: lm_studio/qwen2.5-coder-7b-instruct
      api_base: http://localhost:1234/v1
      api_key: "lm-studio"

# ===== Configurações gerais =====
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: "sqlite:///C:/litellm/litellm.db"
  store_model_in_db: true

# ===== Configurações do LiteLLM =====
litellm_settings:
  drop_params: true
  fallbacks:
    - claude-opus: ["claude-sonnet", "gemini-3-1-pro"]
    - claude-sonnet: ["gemini-3-1-pro", "qwen-14b-local"]
    - gemini-3-1-pro: ["gemini-2-5-pro", "claude-sonnet"]

# ===== Configurações de roteamento =====
router_settings:
  num_retries: 2
  timeout: 600
  routing_strategy: simple-shuffle
```

> 📝 **Notas sobre o arquivo:**
>
> - Os identificadores em `model_name` são os nomes que os devs usarão (ex: `claude-sonnet`)
> - Os identificadores em `model:` são os nomes reais dos modelos nos providers
> - **Confirme os nomes dos modelos locais** no LM Studio (aba **My Models**) e ajuste se necessário
> - O fallback automático: se Claude Opus falhar, tenta Sonnet, depois Gemini 3.1 Pro
> - SQLite como banco é suficiente para equipes pequenas (até ~10 devs); acima disso, considere PostgreSQL

## 3.9 Teste manual antes de virar serviço

Antes de instalar como serviço, valide manualmente.

**Em um PowerShell**, no diretório `C:\litellm`:

```powershell
cd C:\litellm
litellm --config config.yaml --host 0.0.0.0 --port 4000
```

Você deve ver logs do tipo:

```
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:4000
```

**Em outro PowerShell**, teste:

```powershell
$headers = @{
  "Authorization" = "Bearer $env:LITELLM_MASTER_KEY"
  "Content-Type" = "application/json"
}
$body = @{
  model = "claude-sonnet"
  messages = @(@{role="user"; content="Diga 'oi' em português"})
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "http://localhost:4000/v1/chat/completions" `
  -Method Post -Headers $headers -Body $body
```

Se retornar uma resposta JSON com o texto do Claude, está funcionando. Pare o servidor com `Ctrl+C` no primeiro PowerShell e prossiga.

## 3.10 Configuração do Firewall do Windows

Permitir conexões na porta 4000 **somente** vindas da rede local.

**PowerShell como administrador:**

```powershell
New-NetFirewallRule `
  -DisplayName "LiteLLM Gateway (LAN only)" `
  -Direction Inbound `
  -LocalPort 4000 `
  -Protocol TCP `
  -Action Allow `
  -Profile Private,Domain `
  -RemoteAddress 10.150.0.0/24
```

**Explicação dos parâmetros:**

- `-Profile Private,Domain` — regra ativa apenas em redes classificadas como "Privada" ou "Domínio" (nunca em redes "Públicas")
- `-RemoteAddress 10.150.0.0/24` — aceita conexões **somente** da sua sub-rede `10.150.0.0` até `10.150.0.255` (ajuste se sua máscara for diferente)

> ⚠️ **Verificação crítica:** Confirme que sua interface de rede está classificada como "Privada":
> ```powershell
> Get-NetConnectionProfile
> ```
> Se aparecer "Public", mude para "Private":
> ```powershell
> Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
> ```

**Para verificar a regra criada:**

```powershell
Get-NetFirewallRule -DisplayName "LiteLLM Gateway (LAN only)"
```

## 3.11 Instalação como serviço Windows com NSSM

NSSM (Non-Sucking Service Manager) é a ferramenta padrão para transformar qualquer executável em serviço Windows.

### Instalação do NSSM

1. Baixe NSSM em https://nssm.cc/download (versão `nssm 2.24` ou superior)
2. Extraia o ZIP
3. Copie `nssm.exe` (da pasta `win64`) para `C:\nssm\nssm.exe`
4. Adicione `C:\nssm` ao PATH do sistema:
   ```powershell
   [Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\nssm", "Machine")
   ```
5. Feche e reabra o PowerShell, valide:
   ```powershell
   nssm --version
   ```

### Criação do serviço LiteLLM

**PowerShell como administrador:**

```powershell
# Descobrir caminho exato do executável litellm
$litellmPath = (Get-Command litellm).Source
Write-Host "LiteLLM em: $litellmPath"

# Instalar serviço
nssm install LiteLLMGateway $litellmPath
```

Vai abrir uma janela GUI do NSSM. Configure as abas:

**Aba "Application":**

- **Path:** preenchido automaticamente (ex: `C:\Python312\Scripts\litellm.exe`)
- **Startup directory:** `C:\litellm`
- **Arguments:** `--config config.yaml --host 0.0.0.0 --port 4000`

**Aba "Details":**

- **Display name:** `LiteLLM Gateway`
- **Description:** `LLM proxy gateway compartilhado em rede local`
- **Startup type:** `Automatic`

**Aba "Log on":**

- **Log on as:** `Local System account` (deixe marcado)

**Aba "I/O":**

- **Output (stdout):** `C:\litellm\logs\stdout.log`
- **Error (stderr):** `C:\litellm\logs\stderr.log`

Crie a pasta de logs antes:

```powershell
New-Item -ItemType Directory -Path "C:\litellm\logs" -Force
```

**Aba "File rotation":**

- Marque **"Rotate files"**
- **Rotate while service is running:** marque
- **Restrict rotation to files bigger than:** `10485760` (10 MB)
- **Restrict rotation to files older than:** `86400` (1 dia)

**Aba "Environment":**

Adicione cada variável (uma por linha) no formato `NOME=VALOR`:

```
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxx
GEMINI_API_KEY=AIzaSy...
LITELLM_MASTER_KEY=sk-suaMasterKey...
```

> 💡 Mesmo já estando no nível de sistema, definir aqui garante que o serviço sempre as enxergue, independente do contexto de execução.

Clique em **Install service**.

### Iniciar o serviço

```powershell
Start-Service LiteLLMGateway
Get-Service LiteLLMGateway
```

Status deve ser `Running`. Verifique logs:

```powershell
Get-Content C:\litellm\logs\stdout.log -Tail 30 -Wait
```

Você verá os mesmos logs de inicialização do teste manual.

### Comandos úteis do serviço

```powershell
# Parar
Stop-Service LiteLLMGateway

# Iniciar
Start-Service LiteLLMGateway

# Reiniciar
Restart-Service LiteLLMGateway

# Status
Get-Service LiteLLMGateway

# Reinstalar (se mudou config)
nssm restart LiteLLMGateway

# Remover serviço
nssm remove LiteLLMGateway confirm
```

## 3.12 Geração de virtual keys individuais para devs

Cada dev recebe uma chave única, com permissões e budget próprios.

**Pré-requisito:** o serviço `LiteLLMGateway` deve estar rodando.

### Criação via curl/PowerShell

Para criar uma chave para o dev "joao" com acesso a Sonnet, Gemini 3.1 Pro e Qwen 14B local, com teto de US$ 50/mês:

```powershell
$masterKey = $env:LITELLM_MASTER_KEY

$headers = @{
  "Authorization" = "Bearer $masterKey"
  "Content-Type" = "application/json"
}

$body = @{
  models = @("claude-sonnet", "gemini-3-1-pro", "gemini-2-5-flash", "qwen-14b-local")
  max_budget = 50.0
  budget_duration = "30d"
  user_id = "joao.silva"
  key_alias = "joao-dev-laptop"
  metadata = @{
    project = "nt-usina"
    role = "developer"
  }
} | ConvertTo-Json -Depth 5

$response = Invoke-RestMethod -Uri "http://localhost:4000/key/generate" `
  -Method Post -Headers $headers -Body $body

Write-Host "Virtual Key: $($response.key)"
Write-Host "Key ID: $($response.key_name)"
```

Saída exemplo:

```
Virtual Key: sk-9KvP3xZ_qN7vR4tY2mFw8GhJ...
Key ID: sk-9KvP3xZ...
```

**Entregue ao dev `joao.silva`:**

- A chave virtual `sk-9KvP3xZ_qN7vR4tY2mFw8GhJ...`
- O endpoint: `http://10.150.0.69:4000`
- A lista de modelos disponíveis para ele

### Perfis sugeridos por papel

**Dev Junior** (acesso restrito a modelos baratos + locais, budget baixo):

```powershell
$body = @{
  models = @("claude-sonnet", "gemini-2-5-flash", "qwen-7b-local", "qwen-14b-local")
  max_budget = 25.0
  budget_duration = "30d"
  user_id = "junior.dev"
  key_alias = "junior-laptop"
} | ConvertTo-Json -Depth 5
```

**Dev Senior** (acesso completo, budget maior):

```powershell
$body = @{
  models = @("claude-opus", "claude-sonnet", "gemini-3-1-pro", "gemini-2-5-pro", "gemini-2-5-flash", "deepseek-local", "qwen-14b-local", "qwen-7b-local")
  max_budget = 150.0
  budget_duration = "30d"
  user_id = "senior.dev"
  key_alias = "senior-laptop"
} | ConvertTo-Json -Depth 5
```

**Privacy/SERIN** (apenas modelos locais, sem budget pois é gratuito):

```powershell
$body = @{
  models = @("deepseek-local", "qwen-14b-local", "qwen-7b-local")
  user_id = "serin.dev"
  key_alias = "serin-projeto-confidencial"
} | ConvertTo-Json -Depth 5
```

### Operações em chaves existentes

**Listar todas as chaves:**

```powershell
Invoke-RestMethod -Uri "http://localhost:4000/key/info" -Headers $headers
```

**Atualizar uma chave (ex: aumentar budget):**

```powershell
$updateBody = @{
  key = "sk-9KvP3xZ..."
  max_budget = 100.0
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:4000/key/update" `
  -Method Post -Headers $headers -Body $updateBody
```

**Revogar/deletar uma chave (ex: dev saiu da empresa):**

```powershell
$deleteBody = @{
  keys = @("sk-9KvP3xZ...")
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:4000/key/delete" `
  -Method Post -Headers $headers -Body $deleteBody
```

## 3.13 Acesso e uso do dashboard do LiteLLM

O LiteLLM oferece um dashboard web para gerenciar chaves, monitorar gastos e visualizar logs.

### Acesso

A partir de **qualquer máquina na rede local 10.150.0.0/24**:

```
http://10.150.0.69:4000/ui
```

A partir do próprio servidor:

```
http://localhost:4000/ui
```

### Login

- **Username:** `admin`
- **Password:** sua **master key** (`<MASTER_KEY>` definida em 3.7)

### Recursos disponíveis

**Tab "Virtual Keys":**

- Listar todas as chaves criadas
- Criar novas chaves via interface gráfica (alternativa ao curl)
- Editar permissões e budgets
- Revogar chaves
- Ver gasto acumulado por chave

**Tab "Usage":**

- Gráficos de uso por modelo, por dia/semana/mês
- Custo total acumulado (em USD)
- Distribuição de chamadas por dev (`user_id`)
- Identificação de outliers (chaves que estão gastando muito)

**Tab "Models":**

- Lista todos os modelos configurados no `config.yaml`
- Status de health check por modelo
- Latência média e taxa de erro

**Tab "Logs":**

- Histórico de requisições (input, output, modelo, custo, latência)
- Filtros por chave, modelo, período
- **Atenção:** logs contêm prompts e respostas completas — útil para debug, mas considere LGPD se houver dados sensíveis

**Tab "Settings":**

- Adicionar/remover modelos sem editar `config.yaml` (gravado no SQLite)
- Configurar webhooks para alertas de budget

### Recomendações de uso

- **Revisar gastos semanalmente** na aba Usage para identificar tendências
- **Configurar alertas de budget** quando uma chave atinge 80% do teto
- **Auditar logs mensalmente** para verificar uso indevido
- **Backup periódico do `litellm.db`** (contém todas as chaves e histórico de gastos):

  ```powershell
  Copy-Item C:\litellm\litellm.db "C:\litellm\backups\litellm-$(Get-Date -Format 'yyyyMMdd').db"
  ```

---

# Parte 2 — Configuração do Notebook do Dev

Esta seção é o que você entrega para cada desenvolvedor. O único pré-requisito é Node.js — não é necessário Python no notebook.

> 💡 **Como funciona a integração:** O Claude Code tem suporte oficial a LLM gateways. Basta definir `ANTHROPIC_BASE_URL` apontando para o LiteLLM e `ANTHROPIC_AUTH_TOKEN` com a virtual key do dev. O Claude Code continua funcionando exatamente como sempre — incluindo todos os comandos `/opsx:*` do OpenSpec — sem saber que há um gateway no meio.

## 4.1 Instalação do Node.js

O Claude Code CLI e o OpenSpec CLI exigem Node.js 20.19.0 ou superior.

1. Acesse https://nodejs.org/ e baixe a versão **LTS** (Long Term Support) mais recente
2. Execute o instalador marcando **"Add to PATH"** e **"Install necessary tools"**
3. Verifique em um **novo** PowerShell:
   ```powershell
   node --version   # deve ser v20.x ou superior
   npm --version
   ```

## 4.2 Instalação do Claude Code CLI

```powershell
npm install -g @anthropic-ai/claude-code
```

Verificar:

```powershell
claude --version
```

> ⚠️ **Não configure `ANTHROPIC_API_KEY` no notebook.** O dev nunca precisa da API key real da Anthropic — ele usa exclusivamente a virtual key do LiteLLM via `ANTHROPIC_AUTH_TOKEN`. Se a variável `ANTHROPIC_API_KEY` existir no ambiente do dev, o Claude Code tentará ir direto para a Anthropic, ignorando o gateway.

## 4.3 Configuração das variáveis de ambiente do gateway

Estas são as duas variáveis que redirecionam o Claude Code para o gateway LiteLLM.

**Via PowerShell — nível de usuário** (cada dev tem suas próprias, nunca compartilhadas):

```powershell
# Endereço do gateway (igual para todos os devs)
[Environment]::SetEnvironmentVariable(
  "ANTHROPIC_BASE_URL",
  "http://10.150.0.69:4000",
  "User"
)

# Virtual key individual (diferente para cada dev — recebida do administrador)
[Environment]::SetEnvironmentVariable(
  "ANTHROPIC_AUTH_TOKEN",
  "sk-suaVirtualKeyIndividual...",
  "User"
)
```

> 📝 **Por que nível de usuário?** Cada dev tem uma virtual key diferente. Definir no nível "User" garante que cada conta do Windows tenha suas próprias credenciais, sem interferência entre devs que compartilhem a máquina.

**Verificação** (em um novo PowerShell):

```powershell
[Environment]::GetEnvironmentVariable("ANTHROPIC_BASE_URL", "User")
[Environment]::GetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", "User")
```

Ambas devem retornar os valores definidos.

## 4.4 Configuração persistente no perfil do PowerShell

Para que as variáveis sejam carregadas automaticamente em toda nova sessão do PowerShell:

```powershell
# Abrir (ou criar) o perfil do PowerShell
if (!(Test-Path $PROFILE)) { New-Item -Path $PROFILE -Force }
notepad $PROFILE
```

Adicione ao final do arquivo:

```powershell
# ===== LiteLLM Gateway — Natural Tecnologia =====
$env:ANTHROPIC_BASE_URL  = "http://10.150.0.69:4000"
$env:ANTHROPIC_AUTH_TOKEN = "sk-suaVirtualKeyIndividual..."

# Modelo padrão (pode ser sobrescrito por projeto)
$env:ANTHROPIC_MODEL = "claude-sonnet"

# Aliases de conveniência para trocar de modelo rapidamente
function Use-ClaudeOpus   { $env:ANTHROPIC_MODEL = "claude-opus";      Write-Host "Modelo: claude-opus" }
function Use-ClaudeSonnet { $env:ANTHROPIC_MODEL = "claude-sonnet";    Write-Host "Modelo: claude-sonnet" }
function Use-Gemini25Pro  { $env:ANTHROPIC_MODEL = "gemini-2-5-pro";   Write-Host "Modelo: gemini-2-5-pro" }
function Use-Gemini25Flash{ $env:ANTHROPIC_MODEL = "gemini-2-5-flash"; Write-Host "Modelo: gemini-2-5-flash" }
function Use-Qwen14B      { $env:ANTHROPIC_MODEL = "qwen-14b-local";   Write-Host "Modelo: qwen-14b-local (local)" }
function Use-QwenSerin    { $env:ANTHROPIC_MODEL = "deepseek-local";   Write-Host "Modelo: deepseek-local (local/privado)" }
# ================================================
```

Salve e feche o Notepad. Recarregue o perfil:

```powershell
. $PROFILE
```

## 4.5 Instalação e configuração do OpenSpec

O OpenSpec é o framework de Spec-Driven Development (SDD) que integra nativamente com o Claude Code.

### Instalação global do CLI

```powershell
npm install -g @fission-ai/openspec
openspec --version
```

### Inicialização em cada projeto

Dentro da pasta do projeto:

```powershell
cd C:\projetos\meu-projeto
openspec init
```

O assistente interativo vai perguntar:

- **Tools:** selecione `claude` (Claude Code) — outros são opcionais
- **Profile:** `core` para começar (inclui `/opsx:propose`, `/opsx:apply`, `/opsx:archive`, `/opsx:explore`, `/opsx:sync`)
- **Delivery:** `both` (gera tanto skills quanto commands)

Isso cria a estrutura:

```
meu-projeto/
├── openspec/
│   ├── config.yaml        # configuração do projeto (stack, contexto, regras)
│   ├── specs/             # specs consolidadas (source of truth)
│   └── changes/           # mudanças em andamento
│       └── archive/       # mudanças concluídas
└── .claude/
    ├── skills/            # auto-detectado pelo Claude Code
    └── commands/opsx/     # slash commands /opsx:*
```

### Configuração do `openspec/config.yaml`

Edite para refletir o stack da Natural Tecnologia:

```yaml
schema: spec-driven

context: |
  Stack: Laravel, PostgreSQL, Vue.js, GCP
  Linguagem de código: português brasileiro (variáveis, funções, comentários)
  Padrão de branches: feature/*, develop, main
  Merge: --no-ff obrigatório
  Testes: PHPUnit (Laravel), Vitest (Vue)

rules:
  proposal:
    - Incluir impacto em módulos existentes
    - Identificar tabelas do PostgreSQL afetadas
  specs:
    - Usar formato Dado/Quando/Então para cenários
  design:
    - Incluir diagrama de sequência para fluxos com 3+ etapas
  tasks:
    - Tarefas atômicas (máximo 2h cada)
    - Incluir critério de aceite por tarefa
```

### Ativar perfil expandido (opcional)

Para comandos adicionais (`/opsx:new`, `/opsx:continue`, `/opsx:ff`, `/opsx:verify`, `/opsx:bulk-archive`, `/opsx:onboard`):

```powershell
openspec config profile   # selecione "workflows"
openspec update           # regenera os arquivos de comandos
```

## 4.6 Teste de conexão

### Teste direto do gateway (sem Claude Code)

```powershell
$headers = @{
  "Authorization" = "Bearer $env:ANTHROPIC_AUTH_TOKEN"
  "Content-Type"  = "application/json"
}
$body = @{
  model    = "claude-sonnet"
  messages = @(@{role="user"; content="Responda apenas 'Gateway OK' em português"})
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "http://10.150.0.69:4000/v1/chat/completions" `
  -Method Post -Headers $headers -Body $body
```

Deve retornar JSON com `"Gateway OK"`.

### Teste com Claude Code apontando para o gateway

```powershell
claude --print "Diga apenas 'Claude Code via LiteLLM funcionando' em português"
```

A resposta deve vir do Claude via gateway. Verifique nos logs do servidor (`C:\litellm\logs\stdout.log`) se a requisição aparece — isso confirma que o tráfego está passando pelo LiteLLM.

### Verificar modelos disponíveis

```powershell
# Lista todos os modelos que a virtual key do dev pode usar
Invoke-RestMethod -Uri "http://10.150.0.69:4000/v1/models" `
  -Headers @{ "Authorization" = "Bearer $env:ANTHROPIC_AUTH_TOKEN" }
```

## 4.7 Uso prático no dia a dia

### Iniciar uma sessão de desenvolvimento

```powershell
cd C:\projetos\meu-projeto

# Subir o Claude Code (usa ANTHROPIC_BASE_URL e ANTHROPIC_AUTH_TOKEN do perfil)
claude
```

### Workflow OpenSpec completo

Dentro do Claude Code, use os comandos `/opsx:*` normalmente:

```
# 1. Explorar a ideia antes de propor
/opsx:explore adicionar autenticação por CPF no módulo de cadastro

# 2. Criar a proposta com todos os artifacts
/opsx:propose autenticacao-cpf

# 3. Revisar os artifacts gerados:
#    openspec/changes/autenticacao-cpf/proposal.md
#    openspec/changes/autenticacao-cpf/specs/
#    openspec/changes/autenticacao-cpf/design.md
#    openspec/changes/autenticacao-cpf/tasks.md

# 4. Implementar
/opsx:apply

# 5. Sincronizar specs se houve mudanças durante implementação
/opsx:sync

# 6. Arquivar ao concluir
/opsx:archive
```

### Trocar de modelo durante a sessão

```powershell
# Via alias do perfil (fora do Claude Code)
Use-ClaudeOpus       # para tarefas de raciocínio profundo
Use-Gemini25Flash    # para tarefas rápidas e baratas
Use-Qwen14B          # para código com dados sensíveis (local, sem cloud)
Use-QwenSerin        # para projetos SERIN/governo (100% local)

# Reinicia o Claude Code com o novo modelo
claude
```

Ou troque inline passando a flag:

```powershell
claude --model gemini-2-5-pro    # Gemini via LiteLLM
claude --model qwen-14b-local    # modelo local via LM Studio
claude --model deepseek-local    # DeepSeek local para dados sensíveis
```

### Verificar gasto da virtual key

```powershell
$headers = @{ "Authorization" = "Bearer $env:ANTHROPIC_AUTH_TOKEN" }
$info = Invoke-RestMethod -Uri "http://10.150.0.69:4000/key/info" -Headers $headers
Write-Host "Gasto: `$$($info.info.spend) / `$$($info.info.max_budget)"
```

### Projeto SERIN / dados sensíveis de governo

Para projetos onde nenhum dado pode sair da rede local, use exclusivamente modelos locais:

```powershell
# Forçar modelo local para toda a sessão
$env:ANTHROPIC_MODEL = "deepseek-local"
claude
```

O LiteLLM roteia para o LM Studio (`127.0.0.1:1234`) — os prompts e respostas nunca saem do PC servidor.

---

## 5. Troubleshooting

### Serviço LiteLLM não inicia

**Sintoma:** `Start-Service LiteLLMGateway` falha.

**Diagnóstico:**

```powershell
Get-EventLog -LogName Application -Source "LiteLLMGateway" -Newest 10
Get-Content C:\litellm\logs\stderr.log -Tail 50
```

**Causas comuns:**

- Variáveis de ambiente não definidas no nível de sistema → revisar 3.6
- Caminho do `litellm.exe` errado no NSSM → reconfigurar via `nssm edit LiteLLMGateway`
- Porta 4000 já em uso por outro processo:
  ```powershell
  netstat -ano | findstr :4000
  ```

### Dev não consegue conectar

**Sintoma:** `Connection refused` ou `Timeout` ao rodar `claude`.

**Checklist:**

1. Servidor está respondendo no localhost?
   ```powershell
   # No PC servidor
   curl http://localhost:4000/health/liveliness
   ```

2. Firewall está permitindo a conexão?
   ```powershell
   Get-NetFirewallRule -DisplayName "LiteLLM Gateway (LAN only)" | Format-List
   ```

3. Dev consegue alcançar a porta do servidor?
   ```powershell
   # No notebook do dev
   Test-NetConnection -ComputerName 10.150.0.69 -Port 4000
   ```

   `TcpTestSucceeded : True` → ok. Se `False`, é firewall ou IP errado.

4. IP do servidor mudou? Reservar IP no DHCP ou configurar IP fixo.

### Claude Code não usa o gateway

**Sintoma:** Claude Code conecta direto na Anthropic, ignorando o LiteLLM.

**Causas:**

- `ANTHROPIC_BASE_URL` não está definida no nível de usuário, ou PowerShell foi aberto antes de definir a variável:
  ```powershell
  echo $env:ANTHROPIC_BASE_URL   # deve ser http://10.150.0.69:4000
  ```
- Existe `ANTHROPIC_API_KEY` definida no ambiente — o Claude Code a usa como fallback e bypassa o gateway. **Remova-a** do notebook do dev:
  ```powershell
  [Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", $null, "User")
  [Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", $null, "Machine")
  ```
- O perfil do PowerShell não foi recarregado após edição — execute `. $PROFILE`

### Erro 401 Unauthorized

**Sintoma:** `{"error": "Authentication Error"}` ao rodar `claude`.

**Causas:**

- `ANTHROPIC_AUTH_TOKEN` com virtual key errada ou expirada → verificar com:
  ```powershell
  echo $env:ANTHROPIC_AUTH_TOKEN
  ```
- Virtual key revogada → solicitar nova ao administrador via seção 3.12
- Tentando usar modelo que a chave não tem permissão → administrador deve atualizar permissões da chave

### Erro 429 Rate Limit

Vem do provider (Anthropic/Gemini), não do LiteLLM. Soluções:

- Aguardar (rate limit típico expira em 1 minuto)
- O LiteLLM aciona o fallback automático definido no `config.yaml` — troca de modelo sozinho
- Trocar manualmente para modelo alternativo: `claude --model gemini-2-5-pro`

### Modelo local não responde

**Sintoma:** `qwen-14b-local` ou `deepseek-local` retornam erro.

1. LM Studio está rodando? Verifique o ícone na bandeja
2. Servidor do LM Studio está ativo?
   ```powershell
   curl http://localhost:1234/v1/models
   ```
3. O nome do modelo no `config.yaml` bate com o do LM Studio? Confira em **My Models**

### Resposta em idioma errado (Deepseek)

Adicione system prompt fixo na configuração do modelo no `config.yaml`:

```yaml
- model_name: deepseek-local
  litellm_params:
    model: lm_studio/deepseek-coder-v2-lite-instruct
    api_base: http://localhost:1234/v1
    api_key: "lm-studio"
    temperature: 0.2
    extra_body:
      system: "Always respond in Brazilian Portuguese (pt-BR). Never respond in Chinese or English unless explicitly asked."
```

### Comandos `/opsx:*` não aparecem no Claude Code

**Sintoma:** Claude Code não reconhece `/opsx:propose` ou outros comandos do OpenSpec.

1. OpenSpec foi inicializado no projeto? Verifique se existe `.claude/commands/opsx/` na raiz do projeto:
   ```powershell
   ls .claude\commands\opsx\
   ```
2. Se a pasta não existir, rode dentro do projeto:
   ```powershell
   openspec init
   ```
3. Se o perfil foi atualizado mas os comandos sumiram, regenere:
   ```powershell
   openspec update
   ```
4. Reinicie o Claude Code após qualquer alteração nos arquivos de commands.

### Dashboard não abre

- URL correta? `http://10.150.0.69:4000/ui` (com `/ui` no final)
- `database_url` configurado no `config.yaml`?
- Reiniciar serviço após mudanças no config:
  ```powershell
  Restart-Service LiteLLMGateway
  ```

---

## 6. Comandos de manutenção

### Atualizar LiteLLM

```powershell
Stop-Service LiteLLMGateway
pip install --upgrade "litellm[proxy]"
Start-Service LiteLLMGateway
```

Sempre teste após atualizar — APIs internas mudam ocasionalmente.

### Backup do banco de chaves e gastos

```powershell
$date = Get-Date -Format 'yyyyMMdd-HHmm'
Copy-Item C:\litellm\litellm.db "C:\litellm\backups\litellm-$date.db"
```

Automatize via Tarefa Agendada do Windows (diário ou semanal).

### Rotacionar a master key

Para trocar a master key (recomendado a cada 6 meses ou quando suspeitar de comprometimento):

1. Gerar nova:
   ```powershell
   python -c "import secrets; print('sk-' + secrets.token_urlsafe(48))"
   ```
2. Atualizar variável de sistema:
   ```powershell
   [Environment]::SetEnvironmentVariable("LITELLM_MASTER_KEY", "sk-novaMasterKey...", "Machine")
   ```
3. Atualizar variável no NSSM:
   ```powershell
   nssm edit LiteLLMGateway
   ```
   Aba **Environment** → atualizar `LITELLM_MASTER_KEY`
4. Reiniciar serviço:
   ```powershell
   Restart-Service LiteLLMGateway
   ```
5. Atualizar gerenciador de senhas

> ⚠️ Trocar a master key **NÃO invalida** as virtual keys já distribuídas — os devs continuam usando normalmente. Apenas o acesso administrativo (criar/listar/deletar chaves, dashboard) requer a nova master key.

### Verificar saúde geral

```powershell
# Status do serviço
Get-Service LiteLLMGateway

# Endpoints saudáveis (não precisa de auth)
curl http://localhost:4000/health/liveliness
curl http://localhost:4000/health/readiness

# Saúde de todos os modelos configurados (precisa master key)
$headers = @{ "Authorization" = "Bearer $env:LITELLM_MASTER_KEY" }
Invoke-RestMethod -Uri "http://localhost:4000/health" -Headers $headers
```

### Logs em tempo real

```powershell
Get-Content C:\litellm\logs\stdout.log -Tail 50 -Wait
```

`Ctrl+C` para parar.

### Adicionar/remover modelos sem reiniciar

Use o dashboard web (`/ui`) na aba **Models** — alterações são gravadas no SQLite e aplicadas dinamicamente.

Para mudanças permanentes via `config.yaml`, edite o arquivo e reinicie:

```powershell
Restart-Service LiteLLMGateway
```

---

## Anexo — Checklist final de segurança

Antes de considerar a configuração concluída, confirme:

- [ ] API keys da Anthropic e Gemini configuradas **apenas** como variáveis de sistema no PC Servidor, **nunca** dentro do `config.yaml` em texto plano
- [ ] Master key gerada com mínimo 48 bytes de entropia (CSPRNG)
- [ ] Master key armazenada em gerenciador de senhas, **nunca** em arquivos `.txt` ou commits Git
- [ ] Cada dev tem uma virtual key **única**, nunca compartilhada entre dois ou mais
- [ ] Notebooks dos devs têm `ANTHROPIC_BASE_URL` e `ANTHROPIC_AUTH_TOKEN` definidos no nível de **usuário**
- [ ] Notebooks dos devs **não** têm `ANTHROPIC_API_KEY` definida (evita bypass do gateway)
- [ ] Firewall do Windows configurado com escopo de IP `10.150.0.0/24`, **não** `Any`
- [ ] Interface de rede do servidor classificada como **Private** (não Public)
- [ ] Roteador da rede **não** tem port forwarding na porta 4000
- [ ] Serviço NSSM configurado com restart automático em caso de crash
- [ ] LM Studio configurado com servidor em `127.0.0.1:1234` (não exposto à LAN)
- [ ] Backup automatizado do `litellm.db` em pasta separada
- [ ] Logs sendo rotacionados (10 MB ou 1 dia)
- [ ] Dashboard testado e acessível em `http://10.150.0.69:4000/ui`
- [ ] Claude Code testado em notebook do dev com `claude --print "teste"` passando pelo gateway
- [ ] OpenSpec inicializado em pelo menos um projeto (`openspec init`) e comandos `/opsx:*` funcionando
- [ ] Pelo menos 1 dev executou o workflow completo: `/opsx:propose` → `/opsx:apply` → `/opsx:archive`

---

**Documento gerado para Natural Tecnologia**
**Stack:** LiteLLM Gateway + LM Studio + Claude Code CLI + OpenSpec (OPSX)
**Última revisão:** maio 2026
