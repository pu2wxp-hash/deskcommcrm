# RKHL CRM — o fork do DeskcommCRM

Este repositório (`pu2wxp-hash/deskcommcrm`) é um fork do
[DeskcommCRM](https://github.com/melgarafael/DeskcommCRM), de Rafael Melgaço, sob a
licença MIT (`LICENSE`, mantida como veio). Ele roda o **RKHL CRM**
(crm.rkhl.com.br) com as imagens construídas aqui.

A especificação que decide tudo abaixo é a "RKHL CRM — Especificação F1 — Fork e
esteira de atualização" (cérebro, `01 Projetos/RKHL CRM/Especificações/`).

## Ramos

| Ramo | Para quê |
| --- | --- |
| `rkhl` (padrão) | A versão do autor + as nossas mudanças. Toda mudança nossa entra por **PR em `rkhl`**; ninguém empurra direto. |
| `main` | **Não é usada e não é sincronizada.** Nada entra nela, para não disparar os workflows do autor. |
| `sync/vX.Y.Z` | Aberto pelo `rkhl-sincronizar.yml`: a release `vX.Y.Z` do autor juntada em `rkhl`. |

As tags do autor **não** existem aqui. O fork só tem as tags que o
`rkhl-release.yml` cria.

## Versões

A versão do fork tem **o mesmo número** do autor: `vX.Y.Z` do fork é o commit de
`rkhl` que contém a release `vX.Y.Z` do autor, mais as nossas mudanças. É o único
formato que o kit (`hostgator-setup-kit/`) aceita sem mudar nada nele:

- `ultima_release_estavel` lê `/releases/latest` do `origin`;
- `gravar_imagens` grava `ghcr.io/pu2wxp-hash/<imagem>:X.Y.Z` (sem o "v") nas 4 imagens;
- `trio_publicado` exige as 4 imagens públicas;
- `_runtime.sh` mostra a versão como verificada quando a imagem tem `APP_VERSION=X.Y.Z` e
  o label `org.opencontainers.image.revision` igual ao commit da tag.

As nossas mudanças saem junto com a próxima release do autor.

## O que o fork muda no código do autor

Só a lista do `RECADO_AO_FORK` (`tests/unit/namespace-das-imagens.test.ts`), apontando
para `ghcr.io/pu2wxp-hash` e `github.com/pu2wxp-hash/DeskcommCRM`:

- `IMG_NS` e a URL de `ultima_versao_publicada` em `hostgator-setup-kit/_common.sh`;
- `REPO_URL` em `hostgator-setup-kit/install.sh` e `comecar.sh`;
- o default das linhas `image:` das nossas imagens em `docker-compose.prod.yml` (quatro na v1.120.0, com a do `voice-agent`; o recado fala em três porque a do `voice-agent` saiu do compose depois);
- as três `*_IMAGE` de `.env.hostgator.example`;
- `NAMESPACE_DESTE_REPO` em `tests/unit/_identidade-deste-repo.ts`;
- o `LABEL org.opencontainers.image.source` dos 4 Dockerfiles.

Fora disso, só este arquivo e os três workflows `.github/workflows/rkhl-*.yml`. Para
conferir (troque `vX.Y.Z` pela versão do autor em que o `rkhl` está):

```bash
git fetch --no-tags https://github.com/melgarafael/DeskcommCRM.git "+refs/tags/vX.Y.Z:refs/autor/vX.Y.Z"
git diff --stat refs/autor/vX.Y.Z origin/rkhl -- . ':!RKHL.md' ':!.github/workflows/rkhl-*.yml'
```

## Os workflows

| Workflow | Quando | O que faz |
| --- | --- | --- |
| `rkhl-sincronizar.yml` | Segunda, 6h de Brasília, e manual | Escolhe a última release estável do autor com **7 dias ou mais** e abre o PR `sync/vX.Y.Z` → `rkhl` com o CHANGELOG do intervalo, os arquivos alterados em caminhos sensíveis e o alerta de SQL destrutivo. Com conflito, abre o PR marcado **"CONFLITO — PRECISA DO CODE"**, sem resolver nada (o ramo é o commit puro do autor). |
| `rkhl-ci.yml` | Todo PR para `rkhl` | typecheck, lint, testes unitários (7 partes) e build. É o portão do PR de sync. |
| `rkhl-release.yml` | Merge de um PR `sync/vX.Y.Z`, ou manual com o número (primeira versão) | Confere tudo, constrói e publica as 4 imagens (só **amd64**), confere de fora, sem login, que estão públicas, e só então cria a tag e a release com os **digests**. Também aponta `stable` para a versão. |

**As imagens saem como índice.** Cada imagem é construída só para `linux/amd64` e
publicada primeiro em `X.Y.Z-amd64`. Depois, a tag `X.Y.Z` vira um **índice** (OCI
image index) com essa única entrada, como o `juntar-manifestos` do autor faz com
amd64 + arm64.
- **Por que o índice:** sem ele, `X.Y.Z` seria um manifesto OCI de imagem única. O
  `ghcr_status` do kit pede só índice OCI, manifest list e manifesto docker v2, e o
  GHCR responde **404** para um manifesto OCI simples sem esses tipos. O pré-voo do
  `update.sh` na VPS faria o mesmo pedido e recusaria a atualização. Foi o que
  aconteceu no run 46008040971.
- **O que o job `publicar` confere antes de criar a tag e a release:**
  - que `X.Y.Z` é índice, só com `linux/amd64`, e aponta para a imagem construída;
  - `ghcr_status` = 200, `trio_publicado` e `preflight_imagens_crm X.Y.Z linux/amd64`;
  - `APP_VERSION` e `revision` puxados pela própria tag `X.Y.Z`.
- **O que a release registra:** o digest do índice e o da imagem amd64.
- As tags `X.Y.Z-amd64` ficam no GHCR, mas o kit não as usa.

**Juntar um PR de sync: sempre "Create a merge commit".** Squash ou rebase cortam a
história do autor, e a próxima sincronização daria conflito em tudo. O
`rkhl-release.yml` recusa publicar um commit que não contém a release do autor com
o mesmo número, ou que não tem as nossas mudanças (o `IMG_NS` do fork).

**Os workflows do autor ficam desligados no fork** (`gh workflow disable`):
`release.yml`, `publish-image.yml`, `relogio.yml`, `vigia-de-colisao.yml`, `e2e.yml`,
`perf.yml`, `mysql-integracao.yml` e `acolhida.yml`. Eles precisam de segredos do
autor, só servem à `main` dele ou publicariam código sem revisão. Um workflow
**novo** do autor nasce ligado no fork: o PR de sync avisa quando aparece um.

**Testes do autor sobre `.github/workflows`.**
- Dois testes do autor tratam a pasta como um conjunto fechado dele:
  - `workflows-tem-permissions` (escopos de escrita só com justificativa);
  - `gatilho-dos-jobs-de-entrega` (todo job num mapa).
- Os `rkhl-*.yml` não estão nesses mapas. Editar os mapas seria mudar teste do autor e dar conflito a cada sincronização.
- Por isso o `rkhl-ci` tira os `rkhl-*.yml` da pasta antes do `test:unit`: a suíte roda inteira, e esses dois testes seguem vigiando os workflows do autor como na casa dele.
- Para rodar igual na sua máquina: `mkdir -p /tmp/rkhl && mv .github/workflows/rkhl-*.yml /tmp/rkhl/ && pnpm test:unit; mv /tmp/rkhl/rkhl-*.yml .github/workflows/`.

### O segredo `RKHL_BOT_TOKEN`

Um token de acesso **fine-grained**, só deste repositório, com estas permissões de
repositório em **leitura e escrita**:
- Contents;
- Pull requests;
- Workflows.

Validade de 1 ano. Ele fica em Settings › Secrets and variables › Actions ›
`RKHL_BOT_TOKEN` e é usado **só** no `rkhl-sincronizar.yml` e no `rkhl-release.yml`.
O GitHub não deixa o `GITHUB_TOKEN` fazer isso:
- o que o `GITHUB_TOKEN` cria não dispara outros workflows, e o `rkhl-ci` não rodaria no PR;
- o `GITHUB_TOKEN` não pode empurrar mudanças em `.github/workflows`.

O GHCR continua sendo publicado pelo `GITHUB_TOKEN`, porque o token fine-grained não
tem acesso a pacotes. **Ao vencer, a sincronização para**: renove antes.

### Pacotes públicos

Pacote novo no GHCR nasce **privado**, e o repositório ser público não muda isso.
Na primeira publicação, o `rkhl-release.yml` para antes de criar a tag, com o aviso
de 403. Nessa hora, cada uma das 4 imagens precisa ser trocada uma vez para
**Public**:
- onde: github.com/users/pu2wxp-hash/packages/container/`<imagem>`/settings;
- as imagens: `deskcommcrm`, `deskcomm-worker`, `deskcomm-scheduler` e `deskcomm-voice-agent`.

Depois disso, rode de novo os jobs que falharam.

## Atualizar o servidor

Pelo **botão de atualização do painel do CRM** (`/admin`). Com o `origin` do servidor
no fork, o `agent.sh` (cron de root) oferece a última release do fork e roda o
`update.sh`, que faz backup. Quem pode clicar: só platform admin.

Antes de cada atualização:
1. **snapshot do VPS** no painel da Hostinger: é a volta atrás real, porque o
   `restore.sh` só restaura em banco vazio;
2. pelo menos **20 GB livres** no disco.

A ordem é: release publicada → snapshot → botão no painel → conferência.

## Voltar atrás

- O caminho principal é **restaurar o snapshot** pela Hostinger.
- O caminho sem snapshot vale só se o banco não foi afetado, e mantém o mesmo número com as imagens do autor:

```bash
cd /root/deskcommcrm
git remote remove origin                  # o fork
git remote rename upstream-autor origin   # o autor, guardado na virada
git tag -l 'v*' | xargs -r git tag -d
git fetch --tags origin
bash hostgator-setup-kit/update.sh --to vX.Y.Z --force
```

Toda troca de `origin` (a virada e a volta) apaga as tags locais e busca de novo.
O motivo: o `git fetch --tags` não sobrescreve uma tag local que já existe, e o
`update.sh` seguiria a tag velha.
