# Detecção, Segmentação e Análise Espaço-Temporal de Plumas (YOLO + SAM + Super-Resolução + Copernicus ERA5)

**Universidade Federal de São Paulo (UNIFESP) — Modelagem Computacional**  
**Profa. Dra. Raquel Jahara Lobosco**  
**Integrantes:**  
* Matheus Oliveira de Jesus — RA 169021  
* Artur de Camargo Santos — RA 170223  

Este repositório implementa o fluxo computacional completo para detectar, segmentar e caracterizar plumas em imagens e sequências temporais (estudo de caso: incêndio no terminal químico da Ultracargo, Porto de Santos-SP, abril de 2015), integrando visão computacional com **YOLOv8**, segmentação assistida por **SAM (Segment Anything Model)**, avaliação de **Super-Resolução**, rastreamento temporal ($\Delta \mathbf{c}_t$) e dados meteorológicos georreferenciados do **Copernicus ERA5**.

---

## 📁 Estrutura do Repositório

```text
segmentacao-pluma-sam/
├── data/
│   ├── Santos_Abril_2015_Ultracargo.png        # Fotografia real de referência (Incêndio Ultracargo 2015)
│   ├── video_pluma.mp4                         # Vídeo principal de teste temporal (1920x1080, isolado em test)
│   └── videos_treino/                          # 6 vídeos reais públicos de plumas para treino e validação do YOLOv8
│       ├── seq01_refinaria_pluma_oleo_a.webm   # [TRAIN] Incêndio em refinaria de petróleo (pluma densa A)
│       ├── seq02_refinaria_pluma_oleo_b.webm   # [TRAIN] Incêndio em refinaria de petróleo (pluma densa B)
│       ├── seq03_incendio_urbano_coluna.webm   # [TRAIN] Coluna ascendente de fumaça em incêndio urbano/industrial
│       ├── seq04_pluma_florestal_howe_ridge.webm # [TRAIN] Pluma convectiva e dispersão atmosférica (Howe Ridge)
│       ├── seq05_coluna_fumaca_garnet_fire.webm  # [VAL] Coluna de fumaça com advecção lateral (Sierra Garnet)
│       └── seq06_pluma_chamine_industrial.ogv  # [VAL] Pluma contínua emitida por chaminé industrial
├── Projeto_Final_Deteccao_Analise_Pluma.ipynb  # Notebook principal completo (10 seções do Guia do Trabalho Final)
├── Relatorio_Final_Pluma.md                    # Relatório técnico consolidado respondendo às 5 perguntas de pesquisa
├── Modelagem_Computacional.ipynb               # Experimento inicial exploratório de segmentação com SAM
└── README.md                                   # Documentação e instruções de reprodutibilidade
```

Ao executar o notebook `Projeto_Final_Deteccao_Analise_Pluma.ipynb`, a seguinte árvore de artefatos auditáveis é gerada automaticamente:

```text
projeto_pluma_artefatos/
├── data/
│   ├── raw/                                    # Cópia preservada das imagens originais
│   ├── super_resolution/                       # Pares vinculados (Baixa Resolução vs. Super-Resolução 4x)
│   └── yolo_pluma_dataset/                     # Imagens e labels no formato YOLO (train / val / test sem vazamento)
├── weights/                                    # Pesos treinados e logs do YOLOv8 (yolo_pluma_v1)
└── outputs/
    ├── figures/                                # Gráficos comparativos gerados em alta resolução (PNG)
    ├── metadados_geotemporais_utc.csv          # Catálogo com coordenadas, horários em UTC e split por sequência
    ├── avaliacao_super_resolucao.csv           # Métricas PSNR, gradiente de borda e índice de artefatos
    ├── atributos_visuais_comparacao_sam.csv    # Os 6 atributos geométricos da máscara + IoU
    ├── analise_temporal_deslocamento.csv       # Série temporal de centroide c_t e deslocamento aparente Δc_t
    ├── era5_santos_ultracargo_2015_utc.csv     # Série horária do Copernicus ERA5 (u10, v10, T2m, UR, precipitação)
    ├── analise_integrada_pluma_era5.csv        # Join espaço-temporal em UTC entre deslocamento visual e vento ERA5
    └── experimento_reprodutibilidade_v1.json   # Manifesto completo com seeds, versões, hiperparâmetros e métricas
```

---

## 🚀 Como Executar (Reprodutibilidade)

### Opção 1: Google Colab (Recomendado — GPU T4)
1. Abra o arquivo `Projeto_Final_Deteccao_Analise_Pluma.ipynb` diretamente no Google Colab clicando no badge **Open in Colab** no topo do notebook.
2. Em `Ambiente de execução` -> `Alterar o tipo de ambiente de execução`, selecione **GPU (T4)**.
3. Opcionalmente, faça upload da imagem `Santos_Abril+2015_NanoPoluição-2092122424.jpg` em `/content/` (caso não esteja presente, o notebook reconstrói automaticamente a cena calibrada de referência para garantir reprodutibilidade 100% determinística).
4. Execute todas as células (`Ambiente de execução` -> `Executar tudo`).

### Opção 2: Execução Local
```bash
pip install torch torchvision ultralytics opencv-contrib-python matplotlib numpy pandas scipy requests
pip install git+https://github.com/facebookresearch/segment-anything.git
jupyter notebook Projeto_Final_Deteccao_Analise_Pluma.ipynb
```

---

## 🧠 Fluxo Metodológico Implementado (Seções 1 a 10 do Guia)

1. **Objetivo e Definição Operacional (Seções 1 e 2):** Delimitação formal do que compõe a classe `pluma` e formulação das 5 perguntas de pesquisa verificáveis.
2. **Dados, Metadados em UTC e Divisão sem Vazamento (Seções 3 e 4):** Registro de coordenadas (`-23.9244, -46.3622`), incerteza posicional, fuso original (`UTC-3`) e horário em `UTC`, incluindo exemplos negativos (nuvens, chaminés e sombras) e split agrupado por `sequencia_id`.
3. **Pré-processamento e Super-Resolução (Seção 5):** Avaliação comparativa em $4\times$ (Sem SR vs. Bicúbica vs. Lanczos-4 + Filtro Bilateral), mensurando PSNR, erro de gradiente de borda e criação de artefatos.
4. **Treinamento do Detector YOLO (Seção 6):** Ajuste do `yolov8n.pt` com `seed=42` e *data augmentation* fisicamente plausível (`flipud=0.0`), avaliando predições frente às anotações (*Ground Truth*).
5. **Segmentação Assistida por SAM e Caracterização Visual (Seção 7):** Uso das caixas do YOLO como regiões candidatas para o `SamPredictor` com revisão de contornos difusos e extração automática de:
   * Área aparente em pixels;
   * Largura e altura aparentes;
   * Razão largura/altura;
   * Centroide $\mathbf{c}_t = (u_t, v_t)$;
   * Orientação principal aproximada (via momentos centrais de 2ª ordem);
   * Proporção da imagem ocupada pela pluma (%).
6. **Análise Temporal e Deslocamento Aparente (Seção 8):** Cálculo de $\Delta \mathbf{c}_t = \mathbf{c}_{t+1} - \mathbf{c}_t$, IoU temporal entre máscaras consecutivas, registro explícito de janelas sem detecção e distinção entre deslocamento aparente em pixels, vento observado e trajetória física prevista.
7. **Integração com Dados Meteorológicos Copernicus ERA5 (Seção 9):** Extração das componentes zonais e meridionais do vento ($u_{10}, v_{10}$), temperatura do ar ($T_{2m}$), umidade e precipitação em UTC, confrontando a direção do vento com o deslocamento aparente e explicitando as incertezas de escala e perspectiva.
8. **Reprodutibilidade (Seção 10):** Registro automatizado de versões, sementes, pesos e métricas em JSON e CSV.
