<div align="center">
    <h1>
        <img src="https://lightningaidev.wpengine.com/wp-content/uploads/2022/11/Asset-54-15.png">
        <br>
        Finetune large langugage models with Lightning
        </br>
    </h1>

<div align="center">

<p align="center">
  <a href="#run">Run</a> •
  <a href="https://www.lightning.ai/">Lightning AI</a> •
  <a href="https://lightning.ai/lightning-docs/">Docs</a>
</p>

[![ReadTheDocs](https://readthedocs.org/projects/pytorch-lightning/badge/?version=stable)](https://lightning.ai/lightning-docs/)
[![Slack](https://img.shields.io/badge/slack-chat-green.svg?logo=slack)](https://www.pytorchlightning.ai/community)
[![license](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://github.com/Lightning-AI/lightning/blob/master/LICENSE)

</div>
</div>

______________________________________________________________________

Use Lightning Classify to pre-train or fine-tune a large language model for text classification, 
with as many parameters as you want (up to billions!). 

You can do this:
* using multiple GPUs
* across multiple machines
* on your own data
* all without any infrastructure hassle! 

All handled easily with the [Lightning Apps framework](https://lightning.ai/lightning-docs/).

## Run

To run paste the following code snippet in a file `app.py`:


```python
# !pip install git+https://github.com/Lightning-AI/LAI-Text-Classification-Component
import os

import lightning as L
import torchtext.datasets
from transformers import BloomForSequenceClassification, BloomTokenizerFast

from lai_textclf import TextClassification, TextClassificationData, TextClf


class MyTextClassification(TextClf):
    def get_model(self, num_labels: int):
        # choose from: bloom-560m, bloom-1b1, bloom-1b7, bloom-3b
        model_type = "bigscience/bloom-3b"
        tokenizer = BloomTokenizerFast.from_pretrained(model_type)
        tokenizer.pad_token = tokenizer.eos_token
        tokenizer.padding_side = "left"
        model = BloomForSequenceClassification.from_pretrained(
            model_type, num_labels=num_labels, ignore_mismatched_sizes=True
        )
        return model, tokenizer

    def get_dataset(self):
        data_root_path = os.path.expanduser("~/.cache/torchtext/YelpReview")
        train_dset = torchtext.datasets.YelpReviewFull(
            root=data_root_path, split="train"
        )
        val_dset = torchtext.datasets.YelpReviewFull(root=data_root_path, split="test")
        num_labels = 5
        return train_dset, val_dset, num_labels

    def get_trainer_settings(self):
        return dict(strategy="deepspeed_stage_3_offload", precision=16)

    def finetune(self):
        train_dset, val_dset, num_labels = self.get_dataset()
        module, tokenizer = self.get_model(num_labels)
        pl_module = TextClassification(model=module, tokenizer=tokenizer)
        datamodule = TextClassificationData(
            train_dataset=train_dset, val_dataset=val_dset, tokenizer=tokenizer
        )
        trainer = L.Trainer(**self.get_trainer_settings())
        trainer.fit(pl_module, datamodule)


app = L.LightningApp(
    L.app.components.LightningTrainerMultiNode(
        MyTextClassification,
        num_nodes=2,
        cloud_compute=L.CloudCompute("gpu-fast-multi", disk_size=50),
    )
)
```

### Running on cloud

```bash
lightning run app app.py --cloud
```

Don't want to use the public cloud? Contact us at `product@lightning.ai` for early access to run on your private cluster (BYOC)!


### Running locally (limited)
This example is optimized for the cloud. To run it locally, choose a smaller model, change the trainer settings like so:

```python
class MyTextClassification(TextClf):
    ...
    
    def get_trainer_settings(self):
        return dict(accelerator="cpu", devices=1)
```
This will avoid using the deepspeed strategy for training which is only compatible with multiple GPUs for model sharding.
Then run the app with 

```bash
lightning run app app.py --setup
```

