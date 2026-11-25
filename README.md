# Revolution X Vault — Sistema de Backup Incremental, Offline-First e Versionado para Cloudflare R2

Um script monolítico em **Python** projetado para executar backups automatizados, incrementais, resilientes a falhas de conexão (offline-first) e com controle de versionamento visual diretamente para o **Cloudflare R2** (armazenamento de objetos compatível com S3). Ao ser iniciado, o programa abre uma interface web local minimalista para configuração de pastas, chaves API do Cloudflare, agendamento por horário, retenção global e restauração intuitiva de versões.

---

## 1. Visão Geral da Arquitetura

O sistema opera inteiramente a partir de um **único script Python** (`backup_manager.py`) e utiliza arquivos de banco de dados **SQLite (`sqlite3`)** na mesma pasta do executável/script para garantir portabilidade total, suporte a qualquer tipo de arquivo/programa e funcionamento mesmo sem internet:

1. **Interface Web Local (UI Minimalista):** Servidor HTTP embutido que abre automaticamente no navegador padrão com fundo branco limpo (`#FFFFFF`), cartões estruturados e botões quadrados com cantos levemente arredondados (`border-radius: 6px`).
2. **Camada de Buffer Local Offline (`local_buffer.db`):** Antes de enviar qualquer arquivo para a nuvem (ou quando o computador é ligado sem internet), o script armazena os bytes brutos (`BLOB`) de qualquer arquivo ou programa dentro de um banco SQLite local. Assim que o envio ao Cloudflare R2 é confirmado, os dados são excluídos do `.db` local e o espaço em disco é liberado (`VACUUM`).
3. **Camada de Controle, Histórico e Versões (`backup_control.db`):** Registra credenciais da API Cloudflare, pastas monitoradas, trava de execução diária, relatórios detalhados de uploads e o catálogo completo de versões (`atual/` e `anterior/`) com datas reais de modificação (`mtime`) e prazos de retenção de arquivos excluídos.
4. **Motor de Sincronização & Versionamento R2 (`boto3`):** Compara datas/horários de modificação (`mtime`), preserva toda a hierarquia de pastas original e mantém no Cloudflare R2 a **versão atual**, a **versão anterior** e os **arquivos excluídos dentro do prazo global de retenção**.

---

## 2. Funcionalidades da Interface de Configuração (HTML)

A página exibida ao abrir o programa possui layout limpo em branco com botões quadrados de cantos arredondados e reúne todas as configurações e ferramentas em um único painel intuitivo:

### 2.1. Painel de Chave API e Credenciais Cloudflare R2
Seção dedicada e visível na tela de configuração onde o usuário insere as credenciais da sua conta Cloudflare R2:
* **Campo Chave API / Access Key ID:** Campo dedicado para inserir a chave de acesso gerada no Cloudflare R2.
* **Campo Secret Access Key (Chave Secreta API):** Campo tipo senha com botão quadrado arredondado de **Mostrar / Ocultar** chave.
* **Campo Token API Cloudflare (Opcional / Referência):** Campo para armazenar o token Bearer caso o usuário deseje manter o registro completo da chave API no cofre local.
* **Campo Account ID:** Identificador da conta Cloudflare (usado para montar o endpoint `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`).
* **Campo Bucket Name:** Nome do bucket de destino no Cloudflare R2.
* **Ações:** Botão quadrado arredondado **[Testar Conexão R2]** (valida imediatamente a chave API com o servidor) e botão **[Salvar Credenciais]** (grava no banco de dados local).

### 2.2. Explorador de Arquivos e Pastas
* Navegador interativo embutido na página web que permite percorrer diretórios e unidades do computador local (`C:\`, `D:\`, `/home`, etc.).
* Caixas de seleção (*checkboxes*) para marcar múltiplas pastas que farão parte da rotina de backup.
* **Botão de Confirmação:** Um botão quadrado com cantos arredondados (**[Confirmar Seleção de Pastas]**) que grava os caminhos escolhidos no banco de dados local.

### 2.3. Seletor de Horário (Estilo Relógio Digital)
* Componente visual inspirado em um relógio comum, composto por dois campos numéricos grandes para digitação direta de **Hora (`00` a `23`)** e **Minuto (`00` a `59`)** no formato `HH:MM`.
* **Regra de Disparo:** Sempre que o programa for iniciado (por exemplo, na inicialização do sistema operacional) e o horário atual do computador for **igual ou posterior** ao horário configurado (`hora_atual >= hora_configurada`), a rotina de backup do dia é disparada automaticamente.

### 2.4. Configuração Global de Retenção de Arquivos Excluídos
* Campo numérico único na tela de configuração: **"Manter arquivos excluídos por (X) dias"** (ex.: `7`, `15` ou `30` dias).
* **Escopo Global:** O valor definido aplica-se a **todos os arquivos e pastas** monitorados pelo sistema, garantindo uma janela de segurança uniforme para recuperação caso algum arquivo ou aplicativo seja apagado acidentalmente do computador.

### 2.5. Painel Visual de Versionamento e Restauração
Exibe de forma clara e organizada todos os arquivos sincronizados para que o usuário possa localizar e recuperar qualquer item rapidamente:
* **Filtro e Busca:** Pesquisa instantânea por nome de arquivo, extensão, pasta ou status (**Todos**, **Ativos**, **Excluídos em Retenção**).
* **Endereço e Estrutura de Pastas:** Mostra o nome do arquivo, a árvore de subpastas relativa e o **endereço completo original** no computador (ex.: `C:\Projetos\Sistema\config.json`).
* **Comparativo Lado a Lado das Duas Versões:**
  * **Versão Atual (`atual/`):** Exibe a **data e hora exata de modificação do arquivo (`mtime`)** e o tamanho em disco, acompanhada dos botões quadrados arredondados **[Baixar]** e **[Restaurar no Local Original]**.
  * **Versão Anterior (`anterior/`):** Exibe a **data e hora exata de modificação (`mtime`)** da penúltima versão salva e o tamanho, com os botões **[Baixar]** e **[Restaurar no Local Original]** (ou indicação de que ainda não há versão anterior).
* **Indicador de Arquivos Excluídos:** Caso o arquivo tenha sido removido do PC, exibe o selo de **Excluído**, a data em que a exclusão foi detectada e a data limite de expiração antes da remoção definitiva do servidor.

---

## 3. Fluxo Offline-First (Backup Local em DB antes da Nuvem)

Para proteger os dados mesmo quando o computador é ligado sem acesso à internet:

1. **Captura Preventiva no Banco Local (`local_buffer.db`):**
   * Ao atingir o horário programado, o script varre as pastas configuradas e identifica arquivos novos ou com data de modificação (`mtime`) alterada.
   * Antes de qualquer tentativa de rede, o script lê os bytes binários (`rb`) de cada arquivo alterado (suportando executáveis `.exe`, bibliotecas `.dll`, compactados `.zip`, vídeos, documentos, bancos de dados e qualquer extensão) e os grava em uma tabela SQLite com coluna `BLOB` junto com a estrutura de pastas e o `mtime` original.
   * O dia é imediatamente marcado como **garantido localmente** no banco de controle.
2. **Verificação de Conectividade:**
   * **Se Offline:** O backup permanece protegido dentro de `local_buffer.db`. O painel exibe o status *"Offline — Backup salvo localmente, aguardando conexão com o Cloudflare R2"*. Enquanto o script estiver aberto (ou na próxima inicialização), ele verifica periodicamente o retorno da internet.
   * **Se Online:** O script lê cada item pendente do `local_buffer.db`, realiza o upload versionado para o Cloudflare R2 e, após a confirmação de sucesso (`HTTP 200`), **remove o registro binário do arquivo de `local_buffer.db`**.
3. **Compactação e Limpeza de Disco (`VACUUM`):**
   * Assim que todos os arquivos pendentes são enviados e excluídos do `local_buffer.db`, o script executa o comando SQLite `VACUUM`, reduzindo o tamanho do arquivo `.db` local a zero/mínimo e liberando o espaço no computador.

---

## 4. Integração com o Cloudflare R2

O **Cloudflare R2** utiliza protocolo 100% compatível com a API S3 (via biblioteca `boto3` em Python) e não cobra taxas de transferência de saída (*zero egress fees*).

### 4.1. Como Gerar a Chave API no Cloudflare R2
1. Acesse o painel da Cloudflare (`dash.cloudflare.com`) $\rightarrow$ **R2 Object Storage**.
2. Crie um bucket (ex.: `revolutionx-vault-backups`) e anote o seu **Account ID** (exibido no canto direito da visão geral do R2).
3. Clique em **Manage R2 API Tokens** $\rightarrow$ **Create API Token**.
4. Selecione a permissão **Object Read & Write** aplicada ao bucket criado e confirme.
5. Copie o **Access Key ID**, o **Secret Access Key** e o **Token API** e cole-os nos campos correspondentes da interface de configuração do programa.

### 4.2. Organização de Pastas e Versões no Bucket R2

A estrutura de diretórios do computador é espelhada integralmente dentro de prefixos organizados no R2:

```text
seu-bucket-r2/
├── atual/
│   └── NomeDaPastaSelecionada/
│       └── Subpasta/
│           └── programa.exe        <-- Versão mais recente (com mtime registrado)
├── anterior/
│   └── NomeDaPastaSelecionada/
│       └── Subpasta/
│           └── programa.exe        <-- Penúltima versão válida (com mtime registrado)
└── sistema/
    └── backup_control.db           <-- Cópia de segurança do catálogo e histórico
```

* **Rotação Automática de Versões:** Quando um arquivo já existente no R2 sofre alteração no computador (`mtime` mais recente), o script executa uma cópia interna no servidor (`copy_object`) de `atual/<caminho_relativo>` para `anterior/<caminho_relativo>` e, em seguida, faz o upload da nova versão para `atual/<caminho_relativo>`.
* **Restauração Fiel:** Ao clicar em **[Restaurar no Local Original]**, o script recria toda a árvore de pastas caso tenha sido apagada (`os.makedirs(..., exist_ok=True)`), baixa a versão escolhida (`atual` ou `anterior`) e reaplica a **data/hora de modificação original** no arquivo restaurado (`os.utime`).

---

## 5. Estrutura dos Bancos de Dados SQLite (`.db`)

Ambos os arquivos ficam armazenados na mesma pasta do script Python (`backup_manager.py`):

### 5.1. `backup_control.db` (Configurações, Trava Diária, Versões e Relatórios)
* **`configuracoes`:** Armazena `account_id`, `bucket_name`, `access_key_id`, `secret_access_key`, `api_token`, `horario_backup` (`HH:MM`) e `dias_retencao_excluidos` (inteiro global).
* **`pastas_monitoradas`:** Lista de caminhos absolutos selecionados no explorador de arquivos.
* **`controle_diario`:** Registra a data (`YYYY-MM-DD`), status (`local_ok`, `concluido_nuvem`) e horário de execução para impedir backups duplicados no mesmo dia.
* **`catalogo_versoes`:** Guarda `caminho_relativo`, `caminho_origem_completo`, `versao_atual_mtime`, `versao_atual_tamanho`, `versao_anterior_mtime`, `versao_anterior_tamanho`, `status` (`ativo` ou `excluido`), `data_exclusao` e `data_expiracao`.
* **`relatorio_uploads`:** Log detalhado de cada execução contendo data/hora, arquivo, pasta de origem, ação realizada (`novo`, `atualizado_versionado`, `excluido_retido`, `expirado_removido`), tamanho e status do envio.

### 5.2. `local_buffer.db` (Staging Binário Temporário)
* **`fila_upload_local`:**
  * `id` (`INTEGER PRIMARY KEY AUTOINCREMENT`)
  * `caminho_relativo` (`TEXT NOT NULL`)
  * `caminho_origem_completo` (`TEXT NOT NULL`)
  * `mtime_original` (`REAL NOT NULL`)
  * `tamanho_bytes` (`INTEGER NOT NULL`)
  * `conteudo_binario` (`BLOB NOT NULL` — suporta qualquer tipo de arquivo/programa)
  * `data_captura_local` (`TEXT NOT NULL`)

---

## 6. Requisitos e Execução

### Pré-requisitos
* **Python 3.10+** instalado no sistema.
* Biblioteca **`boto3`** para comunicação com a API S3 do Cloudflare R2:
  ```bash
  pip install boto3
  ```
* Bibliotecas nativas utilizadas (não requerem instalação): `http.server`, `sqlite3`, `os`, `shutil`, `json`, `urllib.parse`, `threading`, `time`, `datetime`, `webbrowser`.

### Como Iniciar
```bash
python backup_manager.py
```
Ao executar o comando:
1. Os bancos `backup_control.db` e `local_buffer.db` são criados/verificados automaticamente no diretório do script.
2. O navegador padrão é aberto exibindo a interface de configuração do **Revolution X Vault** (onde você insere a **Chave API do Cloudflare**, escolhe as **pastas**, define o **horário** e os **dias de retenção**).
3. Caso o horário atual já seja igual ou superior ao horário configurado e o backup do dia ainda não tenha sido feito, o ciclo **Buffer Local (`BLOB`) $\rightarrow$ Upload Versionado R2 $\rightarrow$ Limpeza do `.db` local (`VACUUM`)** inicia automaticamente em segundo plano.