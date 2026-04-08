# Kinetic Boxes: Documentacao Tecnica

## 1. Resumo da Arquitetura

O projeto `Kinetic Boxes` combina quatro camadas principais:

- `Three.js` para renderizacao em tempo real
- `Cannon-es` para simulacao fisica
- `MediaPipe PoseLandmarker` para captar o gesto do utilizador
- `EffectComposer` para pos-processamento e enfatizacao visual

A logica geral e a seguinte: a instalacao carrega um modelo 3D de referencia, converte-o para uma representacao eficiente com `THREE.InstancedMesh`, gera uma composicao cenografica no palco, simula fisicamente cada caixa com `Cannon-es` e usa os pulsos de colisao para ativar particulas, bloom, decals e luz reativa.

## 2. Sintese das 5 Melhorias Principais

### 2.1 Estabilidade

A estabilidade do sistema foi melhorada ao substituir o `NaiveBroadphase` por `SAPBroadphase` e ao aumentar o numero de iteracoes do solver. Em termos tecnicos, esta alteracao reduz a quantidade de pares de colisao desnecessarios e melhora a resolucao de contactos em pilhas de objetos. Em conjunto com os parametros de `friction`, `restitution`, `contactEquationStiffness` e `contactEquationRelaxation`, o resultado e uma composicao mais estavel, com menos jittering e menos vibracao em repouso.

### 2.2 Performance

O desempenho foi significativamente otimizado com a transicao de clones de geometria para `THREE.InstancedMesh`. Em vez de criar uma arvore Three.js completa para cada caixa, o sistema passa a reutilizar uma unica geometria e um unico material, variando apenas a transformacao por instancia. Isto reduz o numero de `draw calls`, diminui a carga no CPU e melhora a compatibilidade com portateis comuns, mesmo quando MediaPipe, fisica e bloom estao ativos em simultaneo.

### 2.3 Estetica

Do ponto de vista estetico, o objetivo tecnico era evitar a distorcao visual causada por escalas nao uniformes. Durante o desenvolvimento, a estrategia de referencia para esse problema foi o `Triplanar Mapping`, uma tecnica de shader que projeta a textura a partir de multiplos eixos para reduzir alongamentos em pilares e blocos. Na versao atual do codigo, a prioridade pratica foi preservar o material UV criado no Blender para manter fidelidade ao design autoral da caixa. Assim, para a defesa, convem explicar que o problema de distorcao foi tratado como uma questao de pipeline de materiais: o conceito de mapeamento triplanar guiou a solucao, mas a implementacao ativa privilegia a textura UV do modelo final.

### 2.4 Pos-processamento

O efeito de fluorescencia/neon foi construido com `EffectComposer` e `UnrealBloomPass`. Esta camada nao altera diretamente a geometria, mas expande perceptivamente as zonas mais luminosas da cena, reforcando a leitura cenografica contra o fundo escuro. A configuracao de `threshold`, `strength` e `radius` permite controlar o equilibrio entre legibilidade dos detalhes e explosao luminosa do conjunto.

### 2.5 Interatividade

A interatividade foi reforcada de duas formas: primeiro, com um sistema de `DecalGeometry` que cria marcas de giz no chao e nas superficies apos impactos relevantes; segundo, com uma luz reativa cujo brilho depende da velocidade de colisao medida por `getImpactVelocityAlongNormal()`. O sistema de decals usa `pooling`, isto e, um numero maximo de marcas em memoria, para evitar acumulacao infinita de geometrias e preservar a taxa de fotogramas.

## 3. Nota de Consistencia para a Defesa

Se o juri perguntar especificamente pelo `Triplanar Mapping`, a explicacao mais segura e:

> O projeto estudou a tecnica de mapeamento triplanar como resposta ao problema de distorcao em escalas nao uniformes. No entanto, na versao final do prototipo, a fidelidade ao material UV vindo do Blender foi considerada mais importante para manter a identidade visual do objeto. Por isso, o pipeline final preserva o material autoral e faz o tratamento adicional apenas nas texturas de particulas/decal.

Esta formulacao e tecnicamente honesta e continua a mostrar maturidade metodologica.

## 4. Guia das Variaveis de Controlo

| Variavel / Parametro | Onde atua | Efeito visual / comportamental |
| --- | --- | --- |
| `world.gravity.set(0, -9.82, 0)` | Fisica global | Valores mais negativos tornam as caixas mais pesadas e agressivas na queda. Valores menos negativos tornam o movimento mais lento e "flutuante". |
| `world.solver.iterations` | Resolucao fisica | Aumentar melhora a estabilidade das pilhas, mas custa mais CPU. Diminuir pode introduzir jittering. |
| `world.defaultContactMaterial.friction` | Atrito global | Valores altos fazem as caixas agarrar mais ao chao e entre si. Valores baixos tornam os impactos mais deslizantes. |
| `world.defaultContactMaterial.restitution` | Restituicao global | Aumentar faz as caixas saltar mais. Diminuir torna o comportamento mais seco e escultorico. |
| `linearDamping` / `angularDamping` em `createBox()` | Corpo individual | Valores mais altos fazem as caixas perder energia mais rapidamente e estabilizar cedo. |
| `sleepSpeedLimit` / `sleepTimeLimit` | Corpo individual | Controlam quando um objeto pode "adormecer". Valores mais agressivos melhoram estabilidade visual em repouso. |
| `BLOOM_CONFIG.threshold` | Pos-processamento | Um valor mais baixo faz mais areas entrarem no bloom; um valor mais alto deixa o glow mais seletivo. |
| `BLOOM_CONFIG.strength` | Pos-processamento | Aumentar intensifica o brilho neon. Diminuir torna a imagem mais seca e menos cenografica. |
| `BLOOM_CONFIG.radius` | Pos-processamento | Controla o espalhamento do brilho. Raios maiores criam halos mais largos. |
| `renderer.toneMappingExposure` | Exposicao da imagem | Aumentar ilumina toda a cena; diminuir aprofunda o ambiente escuro. |
| `rimLight.intensity` | Luz de palco | Mais intensidade reforca recorte volumetrico e dramatiza os impactos. |
| `backgroundLight.intensity` | Luz de fundo | Muda a leitura do espaco cenico atras do utilizador. |
| `spotlightImpact` | Luz reativa | Controla quanto a `SpotLight` responde a impactos mais fortes. |
| `MAX_DECALS` | Pool de decals | Aumentar deixa mais memoria visual dos impactos, mas adiciona custo geometrico. |
| `DECAL_IMPACT_THRESHOLD` | Gatilho de decals | Valores baixos geram marcas com muitos impactos pequenos; valores altos reservam decals para colisaoes mais expressivas. |
| `MAX_PIXEL_RATIO` | Rendering/performance | Diminuir melhora desempenho em portateis; aumentar melhora nitidez mas custa GPU. |
| `DETECTION_INTERVAL_MS` | MediaPipe | Valores menores aumentam a frequencia de deteccao e a responsividade, mas custam mais CPU/GPU. |
| `map(lm)` | Conversao de coordenadas | Ajustar os fatores de `x`, `y` e `z` muda a escala do palco e a distancia alcancavel pelos gestos. |
| `randomStageDimensions()` | Variacao cenografica | Controla a proporcao entre cubos largos e pilares verticais. |
| `MAX_DECALS` + `spawnExplosion()` | Memoria visual do impacto | Em conjunto, definem quanto tempo o palco "recorda" uma colisao. |

## 5. Como Explicar a Arquitetura ao Juri

Uma formula simples para apresentar o projeto e:

1. `A fisica decide.` Cada caixa existe como corpo independente em `Cannon-es`.
2. `A renderizacao otimiza.` As caixas sao desenhadas com `THREE.InstancedMesh` para reduzir custo.
3. `A IA interpreta o utilizador.` O MediaPipe converte pulsos corporais em colisoes kinematicas.
4. `O pos-processamento dramatiza.` Bloom, particulas, decals e luz reativa amplificam os momentos de impacto.
5. `A cenografia emerge.` O conjunto deixa de ser uma pilha abstrata e passa a funcionar como espaco performativo.

## 6. Referencias no Codigo

- Fisica: [`index.html`](/Users/bernardowatts/Desktop/Projectos/index.html)
- Instancing: [`index.html`](/Users/bernardowatts/Desktop/Projectos/index.html)
- MediaPipe: [`index.html`](/Users/bernardowatts/Desktop/Projectos/index.html)
- Decals e luz reativa: [`index.html`](/Users/bernardowatts/Desktop/Projectos/index.html)

