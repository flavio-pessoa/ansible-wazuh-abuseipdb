# Cadastrando este projeto no SemaphoreUI

Esta pasta traz um exemplo de como cadastrar e executar o playbook
`../site.yml` (role `wazuh_abuseipdb_cdb`) em uma instância do
[SemaphoreUI](https://semaphoreui.com/) **já instalada e em execução**.

> Nomenclatura: em versões mais recentes do SemaphoreUI a seção antes
> chamada **Environment** foi renomeada para **Variable Groups** no menu.
> A API interna ainda usa o termo `environment`, então os dois nomes
> aparecem por aí — este guia usa o nome atual da UI (**Variable Groups**).

## Tradução dos termos (UI em português)

Se a sua instância estiver configurada em português, os nomes dos menus e
campos aparecem traduzidos. Tabela de correspondência confirmada na versão
usada neste guia:

| Termo em inglês (usado neste README) | Como aparece na UI em português |
|---|---|
| Projects | Projetos |
| Dashboard | Painel |
| Task Templates | **Modelos de Tarefa** |
| Workflows | Workflows *(Beta)* |
| Schedules | Agenda |
| Inventory | **Inventário** |
| Variable Groups | **Grupos de Variáveis** |
| Key Store | **Armazenamento de Chaves** (aparece truncado como "Armazenamento d...") |
| Repositories | **Repositórios** |
| Integrations | Integrações |
| Teams | Equipe |
| Runners | Executores |
| New Template | **Novo Modelo** |
| Ansible Playbook (tipo de template) | Ansible Playbook |
| Name | Nome |
| Repository | Repositório |
| Path to playbook file | Caminho para o arquivo playbook |
| Inventory (campo do template) | Inventário |
| Variable Groups (campo do template) | Grupos de Variáveis |
| **Survey Variables** | **Variáveis de Pesquisa** (dentro de "Opções avançadas", botão **"+ Adicionar variável"**) |
| Limit / Tags / Skip tags | Limite / Tags / Pular tags |
| Vaults | Cofres |
| Run | Executar/Run (varia conforme tela) |

O ponto que mais confunde é o **Survey Variables**: ele não é um item de
menu separado. Ele só aparece **dentro da tela de criação/edição de um
Task Template** ("Modelos de Tarefa" → abrir ou criar um modelo), na
coluna "Opções avançadas", sob o rótulo **"Variáveis de Pesquisa"**, com o
botão **"+ Adicionar variável"**.

## Arquivos desta pasta

| Arquivo | Para que serve |
|---|---|
| `variable_group.json` | Conteúdo **exato** para colar no campo principal de variáveis do Grupo de Variáveis (sem envelope — cole o arquivo inteiro) |
| `variable_group_env.json` | Conteúdo **exato**, opcional, para o campo de variáveis de ambiente do Grupo de Variáveis |
| `survey_vars.json` | Referência de **Survey Variables** / "Variáveis de Pesquisa" do Task Template, no formato aceito pela API (preencha campo a campo na UI; não é para colar) |

> ⚠️ **Atenção**: `variable_group.json` e `variable_group_env.json` devem
> ser colados **inteiros e como estão** no respectivo campo — eles já são
> o JSON final, sem chaves de agrupamento como `extra_vars`. Colar um
> arquivo com uma camada extra de aninhamento (ex.: `{"extra_vars": {...}}`)
> faz a role não enxergar as variáveis, mesmo que pareçam preenchidas.

## O que é necessário, na ordem

### 1. Project (Projetos)

Menu *Projects* / **Projetos** → *New Project* → dê um nome, por exemplo
`Wazuh - AbuseIPDB CDB`.

### 2. Key Store / Armazenamento de Chaves (chave SSH)

No menu do projeto, abra **Key Store** ("Armazenamento de Chaves") e
cadastre uma chave SSH (`Type: SSH Key`) com a chave privada usada para:

- acessar o repositório git deste projeto (se for privado), e/ou
- conectar via SSH nos Wazuh Managers de destino.

Se o repositório for público e o acesso aos hosts usar outra credencial
(ex.: usuário/senha), cadastre as chaves/credenciais correspondentes aqui
também — Repository e Inventory precisam referenciar algo do Key Store.

### 3. Repository (Repositórios)

No menu do projeto, abra **Repository** ("Repositórios") e cadastre:

- `Name` ("Nome"): `wazuh_abuseipdb_role`
- `URL`: URL do repositório git onde este projeto (`site.yml` + `roles/`)
  está versionado
- `Branch`: `main` (ou a branch que você usa)
- `SSH Key`: a chave criada no passo 2 (se o repositório for privado)

### 4. Inventory (Inventário)

No menu do projeto, abra **Inventory** ("Inventário") e cadastre um
inventário. Você pode usar o formato *Static* (INI), colando o conteúdo de
`../inventory.ini`:

```ini
[wazuh_managers]
wazuh-manager-01 ansible_host=10.0.0.10 ansible_user=admin
```

Ou o formato *Static YAML*, colando o conteúdo de `../inventory.yml`:

```yaml
all:
  children:
    wazuh_managers:
      hosts:
        wazuh-manager-01:
          ansible_host: 10.0.0.10
          ansible_port: 22
          ansible_user: root
```

Associe a chave/credencial usada para conectar nos hosts (`User
Credentials` / `Become Credentials`, conforme sua política de acesso SSH e
sudo). Se conectar como `root`, não é necessário configurar
`ansible_become`.

### 5. Variable Group (Grupos de Variáveis)

No menu do projeto, abra **Variable Groups** ("Grupos de Variáveis") e
crie um novo (ex.: `abuseipdb-default`). O Semaphore exige que todo Task
Template referencie um Variable Group, mesmo que vazio (`{}`).

No campo principal de variáveis (JSON, equivalente a `--extra-vars`), cole
o conteúdo **inteiro** de `variable_group.json` (sem chave de agrupamento
— é o JSON final, direto):

```json
{
  "abuseipdb_source_base_url": "https://raw.githubusercontent.com/borestad/blocklist-abuseipdb/refs/heads/main/abuseipdb-s100-7d.ipv4",
  "custom_abuseipdb_srcip": "100010",
  "custom_abuseipdb_dstip": "100011",
  "custom_rule_level": "10"
}
```

O valor acima é apenas um exemplo (repositório público borestad/blocklist-abuseipdb,
lista de 7 dias com confiança ~100%) — substitua `abuseipdb_source_base_url`
pela **URL completa** do arquivo da blocklist que você realmente vai
consumir. Diferente de versões anteriores deste guia, não existe mais uma
variável separada para o nome do arquivo: a role extrai isso
automaticamente do final da URL. `abuseipdb_source_base_url` é
obrigatória: a role falha explicitamente se ficar em branco (ver
`../roles/wazuh_abuseipdb_cdb/defaults/main.yml`).

Opcionalmente, no campo de variáveis de ambiente do processo, cole o
conteúdo de `variable_group_env.json`:

```json
{
  "ANSIBLE_HOST_KEY_CHECKING": "False"
}
```

### 6. Task Template (Modelos de Tarefa)

No menu do projeto, abra **Task Templates** ("Modelos de Tarefa") →
**Novo Modelo** → **Ansible Playbook**. No formulário que abre, aba
"Tarefa" (coluna "Opções comuns"):

- `Nome`: `Aplicar lista AbuseIPDB no Wazuh`
- `Repositório`: o repositório criado no passo 3
- `Caminho para o arquivo playbook`: `site.yml`
- `Inventário`: o inventário criado no passo 4
- `Grupos de Variáveis`: o Variable Group criado no passo 5

Preencha esses campos e **salve o template pela primeira vez** — só depois
de salvo ele aparece na listagem de "Modelos de Tarefa" e você pode
reabri-lo para configurar o restante.

Opcionalmente, na coluna **"Opções avançadas"** do mesmo formulário, seção
**"Variáveis de Pesquisa"** (Survey Variables), clique em **"+ Adicionar
variável"** e cadastre as perguntas descritas em `survey_vars.json` (campos
`name`, `title`, `description`, `type`, `required`, `values` para o tipo
`enum`, e o campo **"Default value"** que aparece na própria UI) — isso
troca a edição manual do Variable Group por um formulário preenchido a
cada execução, permitindo escolher, por exemplo, qual blocklist baixar sem
mexer na configuração salva. Para Ansible, Survey Variables são passadas
como `--extra-vars`, assim como as variáveis do Variable Group. Escolha
**"Extra variable"** em "Pass variable as" para cada uma — "Environment
variable" não é lido pela role.

### 7. Executar

Abra o template criado e clique em **Run**/**Executar**. Se configurou
Variáveis de Pesquisa, preencha o formulário antes de disparar. A execução
(stdout do `ansible-playbook`) aparece em tempo real na própria UI, com
histórico das execuções anteriores.

## Executando múltiplas listas (1d, 7d, 30d...)

Como a role aceita `abuseipdb_source_base_url` (URL completa) e
`custom_abuseipdb_srcip` como parâmetros (ver `../README.md`), a forma mais
simples no Semaphore é criar **um Task Template por lista**, cada um com
seu próprio Variable Group — ou usar um único template com Variáveis de
Pesquisa e trocar os valores a cada execução.

## Agendamento (execução periódica)

No Task Template criado, use a seção **Schedule** ("Agenda") para definir
uma expressão cron (ex.: `0 */6 * * *` para rodar a cada 6 horas),
garantindo que a blocklist no Wazuh seja atualizada periodicamente sem
intervenção manual.
