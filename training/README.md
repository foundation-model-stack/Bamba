# Training

We trained our Bamba model with FSDP using our training repo [here](https://github.com/foundation-model-stack/fms-fsdp/tree/main).
Note that this training effort was started before FSDP2 and also long before we contributed
`Mamba2-Hybrid` to HF, so we were doing FSDP1 training with [official Mamba implementation](https://github.com/state-spaces/mamba).
For users trying to reproduce the training you now have much more options with our newly
contributed [HF-version of Mamba2-Hybrid](https://github.com/huggingface/transformers/tree/main/src/transformers/models/bamba).

Here are the setup details and command on how you can train the model:

``` python
git clone https://github.com/foundation-model-stack/fms-fsdp.git
cd fms-fsdp && pip install -e .

torchrun --nnodes=24 --node_rank=0 --nproc_per_node=8 \
    main_training_mamba.py \
      --model_variant=mamba_9.8b \
      --tokenizer_path="/path/to/tokenizer/" \
      --data_path="/path/to/datasets/" \
      --datasets="subdataset1,subdataset2,subdataset3,subdataset4,..." \
      --weights="1,1,1,1,.." \
      --seq_length=4096 \
      --vocab_size=128256 \
      --logical_shards=960 \
      --ckpt_load_path="/path/to/model/checkpoint" \
      --ckpt_save_path="/path/to/save/model/checkpoint" \
      --sharding_strategy="fsdp" \
      --batch_size=2 \
      --learning_rate=3e-4 \
      --num_steps=1280000 \
      --report_interval=100 \
      --checkpoint_interval=20000 \
```
To reproduce the exact model as Bamba-9B, or train using your own data or models, further config details are [here](data/README.md).

## Continuing Training

Training can be continued from a completed run's final saved checkpoint in multiple ways:
 1. Restore the model state only.
 2. Restore the model and optimizer state.
 3. Restore the entire model, optimizer, and dataloader state.

If the completed run was configured with `--ckpt_save_path="/path/to/prev/ckpt"` and the final checkpoint saved after step `XXX`, then a single
`consolidated.00.pth` file containing the final model weights only is created under `"/path/to/prev/ckpt/pth/step_XXX"`, while
sharded checkpoint files which also capture the optimizer and dataloader state exist under
`"/path/to/prev/ckpt/checkpoints"`. The three scenarios above are then achieved by specifying:
 1. **Model Only**: `--ckpt_load_path="/path/to/prev/ckpt/pth/step_XXX/consolidated.00.pth"`
 2. **Model + Optimizer**: `--ckpt_load_path="/path/to/prev/ckpt/"`
 3. **Model + Optimizer + Dataloader**: `--ckpt_load_path="/path/to/prev/ckpt/" --resuming_dataset`

