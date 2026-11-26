# WRABF · App de Leitura e Registro de Alvos

**Dono:** Nery Signorini Neto · CT CicogNery
**Pasta:** `C:\Users\signon01\Dropbox\Claude AI NSN\Leitura de Alvos WRABF`
**Publicação prevista:** GitHub Pages → `https://signon01.github.io/WRABF-Alvos/` (repositório `WRABF-Alvos`)
**Modelo a seguir:** o app NSN Recarga (`signon01.github.io/CAC-Recargas`): um único `index.html`, sem servidor, dados no aparelho, com backup/restauração e impressão.

---

## 1. Objetivo
Um app de celular para usar no estande, também sem internet, que:
1. lê a foto de um cartão de alvo WRABF/CBTE (25 alvos numerados de 1 a 25, grade 5×5) e calcula a pontuação e os X de cada tiro;
2. analisa o agrupamento: viés, dispersão, extreme spread, fliers e tendência ao longo da série;
3. registra cada cartão com o equipamento e as condições usadas;
4. mostra a evolução até a meta (245+) e compara carabina × luneta × chumbinho.

## 2. Modo de leitura: **A, assistido** (decidido em 26/11/2026)
| Modo | Como funciona | Prós | Contras |
|---|---|---|---|
| **A. Assistido** (recomendado para a v1) | Foto → o app corrige a perspectiva e encaixa a grade → o usuário toca em cada furo (com lupa) → o app pontua | Confiável, rápido de fazer, funciona offline | Exige 25 toques por cartão |
| **B. Automático** | O pipeline da seção 5 roda no navegador (OpenCV.js) → o usuário só corrige os erros | Rápido no uso | Precisa ser validado com fotos reais; sensível a luz e ângulo |
| **C. Híbrido** | A leitura é feita no Claude (skill `wrabf-alvo-reader`) → o resultado vai para o app | Usa a leitura já validada | Depende de enviar a foto ao Claude |

> **Decisão:** a v1 usa o modo **A**. O código deve ficar organizado para incluir o **B** depois, quando a detecção automática passa a preencher os toques e o usuário só confirma.

## 3. Categorias e alvos
| Código | Categoria | Distância | Alvo |
|---|---|---|---|
| AR-LV | Carabina de Ar Light | 25 m | Alvo 25m |
| AR-HV | Carabina de Ar Heavy | 25 m | Alvo 25m |
| AR-SP | Carabina de Ar **Spring** (mola) | 25 m | Alvo 25m (ex.: TX200, HW77) |
| AR-UN | Carabina de Ar Unlimited | 50 m | Alvo 50m (sem limite de energia) |
| RF-LV | Rimfire Light | 50 m | Alvo 50m |
| RF-HV | Rimfire Heavy | 50 m | Alvo 50m |

Padrão de nome da foto: `[CÓDIGO]-AAAAMMDD.jpg` (ex.: `AR-LV-20260829.jpg`).

## 4. Geometria oficial dos anéis
Fonte: WRABF Official Rule Book 2023-2027, Apêndice E.

### Alvo 25m (Ar Comprimido)
| Anel | Diâmetro (mm) | Raio (mm) |
|---|---|---|
| 10 | 2,000 | 1,000 |
| 9 | 8,000 | 4,000 |
| 8 | 14,000 | 7,000 |
| 7 | 22,000 | 11,000 |
| 6 | 30,000 | 15,000 |
| 5 | 39,000 | 19,500 |

- **X (25m):** o furo precisa **obliterar totalmente** o anel do 10 (raio 1,0 mm). Condição: `dist_centro + 1,0 ≤ raio_do_furo`.
- **Anel de calibração:** o do 6 (raio 15,0 mm), que tem a linha azul mais grossa.

### Alvo 50m (Rimfire / Ar Unlimited)
| Anel | Diâmetro (mm) | Raio (mm) |
|---|---|---|
| 10X | 0,792 | 0,396 |
| 10 | 6,350 | 3,175 |
| 9 | 12,700 | 6,350 |
| 8 | 19,050 | 9,525 |
| 7 | 25,400 | 12,700 |
| 6 | 31,750 | 15,875 |
| 5 | 38,100 | 19,050 |

- **X (50m):** basta **tocar** o anel de 0,396 mm. Condição: `best_edge ≤ 0,396`.
- **Anel de calibração:** o do 6 (raio 15,875 mm).

### Regra de pontuação (as duas distâncias)
**Best edge (inward scoring):** `best_edge = máx(dist_centro_do_furo − raio_do_furo, 0)`. A pontuação é o maior anel em que `best_edge ≤ raio_do_anel`. Se a borda do furo tocar a linha externa de um anel, vale esse anel. No cartão físico isso se confere com plug.

**Fora do anel 5:**
| Situação | Pontos |
|---|---|
| Dentro do quadrado do alvo, sem tocar o anel 5 | **4** |
| Borda do furo tocando a linha externa do anel 5 | **5** |
| Fora do quadrado (sem tocar a linha que delimita a caixa) | **0**, e aquele alvo fica sem impacto registrado |

**Exceções:**
- **Erro de primeiro tiro (First Shot Error):** se o **primeiro tiro da série** cair por engano numa caixa de pontuação, em vez do alvo de ensaio, o atirador pode chamar o árbitro **antes do próximo tiro** para anular o impacto, sem perder pontos. Vale **uma única vez por prova**.
- **Dois tiros na mesma caixa:** vale **a menor** das duas pontuações, e o cartão leva **penalidade de 1 ponto**.

Fontes informadas pelo Nery: WRABF Rules 2023-2027 V3; WRABF Scoring Guide; regulamento CBTE (WRABF Regulamento Específico Brasil 2025 e Regulamento prova carabina WRABF); WBSF Rimfire Rulebook 2018; regulamento interno CTCPV Carabina WRABF Benchrest .22 2025; TRV Scoring.

## 5. Pipeline de leitura automática (modo B)
A referência é o script `ler_alvo.py` (Python/OpenCV) da skill `wrabf-alvo-reader`, já validado manualmente. Para rodar no navegador, ele deve ser portado para OpenCV.js.

1. **Localizar a grade:** detectar a borda azul do cartão (HSV 100–130) e dividir em 25 células (5×5).
2. **Calibrar a escala:** em cada célula, achar o anel de calibração (maior contorno azul), usar o círculo envolvente e calcular `px_por_mm = raio_px / raio_mm_do_anel`.
3. **Detectar o furo:** limiar de pixels escuros e um filtro de *fill ratio* (`área ÷ área do círculo envolvente > 0,35`) para separar o furo das linhas impressas. O candidato escolhido é o de maior `fill_ratio × área`.
4. **Pontuar:** best edge contra os raios oficiais, e X conforme a regra da distância.
5. **Validar a qualidade:** comparar o raio médio dos furos detectados com o do calibre (4,5 mm → 2,25 mm; 5,5 mm → 2,75 mm; .22 LR → ~2,8 mm). Se a diferença for grande, avisar: "calibração imprecisa (ângulo ou luz)".

Primeiro teste real da metodologia: alvo 25m, 4,5 mm, Air Arms TX200, **231/250**.

## 6. Modo assistido (v1)
- Tirar a foto com a câmera do celular ou escolher na galeria.
- Ajustar os 4 cantos da grade arrastando os pontos (correção de perspectiva).
- O app desenha os anéis oficiais sobre cada célula, e o usuário confere se encaixaram.
- O usuário toca no **centro de cada furo**, com lupa ao tocar. O raio do furo vem do calibre do cartão.
- Cada alvo pode ser marcado como:
  - **sem impacto / fora do quadrado** → 0;
  - **fora do anel 5, dentro do quadrado** → 4 (o app sugere sozinho quando o toque cai entre o anel 5 e a borda da caixa);
  - **dois tiros na mesma caixa** → o usuário marca os dois furos; o app usa a menor pontuação e aplica −1 no total do cartão;
  - **First Shot Error anulado** → registra que o 1º tiro foi anulado pelo árbitro (uma vez por prova) e não pontua nem penaliza.
- O total do cartão mostra a soma dos alvos − penalidades, e a quantidade de X.
- O app calcula a pontuação de cada alvo, o total e os X na hora.

## 7. Registro de cada cartão (campos)
- **Identificação:** data, local (padrão: CT CicogNery), categoria (código), nº do cartão / prova.
- **Equipamento:** carabina, luneta, munição ou chumbinho (marca, peso, formatação de saia e cabeça), apoio (frontrest e rearest, "bota", "bolsinha").
- **Energia/velocidade:** velocidade média, DP e ES (Garmin C1) e energia em J. Alertar se passar do teto da categoria: AR-LV 16 J, AR-HV 27 J. AR-UN não tem limite de energia.
- **Condições:** clima, vento, temperatura.
- **Resultado:** pontuação por alvo (1–25), total, X, foto do cartão (comprimida).
- **Grupo:** centro do grupo (dx, dy em mm), raio médio, extreme spread, fliers, tendência ao longo da série.
- **Observações:** texto livre (respiração, gatilho, limpeza do cano a cada 2 recargas de ar, etc.).

## 8. Telas
1. **Novo cartão:** foto → leitura → registro.
2. **Histórico:** lista com filtros por categoria, carabina e chumbinho.
3. **Evolução:** gráfico das pontuações com linha da meta (245+), por categoria.
4. **Comparativos:** média, melhor resultado e X por carabina × chumbinho × luneta.
5. **Equipamentos:** cadastro de carabinas, lunetas, chumbinhos e munições (seção 9).
6. **Mapa de calor:** todos os furos de vários cartões sobrepostos, para ver o viés.
7. **Impressão:** relatório do cartão e card 9×13, no mesmo padrão do Cafofo.
8. **Backup / Restaurar / Exportar CSV.**

## 9. Cadastro inicial de equipamentos
| Carabina | Tipo / calibre | Luneta | Categoria típica |
|---|---|---|---|
| Steyr LG110 | PCP 4,5 mm | Weaver T-36x40 | AR-LV (≤16 J; regulada a 16,3 J, rebaixar) |
| Steyr Challenge | PCP 4,5 mm | Vector Optics Sentinel-X 10-40x50 Center Dot | AR-HV (≤27 J) |
| Air Arms TX200 | Spring 4,5 mm | Vector Optics Sentinel-X 10-40x50 Center Dot | AR-SP (25 m) |
| Weihrauch HW77 | Spring 5,5 mm | Vector Optics Hugo 6-24x50 | — |
| Daystate Air Wolf | PCP 5,5 mm | Vector Optics Minotaur GenII 12-60x60 MFL | — |
| JTS Airacuda Max Bigbore | PCP 6,35 mm | Discovery HD Gen II 5-30x56 | — |
| Anschütz 1416 | .22 LR | Sightron SII 36x42 BRD | RF 50m |
| CZ 457 Premium | .22 LR | Vector Optics Continental 6-24x56 | RF 50m |

Outras lunetas: Vector Optics Taurus 3-18x50 FFP, Vector Optics Taurus 5-30x56 FFP e uma segunda Sentinel-X.

**Chumbinhos:** JSB 8,44 gr (0,54 g, formatado para 4,52 mm) na LG110; JSB 13,43 gr na Challenge; a testar na TX200: JSB 7,87, Rifle 8,33, JSB 8,44 e JSB 10,34.

**Pontuações de referência:** LG110 (AR-LV) 241; Challenge (AR-HV) 243; meta 245+.

## 10. Requisitos técnicos
- Um único `index.html`, com HTML, CSS e JS no mesmo arquivo, igual ao NSN Recarga. Bibliotecas externas só se forem indispensáveis (ex.: OpenCV.js no modo B), carregadas de CDN e com o app funcionando sem elas no modo A.
- Pensado para celular: botões grandes, tema escuro, funcionar na luz do estande.
- Dados no aparelho (localStorage/IndexedDB); fotos comprimidas (~1600 px).
- Backup e restauração em .json (para guardar no Dropbox); exportação CSV com `;` e BOM, para abrir no Excel.
- Instalável na tela inicial (iPhone e Android).
- Mensagens honestas: "leitura por foto é uma aproximação, não apuração oficial".

## 11. Validação antes de publicar
- [ ] 3 a 4 fotos de cartões **com o resultado oficial conhecido** (25m e 50m).
- [ ] O app deve bater com o placar oficial, alvo a alvo, nos 25 alvos.
- [ ] Testar foto em ângulo e com luz ruim (o aviso de calibração deve aparecer).
- [ ] Testar backup em um aparelho e restauração em outro.

### Cartões de validação (pasta `validacao\`)
| Foto | Cartão | Resultado oficial | Leitura automática (skill) | Diferença |
|---|---|---|---|---|
| `AR-SP_20260701.jpeg` | 25m Air Rifle, Hobby Air Gun, TX200 Spring (ID 27), 01/07/2026 | **233** (250 − 17) | 231 com o raio do furo medido na foto (2,06 mm); **232 e 2 X** com o raio nominal de 4,5 mm (2,25 mm) | −1 a −2 |

Lição do 1º cartão: o furo medido na foto sai menor que o real, o que puxa a pontuação para baixo. **O app deve usar o raio nominal do calibre** (ou o do plug) no cálculo da melhor borda, não o raio medido na foto. Os alvos 7 e 25 ficaram a 0,08–0,14 mm do limite do 10, abaixo da precisão de uma foto. No modo A, o usuário confirma esses casos com a lupa.

## 12. Como publicar (mesmo processo do NSN Recarga)
1. github.com → **New repository** → `WRABF-Alvos` → Public → Create.
2. **Upload files** → arrastar o `index.html` → Commit changes.
3. **Settings → Pages** → Deploy from a branch → `main` / `(root)` → Save.
4. Em 1 a 2 minutos o app fica no ar em `https://signon01.github.io/WRABF-Alvos/`.
5. No celular: abrir o endereço → **Adicionar à Tela de Início**.

## 13. Pendências do Nery
- [x] Modo de leitura: **A, assistido**.
- [ ] Colocar nesta pasta as fotos de cartões já pontuados (com o resultado oficial). 1 de 3–4 recebida: AR-SP_20260701 (233).
- [x] AR-SP = **Ar Spring** (25 m), categoria usada (TX200).
- [ ] Confirmar se a AR-SP tem limite de energia.
- [x] AR-UN: 50 m, sem limite de energia.
- [x] Tetos de energia: AR-LV 16 J; AR-HV 27 J.
- [x] Regra fora do anel 5: 4 pontos dentro do quadrado, 0 fora; First Shot Error; dois tiros na mesma caixa (menor + −1).
