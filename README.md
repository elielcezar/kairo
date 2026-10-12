# KAIRO — Private volcanic island expeditions

Landing page de um destino fictício. Uma ilha vulcânica **flutuante**, gerada inteiramente em código com Three.js, é coreografada pelo scroll (GSAP + ScrollTrigger + Lenis) ao longo de 8 capítulos. Nesse percurso a câmera entra numa caverna sob a montanha, atravessa uma erupção e acompanha a ilha pousando no oceano.

É uma recriação de estudo de <https://kairo-expedition.aura.build>, feita a partir do briefing [`../kairo.md`](../kairo.md) e corrigida depois contra o site original.

---

## Sumário

1. [Como rodar](#como-rodar)
2. [Arquivos e dependências](#arquivos-e-dependências)
3. [Como o projeto chegou aqui](#como-o-projeto-chegou-aqui)
4. [Arquitetura da página](#arquitetura-da-página)
5. [Os 8 capítulos](#os-8-capítulos)
6. [Como o 3D é gerado](#como-o-3d-é-gerado)
7. [Como o scroll controla a cena](#como-o-scroll-controla-a-cena)
8. [Verificação](#verificação)
9. [Pendências conhecidas](#pendências-conhecidas)
10. [Onde mexer](#onde-mexer)

---

## Como rodar

O `gl.js` é um ES module, então a página **não funciona via `file://`**. Sirva a pasta:

```bash
npx serve kairo
# ou
python -m http.server -d kairo 8000
```

Precisa de internet: as bibliotecas, as fontes, os vídeos e as imagens vêm de CDN e do storage do Supabase.

---

## Arquivos e dependências

| Arquivo | Papel |
|---|---|
| `index.html` | Estrutura dos 8 capítulos, copy, SVGs (marca, mapa da rota, bússola), importmap do Three |
| `style.css` | Tokens de cor e tipografia (temas Day/Night), camadas, layout de cada capítulo, responsivo, reduced motion |
| `scripts.js` | Interface (tema, menu, abas, formulário, relógio, vídeos) + coreografia de scroll que escreve em `window.KAIRO.rig` |
| `gl.js` | A cena 3D inteira: geração da ilha, túnel, efeitos, oceano, luz e o loop de renderização |

| Lib | Versão | Origem |
|---|---|---|
| Three.js (+ `SimplexNoise`, `RoundedBoxGeometry`) | 0.185.1 | jsdelivr, via importmap |
| GSAP + ScrollTrigger | 3.15.0 | jsdelivr |
| Lenis | 1.3.26 | jsdelivr |
| Tailwind CSS | CDN runtime (preflight desligado) | cdn.tailwindcss.com |
| Bodoni Moda + Inter | — | Google Fonts |

**Assets.** São 4 vídeos e 7 imagens, todos no storage `hoirqrkdgbmvpwutwuwj.supabase.co`. O briefing listava as imagens truncadas (`.../<id>_original.jpg`). As URLs completas (bucket `assets/assets/`) foram tiradas do HTML do site original.

---

## Como o projeto chegou aqui

1. **Primeira versão, só com o briefing.** O `kairo.md` descreve uma ilha num oceano, com um hero de 220vh e as seções Two Faces, Passage, Stay e Expedition. A primeira implementação seguiu esse texto ao pé da letra. As imagens ficaram com um fallback porque as URLs estavam truncadas.
2. **Estudo do original.** Comparado com o site real, o briefing estava incompleto. A ilha **flutua no céu**, e depois de Stay a câmera **entra pela montanha**. A página tem 8 capítulos, não 5.
   - O site do aura.build carrega a página num `iframe srcdoc`, que aparece em branco num carregamento simples.
   - Com o Playwright, o HTML desse frame foi extraído e separado em markup, CSS e o módulo JavaScript de 85 KB. Também foram feitos screenshots a cada 1100px de scroll.
3. **Reconstrução.** A página foi reescrita para seguir o original. A copy, o layout e os valores de CSS são os do site. As poses de câmera por capítulo e os tempos de cada efeito (em que fração do scroll a tempestade começa, quando o mar surge etc.) também foram reproduzidos.
   - O **código 3D foi reescrito do zero**. Ele segue a mesma mecânica do original (ilha em grade polar, túnel por curva, mistura de câmera `caveMix`), mas a geometria, os shaders e a iluminação são próprios.
4. **Ajuste visual em loop.** Cada capítulo foi capturado no Chrome real, comparado com o original e corrigido. Os principais problemas encontrados:
   - o túnel ficava invisível por fora;
   - os lampiões estouravam a exposição;
   - a câmera passava metade do capítulo 05 fora da caverna;
   - a paleta da tempestade estava escura demais.

---

## Arquitetura da página

### Pilha de camadas

| Camada | z-index | O quê |
|---|---|---|
| `.loader` | 300 | Tela de carregamento enquanto a ilha é gerada |
| `.header` | 40 | Transparente no topo; com `.is-scrolled` ganha fundo com blur |
| `.menu` | 35 | Menu mobile |
| `.story` | 5 | Todo o conteúdo. `pointer-events: none` nos containers, `auto` só em links, botões, campos e abas |
| `.grain` | 4 | Ruído SVG animado em 3 passos |
| `#scene` | 2 | Canvas WebGL **transparente** |
| `.word` / `#bigWord` | 1 | O "KAIRO" gigante, **atrás** da ilha |
| `.sky` | 0 | Gradiente do céu, sol, nuvens CSS e as camadas `--storm` e `--dawn` |

Como o canvas é transparente, o céu e a palavra em CSS aparecem atrás da ilha, e o texto fica na frente.

### Capítulos "pinados"

Cada capítulo é uma `section.panel` alta (220vh a 380vh) com um `.panel__sticky` de 100vh. O tempo que o usuário passa rolando dentro da seção vira o progresso de 0 a 1 da timeline daquele capítulo.

Faces, Passage, Stay e Expedition têm uma **`.surface` marfim** que cobre a cena enquanto o conteúdo é lido. No fim do capítulo ela desaparece e revela a ilha naquele ponto da jornada. Enquanto uma superfície opaca cobre a tela inteira, o WebGL não renderiza (`KAIRO.coverage`).

### Estado inicial no JS

Os estados iniciais das animações (texto escondido, clip-paths fechados, plates invisíveis) são aplicados com `gsap.set` no `scripts.js`, não no CSS. Sem JavaScript a página continua legível. O loader só aparece com `html.js` e tem uma rede de segurança de 15 s.

---

## Os 8 capítulos

| # | id | Altura | O que acontece na tela | O que acontece na cena |
|---|---|---|---|---|
| 01 | `#island` | 220vh | Hero; as letras de "KAIRO" sobem e somem | Câmera se aproxima e gira a ilha |
| 02 | `#faces` | 260vh | Vídeo dia → noite com wipe `clip-path` e tags Day/Night | Câmera sobe para o cume (coberto pela superfície) |
| 03 | `#passage` | 320vh | Mapa SVG: rota desenhada, barco e itens do itinerário acendendo | A superfície se abre: a ilha vista de longe e de baixo ("The summit, before the sea beneath it") |
| 04 | `#stay` | 320vh | Grid com máscaras e parallax; depois o vídeo do penhasco em tela cheia | Câmera vai até a **boca da caverna** sob o penhasco |
| 05 | `#interior` | 380vh | Plate "Beyond the surface." entra e sai por `clip-path` | Câmera **atravessa o túnel**, passa pela câmara com a piscina termal e sai do outro lado |
| 06 | `#awakening` | 380vh | "The mountain is never still." | Tempestade, erupção, veios de lava e tremor |
| 07 | `#expedition` | 260vh | 5 abas com transição de opacidade e translação | A tempestade se dissipa e a lava esfria; a ilha volta a flutuar entre as ilhotas |
| 08 | `#landfall` | 380vh | "Some places are worth the journey." | O mar surge ao amanhecer, a ilha desce até a linha d'água e as ilhotas afundam |
| — | `#enquire` | auto | Formulário que gera um `mailto:` (nada é enviado) | — |
| — | `#footer` | auto | Marquee, relógio da ilha (UTC−10), capítulos, glow que segue o ponteiro, "KAIRO" vazado | — |

---

## Como o 3D é gerado

Nada é carregado. **Não há modelo, textura de imagem nem HDRI.** Toda a cena nasce de funções matemáticas em `gl.js`.

### 1. Ruído: a matéria-prima

Relevo, contorno, rachaduras, paredes da caverna e até a forma das nuvens vêm do `SimplexNoise` do Three, com uma semente fixa para a ilha sair sempre igual. Em cima dele há três ferramentas:

- **`fbm`**: soma várias camadas de ruído, cada uma com o dobro da frequência e metade da força. Gera o "detalhe dentro do detalhe" de terreno natural.
- **`ridged`**: usa `1 − |ruído|`, que transforma vales suaves em cristas afiadas. São as arestas da montanha.
- **`ss` (smoothstep)**: funciona como máscara ("aplique isto só entre o raio X e Y"). Aceita limites invertidos.

### 2. A ilha (`makeIsland`)

A malha é uma **grade polar**: 110 anéis × 260 segmentos no topo e mais 64 anéis na base, como uma teia de aranha. Esse formato facilita trabalhar com "distância do centro" (`rn`, de 0 no centro a 1 na borda) e com "ângulo".

- **Contorno.** O raio da borda varia com o ângulo via ruído (`edgeR`), por isso a ilha não é um círculo.
- **Topo.** A altura de cada vértice é a soma de várias camadas:
  ```
  base + cone (pico central) + cristas (ridged) + ondulação (fbm) + detalhe fino
       + lábio da cratera − buraco da cratera
  ```
- **Rios.** Para cada um dos 3 ângulos, os vértices perto do leito são puxados até a altura do leito. Isso escava um canal que serpenteia (`riverAngleAt`) até a borda.
- **Base.** Os anéis descem e estreitam até virar uma ponta (`bottomPoint`). Ruído 3D deixa a superfície com cara de rocha rachada. Perto das bocas da caverna o raio é afundado (`caveDent`), criando uma reentrância.
- **Cor por vértice, sem textura.** Cada vértice escolhe a cor pela altura, pela inclinação (derivada da normal) e pela distância do centro:
  - plano: verde;
  - íngreme: rocha;
  - alto: claro;
  - cratera: cinza-escuro;
  - leito: azulado;
  - base: marrom.
- **Veios de lava.** O atributo `aCrack` marca linhas finas de ruído no cone e no fundo da cratera. O `MeshStandardMaterial` recebe um trecho de shader via `onBeforeCompile` que soma uma emissão laranja tremulante proporcional a `uLava`.
- **Costura.** As normais da primeira e da última coluna da grade são mescladas para não aparecer uma linha no ângulo 0.

As **6 ilhotas** usam o mesmo gerador com menos resolução, pico baixo, sem cratera e com outra semente, em escala reduzida.

As **6 suítes** no penhasco são `RoundedBoxGeometry` com um plano emissivo quente fazendo de janela.

### 3. Rios e cachoeiras (`buildRiver`)

- O canal e a queda são **fitas** (`ribbon`): tiras de dois vértices de largura ao longo de uma lista de pontos.
- A queda começa no fim do rio, afasta-se um pouco da borda e desce cerca de 1,9 unidade. Como a base da ilha afina, ela fica pendurada no ar ("three falls that never reach the sea").
- O shader desenha **listras que rolam para baixo**, em faixas com velocidades diferentes, e some nas bordas e na ponta.
- Um sprite de névoa marca o fim da queda.

### 4. A caverna (`buildCave`)

É a peça que permite "entrar pela montanha".

1. **Trajeto.** Uma `CatmullRomCurve3` passa por 9 pontos de controle:
   - de fora da boca de entrada até a boca;
   - para dentro, descendo até a **câmara** perto do centro;
   - subindo até a boca de saída;
   - e para fora de novo.
2. **Tubo feito à mão.** Ao longo da curva, os *Frenet frames* (direções "lado" e "cima" do caminho) posicionam 64 vértices em anel a cada um dos 320 passos.
   - O raio varia com ruído, o que dá paredes irregulares.
   - O chão é achatado.
   - O raio incha numa gaussiana em torno da câmara.
   - A cor por vértice escurece o teto e o chão.
   - O material usa `flatShading` e `DoubleSide`, para ser visível por dentro e também pela abertura, vista de fora.
3. **Furo na ilha.** Com o tubo pronto, a malha da ilha é **reindexada** (`rebuildIndex`). Todo triângulo da base com um vértice dentro do tubo é descartado. É isso que abre a boca real no penhasco.
4. **Interior.**
   - Um disco com relevo faz o chão.
   - A **piscina** usa um shader próprio: azul-esverdeado com cáusticas de ruído e o reflexo do lampião.
   - 16 cones invertidos são as estalactites, e 9 dodecaedros, as pedras.
   - Quatro `PointLight` quentes fazem os lampiões, e uma luz azul-esverdeada sai da piscina. Todas acendem conforme `caveMix`.
5. **Câmera.** `cameraAt(t)` devolve a posição e o ponto de mira sobre a mesma curva. O `t` de 0 a 1 do capítulo é mapeado só para o trecho do túnel, com uma pequena entrada e saída por fora.

### 5. Efeitos

- **Nuvens** (`buildClouds`): 46 sprites com uma textura gerada em canvas (círculo suave deformado por ruído). A maioria fica abaixo da ilha, formando o mar de nuvens. Elas derivam devagar em órbita.
- **Erupção** (`buildEruption`):
  - um brilho aditivo pulsante na cratera;
  - 420 partículas (`Points`) cuja trajetória parabólica é calculada **no vertex shader**, então a CPU não move nenhuma;
  - 14 sprites escuros de fumaça subindo;
  - um valor de `tremor` que sacode a câmera.
- **Oceano** (`buildOcean`): um plano de 180×180 com 280² subdivisões.
  - O vertex shader soma 3 ondas senoidais e calcula a normal.
  - O fragment shader aplica:
    - reflexo do céu nos ângulos rasantes (Fresnel);
    - brilho do sol (dois lóbulos especulares);
    - névoa até o horizonte;
    - um **anel de espuma** onde a ilha toca a água.
  - O raio da ilha na altura do mar é encontrado por **busca binária** sobre a função da base (`radiusAtY`).

### 6. Luz e atmosfera

- Há um sol direcional com sombras (shadow map 2048), uma luz de contorno, uma luz hemisférica, neblina exponencial (`FogExp2`) e tone mapping ACES.
- Três **paletas** guardam cor e intensidade de cada luz, da neblina, das nuvens, do mar e a exposição: `DAY`, `NIGHT` (também usada na tempestade) e `DAWN` (o mar no Landfall).
- A cada frame `applyEnvironment` mistura as paletas conforme:
  - o tema Day/Night, suavizado;
  - `storm`;
  - `sea`;
  - `caveMix`: dentro do túnel o sol e o céu caem quase a zero e a neblina fica escura e densa.

### 7. Construção em etapas (loader)

A geração leva alguns instantes, por isso `build()` roda em etapas com um frame de respiro entre elas. Cada etapa dispara `kairo:progress`, que move a barra do loader e troca o texto:

`Charting the island` → `Raising the falls` → `Cutting the passage` → `Gathering the clouds` → `Filling the sea`

No fim, a cena publica `KAIRO.caveCamera(t)` e dispara `kairo:ready`, que libera a coreografia.

---

## Como o scroll controla a cena

```
scroll (Lenis) ──► ScrollTrigger ──► uma timeline com scrub por capítulo
                                        └─► escreve em window.KAIRO.rig
gl.js, a cada frame ──► lê o rig ──► suaviza a câmera ──► aplica paletas ──► renderiza
```

### O `rig`

| Campo | Efeito |
|---|---|
| `x, y, z` / `lx, ly, lz` | Posição da câmera e ponto para onde ela olha |
| `rot` | Giro da ilha em torno do eixo vertical |
| `caveMix` | 0 = câmera normal; 1 = câmera presa ao trajeto do túnel |
| `caveT` | Posição ao longo do túnel (0 a 1) |
| `storm` | Escurece céu e luz (também move o `--storm` do CSS) |
| `erupt`, `lava`, `cool` | Erupção, veios acesos e resfriamento |
| `sea`, `islandY`, `contact` | Oceano visível, altura da ilha e espuma de contato |
| `idle` | Quanto da flutuação e do parallax do mouse está ativo |

### Poses de câmera

São iguais às do original. Cada capítulo interpola entre elas (`move()` em `scripts.js`):

| Pose | Câmera (x, y, z) | Alvo (lx, ly, lz) | rot | Onde |
|---|---|---|---|---|
| S0 | 0, 0.24, 6.6 | 0, 0.24, 0 | 0 | início do hero |
| S1 | 0.9, 0.55, 4.6 | −0.1, 0.32, 0 | 0.45 | fim do hero |
| S2 | 1.6, 1.05, 3.4 | −0.2, 0.45, 0 | 1.05 | fim de Faces |
| S3 | 0.3, −1.25, 10.5 | 0, 0.15, 0 | 1.9 | Passage, vista de baixo |
| S4 | −1.2, −0.75, 5 | −0.2, −0.05, 0 | 2.3 | fim de Passage |
| CAVE_IN | `caveCamera(0)` | — | 2.15 | boca da caverna (fim de Stay) |
| CAVE_OUT | `caveCamera(1)` | — | 2.15 | saída do túnel |
| S5 | 1.95, 1.5, 2.55 | 0, 0.58, 0 | 2.6 | Awakening, perto do cume |
| S6 | 1.55, 1.7, 2.35 | 0, 0.62, 0 | 2.95 | fim de Awakening |
| S7 | 0.6, 1.9, 6.4 | 0, 0.2, 0 | 3.4 | fim de Expedition |
| S8 | 0.3, 0.9, 8 | 0, −1.3, 0 | 3.9 | meio de Landfall |
| S9 | 0.3, −1.3, 5.6 | 0, −1.95, 0 | 4.25 | ilha na linha d'água |

No mobile (≤ 760px), `fit()` afasta cada pose do alvo e a sobe um pouco, e o FOV passa de 34° para 48°.

### A entrada na montanha, passo a passo

1. Em **Stay**, a câmera viaja de S4 até `CAVE_IN`. Essa pose é calculada pela própria cena como o início exato do trajeto do túnel.
2. No começo de **Interior**, `caveMix` vai de 0 a 1 em 3% do capítulo. A câmera real passa a ser `lerp(pose normal, posição no túnel, caveMix)`.
3. No resto do capítulo, `caveT` vai de 0 a 1 e a câmera percorre o túnel.
4. Em **Awakening**, `caveMix` volta a 0 enquanto a câmera parte de `CAVE_OUT` rumo a S5. A saída é contínua.

### Tempos dos efeitos (fração de cada capítulo)

| Capítulo | Efeito | Início → fim |
|---|---|---|
| Faces | wipe dia → noite | 0.32 → 0.70 |
| Passage | rota no mapa | 0.04 → 0.56 |
| Passage | superfície se abre | 0.62 → 0.78 |
| Stay | vídeo do penhasco | entra 0.72, sai 0.92 |
| Interior | `caveMix` / `caveT` | 0 → 0.03 / 0.03 → 1 |
| Interior | plate do texto | entra 0.42, sai 0.80 |
| Awakening | `storm` | 0.04 → 0.34 |
| Awakening | `erupt` | 0.12 → 0.96 |
| Awakening | `lava` | 0.55 → 0.95 |
| Expedition | tempestade se dissipa / lava esfria | 0 → 0.5 / 0.1 → 0.7 |
| Landfall | `sea` | 0 → 0.30 |
| Landfall | `islandY` 0 → −2.05 | 0.10 → 0.64 |
| Landfall | `contact` (espuma) | 0.50 → 0.74 |
| Landfall | `--dawn` / sol desce | 0 → 0.5 / 0 → 0.6 |

### Suavização e economia

- A câmera persegue o alvo com `lerp` dependente do tempo, e o movimento fica macio mesmo com o scrub.
- Dentro do túnel o seguimento é imediato, para a câmera não atravessar as paredes.
- O render é pulado enquanto uma `.surface` opaca cobre a tela inteira.
- O mouse inclina levemente a ilha e a câmera, em proporção a `idle`.

---

## Verificação

Usei o checker headless da skill `gsap-pagina-do-zero`, mais screenshots de cada capítulo no Chrome instalado:

```bash
python ~/.claude/skills/gsap-pagina-do-zero/scripts/gsap_check.py index.html --width 1440
python ~/.claude/skills/gsap-pagina-do-zero/scripts/gsap_check.py index.html --mobile
```

O resultado esperado é 0 erros, 0 alvos não encontrados e nenhum overflow horizontal. A altura da página dá cerca de 24.206 px, contra 24.186 px no original. Os avisos abaixo são conhecidos:

- **Falhas de rede nos `.mp4`.** O Chromium do Playwright não tem codec H.264. No Chrome real todos os vídeos decodificam.
- **Invisíveis ao final.** São elementos de capítulos que o checker não chegou a revelar no estado em que terminou.
- **Console.** O aviso do Tailwind CDN e "GPU stall due to ReadPixels", que vem do próprio checker.

---

## Pendências conhecidas

- **Mobile, fim do Landfall.** Com as poses afastadas pelo `fit()`, a ilha sai do enquadramento e só aparece o mar com o título.
- **Fidelidade do 3D.** A mecânica e as poses são as do original, mas a geometria, os shaders e a luz foram escritos aqui. O acabamento das rochas, a textura do túnel e o tom das nuvens não são idênticos.
- **O briefing difere do original.** Exemplos: a ilha flutua em vez de ficar no oceano, são 8 capítulos e o relógio fica no footer. Quando os dois divergiam, prevaleceu o **site original**.

---

## Onde mexer

| Quero… | Arquivo / local |
|---|---|
| Mudar o enquadramento de um capítulo | `scripts.js` → poses `S0`…`S9` em `choreograph()` |
| Mudar quando um efeito acontece | `scripts.js` → chamadas `val()` / `move()` do capítulo |
| Altura (duração) de um capítulo | `style.css` → `.panel--*` (e o bloco `@media (max-width: 760px)`) |
| Forma da ilha (pico, cratera, rios) | `gl.js` → `makeIsland()` e as constantes `RIVERS` / `CAVES` |
| Cores do terreno | `gl.js` → objeto `C` dentro de `makeIsland()` |
| Trajeto, largura ou câmara da caverna | `gl.js` → `buildCave()` (pontos `ctrl`, `radiusAt`) |
| Brilho dos lampiões e da piscina | `gl.js` → `setMix()` dentro de `buildCave()` |
| Luz e cores de dia, noite e amanhecer | `gl.js` → paletas `DAY`, `NIGHT`, `DAWN` |
| Ondas, espuma e cor do mar | `gl.js` → shaders em `buildOcean()` |
| Intensidade da erupção | `gl.js` → `buildEruption()` |
| Textos dos passos do loader | `gl.js` → chamadas `progress()` em `build()` |
| Copy, imagens e vídeos | `index.html` |
# kairo
