# Cobrança de XML pendente via WhatsApp (n8n + Evolution API)

Workflow do [n8n](https://n8n.io) que lê uma planilha no SharePoint/OneDrive e envia pelo WhatsApp um lembrete a cada cliente que ainda não mandou o XML do mês. Toda segunda-feira ele também manda a um grupo de WhatsApp a lista de quem está pendente.

```
Microsoft Graph (certificado) → SharePoint (planilha .xlsx/.xlsm) → n8n → Evolution API → WhatsApp
```

## O que ele faz

| Quando | O quê |
| --- | --- |
| Dias 1 a 15, às 9h | Envia o lembrete aos clientes pendentes |
| Dias 6 a 15, às 16h | Segundo lembrete do dia |
| Segunda-feira, às 9h | Envia ao grupo o relatório de pendentes |

- O texto muda com o prazo: dias 1–5 (pedido), 6–10 (lembrete) e 11–15 (último aviso).
- Envia um cliente por vez, com 30 s de pausa entre as mensagens.
- Tem modo de teste: todas as mensagens vão para um único número, com o prefixo `[TESTE → número]`.
- A autenticação na Microsoft usa um **certificado**, sem client secret. A apikey da Evolution fica numa **credencial do n8n**, fora do JSON.

## Requisitos

- n8n 1.x ou 2.x (testado no 2.x, que usa **Publish**), com `NODE_FUNCTION_ALLOW_BUILTIN=fs,crypto`.
- [Evolution API](https://github.com/EvolutionAPI/evolution-api) com uma instância conectada ao WhatsApp.
- Um app registrado no Microsoft Entra ID, com um certificado e a permissão `Sites.Selected` (Application).
- Uma planilha com a aba `Contadores` e as colunas abaixo.

## Planilha

| EMPRESA | WHATSAPP | JANEIRO | FEVEREIRO | … | DEZEMBRO |
| --- | --- | --- | --- | --- | --- |
| Cliente A | 21999999999 | 1 | | … | |
| Cliente B | 21988888888 / 21977777777 | | | … | |

- Coluna do mês **atual** com `1`, `SIM` ou `VERDADEIRO` = já enviou. Vazia, `0`, `FALSO` ou `NÃO` = pendente. Outro valor = pulado e listado em "Verificar na planilha".
- O XML cobrado é o do mês anterior: em novembro, a mensagem cita outubro.
- Vários números na mesma célula vão separados por `/`. O `55` é acrescentado quando falta.
- Linhas cuja EMPRESA começa com "Pendentes" ou "Enviados no mês" (totais) são ignoradas.

## Instalação

### 1. Certificado e app no Entra ID

Gere um certificado autoassinado (PowerShell):

```powershell
$cert = New-SelfSignedCertificate -Subject "CN=n8n-cobranca" -CertStoreLocation "Cert:\CurrentUser\My" `
  -KeyExportPolicy Exportable -KeySpec Signature -KeyLength 2048 -HashAlgorithm SHA256 -NotAfter (Get-Date).AddYears(2)
Export-Certificate -Cert $cert -FilePath .\n8n-cobranca.cer
$pwd = Read-Host -AsSecureString "Senha do .pfx"
Export-PfxCertificate -Cert $cert -FilePath .\n8n-cobranca.pfx -Password $pwd
$cert.Thumbprint
```

Converta a chave para PEM e copie para o container do n8n:

```bash
openssl pkcs12 -in n8n-cobranca.pfx -nocerts -nodes -out chave-privada.pem
docker cp chave-privada.pem <container-n8n>:/home/node/.n8n/chave-privada.pem
docker exec -u root <container-n8n> sh -c "chown node:node /home/node/.n8n/chave-privada.pem && chmod 600 /home/node/.n8n/chave-privada.pem"
```

Depois apague o `.pfx` e o `.pem` locais.

No Entra ID:

1. **App registrations → New registration.**
2. Em **Certificates & secrets → Certificates**, suba o `.cer`.
3. Em **API permissions**, adicione `Microsoft Graph → Application → Sites.Selected` e clique em **Grant admin consent**.
4. Conceda leitura **só no site da planilha** (por exemplo, pelo Graph Explorer):

```http
POST https://graph.microsoft.com/v1.0/sites/{site-id}/permissions
Content-Type: application/json

{
  "roles": ["read"],
  "grantedToIdentities": [{ "application": { "id": "SEU_CLIENT_ID", "displayName": "n8n - Cobrança" } }]
}
```

O `site-id` sai de `GET https://graph.microsoft.com/v1.0/sites/{tenant}.sharepoint.com:/sites/{nome-do-site}`.

### 2. Importar o workflow

1. No n8n: **Workflows → Import from File → `cobranca-xml-n8n.json`**.
2. Crie uma credencial **Header Auth**: Name `apikey`, Value = apikey da Evolution. Selecione-a nos nós **Enviar WhatsApp**, **Enviar relatório** e **Enviar no grupo**.
3. Troque os valores de exemplo:

| Nó | Campo | Valor |
| --- | --- | --- |
| Gerar JWT | `TENANT_ID`, `CLIENT_ID`, `THUMBPRINT`, `KEY_PATH` | Dados do app e do certificado |
| Ler planilha | URL | `SEU_SITE_ID` e o caminho do arquivo (espaços como `%20`) |
| Montar lista de cobrança | `INSTANCIA`, `NUMERO_TESTE`, texto `[SUA EMPRESA]` | Instância da Evolution e número de teste |
| Montar relatório semanal | `INSTANCIA`, `GRUPO_ID` | ID do grupo (`GET /group/fetchAllGroups/{instancia}?getParticipants=false`) |

O `EVOLUTION_URL` vem como `http://evolution-api:8080`, que é o endereço da Evolution na rede Docker. Ajuste se ela estiver em outro lugar.

### 3. Testar e publicar

O arquivo vem em **modo teste** (`MODO_TESTE = true`, `IGNORAR_DATA_HORA = true`, `LIMITE_TESTE = 3`).

1. **Cobrança:** rode **Execute workflow from Teste manual**. Até 3 mensagens chegam no `NUMERO_TESTE`.
2. **Relatório:** na seta do botão **Execute workflow**, escolha **Toda segunda 9h** e execute. Só o grupo recebe; a cobrança sai com 0 itens.
3. No nó **Montar lista de cobrança**, mude `MODO_TESTE` e `IGNORAR_DATA_HORA` para `false`.
4. Clique em **Publish**. Se possível, configure um **Error Workflow** em Settings.

## Configuração (nó Montar lista de cobrança)

| Constante | Produção | O que faz |
| --- | --- | --- |
| `MODO_TESTE` | `false` | `true` envia tudo para o `NUMERO_TESTE` |
| `IGNORAR_DATA_HORA` | `false` | `false` só envia nos dias 1–15, das 9h às 17h |
| `LIMITE_TESTE` | — | Máximo de mensagens no modo teste (0 = sem limite) |
| `RELATORIO_POR_ENVIO` | `false` | `true` manda um resumo ao `NUMERO_TESTE` depois de cada disparo |

## Problemas comuns

| Sintoma | Causa provável |
| --- | --- |
| `Cannot find module 'fs'` | Falta `NODE_FUNCTION_ALLOW_BUILTIN=fs,crypto` no n8n |
| `ENOENT` / `EACCES` no Gerar JWT | `.pem` ausente ou sem permissão para o usuário `node` |
| `AADSTS700027` | Certificado vencido ou thumbprint errado |
| 403 no Ler planilha | Permissão do site não concedida ou sem admin consent |
| `Coluna … não encontrada na planilha` | Falta a coluna do mês atual |
| 401 nos nós de envio | Nó sem a credencial Header Auth, ou Name diferente de `apikey` |
| Relatório não chega no grupo | O número da instância não participa do grupo |

## Segurança

- Nunca versione o `.pem`, o `.pfx`, o `.env` ou a apikey. O `.gitignore` já bloqueia esses arquivos.
- Use `Sites.Selected` com papel `read` em vez de `Sites.Read.All`: o app lê só o site da planilha.
- O export do n8n não leva os valores das credenciais, mas confira o JSON antes de cada commit.
- Lembre de renovar o certificado antes de ele vencer.

## Licença

MIT
