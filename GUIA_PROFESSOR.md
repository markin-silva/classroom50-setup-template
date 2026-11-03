# 🎓 Guia do Professor: Cadastramento e Gestão de Exercícios (`gh teacher`)

Este documento descreve o fluxo de trabalho detalhado para professores e monitores criarem, registrarem e acompanharem atividades (assignments) no ecossistema **Classroom 50** utilizando as extensões oficiais da GitHub CLI (`gh teacher` e `gh student`) e a correção automatizada Zero-Trust (`seguro`).

---

## 📋 Pré-requisitos e Instalação

Instale as extensões oficiais do Classroom 50 diretamente via GitHub CLI (`gh`):

```bash
# Instalação das extensões (Escopo Oficial foundation50)
gh extension install foundation50/gh-teacher
gh extension install foundation50/gh-student

# Verificação da instalação
gh teacher --help
gh student --help

```

### Autenticação na CLI

Faça o login com as permissões OAuth necessárias para o Classroom 50:

```bash
gh teacher login

```

### Inicialização da Organização (Executado 1x por Org)

Caso a organização ainda não tenha sido configurada pelo administrador:

```bash
gh teacher init <NOME_DA_ORGANIZACAO>

```

---

## 🚀 Passo a Passo: Criação e Lançamento de uma Atividade

```text
┌───────────────────────────┐    ┌───────────────────────────┐    ┌───────────────────────────┐
│  1. Cofre de Gabaritos    │    │  2. Template do Aluno     │    │  3. Registro via CLI      │
│  Enunciado + Solução Java │───>│  Starter Code Limpo       │───>│  gh teacher assignment    │
└───────────────────────────┘    └───────────────────────────┘    └───────────────────────────┘

```

### 1. Cadastrar o Gabarito e Enunciado no Cofre Privado

Antes de publicar o exercício para a turma, adicione a solução de referência e o enunciado no repositório de gabaritos da organização:

* **Modelo 1: Repositório Monolítico (`gabaritos-privados`)**
1. Acesse o repositório privado `gabaritos-privados` da organização.
2. Crie uma pasta com a `slug` exata do exercício (ex: `exercicio-livro`).
3. Adicione os arquivos obrigatórios:
* `enunciado.md`: Regras da atividade e instruções de implementação.
* `*.java`: Um ou mais ($N$) arquivos `.java` contendo o código de referência oficial.




* **Modelo 2: Repositório Isolado por Exercício**
1. Crie um repositório privado na organização de gabaritos nomeado com a `slug` da atividade (ex: `org-gabaritos/exercicio-livro`).
2. Adicione o `enunciado.md` e os arquivos `.java` diretamente na raiz do repositório.



---

### 2. Criar o Repositório de Template (Starter Code)

O repositório base fornecido aos alunos deve ser mantido 100% limpo de scripts de CI/CD ou gabaritos:

1. Crie um novo repositório na organização com a estrutura inicial em Java (ex: `template-exercicio-livro`).
2. Adicione apenas o código-fonte básico/esqueleto que o aluno utilizará como ponto de partida.
3. Nas configurações do repositório (**Settings > General**), marque a opção **Template repository**.

---

### 3. Registrar a Atividade na Turma via CLI

No terminal, execute o comando de registro associando a atividade ao repositório template e definindo o autograder customizado (`seguro`):

```bash
gh teacher assignment add <NOME_DA_ORGANIZACAO> <NOME_DA_TURMA> <SLUG_DO_EXERCICIO> \
  --name "Título Exibido da Atividade" \
  --template <NOME_DA_ORGANIZACAO>/<REPO_TEMPLATE> \
  --autograder seguro \
  --submission-mode tag
```

#### Exemplo Prático:

```bash
gh teacher assignment add ufla-classroom50 testes-classroom50 exercicio-livro \
  --name "Exercício 01 - Herança e Polimorfismo" \
  --template ufla-classroom50/template-exercicio-livro \
  --autograder seguro
  --submission-mode tag
```

> **O que ocorre nos bastidores:**
> A flag `--autograder seguro` instrui o Classroom 50 a buscar o arquivo `autograders/seguro.yaml` cadastrado no repositório central `classroom50` da organização e injetá-lo automaticamente no `.github/workflows/autograde.yaml` do repositório do estudante no momento em que a tarefa for aceita.

---

## 👩‍🎓 Fluxo de Execução do Aluno

1. **Autenticação do Aluno**:
```bash
gh student login

```


2. **Aceite da Tarefa**:
```bash
gh student accept <NOME_DA_ORGANIZACAO> <NOME_DA_TURMA> <SLUG_DO_EXERCICIO>

```


3. **Desenvolvimento**: O estudante resolve o exercício em seu repositório privado realizando commits rotineiros (sem acionar a IA desnecessariamente).
4. **Submissão Final**:
```bash
gh student submit

```


Este comando publica uma tag padronizada (`submit/*`), disparando a avaliação isolada no container Docker central e publicando o parecer pedagógico nas Releases e no resumo do GitHub Actions.

---

## 📊 Coleta de Notas e Sincronização

Após os prazos de entrega, o professor pode baixar e consolidar os relatórios de nota calculados pela IA de toda a turma diretamente pela CLI:

```bash
gh teacher download <NOME_DA_ORGANIZACAO> <NOME_DA_TURMA> <SLUG_DO_EXERCICIO>

```

Os resultados serão sincronizados localmente no repositório central da turma, gerando o arquivo `scores.csv` sem a necessidade de clonar manualmente cada repositório de estudante.