---
title: "1-bit Adam: Up to 5x less communication volume and up to 2x faster training"
---

In this tutorial, we are going to introduce the 1-bit Adam optimizer in DeepSpeed. 1-bit Adam can improve model training speed on communication-constrained clusters, especially for communication-intensive large models by reducing the overall communication volume by up to 5x. Detailed description of the 1-bit Adam algorithm, its implementation in DeepSpeed, and performance evaluation is available from our [blog post](https://www.deepspeed.ai/news/2020/09/09/onebit-adam-blog-post.html).

To illustrate the benefits and usage of 1-bit Adam optimizer in DeepSpeed, we use the following two training tasks as examples:

1. BingBertSQuAD Fine-tuning
2. BERT Pre-training

For more details on these tasks, please refer to the tutorial posts on [BingBertSQuAD Fine-tuning](/tutorials/bert-finetuning/) and [BERT Pre-training](/tutorials/bert-pretraining/).

## Overview

If you don't already have a copy of the DeepSpeed repository, please clone in
now and checkout the DeepSpeedExamples submodule that contains the BingBertSQuAD and BERT Pre-training examples.

```shell
git clone https://github.com/microsoft/DeepSpeed
cd DeepSpeed
git submodule update --init --recursive
cd DeepSpeedExamples/
```
## Pre-requisites for 1-bit Adam

1-bit Adam uses advanced communication schemes that are not yet supported by PyTorch distributed and NCCL. We rely on Message Passing Interface (MPI) for these advanced communication primitives.

We package the necessary dependencies in the DeepSpeed docker images. However, if you are using a different build system, please install MPI and mpi4py on your system. We have tested CUDA-Aware MPI communication using the [MVAPICH2-GDR](http://mvapich.cse.ohio-state.edu/userguide/gdr/) library. However, any CUDA-Aware communication library including [OpenMPI](https://www.open-mpi.org/) should work fine with these examples.

An example launch command for 1-bit Adam using the `deepspeed` launcher is as follows:

```shell
deepspeed --launcher=[mvapich|openmpi] script.py
```

Alternatively, the standard mpirun launcher can also be used as follows:

```shell
mpirun -np [#processes] -ppn [#GPUs on each node] -hostfile [hostfile] [MPI flags] bash [training_script.sh]
```

### Configuration
The 1-bit Adam feature can be used by setting the optimizer configuration options as follows. An example json config file is shown below.

```json
{
  "train_batch_size": 4096,
  "train_micro_batch_size_per_gpu": 64,
  "optimizer": {
    "type": "OneBitAdam",
    "params": {
      "lr": 2e-4,
      "freeze_step": 400,
      "cuda_aware": true
    }
  },
  "fp16": {
    "enabled": true,
  }
}
```
Please note two new parameters `freeze_step` and `cuda_aware` that have been added to support the 1-bit Adam feature.

`cuda_aware` is used to indicate that the underlying MPI library support CUDA-Aware communication.
This feature is only supported on systems with InfiniBand interconnect and a CUDA-Aware MPI library like [MVAPICH2-GDR](http://mvapich.cse.ohio-state.edu/userguide/gdr/) or OpenMPI built with CUDA-Aware support. Setting `cuda_aware` to False will allow training on Ethernet based systems. However, the communication will happen using sender as well as receiver side memory copies between CPU and GPU buffers before and after communication.

`freeze_step` is the number of warm up steps before 1-bit compression gets applied to the communication. In order to determine the number of warm up steps, one strategy is to set 15-25% of the total training steps for a given model. If it provides the desired outcome, one can try to extract more performance by reducing the steps systematically. In future, we plan to introduce a threshold that can automatically search and decide for the number of warm up steps for different models. The examples below have been tuned for the number of warm up steps. The `freeze_step` parameter has already been set to the best number we found in the corresponding run scripts.

### Benefits of 1-bit Adam on communication-constrained systems

1-bit Adam offers the same convergence as Adam, incurs up to 5x less communication that enables up to 3.5x higher throughput for BERT-Large pretraining and up to 2.7x higher throughput for SQuAD fine-tuning. This end-to-end throughput improvement is enabled by the 6.6x (Figure 1) and 6.2x (Figure 2) speedup observed during the compression stage.

![BERT-Large Pretraining](/assets/images/bert-scaling.png){: .align-center}

Figure 1: Scalability of 1-bit Adam for BERT-Large Pretraining on V100 GPUs with batch size of 16/GPU.

![SQuAD Finetuning](/assets/images/squad-scaling.png){: .align-center}

Figure 2: Scalability of 1-bit Adam for SQuAD Finetuning on V100 GPUs with batch size of 3/GPU.

For more results, please see our detailed [blog post](https://www.deepspeed.ai/news/2020/09/09/onebit-adam-blog-post.html).

## 1. BingBertSQuAD fine-tuning with 1-bit Adam

* Download the SQuAD dataset:
  * Training set: [train-v1.1.json](https://rajpurkar.github.io/SQuAD-explorer/dataset/train-v1.1.json)
  * Validation set: [dev-v1.1.json](https://rajpurkar.github.io/SQuAD-explorer/dataset/dev-v1.1.json)
* Download the HuggingFace checkpoint and config files:
  * [bert-large-uncased-whole-word-masking](https://s3.amazonaws.com/models.huggingface.co/bert/bert-large-uncased-whole-word-masking-pytorch_model.bin)
  * [bert json config](https://s3.amazonaws.com/models.huggingface.co/bert/bert-large-uncased-whole-word-masking-config.json)

You can also use a pre-trained BERT model checkpoint from either DeepSpeed, [HuggingFace](https://github.com/huggingface/transformers), or [TensorFlow](https://github.com/google-research/bert#pre-trained-models) to run the fine-tuning.

### 1.1 Running BingBertSQuAD with DeepSpeed and 1-bit Adam

The main part of training is done in `nvidia_run_squad_deepspeed.py`, which has
already been modified to use DeepSpeed. The `run_squad_deepspeed.sh` script
helps to invoke training and setup several different hyperparameters relevant
to the training process.

- **DeepSpeed-enabled:** Start training with DeepSpeed by providing the following 4 arguments to this script:

```shell
bash run_squad_deepspeed.sh <NUM_GPUS> <PATH_TO_CHECKPOINT> <PATH_TO_DATA_DIR> <PATH_TO_OUTPUT_DIR>`
```

The first argument is the number of GPUs to train with, second argument is the path to the pre-training checkpoint, third is the path to training and validation sets (e.g., train-v1.1.json), and fourth is path to an output folder where the results will be saved. This script will invoke `nvidia_run_squad_deepspeed.py`.

- **DeepSpeed with 1-bit Adam enabled:** In order to run with 1-bit Adam feature enabled, the same script (`nvidia_run_squad_deepspeed.py`) can be used but there are two options for launching this properly: 1) Launch using deepspeed launcher and 2) Launch with mpirun.

To enable the 1-bit compressed training, 1-bit Adam uses an MPI library (E.g. MVAPICH2-GDR, OpenMPI, etc.) as the communication backend, which means that we can use `mpirun` to launchg the training job. However, our user-friendly launcher called `deepspeed` has been enhanced to launch MPI jobs as well.

### Launch with deepspeed

The following helper script in the DeepSpeedExamples/BingBertSQuAD will launch the training without the need for setting any `mpirun` parameters.

```shell
bash run_squad_deepspeed_onebitadam.sh
```

### Launch with mpirun

Alternatively, we show how the standard `mpirun` launcher can be used for launching the fine-tuning job.

```shell
mpirun -np [#processes] -ppn [#GPUs on each node] -hostfile [hostfile] [MPI flags] bash run_squad_deepspeed_onebitadam.sh
```
For example, in order to use 32 GPUs (4GPUs/node, 8 nodes in total), with the support of InfiniBand, you can use the `mpirun` launcher packaged with the MVAPICH2 library. Please run the folowing command:

```shell
mpirun -np 32 -ppn 4 -hostfile hosts -env MV2_USE_CUDA=1 -env MV2_SUPPORT_DL=1 -env MV2_ENABLE_AFFINITY=0 -env MV2_SMP_USE_CMA=0 bash run_squad_deepspeed_onebitadam.sh
```

### 1.2 Configuration for BingBertSQuAD with DeepSpeed and 1-bit Adam enabled

The `deepspeed_bsz96_onebit_config.json` file gives the user the ability to specify DeepSpeed
options in terms of batch size, micro batch size, optimizer, learning rate, and other parameters.
When running the `nvidia_run_squad_deepspeed.py`, in addition to the
`--deepspeed` flag to enable DeepSpeed, the appropriate DeepSpeed configuration
file must be specified using `--deepspeed_config deepspeed_bsz96_config.json`.

Table 1 shows the fine-tuning configuration we used in our experiments.

| Parameters                     | Value 		|
| ------------------------------ | ---------------------|
| Total batch size               | 96    		|
| Train micro batch size per GPU | 3     		|
| Optimizer                      | **OnebitAdam**  	|
| Learning rate                  | 3e-5  		|
| Sequence-length                | 384   		|
| Weight-decay                   | 0.0   		|
| Epoch count                    | 2     		|
| **freeze_step**                | 400     	   	|
| **cuda_aware**                 | True     		|

Table 1. Fine-tuning configuration

### 1.3 Results for BingBertSQuAD Fine-tuning

The results are summarized in the table below. The total batch size is set to 96 and training is conducted
on 32 GPUs for 2 epochs. A set of parameters (seeds and learning rates) were tried and the best ones were selected.
We fixed the learning rate to 3e-5. The table below shows the F1 and the EM scores we achieved that are on-par or better than the [HuggingFace results](https://github.com/huggingface/transformers/tree/master/examples/question-answering).

| Case        | Model                                 | Precision | EM    | F1    |
| ----------- | ------------------------------------- | --------- | ----- | ----- |
| HuggingFace | [Bert-large-uncased-whole-word-masking](https://s3.amazonaws.com/models.huggingface.co/bert/bert-large-uncased-whole-word-masking-pytorch_model.bin) | FP16      | 87.26 | 93.32 |

**Note:** For more details about loading checkpoint, argument parsing, initialization, forward pass, backward pass, weight update and evaluation, please refer to the [BingBertSQuAD Fine-tuning](https://www.deepspeed.ai/tutorials/bert-finetuning/) tutorial.


## 2. BERT Pre-training with 1-bit Adam
For data downloading and pre-processing, please refer to [BERT Pre-training](https://www.deepspeed.ai/tutorials/bert-pretraining/) posts
for more details.

### 2.1 Running Pre-training with DeepSpeed and 1-bit Adam

The main part of training is done in `deepspeed_train.py`, which has
already been modified to use DeepSpeed. The `ds_train_bert_onebitadam_bsz4k_seq128.sh` and `ds_train_bert_bsz64k_seq128.sh` are the
 shell scripts that
help to invoke training and setup several different hyperparameters relevant
to the training process.

- **DeepSpeed-enabled:** Start training with DeepSpeed by running the command below:

```shell
bash ds_train_bert_bsz64k_seq128.sh
```

- **DeepSpeed with 1-bit Adam enabled:** In order to run with 1-bit Adam feature enabled, the same script (`deepspeed_train.py`) can be used but there are two options for launching this properly:

### Launch with deepspeed

As discussed for BingBertSQuAD fine-tuning, we can simply use the `deepspeed` launcher to launch our BERT pre-training jobs as follows.

```shell
bash ds_train_bert_onebitadam_bsz4k_seq128.sh
```

### Launch with mpirun

Alternatively, use the following command to launch using `mpirun`.

```shell
mpirun -np [#processes] -ppn [#GPUs on each node] -hostfile [hostfile] [MPI flags] bash ds_train_bert_onebitadam_bsz4k_seq128.sh
```

For example, in order to use 32 GPUs (4GPUs/node, 8 nodes in total), with the support of InfiniBand, you can use MVAPICH2 as the launcher and run the following command:
```shell
mpirun -np 32 -ppn 4 -hostfile hosts -env MV2_USE_CUDA=1 -env MV2_SUPPORT_DL=1 -env MV2_ENABLE_AFFINITY=0 -env MV2_SMP_USE_CMA=0 bash ds_train_bert_onebitadam_bsz4k_seq128.sh
```

### 2.2 Configuration for BingBertSQuAD with DeepSpeed and 1-bit Adam enabled

The `deepspeed_bsz4k_onebit_config_seq128.json` file gives the user the ability to specify DeepSpeed
options in terms of batch size, micro batch size, optimizer, learning rate, and other parameters.

Below is the DeepSpeed configuration file for running BERT-large pre-training with sequence length of 128.
```json
{
  "train_batch_size": 4096,
  "train_micro_batch_size_per_gpu": 64,
  "steps_per_print": 1000,
  "optimizer": {
    "type": "Adam",
    "params": {
      "lr": 2e-4,
      "max_grad_norm": 1.0,
      "weight_decay": 0.01,
      "bias_correction": false,
      "freeze_step": 23000,
      "cuda_aware": true
    }
  },
  "fp16": {
    "enabled": true,
    "loss_scale": 0,
    "initial_scale_power": 16
  }
}
```
Notice that for BERT-base training (sequence length 128), the suggested freeze_step is 16000. For the rest of the pre-training using sequence 512, we suggest to use a freeze_step of 1500.