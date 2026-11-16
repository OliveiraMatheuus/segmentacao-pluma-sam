# Relatório Técnico Final: Projeto de Detecção e Análise Espaço-Temporal de Pluma

**Universidade Federal de São Paulo (UNIFESP) — Instituto de Ciência e Tecnologia (ICT)**  
**Disciplina:** Modelagem Computacional — Profa. Dra. Raquel Jahara Lobosco  
**Integrantes:**  
* Matheus Oliveira de Jesus — RA 169021  
* Artur de Camargo Santos — RA 170223  

---

## Resumo
Este documento apresenta a metodologia, a implementação computacional e a análise crítica do fluxo de trabalho reprodutível desenvolvido para detectar, segmentar e caracterizar plumas de fumaça em imagens e sequências de vídeo. O estudo de caso central aborda o incêndio no terminal químico da Ultracargo (Bairro Alemoa, Porto de Santos-SP, abril de 2015). O pipeline integra catalogação de metadados georreferenciados em UTC, divisão de dados sem vazamento temporal (*data leakage*), avaliação de super-resolução, detecção automática com **YOLOv8**, segmentação assistida pelo **Segment Anything Model (SAM)** guiada pelas caixas do YOLO, extração de seis descritores geométricos, rastreamento temporal de centroide ($\Delta \mathbf{c}_t$) e integração com dados meteorológicos de reanálise do **Copernicus ERA5**.

---

## 1. Objetivo, Escopo e Definição Operacional da Pluma

O objetivo do projeto é identificar automaticamente regiões ocupadas por plumas em imagens e acompanhar a evolução de suas propriedades geométricas ao longo do espaço e do tempo, confrontando o deslocamento visual com condições atmosféricas observadas.

* **Definição da Classe `pluma`:** Compreende a coluna ascendente e a nuvem de dispersão advectiva-difusiva de material particulado gerada pela queima de hidrocarbonetos, incluindo tanto o núcleo opaco de alta concentração de fuligem (preto/cinza-escuro na base) quanto a região periférica translúcida (cinza-claro).
* **Estruturas Excluídas (Exemplos Negativos):** Nuvens meteorológicas (cúmulos e estratos costeiros), estruturas industriais brancas/metálicas (tanques, tubulações e guindastes), sombras topográficas da Serra do Mar e reflexos na lâmina d'água do estuário.

---

## 2. Dados, Metadados Geotemporais e Divisão sem Vazamento (Seções 3 e 4)

Cada observação do catálogo (`metadados_geotemporais_utc.csv`) documenta:
1. **Identificação e Origem:** `image_id`, `sequencia_id`, fonte documental, tipo de sensor/câmera e resolução nativa.
2. **Referência Geográfica e Incerteza:** Coordenadas da área observada no Terminal da Alemoa ($\text{lat} = -23{,}9244^\circ$, $\text{lon} = -46{,}3622^\circ$) com incerteza posicional documentada entre $\pm 200\text{ m}$ e $\pm 400\text{ m}$.
3. **Referência Temporal Padronizada:** Registro simultâneo no fuso original (`America/Sao_Paulo`, UTC-3) e em **Tempo Universal Coordenado (UTC)**, requisito indispensável para o acoplamento com o Copernicus ERA5.
4. **Prevenção de Vazamento Temporal (*Data Leakage*):** A separação em `train`, `val` e `test` foi realizada agrupando-se estritamente por `sequencia_id`. Nenhum quadro consecutivo de uma mesma sequência compartilha partições distintas.

---

## 3. Pré-processamento e Avaliação de Super-Resolução (Seção 5)

O pré-processamento preserva cópia integral dos arquivos brutos (`data/raw`) e aplica conversão para o espaço RGB e normalização radiométrica. Para avaliar o impacto da **Super-Resolução** em cenários onde a pluma ocupa poucos pixels:
* A imagem de referência foi subamostrada em $4\times$ ($235 \times 135\text{ px}$);
* Comparou-se a imagem sem super-resolução, a interpolação Bicúbica ($4\times$) e a reconstrução por **Super-Resolução Lanczos-4 acoplada a filtro bilateral preservador de bordas** ($4\times$).
* **Resultado (PQ-05):** A super-resolução elevou o PSNR e aumentou o IoU da máscara segmentada pelo SAM em relação à referência original, reduzindo o efeito de blocos (*aliasing*) nos contornos difusos sem introduzir artefatos significativos de *overshoot* nas fronteiras com os tanques industriais.

---

## 4. Treinamento do Detector YOLO e Segmentação Assistida por SAM (Seções 6 e 7)

### 4.1 Treinamento do YOLO
Utilizou-se a arquitetura `yolov8n.pt` com resolução de entrada de $640 \times 640\text{ px}$, semente fixa (`SEED = 42`) e hiperparâmetros de aumento de dados fisicamente plausíveis: variações moderadas de brilho/contraste (`hsv_s = 0.35`, `hsv_v = 0.30`), escala (`scale = 0.20`), translação (`translate = 0.08`) e inversão horizontal (`fliplr = 0.50`), mantendo **`flipud = 0.0`** (visto que plumas térmicas possuem orientação gravitacional ascendente).

### 4.2 Acoplamento YOLO → SAM e Atributos Visuais
As caixas delimitadoras preditas pelo YOLO alimentam automaticamente o `SamPredictor`. Como a máscara puramente automática dentro da caixa tende a sofrer sobre-segmentação no contorno difuso superior (incorporando nuvens adjacentes), adicionou-se revisão assistida por pontos positivos e negativos. Da máscara validada são extraídos os **6 atributos obrigatórios**:

1. **Área aparente em pixels ($A_{\text{px}}$)**;
2. **Largura ($L_{\text{px}}$) e altura ($H_{\text{px}}$) aparentes**;
3. **Razão entre largura e altura ($L_{\text{px}} / H_{\text{px}}$)**;
4. **Centroide da região segmentada:** $\mathbf{c}_t = (u_t, v_t) = (M_{10}/M_{00}, \; M_{01}/M_{00})$;
5. **Orientação principal aproximada ($\theta$):** obtida pelos momentos centrais de 2ª ordem $\theta = \frac{1}{2}\arctan2(2\mu_{11}, \mu_{20} - \mu_{02})$;
6. **Proporção da imagem ocupada pela pluma (%).**

*(Na ausência de parâmetros intrínsecos e extrínsecos de calibração da câmera, todas as medidas são reportadas rigorosamente em pixels e proporções aparentes da imagem).*

---

## 5. Análise Temporal e Diferenciação Física (Seção 8)

Na sequência temporal de teste (`SEQ_05_VIDEO_TESTE_TEMPORAL`), as detecções consecutivas foram associadas calculando-se o vetor de deslocamento aparente:

$$\Delta \mathbf{c}_t = \mathbf{c}_{t+1} - \mathbf{c}_t = (u_{t+1} - u_t, \; v_{t+1} - v_t)$$

* **Registro de Intervalos sem Observação:** No instante $t = 3$ (`FRAME_TESTE_T03_OCLUSAO`), o quadro sem detecção foi registrado explicitamente como `SEM_DETECCAO`, interrompendo o cálculo de $\Delta \mathbf{c}_t$ imediato para não simular uma trajetória conhecida durante o intervalo oculto.
* **Diferenciação Conceitual:**
  1. **Deslocamento Aparente ($\Delta \mathbf{c}_t$):** Mudança geométrica bidimensional em pixels no plano do sensor, influenciada pela perspectiva e pela variação de opacidade da fumaça.
  2. **Vento Meteorológico (ERA5):** Vetor físico $\mathbf{V} = (u, v)$ em $\text{m/s}$ estimado em escala de grade sinótica ($0{,}25^\circ \approx 28\text{ km}$).
  3. **Trajetória Prevista por Modelo Físico:** Solução de equações de transporte advectivo-difusivo que exige taxa de emissão na fonte, empuxo térmico (elevação de pluma de Briggs), coeficientes de dispersão turbulenta e condições de contorno, não sendo redutível apenas à imagem ou ao vento superficial isoladamente.

---

## 6. Integração com Dados Meteorológicos do Copernicus ERA5 e Incertezas (Seção 9)

Foram extraídos dados horários em **UTC** do produto `reanalysis-era5-single-levels` para a célula de grade mais próxima de Santos-SP ($\text{lat} = -23{,}9244^\circ, \text{lon} = -46{,}3622^\circ$) durante o evento de abril de 2015:
* Componentes do vento a $10\text{ m}$: $u_{10} > 0$ (escoamento para Leste) e $v_{10} > 0$ (escoamento para Norte), caracterizando vento predominante de Sudoeste/Sul-Sudoeste ($\approx 200^\circ\text{–}222^\circ$) com velocidades entre $3{,}8\text{ m/s}$ e $5{,}6\text{ m/s}$, temperatura do ar $T_{2m}$ entre $23{,}0^\circ\text{C}$ e $27{,}6^\circ\text{C}$ e umidade relativa entre $69\%$ e $86\%$.
* **Compatibilidade e Limites:** O sentido de transporte para Nordeste ($u_{10} > 0, v_{10} > 0$) é fisicamente compatível com o deslocamento aparente da pluma para o quadrante superior direito da imagem ($\Delta u_t > 0, \Delta v_t < 0$). Ressalta-se que essa compatibilidade constitui evidência de associação física plausível, mas está sujeita às incertezas de projeção 2D da câmera, diferença entre vento a $10\text{ m}$ e vento na altitude de equilíbrio térmico da pluma, e resolução espacial da grade do ERA5 frente à topografia costeira de Santos.

---

## 7. Reprodutibilidade (Seção 10)

Todo o experimento é autônomo e executável no notebook `Projeto_Final_Deteccao_Analise_Pluma.ipynb`, que fixa `SEED = 42` em todas as bibliotecas e exporta automaticamente os artefatos auditáveis em `./projeto_pluma_artefatos/outputs/`:
* `metadados_geotemporais_utc.csv`
* `avaliacao_super_resolucao.csv`
* `atributos_visuais_comparacao_sam.csv`
* `analise_temporal_deslocamento.csv`
* `era5_santos_ultracargo_2015_utc.csv`
* `analise_integrada_pluma_era5.csv`
* `experimento_reprodutibilidade_v1.json`
