# 猫狗大战

### 题目来源

2014年的Kaggle竞赛[Dogs vs. Cats](https://www.kaggle.com/c/dogs-vs-cats/overview)

### 数据集来源

[Dogs vs. Cats | Kaggle](https://www.kaggle.com/c/dogs-vs-cats/data)

**训练集**：训练集由标记为**cat**和**dog**的猫狗图片组成，各 `12500`张，总共 `25000`张，图片为24位jpg格式，即RGB三通道图像，**图片尺寸不一**。

**测试集**：测试集由 `12500`张的cat或dog图片组成，**未标记**，图片也为24位jpg格式，RGB三通道图像，**图像尺寸不一**。

使用ResNet18二分类网络实现，

### 导入数据

训练数据和测试数据存在 `CatsVSDogsDataset`类中，初始化时根据数据集类别，train数据集导入图片的完整相对地址 `list_img` 和 图片的标签 `list_label` ，标签0表示猫，1表示狗，test数据集只导入图片的完整相对地址。

图片的转换操作包括：图片大小统一、中心裁剪、转换为Tensor、标准化。并将数据的label转换为LongTensor。

```python
IMAGE_SIZE = 200

dataTransform = transforms.Compose([
    transforms.Resize(IMAGE_SIZE), # 将图片resize成统一大小
    transforms.CenterCrop((IMAGE_SIZE, IMAGE_SIZE)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])
```

`DataLoader()`的作用是对定义好的数据集类做一次封装，有以下作用：

- 可以定义为打乱数据集分布，使各个类型的样本均匀地参与网络训练；
- 可以设定多线程数据读取(指数据载入内存)，提高训练效率，因为训练过程中文件的读取是比较耗时的
- 可以一次获得batch size大小的数据，并且是Tensor形式。

```python
dataloader = DataLoader(datafile, 
                        batch_size=opt.batch_size, 
                        shuffle=True, 
                        num_workers=opt.workers)
```



### 训练过程

使用 `torchvision.models`下的 `resnet18`模型，使用Adam优化方法，loss计算方法为交叉熵，可以理解为两者数值越接近其值越小。

使用 `tqdm` 完成训练进度以及loss的可视化。

>  `Variable()`的理解，在代码出现了这个，可以把它理解为定义一个符号，一个未知数，然后网络中的所有计算方式都用这个符号来表示，最终网络计算形成一个由Variable()组成的复杂计算公式，只要将实际数据代入Variable，便可快速求出结果，并且，求导也十分方便，因为都是符号，可以调用求导公式，当然这些都是内部计算过程，外部看不到。

需要注意的是代码`loss = criterion(out, label.squeeze())`中的`.squeeze()`方法，因为从dataloader中获取的label是一个[batch_size × 1]的Tensor，而实际输入的应是1维Tensor，所以需要做一个维度变换。

最后保存模型即可。

### 测试过程

加载模型

```python
# setting model
model = models.resnet18(num_classes=2) # 实例化网络
if opt.use_gpu: # gpu优化
    model = model.cuda()
model = nn.DataParallel(model) # 多线程
model.load_state_dict(torch.load(opt.model_file)) # 加载训练模型
model.eval() # 设定为评估模式，计算过程不要dropout
```

随机加载N个图片做测试

```python
files = random.sample(os.listdir(opt.test_data_dir), opt.N) # 随机加载N个图片做测试
imgs = []  # 原始图片，用来plt.imshow(imgs[index])
imgs_data = []  # 训练图片，dataTransform(img)
for file in files:
    img = Image.open(opt.test_data_dir + file)
    imgs.append(img)
    imgs_data.append(dataset.dataTransform(img))
imgs_data = torch.stack(imgs_data) # 把Tensor list 拼接成一个Tensor
if opt.use_gpu: # gpu优化
    imgs_data = imgs_data.cuda()
```

计算

```python
# with可以对资源闭合访问，进行必要的清理（close等操作）
with torch.no_grad(): # 不自动求导
    out = model(imgs_data)
out = F.softmax(out, dim=1) # 将分类结果转换为[0, 1]的概率，dim=1表示让第一个维度的数据之和等于1
out = out.data.cpu().numpy()
```

结果展示

```python
# 展示测试图片以及预测结果
for idx in range(opt.N):
    plt.figure()
    if out[idx, 0] > out[idx, 1]:
        plt.suptitle('cat:{:.1%},dog:{:.1%}'.format(out[idx, 0], out[idx, 1]))
    else:
        plt.suptitle('dog:{:.1%},cat:{:.1%}'.format(out[idx, 1], out[idx, 0]))
    plt.imshow(imgs[idx])
plt.show()
```

<img src="https://s2.loli.net/2022/01/14/JYEO6W8dTcC1HlN.png" alt="image-20220114195827079" style="zoom:50%;" />

<img src="https://s2.loli.net/2022/01/14/EATguxjyVqwCnXh.png" alt="image-20220114195852116" style="zoom:50%;" />

<img src="https://s2.loli.net/2022/01/14/rJQOEhibP7S368a.png" alt="image-20220114195921089" style="zoom:50%;" />
