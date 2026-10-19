# Segmentação de Plumas com SAM (Segment Anything Model)

Este repositório contém a modelagem computacional e os experimentos voltados para a detecção e segmentação automática de plumas (como plumas de fumaça, emissões atmosféricas ou dispersão de poluentes/fluidos) em imagens, utilizando o modelo **SAM (Segment Anything Model)** da Meta AI.

---

## 📋 Sobre o Projeto

A identificação e delimitação precisa de plumas em imagens ou vídeos é um desafio importante em áreas como monitoramento ambiental, sensoriamento remoto e dinâmica de fluidos computacional. Métodos tradicionais de limiarização (thresholding) muitas vezes falham devido à variação de iluminação, transparência da pluma e fundos complexos.

Este projeto explora a capacidade *zero-shot* e de *prompting* (pontos, caixas delimitadoras ou máscaras) do **Segment Anything Model (SAM)** para isolar e segmentar regiões de pluma de maneira robusta e automatizada.

---

## 📁 Estrutura do Repositório

```text
segmentacao-pluma-sam/
├── Modelagem_Computacional.ipynb  # Notebook principal com o pipeline de pré-processamento, inferência com SAM e análise de resultados
└── README.md                      # Documentação do projeto
```

---

## 🚀 Tecnologias e Bibliotecas Utilizadas

O desenvolvimento deste projeto baseia-se na linguagem **Python** e nas seguintes ferramentas principais:

* **PyTorch / Torchvision**: Backend de Deep Learning para execução do modelo.
* **Segment Anything (`segment-anything`)**: Implementação oficial do modelo SAM.
* **OpenCV (`cv2`) & Pillow**: Carregamento, manipulação e pré-processamento de imagens.
* **NumPy**: Operações matriciais e manipulação das máscaras binárias geradas.
* **Matplotlib**: Visualização das imagens originais, prompts aplicados e sobreposição das máscaras de segmentação.
* **Jupyter Notebook / Google Colab**: Ambiente interativo de modelagem computacional.

---

## ⚙️ Como Executar o Projeto

### 1. Pré-requisitos
Certifique-se de ter o Python (3.8+) instalado. É altamente recomendável o uso de uma GPU (CUDA) para acelerar a inferência do modelo SAM (ou executar diretamente via Google Colab).

### 2. Clonar o Repositório
```bash
git clone https://github.com/SEU-USUARIO/segmentacao-pluma-sam.git
cd segmentacao-pluma-sam
```

### 3. Instalar as Dependências
Instale os pacotes necessários no seu ambiente virtual:
```bash
pip install torch torchvision
pip install git+https://github.com/facebookresearch/segment-anything.git
pip install opencv-python matplotlib numpy jupyter
```

### 4. Baixar os Pesos do Modelo (Checkpoints)
Para rodar o SAM, é necessário baixar um dos *checkpoints* oficiais pré-treinados (se o próprio notebook já não fizer o download automático via `wget`):
* **ViT-H (Huge - Recomendado):** [sam_vit_h_4b8939.pth](https://dl.fbaipublicfiles.com/segment_anything/sam_vit_h_4b8939.pth)
* **ViT-L (Large):** [sam_vit_l_0b3195.pth](https://dl.fbaipublicfiles.com/segment_anything/sam_vit_l_0b3195.pth)
* **ViT-B (Base - Mais leve):** [sam_vit_b_01ec64.pth](https://dl.fbaipublicfiles.com/segment_anything/sam_vit_b_01ec64.pth)

### 5. Executar o Notebook
Inicie o Jupyter Notebook e abra o arquivo `Modelagem_Computacional.ipynb`:
```bash
jupyter notebook Modelagem_Computacional.ipynb
```

---

## 🧠 Metodologia Geral

1. **Aquisição e Pré-processamento**: Carregamento das imagens das plumas, conversão do espaço de cores (BGR para RGB) e normalização, se aplicável.
2. **Configuração do SAM**: Inicialização da arquitetura escolhida (`vit_h`, `vit_l` ou `vit_b`) e carregamento dos pesos no dispositivo (`cuda` ou `cpu`).
3. **Geração de Máscaras**:
   * Uso do `SamAutomaticMaskGenerator` para segmentação global da cena, ou;
   * Uso do `SamPredictor` guiado por *prompts* (pontos positivos/negativos na região da pluma ou *bounding boxes*).
4. **Pós-processamento e Avaliação**: Filtragem das máscaras geradas, cálculo de área/contorno da pluma e visualização dos resultados sobrepostos à imagem original.

---

## 📊 Resultados Esperados

* Extração de máscaras binárias precisas delimitando o contorno da pluma.
* Separação eficiente entre a pluma e elementos de fundo (céu, vegetação, estruturas industriais ou solo).
* Base quantitativa (como contagem de pixels da máscara) para estimativa de dispersão, área ou evolução temporal do fenômeno.

---

## 📝 Licença

Este projeto é distribuído sob a licença MIT. Consulte o arquivo `LICENSE` (se aplicável) para mais detalhes.
