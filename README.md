# 🛠️ Classroom50 Setup Template

Este repositório contém a infraestrutura base de workflows e configurações necessária para habilitar a avaliação automatizada com IA (Google Gemini API + LangGraph) no ecossistema **Classroom 50**.

Ele serve como molde para que professores e administradores de turma provisionem a correção automatizada em novas organizações do GitHub.

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
├── GUIA_PROFESSOR.md                            # Guia de cadastramento e gestão de exercícios via CLI
└── README.md                                    # Guia de implantação e infraestrutura do template
```

---

## 🚀 Como Configurar uma Nova Organização

### 1. Inicializar a Organização via CLI (`gh teacher`)

Antes de copiar os arquivos de infraestrutura, certifique-se de que a organização foi inicializada pelo administrador utilizando a extensão oficial do Classroom 50:

```bash
# Instalar a extensão gh teacher (caso ainda não possua)
gh extension install foundation50/gh-teacher

# Autenticar com escopo OAuth administrativo
gh teacher login

# Inicializar a organização
gh teacher init <NOME_DA_ORGANIZACAO>
```

---

### 2. Configurar o Arquivo Gatilho (`autograders/seguro.yaml`)

Devido às restrições nativas do GitHub Actions, a propriedade `uses:` exige o caminho estático e literal da organização. 

Altere a linha do arquivo `autograders/seguro.yaml` para conter o nome exato da sua organização no GitHub:

```yaml
jobs:
  run-evaluator:
    # Substitua SUA_ORGANIZACAO pelo nome real da org (ex: ufla-classroom50)
    uses: SUA_ORGANIZACAO/classroom50/.github/workflows/secure-evaluator-monolitico.yaml@main
    secrets: inherit
```

---

### 3. Configurar os Segredos da Organização (Organization Secrets)

Acesse **Organization Settings > Secrets and variables > Actions** na sua organização no GitHub e cadastre as credenciais abaixo:

#### A. `GEMINI_API_KEY`
Chave de API do Google Gemini utilizada para autenticação e geração dos feedbacks pedagógicos automatizados via IA.

**Como obter a chave:**
1. Acesse o [Google AI Studio](https://aistudio.google.com/) e faça login com sua conta Google.
2. Clique no botão **Get API key** no painel inicial.
3. Clique em **Create API key** (selecione um projeto existente no Google Cloud ou crie um novo).
4. Copie o código alfanumérico gerado.

#### B. `GABARITOS_READ_TOKEN`
Personal Access Token (PAT) do GitHub com permissão de leitura nos repositórios privados de gabarito da organização.

**Como gerar o token no GitHub (Fine-grained PAT - Recomendado):**
1. No GitHub, clique na foto de perfil (canto superior direito) e acesse **Settings**.
2. No menu lateral esquerdo, role até o final e clique em **Developer settings**.
3. Navegue até **Personal access tokens > Fine-grained tokens** e clique em **Generate new token**.
4. Preencha os campos da seguinte forma:
   * **Token name:** `Classroom50-Gabaritos-ReadOnly`
   * **Expiration:** Selecione a validade desejada (ex: *90 days* ou duração do semestre).
   * **Resource owner:** Selecione a Organização da disciplina (ex: `ufla-classroom50`).
   * **Repository access:** 
     * Escolha **Only select repositories** e marque o repositório `gabaritos-privados` (caso use um único repositório para a turma), ou:
     * Escolha **All repositories** (caso a organização seja dedicada exclusivamente a abrigar gabaritos).
   * **Repository permissions:** Expanda a permissão **Contents** e marque **Access: Read-only**.
5. Clique em **Generate token** no rodapé e copie o token gerado (iniciado por `github_pat_...`).

#### C. Cadastro dos Segredos na Organização (GitHub):
1. Entre na página da Organização no GitHub e acesse **Settings**.
2. No menu lateral esquerdo, navegue até **Secrets and variables > Actions**.
3. Clique em **New organization secret** e cadastre cada item:
   * **Name:** `GEMINI_API_KEY` | **Secret:** *(Cole a chave do AI Studio)* | **Repository access:** *All repositories*.
   * **Name:** `GABARITOS_READ_TOKEN` | **Secret:** *(Cole o token github_pat_...)* | **Repository access:** *All repositories*.

---

### 4. Escolher o Modelo de Organização dos Gabaritos

Copie o arquivo de workflow adequado para o diretório `.github/workflows/` do seu repositório central `classroom50`:

* **Modelo 1: Repositório Monolítico (`gabaritos-privados`) — Padrão**
  * Use o arquivo `secure-evaluator-monolitico.yaml`.
  * Os gabaritos de todos os exercícios ficam em um único repositório privado chamado `gabaritos-privados`, organizados em pastas com o nome da `slug` da atividade (ex: `gabaritos-privados/exercicio-livro/`).
  * O checkout busca o repositório `${{ github.repository_owner }}/gabaritos-privados` dinamicamente.

* **Modelo 2: Organização Dedicada (1 Repositório por Exercício)**
  * Use o arquivo `secure-evaluator-por-repositorio.yaml`.
  * Cada atividade é um repositório próprio em uma organização de gabaritos exclusiva (ex: `org-gabaritos/exercicio-livro`).
  * Permite isolar o acesso por exercício utilizando a variável de organização `GABARITOS_ORG`.

---

## 📖 Leitura de Enunciados e Gabaritos: Estrutura de Arquivos

O motor de avaliação combina o código enviado pelo aluno com a solução oficial e o arquivo `enunciado.md` do professor.

### Estrutura do Repositório de Gabaritos (Modelo Monolítico)

Na pasta correspondente à `slug` do exercício (ex: `gabaritos-privados/exercicio-livro/`), os seguintes arquivos devem existir:
* `enunciado.md`: Regras e instruções da atividade passadas ao prompt da IA.
* `*.java`: Um ou mais ($N$) arquivos `.java` contendo a solução de referência oficial do professor.

---

## 🐳 Customização e Contrato da Imagem Docker do Autograder

A imagem padrão utilizada pelos workflows do Classroom 50 é `markinsilva/classroom50-ai-autograder:v2.0`.

### Alterando a Imagem via GitHub Actions

Para substituir a imagem por uma versão customizada sem alterar o código dos workflows, adicione uma **Organization Variable** ou **Repository Variable** no GitHub:
* **Nome:** `AUTOGRADER_DOCKER_IMAGE`
* **Valor:** `seu-usuario-docker/sua-imagem:v2.0`

---

### Contrato de Montagem de Volumes (Container)

O contêiner do autograder é executado mapeando 3 diretórios obrigatórios:

| Diretório no Container | Origem | Descrição |
| --- | --- | --- |
| `/app/submission` | Código do aluno | Solução enviada pelo estudante (varredura recursiva de arquivos `.java`). |
| `/app/reference` | Gabarito do professor | Contém o `enunciado.md` e os arquivos `.java` de referência da atividade. |
| `/app/output` | Saída | Diretório onde o contêiner DEVE gerar os arquivos de resultado (`results.json` e `feedback.md`). |

---

### Contrato dos Arquivos de Saída (`output/`)

Ao finalizar a análise, a imagem Docker deve gerar no diretório `/app/output` os seguintes arquivos:

#### 1. `output/results.json`
Arquivo JSON lido pelo workflow para publicação e registro de notas no contrato `classroom50/result/v1`:

```json
{
  "avaliacao": "Certo",
  "justificativa": "O aluno aplicou corretamente os conceitos de herança e encapsulamento exigidos.",
  "dica_correcao": "Para melhorar ainda mais seu código, considere tornar os atributos de classe privados.",
  "sugestao_correcao": "private String titulo;\nprivate double preco;"
}
```

> **Valores aceitos na chave `avaliacao`:** `"Certo"`, `"Parcialmente Certo"` ou `"Errado"`.

#### 2. `output/feedback.md`
Arquivo em formato Markdown contendo a mensagem a ser formatada e postada como comentário no Pull Request do estudante.

---

### 🛡️ Resiliência e Tratamento de Erros (Fallback da API Gemini)

O motor contido no contêiner possui mecanismo de resiliência com **tentativas de retry (backoff)** para contornar instabilidades transitórias da API do Gemini (status `503`, `429` ou *timeouts*):

1. O script tenta realizar a chamada até **3 vezes** com intervalo progressivo entre as falhas.
2. Caso a API permaneça indisponível ou a chave não esteja configurada, o motor ativa um **fallback de contingência**:
   * Gera um `feedback.md` amigável no PR orientando o aluno a re-executar `gh student submit` em alguns minutos.
   * Gera o `results.json` de fallback no schema oficial com `avaliacao`: `"Parcialmente Certo"` e pontuação zerada.
   * Encerra a execução com código de saída `0` (`sys.exit(0)`), mantendo o Job no GitHub Actions em **Verde (Sucesso 🟢)** para evitar falsos alertas nos painéis dos monitores.

---

## 📤 Fluxo de Submissão do Aluno e Gatilho `submit/*`

Para evitar a execução desnecessária da IA durante commits intermediários de desenvolvimento:

1. **Restrição do Gatilho (`autograders/seguro.yaml`)**: O autograder é configurado para ser acionado apenas quando houver o envio de uma tag padronizada:
   ```yaml
   on:
     push:
       tags:
         - 'submit/*'
   ```

2. **Comando de Submissão pelo Aluno**: O estudante envia a versão final executando no terminal o comando da CLI do Classroom 50:
   ```bash
   gh student submit
   ```

Este comando gera uma tag de versão (ex: `submit/2026-11-09T14-57-00`), disparando a avaliação isolada no contêiner central e postando o parecer pedagógico no Pull Request e nas Releases do repositório do aluno.