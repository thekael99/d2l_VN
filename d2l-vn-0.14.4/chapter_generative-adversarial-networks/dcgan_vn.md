<!--
# Deep Convolutional Generative Adversarial Networks
-->

# Mạng Đối sinh Tích chập Sâu
:label:`sec_dcgan`


<!--
In :numref:`sec_basic_gan`, we introduced the basic ideas behind how GANs work.
We showed that they can draw samples from some simple, easy-to-sample distribution, like a uniform or normal distribution, 
and transform them into samples that appear to match the distribution of some dataset.
And while our example of matching a 2D Gaussian distribution got the point across, it is not especially exciting.
-->

Trong :numref:`sec_basic_gan`, ta đã giới thiệu về những ý tưởng cơ bản ẩn sau cách hoạt động của GAN.
Ta đã thấy được quá trình tạo mẫu từ các phân phối đơn giản, dễ-lấy-mẫu như phân phối đều hay phân phối chuẩn, và biến đổi chúng thành các mẫu phù hợp với phân phối của tập dữ liệu nào đó.
Dù ví dụ cho GAN khớp với phân phối Gauss 2 chiều là một minh họa rõ ràng, nhưng nó không thật sự thú vị.


<!--
In this section, we will demonstrate how you can use GANs to generate photorealistic images.
We will be basing our models on the deep convolutional GANs (DCGAN) introduced in :cite:`Radford.Metz.Chintala.2015`.
We will borrow the convolutional architecture that have proven so successful for discriminative computer vision problems and show how via GANs,
they can be leveraged to generate photorealistic images.
-->

Trong phần này, chúng tôi sẽ trình bày cách dùng GAN để tạo ra những bức ảnh chân thực.
Ta sẽ xây dựng mô hình dựa theo các mô hình GAN tích chập sâu (*deep convolutional GAN - DCGAN*) được giới thiệu trong :cite:`Radford.Metz.Chintala.2015`.
Bằng cách mượn kiến trúc tích chập đã được chứng minh là thành công với bài toán thị giác máy tính phân biệt, 
và bằng cách thông qua GAN, ta có thể dùng chúng làm đòn bẩy để tạo ra các hình ảnh chân thực.


```{.python .input}
from mxnet import gluon, init, np, npx
from mxnet.gluon import nn
from d2l import mxnet as d2l

npx.set_np()
```

```{.python .input}
#@tab pytorch
from d2l import torch as d2l
import torch
import torchvision
from torch import nn
import warnings
```


<!--
## The Pokemon Dataset
-->

## Tập dữ liệu Pokemon


<!--
The dataset we will use is a collection of Pokemon sprites obtained from [pokemondb](https://pokemondb.net/sprites).
First download, extract and load this dataset.
-->

Ta sẽ sử dụng tập dữ liệu các nhân vật Pokemon từ [pokemondb](https://pokemondb.net/sprites).
Đầu tiên ta tải xuống, giải nén và nạp tập dữ liệu.


```{.python .input}
#@save
d2l.DATA_HUB['pokemon'] = (d2l.DATA_URL + 'pokemon.zip',
                           'c065c0e2593b8b161a2d7873e42418bf6a21106c')

data_dir = d2l.download_extract('pokemon')
pokemon = gluon.data.vision.datasets.ImageFolderDataset(data_dir)
```

```{.python .input}
#@tab pytorch
#@save
d2l.DATA_HUB['pokemon'] = (d2l.DATA_URL + 'pokemon.zip',
                           'c065c0e2593b8b161a2d7873e42418bf6a21106c')

data_dir = d2l.download_extract('pokemon')
pokemon = torchvision.datasets.ImageFolder(data_dir)
```


<!--
We resize each image into $64\times 64$.
The `ToTensor` transformation will project the pixel value into $[0, 1]$, while our generator will use the tanh function to obtain outputs in $[-1, 1]$.
Therefore we normalize the data with $0.5$ mean and $0.5$ standard deviation to match the value range.
-->

Ta thay đổi kích thước ảnh thành $64\times 64$.
Phép biến đổi `ToTensor` sẽ chiếu từng giá trị điểm ảnh vào khoảng $[0,1]$, trong đó mạng sinh của ta sẽ dùng hàm tanh để thu được đầu ra trong khoảng $[-1,1]$.
Do đó ta chuẩn hóa dữ liệu với trung bình $0.5$ và độ lệch chuẩn $0.5$ để khớp với miền giá trị.


```{.python .input}
batch_size = 256
transformer = gluon.data.vision.transforms.Compose([
    gluon.data.vision.transforms.Resize(64),
    gluon.data.vision.transforms.ToTensor(),
    gluon.data.vision.transforms.Normalize(0.5, 0.5)
])
data_iter = gluon.data.DataLoader(
    pokemon.transform_first(transformer), batch_size=batch_size,
    shuffle=True, num_workers=d2l.get_dataloader_workers())
```

```{.python .input}
#@tab pytorch
batch_size = 256
transformer = torchvision.transforms.Compose([
    torchvision.transforms.Resize((64, 64)),
    torchvision.transforms.ToTensor(),
    torchvision.transforms.Normalize(0.5, 0.5)
])
pokemon.transform = transformer
data_iter = torch.utils.data.DataLoader(
    pokemon, batch_size=batch_size,
    shuffle=True, num_workers=d2l.get_dataloader_workers())
```


<!--
Let us visualize the first 20 images.
-->

Hãy xem thử 20 hình đầu tiên.


```{.python .input}
d2l.set_figsize((4, 4))
for X, y in data_iter:
    imgs = X[0:20,:,:,:].transpose(0, 2, 3, 1)/2+0.5
    d2l.show_images(imgs, num_rows=4, num_cols=5)
    break
```

```{.python .input}
#@tab pytorch
warnings.filterwarnings('ignore')
d2l.set_figsize((4, 4))
for X, y in data_iter:
    imgs = X[0:20,:,:,:].permute(0, 2, 3, 1)/2+0.5
    d2l.show_images(imgs, num_rows=4, num_cols=5)
    break
```


<!--
## The Generator
-->

## Bộ Sinh

<!--
The generator needs to map the noise variable $\mathbf z\in\mathbb R^d$, a length-$d$ vector, to a RGB image with width and height to be $64\times 64$.
In :numref:`sec_fcn` we introduced the fully convolutional network that uses transposed convolution layer (refer to :numref:`sec_transposed_conv`) to enlarge input size.
The basic block of the generator contains a transposed convolution layer followed by the batch normalization and ReLU activation.
-->

Bộ sinh sẽ ánh xạ biến nhiễu $\mathbf z\in\mathbb R^d$, một vector $d$ chiều sang hình ảnh RGB với chiều rộng và chiều cao tương ứng là $64 \times 64$.
Trong :numref:`sec_fcn` ta đã giới thiệu về mạng tích chập đầy đủ, sử dụng tầng tích chập chuyển vị (tham khảo :numref:`sec_transposed_conv`) để phóng to kích thước đầu vào.
Khối cơ bản của bộ sinh gồm tầng tích chập chuyển vị, theo sau là chuẩn hóa theo batch và hàm kích hoạt ReLU.


```{.python .input}
class G_block(nn.Block):
    def __init__(self, channels, kernel_size=4,
                 strides=2, padding=1, **kwargs):
        super(G_block, self).__init__(**kwargs)
        self.conv2d_trans = nn.Conv2DTranspose(
            channels, kernel_size, strides, padding, use_bias=False)
        self.batch_norm = nn.BatchNorm()
        self.activation = nn.Activation('relu')

    def forward(self, X):
        return self.activation(self.batch_norm(self.conv2d_trans(X)))
```

```{.python .input}
#@tab pytorch
class G_block(nn.Module):
    def __init__(self, out_channels, in_channels=3, kernel_size=4, strides=2,
                 padding=1, **kwargs):
        super(G_block, self).__init__(**kwargs)
        self.conv2d_trans = nn.ConvTranspose2d(in_channels, out_channels,
                                kernel_size, strides, padding, bias=False)
        self.batch_norm = nn.BatchNorm2d(out_channels)
        self.activation = nn.ReLU()

    def forward(self, X):
        return self.activation(self.batch_norm(self.conv2d_trans(X)))
```


<!--
In default, the transposed convolution layer uses a $k_h = k_w = 4$ kernel, a $s_h = s_w = 2$ strides, and a $p_h = p_w = 1$ padding.
With a input shape of $n_h^{'} \times n_w^{'} = 16 \times 16$, the generator block will double input's width and height.
-->

Mặc định, tầng tích chập chuyển vị dùng hạt nhân $k_h = k_w = 4$, sải bước $s_h = s_w = 2$ và đệm $p_h = p_w = 1$.
Với kích thước đầu vào $n_h^{'} \times n_w^{'} = 16 \times 16$, khối bộ sinh sẽ nhân đôi chiều rộng và chiều cao của đầu vào.


$$
\begin{aligned}
n_h^{'} \times n_w^{'} &= [(n_h k_h - (n_h-1)(k_h-s_h)- 2p_h] \times [(n_w k_w - (n_w-1)(k_w-s_w)- 2p_w]\\
  &= [(k_h + s_h (n_h-1)- 2p_h] \times [(k_w + s_w (n_w-1)- 2p_w]\\
  &= [(4 + 2 \times (16-1)- 2 \times 1] \times [(4 + 2 \times (16-1)- 2 \times 1]\\
  &= 32 \times 32 .\\
\end{aligned}
$$


```{.python .input}
x = np.zeros((2, 3, 16, 16))
g_blk = G_block(20)
g_blk.initialize()
g_blk(x).shape
```

```{.python .input}
#@tab pytorch
x = torch.zeros((2, 3, 16, 16))
g_blk = G_block(20)
g_blk(x).shape
```


<!--
If changing the transposed convolution layer to a $4\times 4$ kernel, $1\times 1$ strides and zero padding.
With a input size of $1 \times 1$, the output will have its width and height increased by 3 respectively.
-->

Giả sử, ta đổi tầng tích chập chuyển vị này thành một hạt nhân $4\times 4$, sải bước $1\times 1$ và đệm không.
Với kích thước đầu vào là $1 \times 1$, chiều rộng và chiều cao của đầu ra sẽ tăng thêm 3 giá trị.


```{.python .input}
x = np.zeros((2, 3, 1, 1))
g_blk = G_block(20, strides=1, padding=0)
g_blk.initialize()
g_blk(x).shape
```

```{.python .input}
#@tab pytorch
x = torch.zeros((2, 3, 1, 1))
g_blk = G_block(20, strides=1, padding=0)
g_blk(x).shape
```


<!--
The generator consists of four basic blocks that increase input's both width and height from 1 to 32.
At the same time, it first projects the latent variable into $64\times 8$ channels, and then halve the channels each time.
At last, a transposed convolution layer is used to generate the output.
It further doubles the width and height to match the desired $64\times 64$ shape, and reduces the channel size to $3$.
The tanh activation function is applied to project output values into the $(-1, 1)$ range.
-->

Bộ sinh bao gồm bốn khối cơ bản thực hiện tăng cả chiều rộng và chiều cao của đầu vào từ 1 lên 32.
Cùng lúc đó, nó trước tiên chiếu biến tiềm ẩn này về $64\times 8$ kênh, rồi giảm một nửa số kênh sau mỗi lần.
Cuối cùng, một tầng tích chập chuyển vị được sử dụng để sinh đầu ra.
Nó tăng gấp đôi chiều rộng và chiều cao để khớp với kích thước mong muốn $64\times 64$, và giảm kích thước kênh xuống $3$.
Hàm kích hoạt tanh được áp dụng để đưa giá trị đầu ra về khoảng $(-1, 1)$.


```{.python .input}
n_G = 64
net_G = nn.Sequential()
net_G.add(G_block(n_G*8, strides=1, padding=0),  # Output: (64 * 8, 4, 4)
          G_block(n_G*4),  # Output: (64 * 4, 8, 8)
          G_block(n_G*2),  # Output: (64 * 2, 16, 16)
          G_block(n_G),    # Output: (64, 32, 32)
          nn.Conv2DTranspose(
              3, kernel_size=4, strides=2, padding=1, use_bias=False,
              activation='tanh'))  # Output: (3, 64, 64)
```

```{.python .input}
#@tab pytorch
n_G = 64
net_G = nn.Sequential(
    G_block(in_channels=100, out_channels=n_G*8,
            strides=1, padding=0),                  # Output: (64 * 8, 4, 4)
    G_block(in_channels=n_G*8, out_channels=n_G*4), # Output: (64 * 4, 8, 8)
    G_block(in_channels=n_G*4, out_channels=n_G*2), # Output: (64 * 2, 16, 16)
    G_block(in_channels=n_G*2, out_channels=n_G),   # Output: (64, 32, 32)
    nn.ConvTranspose2d(in_channels=n_G, out_channels=3, 
                       kernel_size=4, stride=2, padding=1, bias=False),
    nn.Tanh())  # Output: (3, 64, 64)
```


<!--
Generate a 100 dimensional latent variable to verify the generator's output shape.
-->

Hãy sinh một biến tiềm ẩn có số chiều là 100 để xác thực kích thước đầu ra của bộ sinh.


```{.python .input}
x = np.zeros((1, 100, 1, 1))
net_G.initialize()
net_G(x).shape
```

```{.python .input}
#@tab pytorch
x = torch.zeros((1, 100, 1, 1))
net_G(x).shape
```


<!--
## Discriminator
-->

## Bộ Phân biệt


<!--
The discriminator is a normal convolutional network network except that it uses a leaky ReLU as its activation function.
Given $\alpha \in[0, 1]$, its definition is
-->

Bộ phân biệt là một mạng tích chập thông thường ngoại trừ việc nó dùng hàm kích hoạt ReLU rò rỉ.
Với $\alpha \in[0, 1]$ cho trước, định nghĩa của nó là 


$$\textrm{ReLU rò rỉ}(x) = \begin{cases}x & \text{nếu}\ x > 0\\ \alpha x &\text{ngược lại}\end{cases}.$$
<!--dịch-->


<!--
As it can be seen, it is normal ReLU if $\alpha=0$, and an identity function if $\alpha=1$.
For $\alpha \in (0, 1)$, leaky ReLU is a nonlinear function that give a non-zero output for a negative input.
It aims to fix the "dying ReLU" problem that a neuron might always output a negative value and therefore cannot make any progress since the gradient of ReLU is 0.
-->

Như có thể thấy, nó là ReLU thông thường nếu $\alpha=0$, và là hàm đồng nhất nếu $\alpha=1$.
Cho $\alpha \in (0, 1)$, ReLU rò rỉ là một hàm phi tuyến cho đầu ra khác không với giá trị đầu vào âm.
Mục đích của hàm này là khắc phục vấn đề "ReLU chết", khi mà một nơ-ron có thể luôn xuất giá trị âm và do đó không thể được cập nhật (gradient của ReLU luôn bằng 0).


```{.python .input}
#@tab all
alphas = [0, .2, .4, .6, .8, 1]
x = d2l.arange(-2, 1, 0.1)
Y = [d2l.numpy(nn.LeakyReLU(alpha)(x)) for alpha in alphas]
d2l.plot(d2l.numpy(x), Y, 'x', 'y', alphas)
```


<!--
The basic block of the discriminator is a convolution layer followed by a batch normalization layer and a leaky ReLU activation.
The hyperparameters of the convolution layer are similar to the transpose convolution layer in the generator block.
-->

Khối cơ bản của bộ phân biệt là một tầng tích chập, theo sau bởi tầng chuẩn hóa theo batch và một hàm kích hoạt ReLU rò rỉ.
Các siêu tham số của tầng tích chập này tương tự như tầng tích chập chuyển vị trong khối sinh.


```{.python .input}
class D_block(nn.Block):
    def __init__(self, channels, kernel_size=4, strides=2,
                 padding=1, alpha=0.2, **kwargs):
        super(D_block, self).__init__(**kwargs)
        self.conv2d = nn.Conv2D(
            channels, kernel_size, strides, padding, use_bias=False)
        self.batch_norm = nn.BatchNorm()
        self.activation = nn.LeakyReLU(alpha)

    def forward(self, X):
        return self.activation(self.batch_norm(self.conv2d(X)))
```

```{.python .input}
#@tab pytorch
class D_block(nn.Module):
    def __init__(self, out_channels, in_channels=3, kernel_size=4, strides=2,
                padding=1, alpha=0.2, **kwargs):
        super(D_block, self).__init__(**kwargs)
        self.conv2d = nn.Conv2d(in_channels, out_channels, kernel_size,
                                strides, padding, bias=False)
        self.batch_norm = nn.BatchNorm2d(out_channels)
        self.activation = nn.LeakyReLU(alpha, inplace=True)

    def forward(self, X):
        return self.activation(self.batch_norm(self.conv2d(X)))
```


<!--
A basic block with default settings will halve the width and height of the inputs, as we demonstrated in :numref:`sec_padding`.
For example, given a input shape $n_h = n_w = 16$, with a kernel shape $k_h = k_w = 4$, 
a stride shape $s_h = s_w = 2$, and a padding shape $p_h = p_w = 1$, the output shape will be:
-->

Khối cơ bản với thiết lập mặc định sẽ giảm một nửa chiều rộng và chiều cao của đầu vào, như ta đã chứng tỏ trong :numref:`sec_padding`.
Chẳng hạn, cho kích thước đầu vào là $n_h = n_w = 16$, với một hạt nhân có kích thước $k_h = k_w = 4$,
sải bước $s_h = s_w = 2$, và đệm $p_h = p_w = 1$, kích thước đầu ra sẽ là:


$$
\begin{aligned}
n_h^{'} \times n_w^{'} &= \lfloor(n_h-k_h+2p_h+s_h)/s_h\rfloor \times \lfloor(n_w-k_w+2p_w+s_w)/s_w\rfloor\\
  &= \lfloor(16-4+2\times 1+2)/2\rfloor \times \lfloor(16-4+2\times 1+2)/2\rfloor\\
  &= 8 \times 8 .\\
\end{aligned}
$$


```{.python .input}
x = np.zeros((2, 3, 16, 16))
d_blk = D_block(20)
d_blk.initialize()
d_blk(x).shape
```

```{.python .input}
#@tab pytorch
x = torch.zeros((2, 3, 16, 16))
d_blk = D_block(20)
d_blk(x).shape
```


<!--
The discriminator is a mirror of the generator.
-->

Bộ phân biệt là một tấm gương phản chiếu của bộ sinh.


```{.python .input}
n_D = 64
net_D = nn.Sequential()
net_D.add(D_block(n_D),   # Output: (64, 32, 32)
          D_block(n_D*2),  # Output: (64 * 2, 16, 16)
          D_block(n_D*4),  # Output: (64 * 4, 8, 8)
          D_block(n_D*8),  # Output: (64 * 8, 4, 4)
          nn.Conv2D(1, kernel_size=4, use_bias=False))  # Output: (1, 1, 1)
```

```{.python .input}
#@tab pytorch
n_D = 64
net_D = nn.Sequential(
    D_block(n_D),  # Output: (64, 32, 32)
    D_block(in_channels=n_D, out_channels=n_D*2),  # Output: (64 * 2, 16, 16)
    D_block(in_channels=n_D*2, out_channels=n_D*4),  # Output: (64 * 4, 8, 8)
    D_block(in_channels=n_D*4, out_channels=n_D*8),  # Output: (64 * 8, 4, 4)
    nn.Conv2d(in_channels=n_D*8, out_channels=1,
              kernel_size=4, bias=False))  # Output: (1, 1, 1)
```


<!--
It uses a convolution layer with output channel $1$ as the last layer to obtain a single prediction value.
-->

Nó sử dụng một tầng tích chập với kênh đầu ra $1$ làm tầng cuối cùng để có được giá trị dự đoán duy nhất.


```{.python .input}
x = np.zeros((1, 3, 64, 64))
net_D.initialize()
net_D(x).shape
```

```{.python .input}
#@tab pytorch
x = torch.zeros((1, 3, 64, 64))
net_D(x).shape
```


<!--
## Training
-->

## Huấn luyện


<!--
Compared to the basic GAN in :numref:`sec_basic_gan`, 
we use the same learning rate for both generator and discriminator since they are similar to each other.
In addition, we change $\beta_1$ in Adam (:numref:`sec_adam`) from $0.9$ to $0.5$.
It decreases the smoothness of the momentum, the exponentially weighted moving average of past gradients, 
to take care of the rapid changing gradients because the generator and the discriminator fight with each other.
Besides, the random generated noise `Z`, is a 4-D tensor and we are using GPU to accelerate the computation.
-->

So với mô hình GAN cơ bản trong :numref:`sec_basic_gan`,
ta sử dụng cùng tốc độ học cho cả bộ sinh và bộ phân biệt do chúng tương đồng với nhau.
Thêm nữa, ta thay đổi $\beta_1$ trong Adam (:numref:`sec_adam`) từ $0.9$ về $0.5$.
Việc này làm giảm độ mượt của động lượng, tức là trung bình động trọng số mũ của các gradient trước đó,
nhằm đáp ứng sự thay đổi nhanh chóng của gradient do bộ sinh và bộ phân biệt đối kháng lẫn nhau.
Bên cạnh đó, nhiễu ngẫu nhiên `Z` là một tensor 4-D và ta sử dụng GPU để tăng tốc độ tính toán.


```{.python .input}
def train(net_D, net_G, data_iter, num_epochs, lr, latent_dim,
          device=d2l.try_gpu()):
    loss = gluon.loss.SigmoidBCELoss()
    net_D.initialize(init=init.Normal(0.02), force_reinit=True, ctx=device)
    net_G.initialize(init=init.Normal(0.02), force_reinit=True, ctx=device)
    trainer_hp = {'learning_rate': lr, 'beta1': 0.5}
    trainer_D = gluon.Trainer(net_D.collect_params(), 'adam', trainer_hp)
    trainer_G = gluon.Trainer(net_G.collect_params(), 'adam', trainer_hp)
    animator = d2l.Animator(xlabel='epoch', ylabel='loss',
                            xlim=[1, num_epochs], nrows=2, figsize=(5, 5),
                            legend=['discriminator', 'generator'])
    animator.fig.subplots_adjust(hspace=0.3)
    for epoch in range(1, num_epochs + 1):
        # Train one epoch
        timer = d2l.Timer()
        metric = d2l.Accumulator(3)  # loss_D, loss_G, num_examples
        for X, _ in data_iter:
            batch_size = X.shape[0]
            Z = np.random.normal(0, 1, size=(batch_size, latent_dim, 1, 1))
            X, Z = X.as_in_ctx(device), Z.as_in_ctx(device),
            metric.add(d2l.update_D(X, Z, net_D, net_G, loss, trainer_D),
                       d2l.update_G(Z, net_D, net_G, loss, trainer_G),
                       batch_size)
        # Show generated examples
        Z = np.random.normal(0, 1, size=(21, latent_dim, 1, 1), ctx=device)
        # Normalize the synthetic data to N(0, 1)
        fake_x = net_G(Z).transpose(0, 2, 3, 1) / 2 + 0.5
        imgs = np.concatenate(
            [np.concatenate([fake_x[i * 7 + j] for j in range(7)], axis=1)
             for i in range(len(fake_x)//7)], axis=0)
        animator.axes[1].cla()
        animator.axes[1].imshow(imgs.asnumpy())
        # Show the losses
        loss_D, loss_G = metric[0] / metric[2], metric[1] / metric[2]
        animator.add(epoch, (loss_D, loss_G))
    print(f'loss_D {loss_D:.3f}, loss_G {loss_G:.3f}, '
          f'{metric[2] / timer.stop():.1f} examples/sec on {str(device)}')
```

```{.python .input}
#@tab pytorch
def train(net_D, net_G, data_iter, num_epochs, lr, latent_dim, device=d2l.try_gpu()):
    loss = nn.BCEWithLogitsLoss(reduction='sum')
    for w in net_D.parameters():
        nn.init.normal_(w, 0, 0.02)
    for w in net_G.parameters():
        nn.init.normal_(w, 0, 0.02)
    net_D, net_G = net_D.to(device), net_G.to(device)
    trainer_hp = {'lr': lr, 'betas': [0.5,0.999]}
    trainer_D = torch.optim.Adam(net_D.parameters(), **trainer_hp)
    trainer_G = torch.optim.Adam(net_G.parameters(), **trainer_hp)
    animator = d2l.Animator(xlabel='epoch', ylabel='loss',
                            xlim=[1, num_epochs], nrows=2, figsize=(5, 5),
                            legend=['discriminator', 'generator'])
    animator.fig.subplots_adjust(hspace=0.3)
    for epoch in range(1, num_epochs + 1):
        # Train one epoch
        timer = d2l.Timer()
        metric = d2l.Accumulator(3)  # loss_D, loss_G, num_examples
        for X, _ in data_iter:
            batch_size = X.shape[0]
            Z = torch.normal(0, 1, size=(batch_size, latent_dim, 1, 1))
            X, Z = X.to(device), Z.to(device)
            metric.add(d2l.update_D(X, Z, net_D, net_G, loss, trainer_D),
                       d2l.update_G(Z, net_D, net_G, loss, trainer_G),
                       batch_size)
        # Show generated examples
        Z = torch.normal(0, 1, size=(21, latent_dim, 1, 1), device=device)
        # Normalize the synthetic data to N(0, 1)
        fake_x = net_G(Z).permute(0, 2, 3, 1) / 2 + 0.5
        imgs = torch.cat(
            [torch.cat([fake_x[i * 7 + j].cpu().detach() for j in range(7)], dim=1)
             for i in range(len(fake_x)//7)], dim=0)
        animator.axes[1].cla()
        animator.axes[1].imshow(imgs)
        # Show the losses
        loss_D, loss_G = metric[0] / metric[2], metric[1] / metric[2]
        animator.add(epoch, (loss_D, loss_G))
    print(f'loss_D {loss_D:.3f}, loss_G {loss_G:.3f}, '
          f'{metric[2] / timer.stop():.1f} examples/sec on {str(device)}')
```


<!--
We train the model with a small number of epochs just for demonstration.
For better performance, the variable `num_epochs` can be set to a larger number.
-->

Ta sẽ chỉ huấn luyện mô hình với số epoch nhỏ để minh họa.
Để đạt chất lượng mô hình tốt hơn, bạn có thể đặt biến `num_epochs` bằng một giá trị lớn hơn.


```{.python .input}
#@tab all
latent_dim, lr, num_epochs = 100, 0.005, 20
train(net_D, net_G, data_iter, num_epochs, lr, latent_dim)
```

## Tóm tắt

<!--
* DCGAN architecture has four convolutional layers for the Discriminator and four "fractionally-strided" convolutional layers for the Generator.
* The Discriminator is a 4-layer strided convolutions with batch normalization (except its input layer) and leaky ReLU activations.
* Leaky ReLU is a nonlinear function that give a non-zero output for a negative input. 
It aims to fix the “dying ReLU” problem and helps the gradients flow easier through the architecture.
-->

* Kiến trúc DCGAN gồm có bốn tầng tích chập cho Bộ phân biệt, và bốn tầng tích chập "sải bước một phần (*fractionally-strided*)" cho Bộ sinh.
* Bộ phân biệt là một mạng 4 tầng bao gồm các tầng tích chập có sải bước, theo sau bởi tầng chuẩn hoá theo batch (trừ tầng đầu vào) và hàm kích hoạt ReLU rò rỉ.
* ReLU rò rỉ là một hàm phi tuyến trả về kết quả khác không với đầu vào âm.
Hàm này nhằm khắc phục vấn đề "ReLU chết", giúp gradient truyền đi dễ dàng hơn xuyên suốt kiến trúc.


## Bài tập

<!--
1. What will happen if we use standard ReLU activation rather than leaky ReLU?
2. Apply DCGAN on Fashion-MNIST and see which category works well and which does not.
-->

1. Chuyện gì sẽ xảy ra nếu ta sử dụng hàm kích hoạt ReLU phổ thông thay vì ReLU rò rỉ?
2. Áp dụng DCGAN trên Fashion-MNIST và quan sát xem đối với hạng mục nào thì nó hoạt động tốt, hạng mục nào thì không.


## Thảo luận
* Tiếng Anh: [MXNet](https://discuss.d2l.ai/t/409), [PyTorch](https://discuss.d2l.ai/t/1083)
* Tiếng Việt: [Diễn đàn Machine Learning Cơ Bản](https://forum.machinelearningcoban.com/c/d2l)


## Những người thực hiện
Bản dịch trong trang này được thực hiện bởi:

* Đoàn Võ Duy Thanh
* Lý Phi Long
* Nguyễn Mai Hoàng Long
* Phạm Hồng Vinh
* Phạm Minh Đức
* Đỗ Trường Giang
* Nguyễn Lê Quang Nhật
* Nguyễn Văn Cường

*Lần cập nhật gần nhất: 05/10/2020. (Cập nhật lần cuối từ nội dung gốc: 17/09/2020)*