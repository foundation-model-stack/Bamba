# Training Data

Our experiments showed that loss curves are tightly tied to data ordering, making data parity important for fair comparison of base models.
We therefore release the Bamba training data, so that users may perform continued pretraining, or train their own models on the same data sequence.

## Accessing the Data

Our training data is not yet available, but will soon join our checkpoints in the public `bamba-public` AWS bucket. Check back later!

<!---
You can access the bucket using the following commands:
```bash
aws --endpoint-url=https://s3.us-east.cloud-object-storage.appdomain.cloud/ s3 cp s3://TODO/REMOTE-FOLDER ./LOCAL-FOLDER/ --recursive
```

Note that this download is TODO Tb. For testing purposes you can instead download a single file:
```bash
aws --endpoint-url=https://s3.us-east.cloud-object-storage.appdomain.cloud/ s3 cp s3://TODO/REMOTE-FOLDER/REMOTE-FILE ./LOCAL-FOLDER/testing.arrow
```

Shard files are pyarrow collections of tokenized documents using the Llama3 tokenizer. You can view examples from the data using the following in Python:
```python
import pyarrow as pa

doc = 0
with pa.ipc.open_file('testing.arrow') as reader:
    lines = reader.num_record_batches
    if doc >= lines:
        print(f"Doc index exceeds number of documents {lines}")
    else:
        print(reader.get_batch(i)['tokens'].to_pylist())

# [TODO (result)]
```
-->
## Loading the Data

Bamba is trained using a custom stateful distributed dataloader. 
It offers many of the same capabilities as the MosaicML [StreamingDataset](https://docs.mosaicml.com/projects/streaming/en/stable/#), but in the form of a composable pipeline _a la_ [torchvision.transforms](https://pytorch.org/vision/0.9/transforms.html), and with support for different file types.
In particular, the dataloader is:

- Stateful and checkpointable: we guarantee that when reloading from checkpoint, previously seen data is never revisited until the current epoch finishes. When reloading to the same number of workers, the data order is unchanged.
- Distributed: built-in dataset sharding at the document level, with no communication required between workers
- Rescalable: users can save and load checkpoints to different numbers of workers
- Streaming: each worker maintains a single open file at a time, exhausting it before opening the next, minimizing overhead
- Lightweight: a custom random walk generator shuffles hundreds of millions of documents with zero additional overhead
- Asynchronous: data loading and checkpointing do not block model training
- Flexible: users can add support for arbitrary data file types by extending a FileHandler class stub with basic `open`, `length`, `get`, and `slice` operations. The dataloader also supports both tokenized and untokenized input documents.
- Modular: data pipelines are composed of individual processing stages that can be added or removed according to the user's individual needs
- Extensible: users can easily extend the dataset iterator to implement their own stateful or state-free data processing steps, or extend the FileHandler to support arbitrary data file types
- Fast: we have observed throughputs of over 130k tokens/device/second
- Stable: we have run the dataloader over the span of weeks with no slowdown or failure

The code for the data loader in Bamba is available [here](https://github.com/foundation-model-stack/fms-fsdp/blob/mamba-new/fms_fsdp/utils/dataset_utils.py), with a constructor function available [here](https://github.com/foundation-model-stack/fms-fsdp/blob/mamba-new/fms_fsdp/utils/dataloader_utils.py#L60)
The constructor returns a generic PyTorch DataLoader, making it easy to incorporate into other training runs.
A graphical example of Bamba's dataloader structure is below:
<p align="center">
<img src="databranch.jpg", width="800"/>
</p>

We are currently working on contributing our dataloader back into PyTorch. 
More information on our custom dataloader will be published soon.

## Reproducing Bamba Training

To reproduce the exact sequence of training data seen by Bamba, you must use the provided data as-is, without modifying directory structures or file names.
The number of tokens per batch must also be 1.6 million (1,572,864 to be precise - 384 sequences of length 4096).
Bamba was trained on 192 GPUs in parallel.
For training on fewer than 192 GPUs, you can increase `num_workers` and `batch_size` to compensate, and the data sequence will remain unchanged.
For example, using 64 GPUs with `num_workers=3, batch_size=6` will train identically to a model on 96 GPUs with `num_workers=2, batch_size=4`. 
If batches at the adjusted `batch_size` become too large to fit in GPU, gradient accumulation with smaller `batch_size` will also keep the data sequence preserved, so long as total tokens per step remains at 1.6 million.

The dataloader constructor takes a config argument and uses the config object to construct the data pipeline stages. Bamba uses the following values:
```python
from dataclasses import dataclass

@dataclass
class config:
    ckpt_load_path = "[YOUR_CHECKPOINT_PATH]"
    ckpt_save_path = "[YOUR_CHECKPOINT_PATH]"
    data_path = "[YOUR_DATA_PATH]"
    file_type = "arrow"
    col_name = "tokens"
    datasets = "dataset=cc_en_head,dataset=cc_en_middle,dataset=cc_en_tail,dataset=falcon,dataset=starcoder,dataset=c4,dataset=reddit,dataset=pes2o,dataset=arxiv,dataset=stackexchange,dataset=tulu_flan,dataset=cc_news_head,dataset=cc_news_middle,dataset=cc_news_tail,dataset=open,dataset=algebraic,dataset=books,dataset=megawika,dataset=wiki"
    weights = "1456,1835,1558,1851,450,1500,300,236,114,200,100,62,27,11,51,49,100,55,45"
    seq_length = 4096
    bos_token = None
    eos_token = 0
    bol_token = None
    eol_token = None
    strip_tokens: str = ""
    logical_shards = 960
    num_workers = 1
    batch_size = 2
    seed = 2023
```

This config was used to produce the checkpoints trained up to 2T tokens. The final checkpoint is annealed on a different data mixture, with prior dataset checkpoint information dropped (via the `resuming_dataset` flag). Annealing took place on 160 gpus, with 1,310,720 tokens per batch. For running on fewer GPUs, follow the same logic as above. Use the following config for annealing:
```python
from dataclasses import dataclass

@dataclass
class config:
    ckpt_load_path = "[PRIOR_CHECKPOINT_PATH]"
    ckpt_save_path = "[YOUR_NEW_CHECKPOINT_PATH]"
    data_path = "[YOUR_DATA_PATH]"
    file_type = "arrow"
    col_name = "tokens"
    datasets = "smollm-corpus/tokens/llama3/cosmopedia-v2,smollm-corpus/tokens/llama3/fineweb-edu-dedup"
    weights = "0.6,0.4"
    seq_length = 4096
    bos_token = None
    eos_token = 128000
    bol_token = None
    eol_token = None
    strip_tokens: str = ""
    logical_shards = 960
    num_workers = 1
    batch_size = 2
    seed = 2023
    resuming_dataset = False
```
All values from these configs can be passed in as flags to the `torchrun` training command e.g. `--eos_token=128000 --num_workers=2`.

## Using Our DataLoader Checkpoints
You can also continue training from our model checkpoints in the `bamba-public` AWS bucket, which include the dataloader state. Simply point `config.ckpt_load_path` to the existing checkpoint folder, and `config.resuming_dataset` to True. You must keep the `file_type, col_name, datasets, logical_shards, seed` fields of `config` the same as provided above. Other values can be changed to suit your training job.

One perk of our distributed dataloader checkpoints is that they're human-readable! You can inspect the checkpoints manually to see the relevant training statistics. You can run the following to retrieve a single checkpoint shard from our first stage 1 model:
```
aws --endpoint-url=https://s3.us-east.cloud-object-storage.appdomain.cloud/ s3 cp s3://bamba-public/checkpoints/pretraining/phase_one/1_8t/step_1140000_ckp/loader_state_0.pth ./temp_ckpt.pth
```
Then in python, load the state dictionary:
```python
import torch

state_d = torch.load('temp_ckpt.pth')
print(list(state_d.keys()))

# ['Sampling_Dataset.sample_iterator_states', 'Sampling_Dataset.tokens_seen', 'Sampling_Dataset.current_iterator', 'Buffer_Dataset.buffer', 'Preload_Buffer_Dataset.g_state', 'Preload_Buffer_Dataset.buffer']
```
Each of these fields contains statistics from the relevant layer of the pipeline diagram shown above. We can check that behavior is tracking as expected - for example, `Sampling_Dataset.tokens_seen` tracks the number of tokens consumed from each subdataset. Let's check those proportions against the weights we asked for in our config:
```python
subdata_tokens = state_d['Sampling_Dataset.tokens_seen']
subdata_tokens = [x/sum(subdata_tokens) for x in subdata_tokens]
subdata_tokens = [round(x*10000,2) for x in subdata_tokens]
print(subdata_tokens)

# [1456.0, 1834.99, 1557.99, 1850.99, 450.02, 1499.99, 299.99, 235.99, 114.01, 199.99, 99.99, 61.99, 26.99, 10.99, 50.99, 49.0, 100.06, 55.0, 45.0]
```
These weights do indeed match the weights in our config - to within two thousandths of a percent!

Note that this state dict is not flat - some of these fields contain further state dicts. In particular, `Sampling_Dataset.sample_iterator_states` contains the state dicts for each of the subdataset iterators on this shard. Following the diagram above, each of these iterators in turn contains subiterators for each logical partition of the dataset:
```python
subdata_states = state_d['Sampling_Dataset.sample_iterator_states']

print(len(subdata_states))

# 19

print(list(subdata_states[0].keys()))

# ['Scalable_Shard_Dataset.current_reader', 'Scalable_Shard_Dataset.g_state', 'Scalable_Shard_Dataset.n_docs_remaining', 'Scalable_Shard_Dataset.logical_shard_states']

print(len(subdata_states[0]['Scalable_Shard_Dataset.logical_shard_states']))

# 5
```
Checking against our config, there are indeed 19 subdatasets that we're sampling over. And with 5 data partitions per loader checkpoint file, and 192 loader checkpoint files, that comes out to 960 total data partitions - again matching the config (`logical_shards`).

As a final example, we can dig deep into the base levels of the pipeline and gather the number of epochs completed on different datasets:
```python
print(list(subdata_states[0]['Scalable_Shard_Dataset.logical_shard_states'][0].keys()))

# ['Streaming_Doc_Dataset.dataset', 'Streaming_Doc_Dataset.docset_index', 'Streaming_Doc_Dataset.chunk_index', 'Streaming_Doc_Dataset.epochs_seen', 'Streaming_Doc_Dataset.tokens_seen', 'Streaming_Doc_Dataset.docs_seen', 'Streaming_Doc_Dataset.percent_seen', 'Streaming_Doc_Dataset.lcg_state']

# Falcon RefinedWeb dataset is #3
print(subdata_states[3]['Scalable_Shard_Dataset.logical_shard_states'][0]['Streaming_Doc_Dataset.percent_seen'])

# 74.7763942476323

# Wiki dataset is #18
print(subdata_states[18]['Scalable_Shard_Dataset.logical_shard_states'][0]['Streaming_Doc_Dataset.percent_seen'])

# 129.71079308286676
```
We've completed 1 epoch on Wiki, plus an extra 30%, but only 3/4 of an epoch on the Falcon RefinedWeb subdataset - at least on this logical partition, in this particular checkpoint shard. But they tend to be pretty consistent:
```python
for i in range(5):
    print(subdata_states[3]['Scalable_Shard_Dataset.logical_shard_states'][i]['Streaming_Doc_Dataset.percent_seen'])

# 74.7763942476323
# 74.79928004007313
# 74.80668911169474
# 74.81829933007866
# 74.93420931428876
```
In sum, we could continue pretraining on our checkpoints for a lot longer!

## Training on Your Own Data

Our data files contain pre-tokenized documents in a non-standard pyarrow file format, allowing for particularly efficient loading and streaming. However, support for different file types and on-the-fly tokenization is also available for use on your own data. Three options are available:

### 1) Write your data into pyarrow format
Pre-tokenize your documents and write them into pyarrow format using the following code:
```python
import os
import pyarrow as pa

data_dir = "my/data/path"
shard_file_name = "my_file.arrow"
tokenized_docs = [[1,2,3],[4,5],[6,7,8,9]]
# "tokens" is an arbitrary header. You can use any header, and simply update config.col_name above to match
schema = pa.schema([pa.field("tokens", pa.uint32())])

with pa.ipc.new_file(os.path.join(data_dir, shard_file_name), schema) as writer:
    for doc in tokenized_docs:
        writer.write(pa.record_batch([doc], schema=schema))
```
The resulting data shard files can be fed directly into the dataloader as outlined above.

### 2) Use HuggingFace's parquet format and tokenize on the fly
Our dataloader also supports the more standard untokenized parquet data files used in many HuggingFace datasets (i.e. [Cosmopedia](https://huggingface.co/datasets/HuggingFaceTB/cosmopedia)). Simply update the following fields in the config:
```python
config.file_type = "hf_parquet"
config.col_name = "text"
config.tokenizer_path = "path/to/hf/tokenizer"
```
Note that `tokenizer_path` must point to a HuggingFace tokenizer, and `col_name` may vary from dataset to dataset. `"text"` is common but not universal. `"auto"` will also work for `file_type`.

### 3) Add support for your own file type
The FileHandler [class stub](https://github.com/foundation-model-stack/fms-fsdp/blob/0e47e934e69a340a7a2183b15b5f24979eb9db9c/fms_fsdp/utils/dataset_utils.py#L286) allows support for arbitrary file types. Simply implement 5 basic functions:
1. `is_legal`: given a file name, is this a parseable shard file?
2. `open`: given a path to a file, return an indexable object
3. `length`: given a path to a file, return the number of documents in the file
4. `get`: given an indexable object and an index, return the document at that index. Remove BOS/EOS tokens as needed
5. `slice`: given a returned document, a starting index, and a number of tokens, return those tokens as a list

Simply implement a FileHandler class for your desired shard file format with the above operations. An example incorporating a tokenizer can be found [here](https://github.com/foundation-model-stack/fms-fsdp/blob/0e47e934e69a340a7a2183b15b5f24979eb9db9c/fms_fsdp/utils/dataset_utils.py#L371). Then override the existing FileHandler [here](https://github.com/foundation-model-stack/fms-fsdp/blob/0e47e934e69a340a7a2183b15b5f24979eb9db9c/fms_fsdp/utils/dataloader_utils.py#L98) with your new class, and your data files are now supported.

Happy training!
