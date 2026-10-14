# 4th Federated Machine Learning International Summer School: Federated Learning, Privacy Attacks, Poisoning, and Robust Aggregation

This repository contains a sequence of hands-on notebooks for studying **federated learning (FL)**, **privacy leakage from model updates**, **data poisoning attacks**, and **robust aggregation defenses**.

The notebooks use a common image-classification task based on the **Google QuickDraw** dataset and a lightweight **MobileNetV4** model. The sequence starts from centralized fine-tuning, moves to local and federated training, demonstrates privacy and poisoning attacks, and finishes with an FLTrust-based defense.

The material is designed for teaching, experimentation, and reproducible research.

## Learning path

| # | Notebook | Main topic | What it demonstrates |
|---|---|---|---|
| 1 | `01.Centralized_Fine_tuning_of_QuickDraw_Dataset_MobileNet_V4.ipynb` | Centralized fine-tuning | Builds the common QuickDraw classifier and saves the initial checkpoint used by later notebooks. |
| 2 | `02.Single_User_Finetuning_and_Server_Attack.ipynb` | Single-client privacy leakage | Simulates local single-client fine-tuning, constructs the uploaded model update, recovers label information, and performs image reconstruction from the observed update. |
| 3 | `03.One_Round_FedAvg_Global_Update_Attack.ipynb` | FedAvg update inversion | Studies reconstruction from a one-round aggregated FedAvg update while varying the number of participating clients. |
| 4 | `04.Full_Federated_Learning_No_Attack.ipynb` | Clean federated learning | Creates a heterogeneous 10-client population and establishes the clean FedAvg baseline. |
| 5 | `05.Federated_Learning_Data_Poisoning.ipynb` | Data poisoning | Reuses the exact clients from Notebook 4 and evaluates label-flipping and backdoor poisoning under FedAvg. |
| 6 | `06.Federated_Learning_Data_Poisoning_FLTrust.ipynb` | Robust aggregation | Repeats the poisoning experiments with FLTrust and compares robustness with the FedAvg baseline. |

## Fine-Tuned Model

The repository includes the centrally fine-tuned MobileNetV4-Conv-Small-0.5 model used throughout the experiments.

| File | Description |
|---|---|
| [mobilenetv4_small_quickdraw10.safetensors](artifacts/centralized_finetuning/mobilenetv4_small_quickdraw10.safetensors) | Complete fine-tuned model checkpoint |
| [config.json](artifacts/centralized_finetuning/config.json) | Model configuration, class labels, and preprocessing settings |

The checkpoint was initialized from the ImageNet-1k pretrained [`mobilenetv4_conv_small_050.e3000_r224_in1k`](https://huggingface.co/timm/mobilenetv4_conv_small_050.e3000_r224_in1k) model and centrally fine-tuned on ten classes of Google's Quick, Draw! dataset.

The resulting model achieves **97.50% test accuracy** and serves as the common initialization for the federated fine-tuning, privacy leakage, and data poisoning experiments.

## Experimental task

The common classification task contains 10 QuickDraw classes:

- `line`
- `circle`
- `square`
- `triangle`
- `star`
- `tshirt`
- `smiley_face`
- `sun`
- `cloud`
- `donut`

Notebook 1 fine-tunes the ImageNet-pretrained model:

```text
mobilenetv4_conv_small_050.e3000_r224_in1k
```

implemented through `timm`.

QuickDraw images are originally 28 x 28 grayscale bitmaps. The notebooks adapt them to the input representation expected by the pretrained MobileNetV4 model.

## Repository workflow

The notebooks are intentionally connected.

```text
Notebook 1
Centralized QuickDraw fine-tuning
        |
        +-----------------------+
        |                       |
        v                       v
Notebook 2                  Notebook 3
Single-client               One-round FedAvg
update inversion            global-update inversion
        |
        |
        v
Notebook 4
Clean federated baseline
        |
        +-----------------------+
        |                       |
        v                       v
Notebook 5                  Notebook 6
FedAvg + poisoning          FLTrust + poisoning
```

### Important dependency

Run **Notebook 1 before the other experiments** because it creates the shared model checkpoint and configuration.

Run **Notebook 4 before Notebooks 5 and 6**. Notebooks 5 and 6 load the exact client datasets persisted by Notebook 4 instead of generating a new population. This keeps the attack and defense experiments directly comparable.

## Notebook details

### 1. Centralized fine-tuning

Notebook 1 creates the common starting point for the complete lab sequence.

It:

- downloads the selected public QuickDraw classes;
- constructs stratified training, validation, and test splits;
- converts the grayscale sketches for MobileNetV4;
- loads ImageNet-pretrained MobileNetV4 weights;
- fine-tunes the model on the 10-class task;
- selects the checkpoint using validation performance;
- evaluates the final model;
- saves the model as `safetensors`;
- saves the task and preprocessing configuration to JSON.

Default output directory:

```text
./outputs/
```

Later notebooks expect the Stage 1 configuration and checkpoint from this directory.

### 2. Single-client fine-tuning and server-side privacy attack

Notebook 2 simulates a client holding a very small private local dataset containing one example from each of five classes:

```text
line, circle, square, triangle, star
```

The client receives the common model, performs a local update, and uploads its parameter delta.

The notebook then studies what an honest-but-curious server may infer from that update. It includes:

- local fine-tuning;
- client runtime and memory measurements;
- construction of the uploaded model delta;
- recovery of the first-step gradient under the experiment assumptions;
- label-set inference;
- image reconstruction from the observed classifier-head gradient;
- quantitative reconstruction metrics;
- saved attack artifacts.

Default output directory:

```text
./outputs_user1/
```

### 3. One-round FedAvg global-update inversion

Notebook 3 moves from a single local update to an **aggregated FedAvg update**.

It constructs a simulated client pool, performs one federated round, and evaluates how aggregation affects information leakage. The experiment sweeps:

```text
1, 2, 5, 10 participating clients
```

The attacker is not assumed to know which globally available classes were excluded from the simulated clients unless the corresponding oracle/ablation option is explicitly enabled.

Default output directory:

```text
./outputs_attack2_multi_user_v2/
```

### 4. Plain federated learning baseline

Notebook 4 establishes the clean FL reference experiment.

The default setup uses:

```text
Clients:       10
FL rounds:     50
Local epochs:  1
Aggregator:    sample-weighted FedAvg
```

The notebook creates heterogeneous client datasets, persists those datasets for later controlled experiments, performs clean federated training, and records convergence and system metrics.

Default output directory:

```text
./outputs_federated_no_attack_v2/
```

Among the saved artifacts are the client datasets and their manifest. These files are reused directly by Notebooks 5 and 6.

### 5. Federated data poisoning with FedAvg

Notebook 5 keeps the training rule and client population from Notebook 4 and introduces malicious clients.

Two poisoning mechanisms are evaluated:

**Label flipping**

```text
source class: circle
target class: donut
flip rate:    100%
```

**Backdoor poisoning**

A small trigger is inserted into a fraction of malicious-client training images and those samples are assigned to the target class.

The default trigger configuration uses a 4 x 4 patch and a 20% malicious-client sample poisoning rate.

The malicious-client fractions evaluated are:

```text
0%, 10%, 30%, 50%, 100%
```

The main metrics are:

- clean test loss;
- clean test accuracy;
- attack success rate (ASR);
- poisoning counts;
- runtime and resource measurements.

Default output directory:

```text
./outputs_attack3_poisoning/
```

### 6. Federated data poisoning with FLTrust

Notebook 6 repeats the poisoning study using **FLTrust** instead of ordinary FedAvg aggregation.

The client datasets, model initialization, local optimizer, number of rounds, local epochs, poisoning rules, malicious fractions, and evaluation task are kept aligned with Notebook 5.

FLTrust introduces a small trusted server-side root dataset. The notebook requests up to:

```text
100 trusted samples per class
```

and uses the trusted update to score and rescale client updates before aggregation.

In addition to the clean-accuracy and ASR metrics from Notebook 5, the notebook records FLTrust diagnostics such as:

- client cosine similarity with the trusted update;
- non-negative trust score;
- client update norm;
- trusted-update norm;
- trust behavior for honest and malicious clients.

Default output directory:

```text
./outputs_attack4_fltrust/
```

## Recommended execution order

For the complete summer school workflow:

```text
1 -> 2 -> 3 -> 4 -> 5 -> 6
```

For only the poisoning and robust-aggregation experiments:

```text
1 -> 4 -> 5 -> 6
```

Notebooks 2 and 3 are independent attack demonstrations after Notebook 1 and are not prerequisites for Notebook 4.

## Installation

Each notebook contains its own dependency-installation cell. The main Python packages used across the series are:

```text
torch
torchvision
timm
safetensors
numpy
pandas
matplotlib
Pillow
requests
tqdm
psutil
```

The notebooks are written in a Jupyter/Google-Colab style and automatically use CUDA when it is available. CPU execution is also possible, although model training and reconstruction attacks can take substantially longer.

No fixed package lockfile is currently included, so exact results can vary slightly across library and hardware versions.

## Data

The notebooks download the required QuickDraw bitmap files directly from the public QuickDraw data source.

To keep the experiments manageable for teaching, Notebook 1 caps the working centralized dataset instead of loading the full QuickDraw corpus.

Later FL notebooks reconstruct the same task definition and use deterministic seeds and persisted client data where required for controlled comparisons.

## Reproducibility

The notebooks use fixed seeds for Python, NumPy, and PyTorch where applicable.

For the most comparable results:

1. start from a clean runtime;
2. run the notebooks in the recommended order;
3. do not change the class ordering in Notebook 1;
4. keep the Stage 1 checkpoint and `config.json` together;
5. do not regenerate the Notebook 4 clients before comparing Notebooks 5 and 6;
6. compare attack and defense runs using the same malicious fraction and poisoning type.

GPU operations and differences between software versions may still introduce small numerical variations.

## Key metrics

Across the notebooks, the main quantities of interest include:

- classification loss and accuracy;
- local and federated training time;
- RAM and, when CUDA is available, GPU-memory use;
- reconstruction quality for privacy attacks;
- attack success rate for poisoning attacks;
- FLTrust trust and cosine-similarity diagnostics.

The poisoning notebooks should always be interpreted using **both clean utility and attack success rate**. A defense that reduces ASR while severely damaging clean accuracy is not automatically a useful defense.

## Educational and responsible use notice

The privacy-inversion and poisoning notebooks implement adversarial techniques so that students and researchers can understand information leakage and robustness in federated learning.

Use these experiments only on systems, models, and data for which you have authorization. They are intended for controlled research, education, benchmarking, and defensive evaluation.

The reconstruction results are experiment-specific demonstrations and should not be interpreted as showing that every federated-learning configuration leaks the same information. Leakage depends on the model, observable update, local training procedure, batch size, aggregation, prior knowledge, and other assumptions.

Likewise, the poisoning and FLTrust results characterize the controlled setup implemented here and should not be generalized to all FL deployments without additional evaluation.

## License

The repository code is released under the **Apache License 2.0**. See [`LICENSE`](LICENSE).

This repository includes third-party pretrained model weights and uses the Google Quick, Draw! dataset. See [`NOTICE`](NOTICE) for attribution and licensing information.

## Acknowledgment

This material was prepared as a hands-on sequence for the **4th Federated Machine Learning International Summer School**, focusing on federated learning, privacy leakage, adversarial data poisoning, and robust aggregation.

