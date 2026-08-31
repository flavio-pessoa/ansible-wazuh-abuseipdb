# ansible-wazuh-abuseipdb

Role Ansible que automatiza a integração de uma blocklist de reputação de IP
(padrão AbuseIPDB/FireHOL) ao **Wazuh Manager**, como lista CDB, incluindo:

1. Download e formatação da lista (`chave:valor`)
2. Ajuste de permissões do arquivo (`wazuh:wazuh`, `0640`)
3. Declaração da lista dentro de `<ruleset>` no `ossec.conf`
4. Criação de uma regra customizada de alerta em `local_rules.xml`
5. Restart do serviço `wazuh-manager`, apenas quando algo muda

Todas as etapas são **idempotentes**: rodar a role várias vezes não duplica
entradas em `ossec.conf` nem em `local_rules.xml`, e o `wazuh-manager` só é
reiniciado quando a lista, a configuração ou a regra de fato mudam.

## Estrutura

```
ansible-wazuh-abuseipdb/
├── inventory.ini                    # inventário de exemplo (formato INI)
├── inventory.yml                    # inventário de exemplo (formato YAML)
├── site.yml                         # playbook de exemplo que usa a role
├── group_vars/
│   └── wazuh_managers.yml           # overrides de variáveis por grupo (opcional)
└── roles/
    └── wazuh_abuseipdb_cdb/
        ├── defaults/main.yml        # variáveis padrão (sobrescrevíveis)
        ├── tasks/main.yml           # lógica principal
        ├── handlers/main.yml        # restart do serviço quando algo muda
        └── meta/main.yml            # metadados da role
```

## Pré-requisitos

- Ansible >= 2.14
- Acesso SSH com privilégios `sudo` ao(s) Wazuh Manager(s)
- Conectividade do host de destino até a URL da blocklist escolhida
- Wazuh já instalado nos hosts de destino

## Uso rápido

`abuseipdb_source_base_url` **não tem valor padrão** e precisa ser definida
antes de rodar — é a **URL completa** do arquivo da blocklist (não uma
URL "base" para compor com outro nome). Veja a seção
[Exemplo de valores](#exemplo-de-valores) para uma URL de referência.

```bash
# 1. Ajuste o inventário com o(s) host(s) real(is)
vim inventory.ini   # ou inventory.yml

# 2. Defina abuseipdb_source_base_url:
#    em site.yml, em group_vars/wazuh_managers.yml, ou via --extra-vars
vim group_vars/wazuh_managers.yml

# 3. Rode o playbook
ansible-playbook -i inventory.ini site.yml

# Ou definindo direto na linha de comando:
ansible-playbook -i inventory.ini site.yml \
  -e "abuseipdb_source_base_url=<URL_COMPLETA_DO_ARQUIVO>"

# Dry-run (não aplica mudanças, só mostra o que mudaria)
ansible-playbook -i inventory.ini site.yml --check --diff
```

## Principais variáveis (defaults/main.yml)

| Variável | Padrão | Descrição |
|---|---|---|
| `wazuh_lists_dir` | `/var/ossec/etc/lists` | Diretório de listas do Wazuh |
| `wazuh_rules_dir` | `/var/ossec/etc/rules` | Diretório de regras locais |
| `ossec_conf_path` | `/var/ossec/etc/ossec.conf` | Caminho do `ossec.conf` |
| `local_rules_path` | `{{ wazuh_rules_dir }}/local_rules.xml` | Caminho do `local_rules.xml` |
| `abuseipdb_source_base_url` | *(vazio — obrigatório)* | **URL completa** do arquivo remoto da blocklist (ex.: `.../abuseipdb-s100-7d.ipv4`) |
| `abuseipdb_source_url` | `{{ abuseipdb_source_base_url }}` | URL usada para o download; por padrão é igual à anterior, só sobrescreva em cenários avançados |
| `abuseipdb_list_name` | derivado | Nome do arquivo, extraído automaticamente do final da URL (filtro `basename`) |
| `abuseipdb_list_basename` | derivado | `abuseipdb_list_name` sem a extensão (calculado automaticamente) |
| `custom_abuseipdb_srcip` | `100010` | ID da regra de **entrada** (srcip) — use a faixa 100000–120000 |
| `custom_abuseipdb_dstip` | `100011` | ID da regra de **saída** (dstip); deixe `""` para desativá-la |
| `custom_rule_level` | `10` | Nível de criticidade do alerta |
| `custom_rule_if_group` | `pfsense` | Grupo do Wazuh ao qual as regras ficam atreladas via `<if_group>` — **obrigatório para as regras serem avaliadas** (ver [Observações importantes](#observações-importantes)) |
| `wazuh_user` / `wazuh_group` | `wazuh` | Dono/grupo dos arquivos gerados |
| `wazuh_service_name` | `wazuh-manager` | Nome do serviço systemd |

A role falha explicitamente (`assert`) se `abuseipdb_source_base_url` não
for definida, para evitar rodar contra uma URL vazia.

`abuseipdb_list_name` (nome do arquivo `.txt` local e a entrada em
`ossec.conf`/`local_rules.xml`) é **derivado automaticamente** do último
trecho da URL — você não precisa informá-lo. Só sobrescreva manualmente se
a URL não terminar exatamente no nome que você quer usar localmente.

### Exemplo de valores

Uma fonte pública de blocklists compatível com este formato é o repositório
[borestad/blocklist-abuseipdb](https://github.com/borestad/blocklist-abuseipdb),
que agrega denúncias do AbuseIPDB por janela de tempo (`1d`, `7d`, `14d`,
`30d`, `90d`) e nível de confiança. Para consumir a lista de 7 dias com
confiança ~100%, por exemplo:

```yaml
abuseipdb_source_base_url: "https://raw.githubusercontent.com/borestad/blocklist-abuseipdb/refs/heads/main/abuseipdb-s100-7d.ipv4"
```

Isso é apenas um exemplo — use a URL completa da blocklist que você
realmente pretende consumir.

## Reutilizando para múltiplas listas

Dá para chamar a role mais de uma vez no mesmo playbook para consumir
várias blocklists de uma vez, desde que cada chamada use uma URL e um
`custom_abuseipdb_srcip`/`custom_abuseipdb_dstip` diferentes (evite colisão com os
IDs `100010`/`100011` usados pela primeira chamada):

```yaml
- hosts: wazuh_managers
  become: true
  roles:
    - role: wazuh_abuseipdb_cdb
      vars:
        abuseipdb_source_base_url: "<URL_COMPLETA_DO_ARQUIVO_1>"
        custom_abuseipdb_srcip: "100010"
        custom_abuseipdb_dstip: "100011"

    - role: wazuh_abuseipdb_cdb
      vars:
        abuseipdb_source_base_url: "<URL_COMPLETA_DO_ARQUIVO_2>"
        custom_abuseipdb_srcip: "100012"
        custom_abuseipdb_dstip: "100013"
```

## Observações importantes

- **Compilação da lista CDB**: o Wazuh Manager compila a lista `.txt`
  automaticamente ao reiniciar — não há (nem existe) um binário para
  compilar manualmente. A entrada em `ossec.conf`/`local_rules.xml`
  referencia o próprio arquivo `.txt`, e não um `.cdb` gerado à parte.
  Ref.: [documentação oficial](https://documentation.wazuh.com/current/user-manual/ruleset/cdb-list.html).
- **`lookup` da regra**: como o campo filtrado é um endereço IP (`srcip`),
  a regra usa `lookup="address_match_key"` (que entende notação CIDR), e
  não `match_key` (comparação de string exata, usada para hashes/usuários).
- **Comentário inline nas fontes de blocklist**: algumas listas (ex.:
  `borestad/blocklist-abuseipdb`) trazem um comentário `# país AS...` no
  fim de cada linha de dado, não só no cabeçalho. A formatação remove tudo
  a partir do primeiro `#` em cada linha (não só linhas inteiras de
  comentário), senão o comentário vira parte da chave e a lista nunca bate
  com nenhum IP real extraído dos logs.
- **`srcip` é um campo estático**: a regra usa somente
  `<list field="srcip" lookup="address_match_key">` como condição de
  lookup. Uma tag `<field name="srcip">` (usada em versões anteriores
  deste projeto) quebra o carregamento do ruleset com o erro `Field
  'srcip' is static` — fatal, derruba o `wazuh-analysisd` e impede o
  `wazuh-manager` de subir. A tag `<srcip>` também não serve como
  substituto genérico, pois só aceita IP/CIDR literal, não regex.
- **A regra precisa de `<if_group>` para ser avaliada**: no Wazuh, uma
  regra só é testada se estiver encaixada na árvore de avaliação — via
  `<if_sid>`, `<if_group>`, `<category>` ou `<decoded_as>`. Sem nenhuma
  dessas tags, a regra nunca chega a rodar (mesmo sendo XML 100% válido,
  sem erro nenhum no log), então a busca na lista CDB nunca dispara. A
  role usa `custom_rule_if_group` (padrão `pfsense`) para isso — **ajuste
  essa variável para o grupo correto da sua fonte de logs** se não for
  pfSense/OPNsense. Para descobrir o grupo certo no seu ambiente, rode:
  ```bash
  sudo /var/ossec/bin/wazuh-logtest
  ```
  cole um log real da fonte que você quer monitorar, e veja o campo
  `groups` na Fase 3 (regra que casar primeiro) — um desses grupos é o
  valor a usar em `custom_rule_if_group`.
- **Entrada vs. saída (`srcip` vs. `dstip`)**: a role cria duas regras
  irmãs — uma verifica o IP de **origem** (`srcip`, id `custom_abuseipdb_srcip`,
  conexões chegando de um IP malicioso) e outra o IP de **destino**
  (`dstip`, id `custom_abuseipdb_dstip`, conexões saindo para um IP
  malicioso — útil para detectar hosts internos comprometidos). Um
  `<list>` só verifica um campo por vez, por isso são duas regras, não
  uma. Defina `custom_abuseipdb_dstip: ""` se só quiser a de entrada.
- **URL da fonte**: `abuseipdb_source_base_url` é obrigatória e não vem
  preenchida por padrão — defina-a com a URL completa da blocklist que
  você deseja consumir (veja [Exemplo de valores](#exemplo-de-valores)).
- **Local da regra em `local_rules.xml`**: a task usa `blockinfile` com
  `insertbefore: "{{ custom_rule_group_marker }}"` (por padrão `</group>`).
  Se o arquivo tiver **múltiplos** blocos `<group>`, ajuste
  `custom_rule_group_marker` para algo mais específico (ex.: um comentário
  único antes do fechamento do grupo desejado), para não inserir a regra no
  grupo errado.
- **Atualização periódica**: para manter a lista sempre atualizada, agende
  este playbook via `cron`/`systemd timer` no controlador Ansible, ou use
  AWX/Tower/AAP/SemaphoreUI com um schedule.
- **Segurança**: a role baixa conteúdo de uma URL externa e o compila
  diretamente na configuração do Wazuh. Garanta que `abuseipdb_source_url`
  seja uma fonte confiável (idealmente HTTPS com verificação de certificado,
  que é o padrão do módulo `get_url`).

## Testando antes de aplicar em produção

```bash
# Sintaxe
ansible-playbook -i inventory.ini site.yml --syntax-check

# Lint (requer ansible-lint instalado)
ansible-lint roles/wazuh_abuseipdb_cdb

# Execução limitada a um único host
ansible-playbook -i inventory.ini site.yml --limit wazuh-manager-01
```

## Usando com SemaphoreUI

Se você já tem uma instância do SemaphoreUI rodando e quer executar este
projeto nela (com histórico de execuções, formulário de variáveis e
agendamento cron), veja o passo a passo e os arquivos de exemplo em
[`semaphoreui/`](semaphoreui/README.md): como cadastrar Project,
Repository, Inventory, Variable Group e Task Template.
