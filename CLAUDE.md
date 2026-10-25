# CLAUDE.md

Este arquivo orienta o Claude Code (claude.ai/code) ao trabalhar com o código deste repositório.

## O que é

SisVip ("SisVip — Atendimentos") é um sistema de gestão em página única: agenda/atendimentos, clientes, operadores, caixa/PDV, catálogo de produtos com catálogos fiscais, boletos/PIX, dashboard de indicadores e WhatsApp via Digisac. **O app inteiro está em um único arquivo, `index.html`** (~8,6 mil linhas de HTML/CSS/JS inline). Não há build, gerenciador de pacotes, lint nem testes. Textos da interface, identificadores do código e mensagens de commit são em português.

## Rodar

Sirva a pasta como estática (configuração em `.claude/launch.json`):

```
python -m http.server 8123
```

e abra http://localhost:8123. Abrir o `index.html` direto no navegador também funciona quase sempre, mas as chamadas `fetch` externas (ViaCEP, BrasilAPI/CNPJ, Digisac e o worker de sincronização) se comportam melhor via http.

As dependências externas vêm só de CDNs: `qrcodejs` (QR codes do PIX) e `@emailjs/browser` (e-mail de recuperação de senha).

## Arquitetura

- **Vários blocos `<script>` em sequência, todos compartilhando o `window`.** Os módulos foram sendo adicionados como blocos `<script>` separados, que se comunicam por variáveis globais (`window.records`, `window.clientes`, `window.operadores`, `window.lancamentos`, `window.historyLog`, `window.appConfig`, `window.produtos` etc.). Blocos posteriores muitas vezes alteram ou embrulham funções definidas antes. Ao adicionar código, coloque no `window` todo estado que o resto do app precisa enxergar. Um bug antigo ("taxa de entrega não atualizava o total do carrinho") vinha justamente de um valor que ficou no escopo local de um bloco em vez de ficar no `window`.
- **Navegação:** as páginas são seções `<div id="page-<nome>">`, trocadas por `switchPage(id)`. O `switchPage` foi sobrescrito várias vezes. Perto da seção "CONSOLIDAR switchPage" existe um mapa `_allPageHandlers` (`lista`, `novo`, `agenda`, `clientes`, `operadores`, `painel`, `historico`, `dashboard`, `permissoes` etc.) que chama a função `render*` de cada página. Registre páginas novas nesse mapa, em vez de criar mais um wrapper de `switchPage`.
- **A persistência é só em localStorage** (por navegador/aparelho; não há banco de dados no servidor). O bloco "PERSISTÊNCIA LOCAL" define `window.sisvipSave()` / `window.sisvipLoad()` para as coleções principais (`sisvip_records`, `sisvip_clientes`, `sisvip_operadores`, `sisvip_lancamentos`, `sisvip_history`, `sisvip_config`, `sisvip_version`). Outros módulos usam chaves `sisvip_*` próprias (`sisvip_produtos`, `sisvip_catalogos`, `sisvip_caixa_sessoes`, `sisvip_senhas`, `sisvip_*_cfg` para integrações e flags `*_seed` para popular catálogos uma única vez). Chame `sisvipSave()` depois de alterar as coleções principais. O backup/restauração (protegido por senha, inclui as credenciais do Digisac/Sicoob) precisa ser atualizado quando novas chaves forem criadas.
- **Sincronização automática com o Digisac:** os webhooks do Digisac chegam a um Cloudflare Worker (`window.SYNC_WORKER_URL`, `sisvip-digisac-sync.kleber-netfix.workers.dev`, fora deste repositório). O app consulta esse worker periodicamente e transforma os eventos em Atendimentos. A diferença entre `sisvip_sync_last` ser `null` ou `'0'` importa (aparelho que nunca sincronizou × reprocessamento proposital do histórico); veja `digisacSyncUltimoTimestamp` / `digisacSyncPoll`.
- **Os modais não podem fechar com clique fora deles** (regra proposital em todo o projeto: clicar no fundo chegou a apagar vendas em andamento). Mantenha esse comportamento em modais novos; feche por botões explícitos ou por `fecharModal`.
- **Impressão** (boletos, extratos, relatórios, comandas/pré-contas em térmica de 80 mm) é feita renderizando uma visão limpa só para impressão. Correções anteriores trataram de garantir que só o conteúdo imprimível saia na impressão.

## Trabalhando no arquivo

O `index.html` é grande: use Grep para achar o bloco `<script>` ou a seção `id="page-…"` / `id="modal-…"` relevante e leia só esse trecho. Cuidado com funções duplicadas em blocos diferentes (vale a última carregada); confira com Grep antes de adicionar ou editar uma função.
