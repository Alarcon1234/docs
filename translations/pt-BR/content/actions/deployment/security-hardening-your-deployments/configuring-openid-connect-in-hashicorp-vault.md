---
title: Configurando o OpenID Connect no HashiCorp Vault
shortTitle: Configurando o OpenID Connect no HashiCorp Vault
intro: Use o OpenID Connect nos seus fluxos de trabalho para efetuar a autenticação com o HashiCorp Vault.
miniTocMaxHeadingLevel: 3
versions:
  fpt: '*'
  ghae: issue-4856
  ghec: '*'
  ghes: '>=3.5'
type: tutorial
topics:
  - Security
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

## Visão Geral

O OpenID Connect (OIDC) permite aos seus fluxos de trabalho de {% data variables.product.prodname_actions %} efetuar a autenticação com um HashiCorp Vault para recuperar segredos.

Este guia fornece uma visão geral sobre como configurar o cofre HashiCorp para confiar no OIDC de {% data variables.product.prodname_dotcom %} como uma identidade federada e mostra como usar essa configuração na ação [hashicorp/vault-action](https://github.com/hashicorp/vault-action) para recuperar segredos do cofre HashiCorp.

## Pré-requisitos

{% data reusables.actions.oidc-link-to-intro %}

{% data reusables.actions.oidc-security-notice %}

## Adicionando o provedor de identidade ao HashiCorp Vault

Para usar OIDC com oHashiCorp Vault, você deverá adicionar uma configuração de confiança ao provedor do OIDC de {% data variables.product.prodname_dotcom %}. Para obter mais informações, consulte a [documentação](https://www.vaultproject.io/docs/auth/jwt) do HashiCorp Vault.

Para configurar o seu servidor Vault para aceitar o JSON Web Tokens (JWT) para autenticação:

1. Habilite o método de autenticação `JWT` e use `gravar` para aplicar a configuração do seu Vault. Para os parâmetros `oidc_discovery_url` e `bound_issuer`, use {% ifversion ghes %}`https://HOSTNAME/_services/token`{% else %}`https://token.actions.githubusercontent.com`{% endif %}. Estes parâmetros permitem que o servidor do Vault verifique os JSON Web Tokens recebidos (JWT) durante o processo de autenticação.

    ```sh{:copy}
    vault auth enable jwt
    ```

    ```sh{:copy}
    vault write auth/jwt/config \
      bound_issuer="{% ifversion ghes %}https://HOSTNAME/_services/token{% else %}https://token.actions.githubusercontent.com{% endif %}" \
      oidc_discovery_url="{% ifversion ghes %}https://HOSTNAME/_services/token{% else %}https://token.actions.githubusercontent.com{% endif %}"
    ```
2. Configure uma política que concede apenas acesso aos caminhos específicos que seus fluxos de trabalho serão usados para recuperar segredos. Para políticas mais avançadas, consulte o as [Políticas documentação](https://www.vaultproject.io/docs/concepts/policies) do HashiCorp Vault.

    ```sh{:copy}
    vault policy write myproject-production - <<EOF
    # Read-only permission on 'secret/data/production/*' path

    path "secret/data/production/*" {
      capabilities = [ "read" ]
    }
    EOF
    ```
3. Configure funções para agrupar diferentes políticas. Se a autenticação for bem-sucedida, essas políticas serão anexadas ao token de acesso do Vault resultante.

    ```sh{:copy}
    vault write auth/jwt/role/myproject-production -<<EOF
    {
      "role_type": "jwt",
      "user_claim": "actor",
      "bound_claims": {
        "repository": "user-or-org-name/repo-name"
      },
      "policies": ["myproject-production"],
      "ttl": "10m"
    }
    EOF
    ```

- `ttl` define a validade do token de acesso resultante.
- Certifique-se de que o parâmetro `bound_claims` está definido para seus requisitos de segurança e tem pelo menos uma condição. Opcionalmente, você também pode definir os partâmetros `bound_subject` e `bound_audiences`.
- Para verificar reivindicações arbitrárias na carga do JWT recebido, o parâmetro `bound_claims` contém um conjunto de reivindicações e seus valores obrigatórios. No exemplo acima, a função vai aceitar qualquer solicitação de autenticação de entrada do repositório `repo-name` pertencente à conta `user-or-org-name`.
- Para ver todas as reivindicações disponíveis compatíveis com o provedor do OIDC do {% data variables.product.prodname_dotcom %}, consulte ["Configurando a confiança do OIDC com a nuvem"](/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#configuring-the-oidc-trust-with-the-cloud).

Para obter mais informações, consulte a [documentação](https://www.vaultproject.io/docs/auth/jwt) do HashiCorp Vault.

## Atualizar o seu fluxo de trabalho de {% data variables.product.prodname_actions %}

Para atualizar seus fluxos de trabalho para o OIDC, você deverá fazer duas alterações no seu YAML:
1. Adicionar configurações de permissões para o token.
2. Use a ação [`hashicorp/vault-ação`](https://github.com/hashicorp/vault-action) para trocar o token do OIDC (JWT) por um token de acesso na nuvem.


Para adicionar a integração do OIDC a seus fluxos de trabalho que lhes permitem acessar os segredos no Vault, você deverá adicionar as seguintes alterações de código:

- Conceder permissão para obter o token do provedor do OIDC de {% data variables.product.prodname_dotcom %}:
  - O fluxo de trabalho precisa de configurações de `permissions:` com o valor `id-token` definido como `write`. Isso permite obter o token do OIDC de cada trabalho do fluxo de trabalho.
- Solicite o JWT do provedor do OIDC {% data variables.product.prodname_dotcom %} e apresente-o ao HashiCorp Vault para receber um token de acesso:
  - Você pode usar a ação [`hashicorp/vault-ação`](https://github.com/hashicorp/vault-action) para buscar o JWT e receber o token de acesso do Vault, ou você poderia usar o [conjunto de ferramentas de Ações](https://github.com/actions/toolkit/) para buscar os tokens para o seu trabalho.

Este exemplo demonstra como usar o OIDC com a ação oficial para solicitar um segredo ao HashiCorp Vault.

### Adicionando configurações de permissões

 {% data reusables.actions.oidc-permissions-token %}

{% note %}

**Observação**:

Quando a chave `permissões` é usada, todas as permissões não especificadas estão configuradas como _sem acesso_, com exceção do escopo de metadados, que sempre recebe acesso de _leitura_. Como resultado, você pode precisar adicionar outras permissões, como `contents: read`. Consulte [Autenticação Automática de Tokens](/actions/security-guides/automatic-token-authentication) para mais informações.

{% endnote %}

### Solicitando o token de acesso

A ação `hashicorp/vault-ação` recebe um JWT do provedor de OIDC de {% data variables.product.prodname_dotcom %} e, em seguida, solicita um token de acesso da sua instância do HashiCorp Vault para recuperar segredos. Para obter mais informações, consulte a documentação do [HashiCorp Vault do GitHub Actions](https://github.com/hashicorp/vault-action).

Este exemplo demonstra como criar um trabalho que solicita um segredo para o HashiCorp Vault.

- `<Vault URL>`: Substitua isso pela URL do seu HashiCorp Vault.
- `<Vault Namespace>`: Substitu-o pelo Namespace que você colocou no HashiCorp Vault. Por exemplo: `administrador`.
- `<Role name>`: Substitua isso pela função que você definiu no relacionamento de confiança do HashiCorp Vault.
- `<Secret-Path>`: Substitua isso pelo caminho para o segredo que você está recuperando do HashiCorp Vault. Por exemplo: `secret/data/production/ci npmToken`.

```yaml{:copy}
jobs:
  retrieve-secret:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Retrieve secret from Vault
        uses: hashicorp/vault-action@v2.4.0
          with:
            method: jwt
            url: <Vault URL>
            namespace: <Vault Namespace - HCP Vault and Vault Enterprise only>
            role: <Role name>
            secrets: <Secret-Path>

      - name: Use secret from Vault
        run: |
          # This step has access to the secret retrieved above; see hashicorp/vault-action for more details.
```

{% note %}

**Observação**:

- Se o seu servidor do Vault não puder ser acessado a partir da rede pública, considere usar um executor auto-hospedado com outros [auth methods](https://www.vaultproject.io/docs/auth) disponíveis para o Vault. Para obter mais informações, consulte "[Sobre os executores auto-hospedados](/actions/hosting-your-own-runners/about-self-hosted-runners)."
- `<Vault Namespace>` deve ser definido para uma implantação da empresa do Vault (incluindo HCP Vault). Para obter mais informações, consulte [Namespace do Vault](https://www.vaultproject.io/docs/enterprise/namespaces).

{% endnote %}

### Revogando o token de acesso

Por padrão, o servidor do Vault irá revogar automaticamente os tokens de acesso quando seu TTL vencer, portanto, você não precisa revogar manualmente os tokens de acesso. No entanto, se você deseja revogar os tokens de acesso imediatamente após o seu trabalho terminar ou falhar, você poderá revogar manualmente o token emitido usando a API do Vault [](https://www.vaultproject.io/api/auth/token#revoke-a-token-self).

1. Defina a opção `exportToken` como `verdadeiro` (padrão: `falso`). Isso exporta o token de acesso do Vault emitido como uma variável de ambiente: `VAULT_TOKEN`.
2. Adicione um passo para chamar a API do Vault [Revogar um Token (Self)](https://www.vaultproject.io/api/auth/token#revoke-a-token-self) para revogar o token de acesso.

```yaml{:copy}
jobs:
  retrieve-secret:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Retrieve secret from Vault
        uses: hashicorp/vault-action@v2.4.0
          with:
            exportToken: true
            method: jwt
            url: <Vault URL>
            role: <Role name>
            secrets: <Secret-Path>

      - name: Use secret from Vault
        run: |
          # This step has access to the secret retrieved above; see hashicorp/vault-action for more details.

      - name: Revoke token
        # This step always runs at the end regardless of the previous steps result
        if: always()
        run: |
          curl -X POST -sv -H "X-Vault-Token: {% raw %}${{ env.VAULT_TOKEN }}{% endraw %}" \
            <Vault URL>/v1/auth/token/revoke-self
```
