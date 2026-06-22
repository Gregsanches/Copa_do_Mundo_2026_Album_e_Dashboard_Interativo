# 📊 Dashboard Interativo — Álbum Panini Copa do Mundo 2026

Dashboard analítico single-page (HTML/CSS/JS puro) para gerenciamento e análise estatística de figurinhas do álbum oficial Panini da Copa do Mundo 2026. Construído sem dependências de backend — roda 100% no navegador, sem necessidade de servidor ou instalação.

**[→ Demo ao vivo](https://gregsanches.github.io/Copa_do_Mundo_2026_Album_e_Dashboard_Interativo/)**

---

## 🗂️ Estrutura do Projeto

```
album-copa-2026/
├── index.html          # Aplicação completa (single-file)
└── README.md           # Este documento
```

O projeto é propositalmente contido em um único arquivo HTML. Isso facilita o compartilhamento, hospedagem via GitHub Pages e a portabilidade — qualquer pessoa pode baixar e abrir no navegador sem configuração adicional.

---

## ✨ Funcionalidades

### Visão Geral
- KPIs em tempo real: total colado, faltando, completude percentual
- Barra de progresso geral do álbum
- Breakdown por confederação, tipo de categoria (jogador, escudo, foto de time, especial, Coca-Cola) e Top 10 seleções mais completas
- Exportação e importação de progresso em JSON — compatível entre dispositivos

### Por Seleção
- Listagem de todas as 48 seleções com barra de progresso individual
- Filtros por grupo (A–L) e confederação (UEFA, CONMEBOL, CONCACAF, CAF, AFC, OFC)

### Por Grupo (Heatmap)
- Mapa de calor de completude por grupo e seleção, com escala de cor semântica (vermelho → amarelo → verde)

### Galeria Interativa
- Grade visual com imagens das 994 figurinhas
- Marcar/desmarcar figurinha como colada via clique no card ou checkbox dedicado
- Borda verde + checkbox preenchido para figurinhas coladas
- Filtros combinados: seleção, tipo, status (coladas/faltando), busca textual
- Ordenação por **ordem alfabética** (com normalização de acentos — Á, É, Ó tratados como A, E, O) ou por **sequência do álbum** (grupos A→L)
- Ações em lote: "Colar filtradas" e "Descolar filtradas"

### Análise de Dados
- Previsão dinâmica de conclusão com base nos parâmetros configurados
- Custo das figurinhas restantes com cálculo individual por figurinha
- Curva de custo marginal dinâmica
- Simulação Monte Carlo (1.000 iterações) para distribuição probabilística de custo total

---

## 📐 Metodologias de Análise

Esta seção detalha os modelos estatísticos implementados. Contribuições, críticas e aprimoramentos são bem-vindos — veja a seção [Contribuindo](#-contribuindo).

### 1. Modelo Coupon Collector (Colecionador de Cupons)

O problema central do colecionador de figurinhas é uma instância clássica do **Coupon Collector's Problem**: dado um conjunto de *N* itens distintos, quantas compras aleatórias (com reposição) são necessárias para obter todos?

O valor esperado de tentativas para completar o conjunto a partir do zero é:

```
E[tentativas] = N × H(N) = N × Σ(1/k) para k = 1 até N
```

onde H(N) é o N-ésimo número harmônico. Para N = 994 figurinhas, isso resulta em aproximadamente **7.375 figurinhas compradas** para completar o álbum do zero — ou seja, mais de 7× o tamanho total do álbum, em média.

O dashboard adapta esse modelo para o **estado atual** do álbum (já com algumas figurinhas coladas), recalculando a partir da posição corrente.

### 2. Custo Marginal por Figurinha

Para cada estado do álbum (com `coladas` de `N` figurinhas), o custo marginal esperado de obter **uma** nova figurinha única é calculado via distribuição geométrica adaptada para pacotes:

```
P(pacote ter ≥1 nova) = 1 - (coladas / N) ^ figurinhas_por_pacote

E[pacotes necessários] = 1 / P(pacote ter ≥1 nova)

Custo marginal = E[pacotes] × preço_do_pacote
```

> **Nota de revisão (v2):** uma versão anterior usava `P = faltando / N` (probabilidade por *slot* individual), o que subestimava o custo quando o pacote tem mais de uma figurinha, por ignorar duplicatas *dentro* do mesmo pacote. A fórmula foi unificada em todos os módulos para a formulação correta por pacote — `1 - (coladas/N)^fpac`. Isso garante consistência entre o custo marginal, o custo das figurinhas restantes e a previsão de conclusão.

**Intuição:** no início do álbum, quase qualquer figurinha do pacote é nova — o custo marginal tende ao preço de compra. À medida que o álbum se completa, a probabilidade de tirar uma figurinha nova cai dramaticamente, e o custo por figurinha única dispara de forma exponencial.

O gráfico de custo marginal traça essa curva ao longo do progresso. Os pontos de amostragem foram **adensados na região assintótica** (99,5%, 99,8%, 99,9%), onde a curvatura é mais acentuada e o custo explode — capturando o comportamento de cauda que pontos esparsos perderiam.

### 3. Custo das Figurinhas Restantes

O indicador de custo das figurinhas restantes itera sobre **cada uma** das figurinhas que ainda faltam, computando o custo marginal individual conforme o álbum evolui:

```javascript
for (k = 1; k <= missing; k++) {
  coladas = already + k - 1
  pHit = 1 - (coladas / TOTAL)^fpac
  custoK = (1 / pHit) × precoPacote
  custoTotal += custoK
}
custoMedio = custoTotal / missing
```

O resultado mostra:
- **Custo médio** por figurinha entre todas as restantes
- **Custo da próxima** (a mais barata — estado atual do álbum)
- **Custo da última** (a mais cara — quando só ela falta)
- **Custo total estimado** para completar sem trocas

> **Limitação conhecida:** o modelo trata a obtenção de cada figurinha nova como evento sequencial independente, sem considerar que um único pacote pode avançar várias posições de uma vez. Isso torna o custo total **levemente conservador** (superestimado em ~3–8%, maior quando `fpac` é alto e faltam poucas figurinhas). A simulação de Monte Carlo (seção 5) não tem essa limitação e serve como contraponto mais fiel.

### 4. Previsão de Conclusão

A previsão de tempo usa um **modelo integrado de taxa variável**. Em vez de assumir uma taxa de novas figurinhas constante, soma o número esperado de pacotes para cada figurinha restante — refletindo a desaceleração real conforme o álbum se completa:

```
pacotes_totais = Σ (i = already até N-1) [ 1 / (1 - (i/N)^fpac) ]

semanas_restantes = ⌈ pacotes_totais / pacotes_por_semana ⌉
```

> **Nota de revisão (v2):** a versão anterior usava `taxa = faltando/N` calculada uma única vez no estado atual e aplicada como constante, o que tornava a previsão progressivamente **otimista** (subestimava o tempo) quanto mais completo o álbum. O modelo integrado corrige isso usando a mesma esperança por pacote da seção 2, agora consistente em toda a aplicação.

### 5. Simulação Monte Carlo

A simulação roda **1.000 iterações** independentes, cada uma simulando a compra de pacotes até completar o álbum a partir do **estado real** do usuário:

```javascript
possuiIdx = índices reais das figurinhas que o usuário já tem
for cada iteração:
  coladas = Set(possuiIdx)        // estado real, não índices genéricos
  pacotes = 0
  while (coladas.size < TOTAL):
    para cada slot do pacote:
      sorteia figurinha aleatória (uniforme, sem raridades)
      adiciona ao Set
    pacotes++
  registra: pacotes necessários nesta iteração
```

> **Nota de revisão (v2):** a versão anterior pré-populava o conjunto com índices genéricos `0..already-1`. Embora o resultado numérico fosse equivalente (o número de pacotes depende apenas de *quantas* figurinhas faltam, não de *quais*), o modelo agora usa o conjunto real de figurinhas possuídas. Isso torna a simulação fiel ao estado do álbum e abre caminho para modelar **estratégias de troca por figurinha específica** no futuro.

A distribuição resultante permite calcular:
- **Mediana** (pacotes no percentil 50)
- **Intervalo de confiança 25–75%** (metade das simulações cai aqui)
- **Histograma de distribuição** de custo total

**Premissas do modelo:** distribuição uniforme entre todas as figurinhas (sem raridades, sem hierarquia de probabilidades), independência entre pacotes.

---

## 🛠️ Stack Técnica

| Componente | Tecnologia |
|---|---|
| Estrutura | HTML5 semântico |
| Estilo | CSS3 com variáveis (design tokens) |
| Lógica | JavaScript ES6+ (vanilla) |
| Gráficos | [Chart.js 4.4](https://www.chartjs.org/) |
| Ícones | [Tabler Icons](https://tabler.io/icons) |
| Fontes | Plus Jakarta Sans (Google Fonts) |
| Persistência | localStorage (progresso local) + JSON export/import |

Sem frameworks, sem bundlers, sem Node.js. Zero dependências de build.

---

## ♿ Acessibilidade

Este projeto passou por uma rodada inicial de auditoria com a ferramenta [WAVE (Web Accessibility Evaluation Tool)](https://wave.webaim.org/), e os principais apontamentos identificados foram corrigidos. **Isso não significa que o projeto é totalmente acessível** — é um ponto de partida, não um ponto de chegada.

### O que foi feito

| Categoria | Correção aplicada |
|---|---|
| Idioma | `<html lang="pt-BR">` declarado para leitores de tela |
| Navegação por teclado | Skip link *"Ir para o conteúdo principal"* visível ao foco |
| Landmark semântico | Conteúdo principal envolvido em `<main>` |
| Navegação por abas | `role="tablist"` na nav, `role="tab"` em cada botão, `aria-selected` atualizado dinamicamente via JS |
| Formulários | `aria-label` em todos os `<select>` e no campo de busca textual |
| Botões de ação | `aria-label` nos botões de exportar, importar e ações em lote |
| Checkboxes interativos | `role="checkbox"`, `aria-checked` dinâmico e suporte a teclado (Space/Enter) nos cards da galeria |
| Ícones decorativos | `aria-hidden="true"` em todos os ícones Tabler, evitando leitura desnecessária |
| Contraste | Cor de texto terciário ajustada para passar o critério AA do WCAG em fundos escuros |
| Movimento reduzido | `@media (prefers-reduced-motion)` zera transições e animações para quem ativa essa preferência no sistema |
| Modo alto contraste | `@media (forced-colors: active)` mapeia cores semânticas para primitivas do sistema (Canvas, Highlight, ButtonText) |
| Feedback em tempo real | Região `aria-live="polite"` anuncia cada figurinha colada/removida e o total atualizado, sem mover o foco |
| Gráficos | Cada `<canvas>` tem `role="img"` + `aria-label` e uma tabela de dados equivalente (`sr-only-table`) lida por leitores de tela |
| Painéis (abas) | `role="tabpanel"` + `aria-labelledby`; ao trocar de aba o foco move para o painel, permitindo navegação imediata por Tab |
| Heatmap | `role="grid"`, células com `role="gridcell"`, `aria-label` descritivo (seleção, grupo, %, status) e navegação por setas ←→↑↓ entre grupos |
| Barras de progresso | `role="progressbar"` com `aria-valuenow`/`valuemin`/`valuemax` em todas as barras (confederação, tipo, seleções) |
| Lista de seleções | `role="list"`/`listitem`, cada item focável com `aria-label` completo |
| Foco visível | `:focus-visible` com contorno de contraste AA em todos os elementos interativos; roving tabindex na galeria e no heatmap |
| Alt contextual | Imagens das figurinhas descrevem nome, código, seleção e status; placeholders de imagem ausente usam `role="img"` com `aria-label` |

### O que ainda precisa melhorar

Esta lista é honesta sobre as limitações que permanecem após a segunda rodada de auditoria:

- **Validação com leitores de tela reais:** as correções seguem as práticas WAI-ARIA, mas não foram testadas com NVDA, JAWS ou VoiceOver por usuários reais. Testes com pessoas que dependem dessas tecnologias são o próximo passo essencial.
- **Ordem de leitura dos gráficos:** as tabelas equivalentes existem, mas a transição entre o `<canvas>` e a tabela pode ser confusa em alguns leitores — falta refinar o `aria-describedby`.
- **Navegação por teclado em listas muito longas:** a galeria usa roving tabindex (uma boa prática), mas percorrer centenas de cards ainda é cansativo. Um mecanismo de "pular para seleção" ajudaria.
- **Contraste do texto sobre células do heatmap:** as cores de fundo do heatmap (vermelho/laranja/verde) não foram todas auditadas contra o texto sobreposto em cada estado.
- **Internacionalização do conteúdo ARIA:** todos os rótulos estão em português fixo — não há suporte a outros idiomas.

### Como contribuir com acessibilidade

Contribuições focadas em acessibilidade são especialmente bem-vindas. Se você identificar novos problemas ou quiser implementar qualquer item da lista acima, abra uma issue descrevendo o critério WCAG afetado (ex: [1.4.3 Contraste](https://www.w3.org/TR/WCAG21/#contrast-minimum), [4.1.3 Mensagens de Status](https://www.w3.org/TR/WCAG21/#status-messages)) e o comportamento esperado antes de abrir um PR. **Relatos de teste com leitores de tela reais são particularmente valiosos.**

---

## 🚀 Como usar

### Opção 1 — GitHub Pages (recomendado)
1. Fork este repositório
2. Vá em Settings → Pages → Source: `main` / `root`
3. Acesse `https://seu-usuario.github.io/album-copa-2026`

### Opção 2 — Local
```bash
git clone https://github.com/seu-usuario/album-copa-2026.git
cd album-copa-2026
# Abrir index.html no navegador
open index.html  # macOS
# ou duplo-clique no arquivo
```

### Backup do progresso
Use os botões **Exportar Dados** / **Importar Dados** na aba Visão Geral para salvar e transferir seu progresso entre dispositivos ou navegadores.

---

## 🤝 Contribuindo

Este repositório é aberto a contribuições! Áreas de melhoria identificadas:

**Acessibilidade** 
- [ ] Validação com leitores de tela reais (NVDA, JAWS, VoiceOver)
- [ ] Refinar `aria-describedby` ligando cada gráfico à sua tabela equivalente
- [ ] Mecanismo "pular para seleção" na navegação por teclado da galeria
- [ ] Auditar contraste do texto sobre cada estado de cor do heatmap


**Análise estatística** 
- [ ] **Modelo com trocas:** quanto se economiza com um grupo de N colecionadores trocando duplicatas? (estimativa: ~40–60% com 3–4 pessoas)
- [ ] **Probabilidade de completar com orçamento X:** dado R$ Y restantes, qual a chance de completar? (Monte Carlo com corte de orçamento)
- [ ] **Data de conclusão com intervalo:** usar o Monte Carlo para dar P25/P75 da data prevista, não só um ponto
- [ ] **Curva de retorno decrescente:** gráfico de figurinhas novas por real gasto — mostra quando parar de comprar e migrar para trocas
- [ ] **Estratégia de compra em lote vs. gradual:** comparar qual reduz mais o custo esperado
- [ ] Intervalo de confiança 90% e 95% na simulação Monte Carlo
- [ ] Modelar o avanço de múltiplas posições por pacote 

**Produto / UX**
- [ ] Modo escuro/claro toggle
- [ ] Exportação de relatório em PDF
- [ ] Sincronização via URL (estado codificado em query string)
- [ ] Progressive Web App (PWA) para uso offline no celular
- [ ] Filtro por múltiplas seleções simultâneas na galeria
- [ ] Campo de "repetidas" por figurinha (rastrear duplicatas e seu valor)

**Dados**
- [ ] Validação e atualização das URLs de imagem
- [ ] Validação de integridade na importação (reportar códigos que não correspondem ao álbum atual)
- [ ] Tratamento de `QuotaExceededError` no localStorage
- [ ] Adicionar campo de "repetidas" por figurinha


Para contribuir:
```bash
# 1. Fork o repositório
# 2. Crie uma branch
git checkout -b feature/minha-melhoria

# 3. Faça suas alterações no index.html
# 4. Abra um Pull Request descrevendo a mudança e o racional
```

Por ser um arquivo único, não há processo de build. Teste abrindo o `index.html` diretamente no navegador.

---

## 📄 Dados e Imagens

As imagens das figurinhas são hospedadas externamente via [PostImg](https://postimg.cc/). O código mapeou manualmente as URLs de todas as 994 figurinhas do álbum. Caso alguma imagem esteja indisponível, o card exibe o código da figurinha como fallback.

Os dados estruturais (seleções, grupos, confederações, nomes dos jogadores) foram construídos manualmente a partir do álbum oficial Panini Copa do Mundo 2026.

---

## 📝 Licença

MIT — use, modifique e distribua livremente, com atribuição.

---

*Feito com curiosidade estatística de um pai para brincar junto com seu filho de álbum da Copa do Mundo 2026.*

![Brasil](https://flagcdn.com/24x18/br.png)
🇧🇷
