## 便于学习的数据集

一些可以通过 Python 直接导入使用的机器学习数据集，主要通过常见的机器学习库（如 scikit-learn、Keras、TensorFlow、PyTorch）提供：

### 1. 使用 scikit-learn 导入数据集

scikit-learn 提供了许多内置的数据集，可以直接加载使用：

- **Iris 数据集**：用于分类任务。

```python
from sklearn import datasets
iris = datasets.load_iris()
print(iris.data[:5])  # 查看特征数据
print(iris.target[:5])  # 查看目标数据
```

- **手写数字数据集（Digits）**：用于图像分类任务。
  
```python
digits = datasets.load_digits()
print(digits.data.shape)  # 查看数据形状
```

- **波士顿房价数据集（Boston Housing）**：用于回归任务。
  
  ```python
  boston = datasets.load_boston()
  print(boston.data.shape)  # 查看特征数据形状
  print(boston.target.shape)  # 查看目标数据形状
  ```

### 2. 使用 Keras 导入 MNIST 数据集

Keras 提供了 MNIST 数据集的加载接口：

```python
from keras.datasets import mnist
(train_images, train_labels), (test_images, test_labels) = mnist.load_data()
print(train_images.shape)  # 查看训练集图像形状
print(test_images.shape)  # 查看测试集图像形状
```

### 3. 使用 TensorFlow 导入 MNIST 数据集

TensorFlow 同样提供了 MNIST 数据集的加载接口：

```python
import tensorflow as tf
mnist = tf.keras.datasets.mnist
(train_images, train_labels), (test_images, test_labels) = mnist.load_data()
print(train_images.shape)  # 查看训练集图像形状
print(test_images.shape)  # 查看测试集图像形状
```

### 4. 使用 PyTorch 导入 MNIST 数据集

PyTorch 也提供了 MNIST 数据集的加载接口：

```python
from torchvision import datasets, transforms
transform = transforms.Compose([transforms.ToTensor()])
train_dataset = datasets.MNIST(root='./data', train=True, download=True, transform=transform)
test_dataset = datasets.MNIST(root='./data', train=False, download=True, transform=transform)
print(train_dataset.data.shape)  # 查看训练集图像形状
print(test_dataset.data.shape)  # 查看测试集图像形状
```

### 5. 使用 Pandas 导入 CSV 数据集

如果数据集以 CSV 文件形式提供，可以使用 Pandas 读取：

```python
import pandas as pd
df = pd.read_csv('path/to/your/dataset.csv')
print(df.head())  # 查看数据集前几行
```

### 6. 使用 Kaggle API 导入数据集
Kaggle 提供了丰富的数据集，可以通过 Kaggle API 下载：

1. **安装 Kaggle 库**：
   
   ```bash
   pip install kaggle
   ```

2. **获取 API 密钥**：在 Kaggle 官网登录账号，进入个人账户设置页面，找到 API 密钥选项并生成 API 密钥。将 API 密钥下载到本地，并将其路径添加到环境变量中。
3. **下载数据集**：

   ```python
   import kaggle
   kaggle.api.authenticate()
   kaggle.api.dataset_download_files('username/dataset-name', path='./data', unzip=True)
   ```

4. **加载数据集**：

   ```python
   import pandas as pd
   df = pd.read_csv('./data/dataset.csv')
   print(df.head())
   ```