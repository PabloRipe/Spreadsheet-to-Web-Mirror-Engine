# Spreadsheet-to-Web Mirror Engine

Transforme planilhas `.xlsx` complexas em espelhos web interativos e gere o
arquivo final preenchido com fórmulas e estilos preservados.

O projeto tem duas camadas:

1. **Módulo de processamento local (Python/openpyxl)** — lê o `.xlsx`, clona a
   estrutura de todas as abas, aplica a convenção de cores que separa células
   fixas de editáveis e exporta:
   - `mirror.json`: espelho estruturado (abas, células, estilos, mesclagens);
   - `template.xlsx`: clone limpo, com fórmulas e estilos intactos e apenas as
     células editáveis em branco.
2. **Camada web (Flask + Tailwind CSS + JavaScript puro)** — consome o
   `mirror.json`, renderiza a planilha no navegador, permite preencher as
   células editáveis (com auto-save em `sessionStorage`) e gera o `.xlsx` final
   a partir do `template.xlsx` com os dados informados.

---

## Requisitos

- Python 3.9 ou superior
- `pip`

Navegadores modernos (Chrome, Edge, Firefox ou Safari) são necessários para a
interface web. Não há dependência de Node.js: o Tailwind é carregado via CDN.

---

## Instalação

Clone o repositório e acesse a pasta:

```bash
git clone https://github.com/PabloRipe/Spreadsheet-to-Web-Mirror-Engine.git
cd Spreadsheet-to-Web-Mirror-Engine
```

### Opção 1 — instalação completa (desenvolvimento)

Instala o pacote em modo editável com todas as dependências de desenvolvimento e
da camada web:

```bash
python -m venv .venv
```

Ative o ambiente virtual:

- **Windows (PowerShell):**
  ```powershell
  .venv\Scripts\Activate.ps1
  ```
- **Linux/macOS:**
  ```bash
  source .venv/bin/activate
  ```

Instale o projeto:

```bash
pip install -e ".[dev,web]"
```

### Opção 2 — instalação mínima (apenas executar o servidor)

```bash
pip install -r requirements.txt
```

---

## Uso

### 1. Processar uma planilha pela linha de comando

```bash
spreadsheet-mirror process caminho/planilha.xlsx -o saida/
```

Processar todos os `.xlsx` de um diretório (recursivo):

```bash
spreadsheet-mirror process caminho/pasta -o saida/ --recursive
```

Listar planilhas encontradas:

```bash
spreadsheet-mirror scan caminho/pasta --recursive
```

A saída é uma pasta por projeto:

```text
saida/
├── index.json
└── minha-planilha/
    ├── mirror.json
    └── template.xlsx
```

### 2. Executar a camada web

Com as dependências instaladas:

```bash
python server.py
```

O servidor sobe em `http://localhost:5000` por padrão. Para outra porta, defina
a variável `PORT`:

```powershell
$env:PORT=8000; python server.py
```

Para produção local (servidor WSGI `waitress`):

```bash
waitress-serve --port=8000 server:app
```

Acesse `http://localhost:8000` no navegador.

### 3. Usar a interface web

1. Clique em **Enviar planilha** e selecione um arquivo `.xlsx` (ou arraste e
   solte na área indicada).
2. O processamento gera automaticamente o projeto em `mirror_output/<slug>/`
   com `mirror.json` e `template.xlsx`.
3. Escolha um projeto no seletor superior. As abas da planilha aparecem como
   botões; clique para alternar.
4. Preencha apenas as células editáveis (fundo branco). Células fixas
   (cabeçalhos, fórmulas, mesclagens e preenchimentos coloridos) são exibidas
   sem edição.
5. Os valores são salvos automaticamente no navegador (`sessionStorage`).
6. Clique em **Gerar .xlsx** para baixar o arquivo final. As fórmulas são
   preservadas e serão recalculadas ao abrir no Excel.

### Variáveis de ambiente

| Variável           | Padrão           | Descrição                                                        |
| ------------------ | ---------------- | ---------------------------------------------------------------- |
| `PORT`             | `5000`           | Porta usada por `python server.py`.                              |
| `MIRROR_WORKSPACE` | `./mirror_output` | Diretório onde os projetos processados e templates são armazenados. |

---

## Estrutura do projeto

```text
.
├── server.py                     # API Flask + servidor da interface web
├── web/
│   ├── index.html                # Interface (Tailwind via CDN)
│   ├── app.js                    # Lógica de renderização, edição e exportação
│   └── styles.css                # Estilos da grade e abas
├── src/spreadsheet_mirror/
│   ├── cli.py                    # Comandos scan/process
│   ├── config.py                 # Convenção de cores configurável
│   ├── reader.py                 # Leitura de .xlsx (fórmulas + valores em cache)
│   ├── colors.py                 # Cores RGB, indexadas e de tema
│   ├── styles.py                 # Estilos, dimensões e mesclagens
│   ├── editability.py            # Regras fixo vs. editável
│   ├── serializer.py             # Geração do mirror.json
│   ├── template.py               # Geração do template.xlsx e preenchimento
│   └── pipeline.py               # Orquestração do processamento em lote
├── tests/                        # Testes unitários e de integração (pytest)
├── requirements.txt              # Dependências de execução
├── pyproject.toml                # Empacotamento e dependências do projeto
└── README.md
```

---

## Convenção de cores (fixo vs. editável)

Por padrão, uma célula é **fixa** quando:

- contém fórmula;
- é parte não-âncora de um intervalo mesclado;
- está nas linhas/colunas de cabeçalho configuradas;
- tem preenchimento sólido diferente de branco;
- tem fonte em negrito.

Caso contrário (fundo branco/padrão e sem fórmula), a célula é marcada como
**editável** e seu valor é removido do template.

As regras podem ser ajustadas via arquivo JSON de configuração ou flags da CLI
(`--header-rows`, `--header-cols`, `--no-fixed-if-bold`, entre outras). Consulte
`spreadsheet_mirror.config.MirrorConfig` para todas as opções.

---

## API HTTP

| Método | Rota                             | Descrição                                              |
| ------ | -------------------------------- | ------------------------------------------------------ |
| GET    | `/`                              | Interface web.                                         |
| GET    | `/api/health`                    | Status do servidor e quantidade de projetos.           |
| GET    | `/api/projects`                  | Lista de projetos processados.                         |
| GET    | `/api/projects/<slug>/mirror`    | Retorna o `mirror.json` do projeto.                    |
| POST   | `/api/upload`                    | Envia um `.xlsx` (multipart/form-data, campo `file`).  |
| POST   | `/api/projects/<slug>/export`    | Gera o `.xlsx` final com os dados preenchidos (JSON).  |

O limite de upload é de **32 MB**.

---

## Testes

```bash
python -m pytest
```

---

## Limitações conhecidas

O round-trip feito pelo openpyxl preserva fórmulas, estilos, mesclagens e
dimensões, mas não preserva gráficos, imagens, tabelas dinâmicas, macros VBA ou
determinados elementos de formatação condicional. O espelho JSON e o template
focam na estrutura de células conforme o escopo do projeto.

---

## Licença

MIT.
