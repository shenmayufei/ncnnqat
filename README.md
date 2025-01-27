<div id="ncnnqat"></div>

# ncnnqat

ncnnqat is a quantize aware training package for NCNN on pytorch.

<div id="table-of-contents"></div>

## Table of Contents

- [ncnnqat](#ncnnqat)
  - [Table of Contents](#table-of-contents)
  - [Installation](#installation)
  - [Usage](#usage)
  - [Code Examples](#code-examples)
  - [Results](#results)
  - [Todo](#todo)


<div id="installation"></div>  

## Installation

* Supported Platforms: Linux
* Accelerators and GPUs: NVIDIA GPUs via CUDA driver ***10.1***.
* Dependencies:
  * python >= 3.5, < 4
  * pytorch >= 1.6
  * numpy >= 1.18.1
  * onnx >= 1.7.0
  * onnx-simplifier >= 0.3.6

* Install ncnnqat via pypi:  
  ```shell
  $ pip install ncnnqat (to do....)
  ```
  It is recommended to install from the source code
* or Install ncnnqat via repo：
  ```shell
  $ git clone https://github.com/ChenShisen/ncnnqat
  $ cd ncnnqat
  $ make install
  ```

<div id="usage"></div>

## Usage


* register_quantization_hook and merge_freeze_bn

  (suggest finetuning from a well-trained model, do it after a few epochs of training otherwise.)

  ```python
  from ncnnqat import unquant_weight, merge_freeze_bn, register_quantization_hook
  ...
  ...
      for epoch in range(epoch_train):
          model.train()
	  if epoch==well_epoch:
	      register_quantization_hook(model)
	  if epoch>=well_epoch:
	      model = merge_freeze_bn(model)  #it will change bn to eval() mode during training
  ...
  ```

* Unquantize weight before update it

  ```python
  ...
  ... 
      if epoch>=well_epoch:
          model.apply(unquant_weight)  # using original weight while updating
      optimizer.step()
  ...
  ```

* Save weight and save ncnn quantize table after train


  ```python
  ...
  ...
      onnx_path = "./xxx/model.onnx"
      table_path="./xxx/model.table"
      dummy_input = torch.randn(1, 3, img_size, img_size, device='cuda')
      input_names = [ "input" ]
      output_names = [ "fc" ]
      torch.onnx.export(model, dummy_input, onnx_path, verbose=False, input_names=input_names, output_names=output_names)
      save_table(model,onnx_path=onnx_path,table=table_path)

  ...
  ```
  if use "model = nn.DataParallel(model)",pytorch unsupport torch.onnx.export,you should save state_dict first and  prepare a new model with one gpu,then you will export onnx model.
  
  ```python
  ...
  ...
      model_s = new_net() #
      model_s.cuda()
      register_quantization_hook(model_s)
      #model_s = merge_freeze_bn(model_s)
      onnx_path = "./xxx/model.onnx"
      table_path="./xxx/model.table"
      dummy_input = torch.randn(1, 3, img_size, img_size, device='cuda')
      input_names = [ "input" ]
      output_names = [ "fc" ]
      model_s.load_state_dict({k.replace('module.',''):v for k,v in model.state_dict().items()}) #model_s = model     model = nn.DataParallel(model)
            
      torch.onnx.export(model_s, dummy_input, onnx_path, verbose=False, input_names=input_names, output_names=output_names)
      save_table(model_s,onnx_path=onnx_path,table=table_path)
	  

  ...
  ```


<div id="code-examples"></div>

## Code Examples

  Cifar10 quantization aware training example.

  ```python test/test_cifar10.py```
  
  SSD300 quantization aware training example.
     
  ```
     ln -s /your_coco_path/coco ./tests/ssd300/data
  ```
  ```
     python -m torch.distributed.launch \
      --nproc_per_node=4 \
      --nnodes=1 \
      --node_rank=0 \
      ./tests/ssd300/main.py \
      -d ./tests/ssd300/data/coco
  ```
  ```
      python ./tests/ssd300/main.py --onnx_save  #load model dict, export onnx and ncnn table
  ```

<div id="results"></div>

## Results  

* Cifar10


  result：

    |  net   | fp32(onnx) | ncnnqat     | ncnn aciq     | ncnn kl |
    | -------- |  -------- | -------- | -------- | -------- |
    | mobilenet_v2     | 0.91  | 0.9066  | 0.9033 | 0.9066 |
    | resnet18 | 0.94   | 0.93333   | 0.9367 | 0.937|


* SSD300(resnet18|coco)


    ```
    fp32:
	 Average Precision  (AP) @[ IoU=0.50:0.95 | area=   all | maxDets=100 ] = 0.193
	 Average Precision  (AP) @[ IoU=0.50      | area=   all | maxDets=100 ] = 0.344
	 Average Precision  (AP) @[ IoU=0.75      | area=   all | maxDets=100 ] = 0.191
	 Average Precision  (AP) @[ IoU=0.50:0.95 | area= small | maxDets=100 ] = 0.042
	 Average Precision  (AP) @[ IoU=0.50:0.95 | area=medium | maxDets=100 ] = 0.195
	 Average Precision  (AP) @[ IoU=0.50:0.95 | area= large | maxDets=100 ] = 0.328
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets=  1 ] = 0.199
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets= 10 ] = 0.293
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets=100 ] = 0.309
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area= small | maxDets=100 ] = 0.084
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area=medium | maxDets=100 ] = 0.326
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area= large | maxDets=100 ] = 0.501
	Current AP: 0.19269

    ncnnqat:
	 Average Precision  (AP) @[ IoU=0.50:0.95 | area=   all | maxDets=100 ] = 0.192
	 Average Precision  (AP) @[ IoU=0.50      | area=   all | maxDets=100 ] = 0.342
	 Average Precision  (AP) @[ IoU=0.75      | area=   all | maxDets=100 ] = 0.194
	 Average Precision  (AP) @[ IoU=0.50:0.95 | area= small | maxDets=100 ] = 0.041
	 Average Precision  (AP) @[ IoU=0.50:0.95 | area=medium | maxDets=100 ] = 0.194
	 Average Precision  (AP) @[ IoU=0.50:0.95 | area= large | maxDets=100 ] = 0.327
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets=  1 ] = 0.197
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets= 10 ] = 0.291
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area=   all | maxDets=100 ] = 0.307
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area= small | maxDets=100 ] = 0.082
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area=medium | maxDets=100 ] = 0.325
	 Average Recall     (AR) @[ IoU=0.50:0.95 | area= large | maxDets=100 ] = 0.497
	Current AP: 0.19202
    ```


<div id="todo"></div>

## Todo

   ....


### install and usage

- pre

install cuda and cudnn 

```bash
# 182, for example
export PATH=/usr/local/cuda-10.2/bin:/data/yzh/projects/cuda/include:$PATH
export LD_LIBRARY_PATH=/data/yzh/projects/cuda/lib64:/usr/local/cuda-10.2/lib64:$LD_LIBRARY_PATH
#183
export PATH=/usr/local/cuda-10.2/bin:/data/data1/yzh/projects/software/cuda/include:$PATH
export LD_LIBRARY_PATH=/data/data1/yzh/projects/software/cuda/lib64:/usr/local/cuda-10.2/lib64:$LD_LIBRARY_PATH
```
- build ncnnqat
```bash

cd ncnnqat
make install
```

- install yolox
```bash
cd tests
git clone https://github.com/shenmayufei/YOLOX/tree/qat
cd yolox/
pip install -e .
```

- train

```bash 
first , you nedd to train withou quantization,only set
self.qat = False

second, when your training is ok, then set 
self.qat = True

```


