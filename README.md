# LAPA: Latent Action Pretraining from Videos -- Reproduce Version
[[Original Repo]](https://github.com/LatentActionPretraining/LAPA)
[[Project]](https://latentactionpretraining.github.io/)
[[Paper]](https://arxiv.org/abs/2410.11758)
[[Models]](https://huggingface.co/latent-action-pretraining/LAPA-7B-openx)

## Reproducing Experiments

### Pre-training
Use data from LAPA repo and pretrained [LWM](https://huggingface.co/LargeWorldModel/LWM-Chat-1M-Jax) for 70k steps. The configurations are exactly the same, except using 8 A100 GPUs.
<img width="800" alt="image" src="https://github.com/user-attachments/assets/4eddb92c-cfbe-46a9-ba3f-92210df58e2c" />

### Fine-tuning
Compared fine-tuning results on Simpler trajectories in the repo based on 3 checkpoints for 500 steps. The finetuning scripts follow [[here]](https://github.com/chang-github-00/LAPA-Reproduce/tree/main/scripts).

1. [LWM Checkpoint](https://huggingface.co/LargeWorldModel/LWM-Chat-1M-Jax) (purple line)
2. [LAPA Official Checkpoint](latent-action-pretraining/LAPA-7B-openx) (blue line)
3. LWM Checkpoint Pre-trained on OpenX latent action data (green line)
<img width="400" alt="image" src="https://github.com/user-attachments/assets/6f90892b-3d31-4176-a2a2-44fe55b3453a" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/1aef18b7-a448-40c3-bfd5-845e46085c02" />

### Testing on Simpler
Testing scrips follow [here](https://github.com/chang-github-00/LAPA-Reproduce/blob/main/SimplerEnv/scripts/lapa_bridge.sh).
Note that the original script has a bug as in [issues#29](https://github.com/LatentActionPretraining/LAPA/issues/29), so we add a config `delta_vocab_size=8`.

Results are as follows: 
| Task                                                 | StackGreenCubeOnYellow | PutCarrotOnPlate| PutSpoonOnTableCloth | PutEggplantInBasket | Avg   |
|------------------------------------------------------|---------------------------------------------|----------------------------|--------------------------------|-----------------------------|-------|
| **LWM-1M-Chat Base:**                                |                                             |                            |                                |                             |       |
| + finetuning                                         | 0.625                                       | 0.4167                     | 0.7083                         | 0.6667                      | 0.6042 |
| + finetuning + official pretraining                  | 0.5833                                    | 0.4167                     | 0.7083                         | 0.75                        | 0.6146 |
| + finetuning + openx pretraining                     | 0.375                                       | 0.4589                     | 0.6667                         | 0.4583                      | 0.4896 |

## Getting Started 

```bash
conda create -n lapa python=3.10 -y
conda activate lapa
git clone https://github.com/LatentActionPretraining/LAPA.git
pip install -r requirements.txt 
mkdir lapa_checkpoints && cd lapa_checkpoints
```
Next, download the model checkpoint from [Huggingface](https://huggingface.co/latent-action-pretraining/LAPA-7B-openx) repository. Download, three files under `lapa_checkpoints` directory. 

```bash
wget https://huggingface.co/latent-action-pretraining/LAPA-7B-openx/resolve/main/tokenizer.model
wget https://huggingface.co/latent-action-pretraining/LAPA-7B-openx/resolve/main/vqgan
wget https://huggingface.co/latent-action-pretraining/LAPA-7B-openx/resolve/main/params
```

To run LAPA checkpoint which is pretrained on [Open-X Embodiment dataset](https://arxiv.org/abs/2310.08864), run the following command:
```bash
cd ..
python -m latent_pretraining.inference
```
This will generate the latent action conditioned on the input image and the natural language instruction.
You can change the input image and the instruction to a custom instance. **Note that the output space is the latent action space (which a space size of $8^4$), which is not the real action space**. To evaluate LAPA, fine-tuning is needed to map the latent space to the real action space (e.g. end-effector).

## Fine-tuning LAPA 
For fine-tuning LAPA on real world trajectories, you have to first preprocess the dataset to discretize the action space. We assume that there is a json file (`--input_path`) where the json file has the following row format:
```json
  {
    "id": "data/finetune_data/episode_0/step_0",
    "image": "data/finetune_data/episode_0/step_0.jpg",
    "conversations": [
      {
        "from": "human",
        "value": "<image>\nWhat action should the robot take to `pick up the milk and put it in the sink`"
      },
      {
        "from": "gpt",
        "raw_actions": [
          0.0004934221506118809,
          -0.00011252239346504211,
          -0.001941084861755371,
          0.013634951062806884,
          0.013678191591275368,
          -0.004913635449167675,
          0.0
        ],
        "states": {
          "eef_pos": [
            0.24725835025310516,
            -0.022094586864113808,
            0.9283081889152527
          ],
          "eef_euler": [
            3.1202197128587876,
            -0.7113159765223936,
            -0.10937155062330725
          ],
          "gripper_state": 0.0
        }
      }
    ]
  }
```
where `finetune_data` contains the images of fine-tuning trajectories.

Run the following commands to preprocess the fine-tuning dataset and fine-tune LAPA.
```bash
python data/finetune_preprocess.py --input_path "/path_to_json_file" --output_filename "data/real_finetune.jsonl" --csv_filename "data/real_finetune.csv"
./scripts/finetune_real.sh
```
We ran the experiments with 4 80GB-A100 GPUs. To change the number of GPUs being used, change the second index of `--mesh_dim` in the script to the number of GPUs.

For fine-tuning on SIMPLER rollout trajectories (100 trajecories), run the following command:
```bash
./scripts/finetune_simpler.sh
```

After finetuning, to deploy the model, run the following command:
```bash
python -m latent_pretraining.deploy --load_checkpoint "params::/path_to_the_finetuned_ckpt" --action_scale_file "data/real_finetune.csv"
```
where `load_checkpoint` includes the path to the finet-uned checkpoint and `action_scale_file` includes the path to the csv file constructed during data preprocessing of fine-tuning dataset.
 
## Latent Action Quantization 
We provide the code for latent action quantization pretraining.
```bash
conda create -n laq python=3.10 -y
conda activate laq
cd laq
pip install -e .
accelerate launch train_sthv2.py
```
Note that the current data loader code is based on something-something v2 dataset structure where the directory consists of multiple trajectories and each trajectory contain multiple images. To train on custom dataset, either change the data structure or modify the existing data loading code. 

After training, you can use the trained quantization model as an inverse dynamics model to obtain latent actions for training data. 

```bash
python inference_sthv2.py
```
Add arguments based on the training arguements. For the `input_file` argument, it should be a jsonl file which contains `id`, `image`, `instruction` keys as the metadata and `vision` which is the output of the vqgan model consisting of 256 discrete image tokens as the otuput.


## Latent-Pretraining 
We provide the code to do latent pretraining from pretrained LWM checkpoint. First, download the [LWM-Chat-1M-Jax](https://huggingface.co/LargeWorldModel/LWM-Chat-1M-Jax) model under `lwm_checkpoints` directory. Then, download the pretraining dataset from this [link](https://huggingface.co/latent-action-pretraining/LAPA-7B-openx/resolve/main/latent_action_pretraining_openx.jsonl) under the `data` directory. Run the following command for latent pretraining:
```bash
./scripts/latent_pretrain_openx.sh
```
We experimented with 8 H100 GPUs for 34 hours. We have empirically observed that 70K steps with a batch size of 256 is enough to get decent performance on downstream tasks after fine-tuning.

## SIMPLER
As a reproducible simulation, we release the setup that we tested with. First, install packages required for our latent-pretraining and [SIMPLER](https://github.com/simpler-env/SimplerEnv) following the installation guide. 

The inference script is provided in `scripts/lapa_bridge.sh`.
## Acknowledgement 
The codebase is based on [Large-World-Model](https://github.com/LargeWorldModel/LWM) repository. For latent action quantization, we referred to [Phenaki](https://github.com/lucidrains/phenaki-pytorch) code. For deployment code, we referred to the [OpenVLA](https://github.com/openvla/openvla) code. For the SIMPLER evaluation code, we referred to the [SIMPLER](https://github.com/simpler-env/SimplerEnv) repository.


## Citation

If you use this codebase, or otherwise found our work valuable, please cite:
```
@article{ye2024latent,
  title={Latent Action Pretraining from Videos},
  author={Ye, Seonghyeon and Jang, Joel and Jeon, Byeongguk and Joo, Sejune and Yang, Jianwei and Peng, Baolin and Mandlekar, Ajay and Tan, Reuben and Chao, Yu-Wei and Lin, Bill Yuchen and others},
  journal={arXiv preprint arXiv:2410.11758},
  year={2024}
}
```

If you have additional question, feel free to send an email to latentactionpretraining@gmail.com.

## License

LAPA's code and model weights are released under the MIT License. 
