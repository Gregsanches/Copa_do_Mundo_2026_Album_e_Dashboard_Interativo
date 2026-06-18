# 📊 Dashboard Interativo — Álbum Panini Copa do Mundo 2026

Dashboard analítico single-page (HTML/CSS/JS puro) para gerenciamento e análise estatística de figurinhas do álbum oficial Panini da Copa do Mundo 2026. Construído sem dependências de backend — roda 100% no navegador, sem necessidade de servidor ou instalação.

**[→ Demo ao vivo](https://seu-usuario.github.io/album-copa-2026)** *(substituir com seu link após publicar no GitHub Pages)*

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

Para cada posição *k* do álbum (de 1 até N), o custo marginal esperado de obter uma nova figurinha única é calculado via distribuição geométrica adaptada para pacotes:

```
P(acerto em 1 pacote) = 1 - ((coladas_antes_desta / N) ^ figurinhas_por_pacote)

E[pacotes necessários] = 1 / P(acerto)

Custo marginal = E[pacotes] × preço_do_pacote
```

**Intuição:** no início do álbum, quase qualquer figurinha do pacote é nova — o custo marginal tende ao preço de compra. À medida que o álbum se completa, a probabilidade de tirar uma figurinha nova cai dramaticamente, e o custo por figurinha única dispara de forma exponencial.

O gráfico de custo marginal traça essa curva ao longo do progresso (0% a 99%).

### 3. Custo das Figurinhas Restantes

O indicador de custo das figurinhas restantes itera sobre **cada uma** das figurinhas que ainda faltam, computando o custo marginal individual:

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

### 4. Previsão de Conclusão

A previsão de tempo usa a taxa efetiva de novas figurinhas por semana, estimada a partir do estado atual:

```
P(nova fig por slot) = faltando / total

novas_por_semana = pacotes_por_semana × figs_por_pacote × P(nova fig por slot)

semanas_restantes = faltando / novas_por_semana
```

Esta é uma aproximação linear de primeira ordem — na prática, a taxa cai conforme o álbum é preenchido. Para previsões mais precisas com muitas figurinhas faltando, o modelo é otimista; para álbuns mais completos, é mais preciso.

### 5. Simulação Monte Carlo

A simulação roda **1.000 iterações** independentes, cada uma simulando a compra de pacotes até completar o álbum a partir do estado atual:

```javascript
for cada iteração:
  coladas = Set com as figurinhas já obtidas
  pacotes = 0
  while (coladas.size < TOTAL):
    para cada slot do pacote:
      sorteia figurinha aleatória (uniforme, sem raridades)
      adiciona ao Set
    pacotes++
  registra: pacotes necessários nesta iteração
```

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

**Análise estatística**
- [ ] Modelo de previsão com taxa decrescente ao longo do tempo (não linear)
- [ ] Simulação de estratégia de trocas entre colecionadores
- [ ] Cálculo de valor esperado com grupos de troca (N colecionadores)
- [ ] Intervalo de confiança 90% e 95% na simulação Monte Carlo
- [ ] Sensibilidade dos resultados a variações no preço

**Produto / UX**
- [ ] Modo escuro/claro toggle
- [ ] Exportação de relatório em PDF
- [ ] Sincronização via URL (estado codificado em query string)
- [ ] Progressive Web App (PWA) para uso offline no celular
- [ ] Filtro por múltiplas seleções simultâneas na galeria

**Dados**
- [ ] Validação e atualização das URLs de imagem
- [ ] Adicionar campo de "repetidas" por figurinha
- [ ] Suporte a outros álbuns (extensão do modelo de dados)

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

*Feito com curiosidade estatística para aprendizado pessoal e, principalmente, para um filho e um pai colecionador de figurinhas.* 

![Brasil](https://flagcdn.com/24x18/br.png) 🇧🇷
