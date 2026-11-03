# 🛠️ Classroom50 Setup Template

Este repositório contém a infraestrutura base de workflows e configurações necessária para habilitar a avaliação automática com IA (Gemini API + LangGraph) no ecossistema **Classroom50**.

Ele serve como molde para que professores provisionem a correção automatizada em novas organizações no GitHub.

---

## 📂 Estrutura do Repositório

```text
classroom50-setup-template/
├── .github/
│   └── workflows/
│       ├── secure-evaluator-monolitico.yaml      # Fluxo para gabaritos em subpastas (Modelo Atual)
│       └── secure-evaluator-por-repositorio.yaml  # Fluxo para 1 repo por gabarito (Modelo Futuro)
├── autograders/
│   └── seguro.yaml                              # Gatilho leve injetado nos repositórios dos alunos
└── README.md                                    # Guia de implantação e customização
└── GUIA_PROFESSOR.md                            # Guia para configuração e submissão dos exercícios

```

---

## 🚀 Como Configurar uma Nova Organização

### 1. Ajustar o Arquivo Gatilho (`autograders/seguro.yaml`)

Devido às restrições do GitHub Actions, a propriedade `uses:` exige o caminho exato do repositório da organização.
Altere a linha do arquivo `autograders/seguro.yaml` para conter o nome da sua organização:

```yaml
uses: SUA_ORGANIZACAO/classroom50/.github/workflows/secure-evaluator-monolitico.yaml@main

```

### 2. Configurar os Segredos da Organização (Organization Secrets)

Acesse **Organization Settings > Secrets and variables > Actions** na sua organização e adicione as seguintes credenciais:

* **`GEMINI_API_KEY`**: Chave da API do Google Gemini (gerada no Google AI Studio) utilizada para a geração de feedback.
* **`GABARITOS_READ_TOKEN`**: Personal Access Token (PAT) com permissão de leitura nos repositórios/pastas de gabarito privado da organização.

### 3. Escolher o Modelo de Organização dos Gabaritos

Copie o arquivo de workflow adequado para o diretório `.github/workflows/` do seu repositório central `classroom50`:

* **Modelo 1: Repositório Monolítico (`gabaritos-privados`)**
* Use o arquivo `secure-evaluator-monolitico.yaml`.
* Os gabaritos ficam em um único repositório privado chamado `gabaritos-privados`, organizados em pastas nomeadas por `slug` da atividade (ex: `gabaritos-privados/exercicio-livro/`).
* O checkout utiliza `${{ github.repository_owner }}/gabaritos-privados` dinamicamente.


* **Modelo 2: Organização Dedicada (1 Repositório por Exercício)**
* Use o arquivo `secure-evaluator-por-repositorio.yaml`.
* Cada atividade é um repositório próprio em uma organização de gabaritos (ex: `org-gabaritos/exercicio-livro`).
* Defina a variável de organização `GABARITOS_ORG` nas configurações da organização se a org de gabaritos for diferente da org do Classroom50.



---

## 📖 Leitura de Enunciados e Gabaritos: Atual vs. Futuro

O motor de avaliação combina o código enviado pelo aluno com a solução oficial e o arquivo `enunciado.md` do professor.

### Modelo Atual (Repositório Monolítico `gabaritos-privados`)

* **Estrutura**: Existe um único repositório privado na organização chamado `gabaritos-privados`.
* **Organização**: Cada exercício fica em uma subpasta nomeada com a `slug` exata da atividade (ex: `gabaritos-privados/exercicio-livro/`).
* **Arquivos da Pasta**:
* `enunciado.md`: Descrição das regras da atividade lida dinamicamente como instrução.
* `*.java`: Um ou múltiplos ($N$) arquivos `.java` com o código de referência oficial.



### Modelo Futuro (Organização Exclusiva de Gabaritos — 1 Repo por Exercício)

* **Estrutura**: Uma organização externa dedicada (ex: `org-gabaritos-professor`), onde **cada exercício é um repositório isolado** com o nome da `slug` (ex: `org-gabaritos-professor/exercicio-livro`).
* **Arquivos**: A raiz do repositório individual conterá diretamente o `enunciado.md` e os arquivos `.java`.

---

## 🐳 Customização da Imagem Docker do Autograder

A imagem padrão utilizada pelo workflow é `markinsilva/classroom50-ai-autograder:v2.0`.

### Alterando a Imagem via GitHub Actions

Para substituir a imagem sem alterar o código do workflow, adicione uma **Organization Variable** ou **Repository Variable**:

* **Nome**: `AUTOGRADER_DOCKER_IMAGE`
* **Valor**: `seu-usuario-docker/sua-imagem:v1.0`

---

## 🏗️ Como Criar e Dockerizar uma Nova Imagem de Autograder

Caso deseje criar um novo motor de avaliação (ex: integrando compiladores Java ou outros modelos de IA), a nova imagem Docker deve seguir o **Contrato de Montagem de Volumes**:

### Contrato de Diretórios (Volumes)

O container é executado mapeando 3 diretórios obrigatórios:

| Diretório no Container | Origem | Descrição |
| --- | --- | --- |
| `/app/submission` | Código do aluno | Contém a solução enviada pelo estudante (busca recursiva de `.java`). |
| `/app/reference` | Gabarito do professor | Contém o arquivo `enunciado.md` e os arquivos `.java` oficiais. |
| `/app/output` | Saída | Local onde a imagem DEVE salvar o relatório `results.json`. |

### Variáveis de Ambiente Injetadas

* `GEMINI_API_KEY`: Chave da API enviada de forma isolada ao container.

### Formato Obrigatório do Arquivo `output/results.json`

O motor contido na imagem Docker deve gerar o arquivo `/app/output/results.json` na seguinte estrutura ao finalizar a execução:

```json
{
  "avaliacao": "Certo",
  "justificativa": "Explicação detalhada dos acertos e falhas encontradas no código.",
  "dica_correcao": "Instrução pedagógica para ajudar o aluno a evoluir.",
  "sugestao_correcao": "Exemplo de implementação em Java (opcional)."
}

```

### Exemplo de `Dockerfile` para Nova Imagem

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Instala dependências do sistema e do compilador
RUN apt-get update && apt-get install -y default-jdk && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src

# Garante que os diretórios do contrato existam
RUN mkdir -p /app/submission /app/reference /app/output

ENTRYPOINT ["python", "/app/src/main.py"]

```

### Comandos para Compilar e Publicar a Nova Imagem

```bash
# 1. Build da nova imagem
docker build -t seu-usuario-docker/classroom50-custom-autograder:v1.0 .

# 2. Upload para o Registry (Docker Hub)
docker push seu-usuario-docker/classroom50-custom-autograder:v1.0

```

---