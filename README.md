# Calendário de Leilões — Leiloaria Smart

Calendário web que mostra, dia a dia, quantos lotes têm praça agendada. Clicando
no dia abre a lista com horário, valor e link direto para a página do lote em
`leiloariasmart.com.br`.

Os dados são extraídos automaticamente de https://leiloariasmart.com.br/busca.

---

## Como usar

### 1. Atualizar os dados

```bash
node scraper.js                    # pega tudo que estiver publicado
node scraper.js --ate 2026-09-30   # só até a data indicada
```

Gera dois arquivos em `site/`:

| Arquivo | Para quê |
|---|---|
| `lotes.json` | dados limpos, para integrações (n8n, API, planilha, outro front) |
| `lotes.js`   | mesmo conteúdo em `window.DADOS_LEILAO`, usado pelo calendário |

O `.js` existe porque o navegador bloqueia `fetch()` de arquivo local
(`file://`) — com ele o `index.html` abre com duplo clique, sem servidor.

### 2. Abrir o calendário

Duplo clique em `site/index.html`. Para servir de verdade, suba a pasta `site/`
em qualquer hospedagem estática (é só HTML/CSS/JS, sem build).

Para testar com servidor local:

```bash
npx serve site
```

---

## Estrutura

```
calendario-leiloes/
├── scraper.js          # baixa e parseia o site -> lotes.json + lotes.js
├── site/
│   ├── index.html      # o calendário (arquivo único, sem dependências)
│   ├── lotes.json      # dados gerados
│   └── lotes.js        # dados gerados (window.DADOS_LEILAO)
└── README.md
```

---

## Formato do `lotes.json`

```jsonc
{
  "fonte": "https://leiloariasmart.com.br/busca",
  "atualizadoEm": "2026-08-17T04:44:49.000Z",
  "total": 188,              // imóveis
  "totalPracas": 261,        // datas agendadas (um imóvel pode ter 1ª e 2ª praça)
  "primeiraData": "2026-08-03",
  "ultimaData": "2026-09-18",
  "lotes": [
    {
      "id": 1700,
      "titulo": "Apto 88m² - Goiânia/GO",
      "url": "https://leiloariasmart.com.br/imovel/1700",
      "imagem": "https://leiloariasmart.com.br/_admin_/upload/....jpg",
      "tipo": "Apartamento",          // derivado do título
      "cidade": "Goiânia",
      "uf": "GO",
      "modalidade": "Extrajudicial",
      "comitente": "VERT Companhia Securitizadora",
      "situacao": "Desocupado",
      "desconto": "50%",
      "descontoTexto": "abaixo na 2ª praça",
      "precoDestaque": "R$ 470.000,00",
      "visitas": 1142,
      "pracas": [
        { "ordem": 1, "rotulo": "1ª praça", "data": "2026-08-04",
          "dataBr": "04/08/2026", "hora": "13:55",
          "valor": "R$ 940.000,00", "encerrada": true },
        { "ordem": 2, "rotulo": "2ª praça", "data": "2026-08-25",
          "dataBr": "25/08/2026", "hora": "13:55",
          "valor": "R$ 470.000,00", "encerrada": false }
      ]
    }
  ]
}
```

**Cada praça vira um evento no calendário.** Um imóvel com 1ª e 2ª praça aparece
nos dois dias — por isso `totalPracas` (261) é maior que `total` (188).

---

## Como o scraper funciona

O site é server-rendered (HTML pronto, sem API pública), e `/busca` devolve o
catálogo inteiro numa página só — sem paginação. O scraper:

1. baixa o HTML de `/busca` (~1,2 MB);
2. quebra em cards pelo marcador `<div class="caixa-imoveis">`;
3. lê de cada card: `info-imovel-1` (modalidade/comitente), `info-imovel-2`
   (título e link), `info-imovel-3` / `info-imovel-4` / `info-imovel-5`
   (datas das praças), `percentual-praca`, `preco-imovel`, `visitas-imovel`,
   `faixa-*-imovel` (ocupado/desocupado);
4. deriva `tipo`, `cidade` e `uf` a partir do título;
5. grava o JSON.

Sem dependências externas — só `fetch` nativo do Node 18+.

### Se o site mudar o HTML

O parsing depende dessas classes CSS. Se um dia o scraper voltar 0 cards ou
campos vazios, o ponto a conferir é o bloco `parseCard()` em `scraper.js` —
as classes provavelmente mudaram de nome.

---

## Observações sobre os dados

- **Percentuais negativos** (`-8%`) existem no site: significam lance acima da
  avaliação. O calendário mostra como "8% acima", em cinza, em vez de "OFF".
- **Praças já encerradas** vêm marcadas com `"encerrada": true` (o site rasura
  esses blocos). Elas continuam no calendário para manter o histórico do mês;
  os dias passados aparecem esmaecidos.
- O calendário limita a navegação ao intervalo que existe nos dados — não dá
  para navegar para meses vazios.

---

## Automatizar a atualização

Para manter o calendário sempre em dia, agende o scraper. No Windows:

```powershell
schtasks /create /tn "CalendarioLeiloes" /tr "node C:\Users\santo\projetos\calendario-leiloes\scraper.js" /sc daily /st 06:00
```

Depois é só publicar a pasta `site/` novamente (ou apontar a hospedagem para
ela, se o deploy for automático).
