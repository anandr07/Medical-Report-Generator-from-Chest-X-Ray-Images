# Automated Radiology Report Generation from Chest X-Rays

## Table of Contents
- [Introduction](#introduction)
- [Dataset Description](#dataset-description)
- [Methodology](#methodology)
  - [1. Data Collection and Preprocessing](#1-data-collection-and-preprocessing)
    - [a. Data Extraction](#a-data-extraction)
    - [b. Data Pre-processing](#b-data-pre-processing)
    - [c. Dataset Split](#c-dataset-split)
  - [2. Extracting Labels Using CheXbert](#2-extracting-labels-using-chexbert)
  - [3. ChexNet for Structural Findings Extraction](#3-chexnet-for-structural-findings-extraction)
  - [4. Model Architectures](#4-model-architectures)
    - [Model 1: BioVilt + Alignment + BioGPT](#model-1-biovilt--alignment--biogpt)
    - [Model 2: BioVilt + ChexNet + Alignment + BioGPT](#model-2-biovilt--chexnet--alignment--biogpt)
- [References](#references)

---

## Introduction

In modern healthcare, radiology plays an essential role in diagnosing and managing numerous medical conditions. **Chest X-rays** are among the most widely used diagnostic tools to detect abnormalities such as **Pneumonia, Hernia, and Cardiomegaly**.

**Project Motivation:**  
This project aims to **automate the generation of preliminary radiology reports** from chest X-ray images by leveraging advanced computer vision techniques and large language models. This system serves as an aid for radiologists by:
- Enhancing productivity
- Reducing delays
- Minimizing errors due to workload fatigue

The report covers:
- An overview of the dataset structure and features
- Detailed methodology and preprocessing steps
- Model design, training, evaluation, and optimization techniques
- Performance metrics and analysis
- Potential further improvements

---

## Dataset Description

The project uses the **MIMIC-CXR** dataset, which includes:
- **15,000 chest X-ray images** (originally in DICOM format, converted to PNG)
- Associated radiology reports in XML format

**Key Dataset Features:**
- **Image File Path:** Link or location of the corresponding chest X-ray image.
- **Findings:** Textual descriptions of abnormalities or observations.
- **Impression:** A concise summary of the primary conclusions.

**Pathology Labels (14 Total):**
- Atelectasis
- Cardiomegaly
- Consolidation
- Edema
- Enlarged Cardiomediastinum
- Fracture
- Lung Lesion
- Lung Opacity
- Pleural Effusion
- Pleural Other
- Pneumonia
- Pneumothorax
- Support Devices
- No Finding

---

## Methodology

The project is structured into several key stages:

### 1. Data Collection and Preprocessing

#### a. Data Extraction
- **DICOM to PNG Conversion:**  
  A custom script converts the original DICOM images to PNG format, reducing file size while preserving image quality for efficient loading and processing.

- **CSV Creation:**  
  A dedicated script extracts the following fields:
  - `image_ID`: Unique identifier for each image.
  - `image_path`: Consolidated file paths to each PNG image.
  - `findings` and `impressions`: Parsed from XML reports.

#### b. Data Pre-processing
- **Text Cleaning:**  
  - Expanding abbreviations (e.g., "lat" → "lateral")
  - Removing special characters
  - Fixing spacing around punctuation

- **Filtering and Label Mapping:**  
  Invalid or missing entries are removed, and findings are mapped to a list of specific disease labels.

- **Image Augmentation:**  
  Applied techniques include:
  - Resizing to (224, 224)
  - Random rotations and flips
  - Noise addition
  - Normalization

#### c. Dataset Split
A custom function `get_dataloaders` creates PyTorch DataLoader objects for training and validation with parameters:
- `batch_size`: Default is 8.
- `train_split`: 85% training, 15% validation.
- `num_workers`: Default is 4 for faster loading.
- `collate_fn`: Custom function to merge samples, particularly for variable-length inputs like text.

---

### 2. Extracting Labels Using CheXbert

**CheXbert** is a transformer-based model fine-tuned for medical text classification using the BERT architecture. It extracts multi-label classifications from chest X-ray radiology reports.

#### Process:
1. **Text Processing:**  
   - Extract "Findings" and "Impressions" from reports.
   - Tokenize and format text for CheXbert.
   - Generate high-dimensional contextual embeddings.

2. **Label Extraction:**  
   - A classification layer predicts probabilities for each clinical condition.
   - Probabilities are thresholded at **0.5** to produce binary labels.

3. **Dataset Preparation:**  
   The binary labels are integrated into a CSV file to enrich the dataset for multi-label classification.

![image](https://github.com/user-attachments/assets/29b4921c-d5e8-431d-ba86-8b73ca16b8b6)

---

### 3. ChexNet for Structural Findings Extraction

**ChexNet** (based on DenseNet-121) is fine-tuned for multi-label classification of chest X-rays, focusing on structural abnormalities.

#### Key Points:
- **Base Model:** DenseNet-121 with pre-trained ImageNet weights.
- **Layer Freezing:**  
  Initial layers are frozen; only the last two dense blocks and the classifier head are fine-tuned.
- **Custom Classifier:**  
  - **Input:** 1024 features from DenseNet-121.
  - **Hidden Layer:** 512 units with ReLU activation.
  - **Dropout:** 0.3 for regularization.
  - **Output:** 14 sigmoid-activated nodes for multi-label classification.
- **Training Procedure:**  
  - **Loss Function:** Custom Weighted Binary Cross-Entropy Loss (WeightedBCELoss)
  - **Optimizer:** Adam with differential learning rates.
  - **Scheduler:** ReduceLROnPlateau.
  - **Metric:** Achieved an F1-micro score of **0.70**.

---

### 4. Model Architectures

Two distinct model architectures were experimented with to generate medical reports:

#### Model 1: BioVilt + Alignment + BioGPT

1. **Components:**
   - **BioVilt:**  
     - Uses a ResNet backbone (ResNet-50/ResNet-18) for feature extraction.
     - Produces a 512-dimensional global embedding.
   - **Alignment Module:**  
     - Bridges image embeddings with textual representations.
   - **BioGPT:**  
     - A powerful GPT-2 based language model pre-trained on biomedical literature (approx. 347M parameters).

2. **Configuration:**
   - **BioVilt:**  
     - Backbone: ResNet-50  
     - Output: 512-dimensional embedding.
   - **Alignment Module:**  
     - Text encoder: Microsoft BioGPT.
     - Projection layers map image embeddings to BioGPT’s 768-dimensional space.
     - Loss Function: Contrastive Loss.
   - **BioGPT (PEFT via LoRA):**  
     - **Rank:** 16  
     - **Alpha:** 32  
     - **Dropout:** 0.1  
   - **Generation Parameters:**  
     - `max_length`: 150 tokens  
     - `temperature`: 0.8  
     - `top_k`: 50  
     - `top_p`: 0.85  

3. **Integration and Flow:**
   - **Image Preprocessing:** Resize and augment PNG images.
   - **Image Encoding:** BioVilt extracts image features.
   - **Alignment:** Projects image embeddings to align with BioGPT's text embeddings.
   - **Report Generation:** The aligned embeddings are fed into BioGPT to generate the final report.

---

#### Model 2: BioVilt + ChexNet + Alignment + BioGPT

1. **Components:**
   - **BioVilt:**  
     - ResNet-50 based image encoder.
   - **ChexNet:**  
     - Multi-label classifier (DenseNet-121) for structural findings.
   - **Alignment Module:**  
     - Integrates image and label embeddings with text embeddings.
   - **BioGPT:**  
     - Fine-tuned for biomedical report generation.

2. **Configuration:**
   - **BioVilt:**  
     - Backbone: ResNet-50  
     - Output: 512-dimensional embedding.
   - **ChexNet:**  
     - Backbone: DenseNet-121  
     - Output: Multi-label predictions for 14 clinical findings.
   - **Alignment Module:**  
     - Text encoder: Microsoft BioGPT.
     - Projection layers map image embeddings to 768 dimensions and separately project text from the ground truth reports.
     - Loss Function: Contrastive Loss.
   - **BioGPT (PEFT via LoRA):**  
     - **Rank:** 16  
     - **Alpha:** 32  
     - **Dropout:** 0.1  
   - **Generation Parameters:**  
     - `max_length`: 150 tokens  
     - `temperature`: 0.8  
     - `top_k`: 50  
     - `top_p`: 0.85  

3. **Integration and Flow:**
   - **Image Preprocessing:** Resize and augment PNG images.
   - **Image Encoding:** BioVilt extracts image features.
   - **ChexNet Classification:** Identifies structural findings and generates binary labels.
   - **Alignment:** Combines image embeddings with label information and projects them to align with BioGPT’s text embeddings.
   - **Concatenation:** The image embeddings and prompt text embeddings (with a `<SEP>` token separator) are concatenated.
   - **Report Generation:** The concatenated embeddings are fed into BioGPT to generate the final report.

---

## References

A comprehensive list of sources and research papers is available upon request or can be found within the project documentation. Key references include:

- **CheXbert:** [Link to CheXbert Paper/Repository]
- **ChexNet:** [Link to ChexNet Paper/Repository]
- **BioVilt:** [Link to BioVilt Paper/Repository]
- **BioGPT:** [Link to BioGPT Paper/Repository]
- **PEFT Techniques (LoRA):** [Link to Relevant Documentation or Research]

---

*This project demonstrates a synergistic approach combining computer vision and natural language processing to assist radiologists by generating detailed preliminary reports from chest X-ray images.*

Feel free to explore the repository for code, experiments, and further documentation.
