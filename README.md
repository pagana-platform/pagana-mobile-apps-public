# pagana-mobile-apps-public

Políticas de versão dos apps mobile da Pagana, publicadas num repositório público para que o gate de versão dos apps consiga lê-las sem autenticação.

## Para que serve

Os apps fazem, ao abrir (e a cada 5 minutos enquanto estão em uso), uma leitura anônima de um arquivo `versions.json` neste repositório. Com base nele, decidem se exibem o modal de atualização ou se bloqueiam o uso até o usuário atualizar.

O app é privado e não consegue servir arquivos "raw" sem token. Um repositório público resolve isso expondo apenas o que já é público: versões nas lojas, URLs das lojas e a mensagem do modal. Nenhum segredo, código ou dado de usuário vive aqui.

O histórico do Git funciona como trilha de auditoria de cada mudança de política.

## Estrutura

```
.
├── README.md
└── <appId>/
    ├── production/
    │   └── versions.json   # escrito automaticamente
    └── preview/
        └── versions.json   # escrito à mão, usado em HML
```

Um arquivo por app e por ambiente. Hoje:

| App       | Ambiente   | Arquivo                          | Quem escreve                     |
| --------- | ---------- | -------------------------------- | -------------------------------- |
| `voucher` | production | `voucher/production/versions.json` | Serviço (GitHubApiWriter); `appId: voucher-prod`    |
| `voucher` | preview    | `voucher/preview/versions.json`    | Manual (apenas para testes/HML); `appId: voucher-preview` |

## Formato do `versions.json`

```json
{
  "appId": "voucher-prod",
  "updatedAt": "2026-09-03T20:26:37.667Z",
  "platforms": {
    "ios": {
      "latestVersion": "1.4.0",
      "minimumVersion": "1.2.0",
      "forceUpdate": false,
      "message": "Uma nova versão está disponível.",
      "storeUrl": "https://apps.apple.com/app/..."
    },
    "android": {
      "latestVersion": "1.4.0",
      "minimumVersion": "1.2.0",
      "forceUpdate": false,
      "message": "Uma nova versão está disponível.",
      "storeUrl": "https://play.google.com/store/apps/details?id=..."
    }
  }
}
```

| Campo            | Descrição                                                                 |
| ---------------- | ------------------------------------------------------------------------- |
| `appId`          | Deve ser igual ao `VERSION_GATE_APP_ID` do build (`voucher-prod` em production, `voucher-preview` em preview). |
| `latestVersion`  | Última versão publicada na loja, sempre `major.minor.patch`.              |
| `minimumVersion` | Versão mínima aceita.                                                     |
| `forceUpdate`    | Ver semântica abaixo.                                                     |
| `message`        | Texto exibido no modal.                                                   |
| `storeUrl`       | URL da loja. **Obrigatório**: sem ele a entrada é tratada como malformada. |

### Semântica de bloqueio

- Versão instalada **abaixo de `minimumVersion`**: o app é **sempre bloqueado**.
- `forceUpdate: false`: versões entre `minimumVersion` e `latestVersion` veem apenas um aviso opcional.
- `forceUpdate: true`: o bloqueio se estende até `latestVersion`.

Como `latestVersion` sobe sozinho quando a loja publica uma nova versão, manter `forceUpdate: true` bloquearia todos os usuários a cada release. Use `true` apenas de forma deliberada e temporária.

## Como alterar uma política

A fonte da verdade é o `APPS_CONFIG` do serviço, não os arquivos deste repositório. A cada execução o serviço recalcula o documento a partir do `APPS_CONFIG` e só faz commit quando o conteúdo muda. Edições manuais em `production/` são desfeitas na próxima execução.

**Mudança de emergência** (exigir uma versão mínima mais alta):

1. Altere `minimumVersion` do app/plataforma no `MOBILE_VERSION_GATE_APPS_CONFIG` do serviço `mobile-version-gate` (srv-integrations). Ela não pode ser maior que a versão publicada na loja.
2. Rode o serviço com `--once` para publicar imediatamente.
3. Confirme o commit neste repositório.

**Testes em HML:** edite `<appId>/preview/versions.json` à mão (por exemplo, suba `minimumVersion` acima da versão instalada) e verifique o modal no app de preview. Esse arquivo não é tocado pelo serviço.

## Configuração no app

Production (EAS environment `production`):

```
VERSION_GATE_URL=https://raw.githubusercontent.com/pagana-platform/pagana-mobile-apps-public/refs/heads/main/voucher/production/versions.json
VERSION_GATE_APP_ID=voucher-prod
```

Preview / HML (EAS environment `preview`):

```
VERSION_GATE_URL=https://raw.githubusercontent.com/pagana-platform/pagana-mobile-apps-public/refs/heads/main/voucher/preview/versions.json
VERSION_GATE_APP_ID=voucher-preview
```

Sem `VERSION_GATE_URL` o gate fica desligado e o app abre normalmente.

## Limitações conhecidas

- **Cache:** `raw.githubusercontent.com` responde com `max-age=300` e ignora `Cache-Control: no-cache`. No pior caso, um bloqueio leva cerca de **10 minutos** para chegar ao usuário (5 min de CDN + 5 min do intervalo de revalidação do app).
- **Sem SLA:** o GitHub não garante disponibilidade nem documenta rate limit para o raw. O volume aqui é baixo (uma leitura por abertura/5 min por dispositivo).
- **Fail-open:** se o arquivo estiver indisponível, malformado ou sem `storeUrl`, o app **não bloqueia** o usuário (e registra um warning). Portanto, não prometa "bloqueio garantido".

## Permissões

- O serviço escreve via GitHub App ou PAT fine-grained restrito a este repositório, com `contents: write`.
- A branch `main` não tem proteção para o bot; a regra de que humanos não editam `production/` é por convenção.
