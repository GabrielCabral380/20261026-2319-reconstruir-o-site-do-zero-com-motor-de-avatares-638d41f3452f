# Plano de reconstrução do site com avatares interativos

## Situação e escopo

A inspeção da pasta atual em 26/10/2026 não encontrou arquivos, inclusive o HTML e CSS mencionados no pedido. Xavier será o único responsável por recuperar e adaptar esse material, se estiver disponível, ou recriar a página. Este documento e PARTES.json são os únicos artefatos desta etapa; nenhuma parte será implementada agora.

O resultado será um site responsivo em português com seleção de avatar, animações de repouso, saudação e celebração, controles por mouse e teclado e mensagens acessíveis de estado. A primeira versão usará avatares vetoriais SVG animados por JavaScript, sem download de modelos 3D. Um contrato de renderização permitirá substituir o renderer por um de modelos 3D posteriormente, sem alterar os eventos públicos.

## Tecnologias e execução

- HTML5, CSS3 e JavaScript com módulos ES nativos, sem framework, bundler ou dependências externas.
- SVG local para os personagens; Web Animations API para animações, com fallback estático quando indisponível.
- Node.js e seu executor nativo de testes para contratos e lógica. Os testes usarão doubles de DOM/renderer quando necessário, sem instalar bibliotecas.
- Servidor HTTP estático: `python -m http.server 8000 --bind 127.0.0.1`, executado na raiz, em PowerShell. A página será aberta em `http://127.0.0.1:8000/`; não usar `file://` para módulos.
- Antes da implementação, confirmar `node --version` (Node 20 ou superior) e `python --version`. Não pressupor Linux, WSL ou comandos Unix. A etapa de planejamento exige apenas escrita na pasta autorizada.

## Estrutura e propriedade exclusiva

```text
PLANO.md                          coordenação (entregue nesta etapa)
PARTES.json                       coordenação (entregue nesta etapa)
index.html                        Xavier
css/styles.css                    Xavier
js/core/avatar-engine.js          Xavier
tests/avatar-engine.test.mjs       Xavier
js/avatars/catalog.js             Maya
js/avatars/svg-renderer.js         Maya
assets/avatars/aurora.svg          Maya
assets/avatars/cosmo.svg           Maya
tests/avatar-renderer.test.mjs     Maya
js/app.js                         Gabriel
js/ui/avatar-controls.js          Gabriel
tests/avatar-controls.test.mjs    Gabriel
TESTES.md                         Gabriel
```

Não haverá package.json: os testes `.mjs` serão executados com `node --experimental-default-type=module --test tests/*.test.mjs`, para importar os módulos `.js` como ES modules. Cada teste deverá também ser executável individualmente, substituindo o glob pelo caminho do arquivo. O suporte a esse comando deve ser confirmado na versão de Node instalada antes de desenvolver.

## Quem faz o quê

1. **Maya — personagens, animações e renderer.** Define dois SVGs originais locais, catálogo e adaptador visual. Como o pedido não registra uma preferência técnica de Maya, ela fica com a camada visual e mantém a referência de coordenação descrita neste plano.
2. **Xavier — página e núcleo de eventos.** Recupera/adapta seu HTML e CSS ou os recria, além de implementar a máquina de estados e os eventos do motor. Isso respeita sua oferta de preparar a estrutura para receber modelos ou animações.
3. **Gabriel — integração e controles.** Conecta o catálogo, renderer, motor e página, cuida das interações e documenta a validação completa. Isso respeita sua preferência por integrar avatares e garantir interatividade.

Xavier e Gabriel propuseram tarefas próximas em JavaScript. A divisão separa infraestrutura reutilizável (Xavier) de uso da infraestrutura na página (Gabriel), evitando dois donos para a mesma lógica. Cada pessoa só edita os arquivos atribuídos acima e em PARTES.json. Alterações em contratos devem ser combinadas antes de serem aplicadas; ninguém modifica o arquivo de outra parte para resolver uma integração.

## Contratos fixos entre as partes

### Página e DOM — Xavier fornece, Gabriel consome

`index.html` carrega apenas `<script type="module" src="./js/app.js"></script>` e `./css/styles.css`. Fornece estes elementos estáveis:

- `#avatar-stage`: área de montagem do renderer, com nome acessível.
- `#avatar-select`: `<select>` com label associado; Gabriel preenche suas opções.
- `#action-wave` e `#action-celebrate`: botões nativos com texto em português.
- `#motion-toggle`: checkbox com label "Reduzir movimento"; marcado significa animação reduzida.
- `#avatar-status`: região textual com `role="status"` e `aria-live="polite"`.

Todos os controles começam desabilitados até a inicialização. A página inclui `<noscript>`. CSS não depende da estrutura interna dos SVGs e cobre foco visível, viewport de 320 px e telas maiores. A única rota é `/`; caminhos `/css/`, `/js/` e `/assets/` são arquivos estáticos, sem API, credenciais ou backend.

### Catálogo e renderer — Maya fornece

`catalog.js` exporta `AVATARS`, array não vazio de `{ id, name, src, actions }`. IDs únicos: `aurora` e `cosmo`; `name` é o nome exibido; `src` é uma URL absoluta calculada com `new URL('../../assets/avatars/aurora.svg', import.meta.url).href` (ou cosmo); `actions` é `['idle', 'wave', 'celebrate']`. O catálogo não toca no DOM.

`svg-renderer.js` exporta `createSvgRenderer({ container })`, que retorna:

- `load(avatar): Promise<void>`: carrega e apresenta o SVG local, cancela animações anteriores e rejeita falhas de rede/arquivo; mantém no máximo um avatar montado.
- `play(action, { reducedMotion }): Promise<void>`: `idle` apresenta repouso e resolve imediatamente; `wave` e `celebrate` são finitas, resolvendo ao terminar. Com movimento reduzido ou API indisponível, apresenta estado estático e resolve sem movimento contínuo.
- `destroy(): void`: idempotente, cancela animações, remove elementos e impede montagem tardia de carregamentos pendentes. Cancelamento rejeita a operação pendente com erro de nome `AbortError`.

O renderer não registra eventos globais nem altera controles. Os SVGs são locais, sem scripts ou referências remotas. Maya define sua estrutura interna para permitir animação e oferece representação estática legível.

### Motor — Xavier fornece

`avatar-engine.js` exporta `createAvatarEngine({ renderer, avatars, reducedMotion = false })`. Recebe o renderer pelo contrato acima, sem importá-lo, e retorna:

- `selectAvatar(id): Promise<void>`: rejeita ID desconhecido sem mudar o estado; carrega o avatar e entra em repouso.
- `perform(action): Promise<void>`: aceita `wave` e `celebrate` apenas quando pronto; executa e retorna a repouso. Rejeita ação desconhecida ou chamada enquanto ocupado, sem iniciar animação concorrente.
- `setReducedMotion(value): void`: atualiza a preferência booleana para a próxima animação; não reinicia a operação atual.
- `getState(): object`: devolve cópia de `{ status, avatarId, action, reducedMotion, error }`.
- `subscribe(listener): () => void`: entrega imediatamente o estado atual e depois cada alteração; retorna função de desinscrição. É o único canal de eventos públicos; não há CustomEvents ou nomes globais extras.
- `destroy(): void`: idempotente; encerra renderer, remove assinantes e ignora resultados assíncronos tardios.

Estados: `uninitialized → loading → ready → playing → ready`; falha operacional leva a `error`; destruição leva a `destroyed`. `avatarId` começa `null` e só muda após carga bem-sucedida; `action` é `idle`, `wave` ou `celebrate`, começando em `idle`. `error` é `null` ou `{ code, message }`, com códigos `LOAD_FAILED` e `PLAY_FAILED`. Carregamento/ação são mutuamente exclusivos: chamadas enquanto ocupado rejeitam com erro `BUSY`. Entradas inválidas rejeitam com `INVALID_AVATAR` ou `INVALID_ACTION`, uso antes de seleção com `NOT_READY`, e uso após destruição com `DESTROYED`. Esses erros de validação não alteram o estado. Uma nova seleção válida em `error` permite recuperação e limpa o erro. Promessas públicas rejeitadas são sempre tratadas por Gabriel.

### Integração — Gabriel fornece

`avatar-controls.js` exporta `bindAvatarControls({ root, engine, avatars })`, que preenche a seleção, registra listeners, assina estado e retorna uma função de limpeza idempotente. `root` é o documento ou um elemento contendo todos os seletores acordados. O estado controla botões, seleção, `aria-busy` da área visual e mensagens de status; nenhum HTML vindo de mensagens é injetado. Em `error`, a seleção fica habilitada para tentar novamente, enquanto ações ficam desabilitadas.

`app.js` importa os três fornecedores pelos caminhos acima, cria o renderer com `#avatar-stage`, inicializa o motor com `matchMedia('(prefers-reduced-motion: reduce)').matches`, conecta os controles e seleciona o primeiro avatar. O checkbox reflete a preferência inicial e permite alterá-la durante a sessão; mudanças do sistema após iniciar a página não precisam ser acompanhadas. Na saída da página, executa limpeza dos controles e destrói o motor. Se faltar um elemento obrigatório, mostra erro compreensível no status disponível e registra diagnóstico no console, sem deixar rejeição não tratada.

## Paralelismo e integração

As três partes começam ao mesmo tempo a partir destes contratos. Maya testa seu renderer com DOM e animações simuladas; Xavier testa o motor com renderer falso; Gabriel testa controles com engine e DOM falsos. Nenhuma parte precisa esperar um módulo alheio para desenvolver seus arquivos. Após os três entregarem, Gabriel executa os fluxos integrados e comunica defeitos ao dono do arquivo; cada dono corrige apenas sua parte. PLANO.md e PARTES.json permanecem sob coordenação, fora dos arquivos de implementação.

## Testes e critérios de conclusão

1. Executar os três arquivos de testes com o comando Node indicado, com saída sem falhas. Verificar transições do motor, bloqueio de concorrência, recuperação após falha, limpeza e resultados tardios; verificar animação reduzida e montagem única; verificar vinculação de controles, estado ocupado e tratamento de rejeições.
2. Servir a raiz via Python e abrir `/` em navegador atual. Confirmar que o primeiro avatar aparece, ambos podem ser selecionados e ambas as ações terminam retornando ao repouso. Não deve haver 404, erro no console ou rejeição não tratada no fluxo normal.
3. Navegar somente com Tab, Enter e Espaço: foco visível, seleção e ações funcionais. Confirmar labels, status anunciado e controles bloqueados durante operações.
4. Ativar redução de movimento no sistema antes de abrir a página e depois testar o checkbox. Não deve haver animação contínua no modo reduzido.
5. Testar largura de 320 px e desktop sem rolagem horizontal. Simular falha de carga bloqueando um SVG nas ferramentas do navegador, confirmar mensagem e recuperação ao selecionar o outro avatar. Clicar repetidamente durante ações não deve sobrepor animações.
6. Gabriel registra em TESTES.md comandos, resultados reais, navegador e eventuais limitações. Testes automatizados com doubles não substituem a verificação visual no navegador.

O trabalho só estará concluído quando esses critérios forem verificados. Nesta etapa, a validação limita-se à existência dos dois artefatos, JSON válido, exatamente três responsáveis distintos, listas de arquivos e aceite não vazias e ausência de propriedade sobreposta.
