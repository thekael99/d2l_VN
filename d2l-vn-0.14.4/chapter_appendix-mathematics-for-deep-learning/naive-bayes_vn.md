<!--
# Naive Bayes
-->

# Bộ phân loại Naive Bayes
:label:`sec_naive_bayes`


<!--
Throughout the previous sections, we learned about the theory of probability and random variables.
To put this theory to work, let us introduce the *naive Bayes* classifier.
This uses nothing but probabilistic fundamentals to allow us to perform classification of digits.
-->

Trong các phần trước, ta đã học lý thuyết về xác suất và biến ngẫu nhiên.
Để áp dụng lý thuyết này, ta sẽ lấy một ví dụ sử dụng bộ phân loại *naive Bayes* cho bài toán phân loại chữ số.
Phương pháp này không sử dụng bất kỳ điều gì khác ngoài các lý thuyết căn bản về xác suất.


<!--
Learning is all about making assumptions. If we want to classify a new data example 
that we have never seen before we have to make some assumptions about which data examples are similar to each other.
The naive Bayes classifier, a popular and remarkably clear algorithm, assumes all features are independent from each other to simplify the computation.
In this section, we will apply this model to recognize characters in images.
-->

Quá trình học hoàn toàn xoay quanh việc đưa ra các giả định.
Nếu muốn phân loại một mẫu dữ liệu mới chưa thấy bao giờ, ta cần phải đưa ra một giả định nào đó về sự tương đồng giữa các mẫu dữ liệu.
Bộ phân loại Naive Bayes, một thuật toán thông dụng và dễ hiểu, giả định rằng tất cả các đặc trưng đều độc lập với nhau nhằm đơn giản hóa việc tính toán.
Trong phần này, chúng tôi sẽ sử dụng mô hình này để nhận dạng ký tự trong ảnh.


```{.python .input}
%matplotlib inline
from d2l import mxnet as d2l
import math
from mxnet import gluon, np, npx
npx.set_np()
d2l.use_svg_display()
```

```{.python .input}
#@tab pytorch
%matplotlib inline
from d2l import torch as d2l
import math
import torch
import torchvision
d2l.use_svg_display()
```

```{.python .input}
#@tab tensorflow
%matplotlib inline
from d2l import tensorflow as d2l
import math
import tensorflow as tf
d2l.use_svg_display()
```


<!--
## Optical Character Recognition
-->

## Nhận diện Ký tự Quang học


<!--
MNIST :cite:`LeCun.Bottou.Bengio.ea.1998` is one of widely used datasets.
It contains 60,000 images for training and 10,000 images for validation.
Each image contains a handwritten digit from 0 to 9.
The task is classifying each image into the corresponding digit.
-->

MNIST :cite:`LeCun.Bottou.Bengio.ea.1998` là một trong những tập dữ liệu được sử dụng rộng rãi.
Nó chứa 60.000 ảnh để huấn luyện và 10.000 ảnh để kiểm định.
Mỗi ảnh chứa một chữ số viết tay từ 0 đến 9.
Nhiệm vụ là phân loại từng ảnh theo chữ số tương ứng.


<!--
Gluon provides a `MNIST` class in the `data.vision` module to
automatically retrieve the dataset from the Internet.
Subsequently, Gluon will use the already-downloaded local copy.
We specify whether we are requesting the training set or the test set
by setting the value of the parameter `train` to `True` or `False`, respectively.
Each image is a grayscale image with both width and height of $28$ with shape ($28$,$28$,$1$).
We use a customized transformation to remove the last channel dimension.
In addition, the dataset represents each pixel by an unsigned $8$-bit integer.
We quantize them into binary features to simplify the problem.
-->

Gluon cung cấp lớp `MNIST` trong mô-đun `data.vision` để tự động lấy dữ liệu từ Internet. 
Sau đó, Gluon sẽ sử dụng dữ liệu đã được tải xuống.
Chúng ta xác định rằng ta đang yêu cầu tập huấn luyện hay tập kiểm tra bằng cách đặt giá trị tham số `train` thành` True` hoặc `False` tương ứng.
Mỗi hình ảnh là một ảnh xám có cả chiều rộng và chiều cao là $28$, kích thước ($28$,$28$,$1$).
Ta sẽ sử dụng một phép biến đổi được tùy chỉnh để loại bỏ chiều của kênh cuối cùng.
Ngoài ra, tập dữ liệu biểu diễn mỗi điểm ảnh bằng một số nguyên $8$-bit không âm.
Ta lượng tử hóa (*quantize*) chúng thành các đặc trưng nhị phân để đơn giản hóa bài toán.



```{.python .input}
def transform(data, label):
    return np.floor(data.astype('float32') / 128).squeeze(axis=-1), label

mnist_train = gluon.data.vision.MNIST(train=True, transform=transform)
mnist_test = gluon.data.vision.MNIST(train=False, transform=transform)
```

```{.python .input}
#@tab pytorch
data_transform = torchvision.transforms.Compose(
    [torchvision.transforms.ToTensor()])

mnist_train = torchvision.datasets.MNIST(
    root='./temp', train=True, transform=data_transform, download=True)
mnist_test = torchvision.datasets.MNIST(
    root='./temp', train=False, transform=data_transform, download=True)
```

```{.python .input}
#@tab tensorflow
((train_images, train_labels), (
    test_images, test_labels)) = tf.keras.datasets.mnist.load_data()
```


<!--
We can access a particular example, which contains the image and the corresponding label.
-->

Ta có thể truy cập vào từng mẫu cụ thể có chứa ảnh và nhãn tương ứng.


```{.python .input}
image, label = mnist_train[2]
image.shape, label
```

```{.python .input}
#@tab pytorch
image, label = mnist_train[2]
image.shape, label
```

```{.python .input}
#@tab tensorflow
image, label = train_images[2], train_labels[2]
image.shape, label
```


<!--
Our example, stored here in the variable `image`, corresponds to an image with a height and width of $28$ pixels.
-->

Mẫu được lưu trữ trong biến `image` trên tương ứng với một ảnh có chiều cao và chiều rộng là $28$ pixel.


```{.python .input}
#@tab all
image.shape, image.dtype
```


<!--
Our code stores the label of each image as a scalar. Its type is a $32$-bit integer.
-->

Đoạn mã lưu nhãn của từng ảnh dưới dạng số nguyên $32$-bit.


```{.python .input}
label, type(label), label.dtype
```

```{.python .input}
#@tab pytorch
label, type(label)
```

```{.python .input}
#@tab tensorflow
label, type(label)
```


<!--
We can also access multiple examples at the same time.
-->

Ta cũng có thể truy cập vào nhiều mẫu cùng một lúc.


```{.python .input}
images, labels = mnist_train[10:38]
images.shape, labels.shape
```

```{.python .input}
#@tab pytorch
images = torch.stack([mnist_train[i][0] for i in range(10,38)], 
                     dim=1).squeeze(0)
labels = torch.tensor([mnist_train[i][1] for i in range(10,38)])
images.shape, labels.shape
```

```{.python .input}
#@tab tensorflow
images = tf.stack([train_images[i] for i in range(10, 38)], axis=0)
labels = tf.constant([train_labels[i] for i in range(10, 38)])
images.shape, labels.shape
```


<!--
Let us visualize these examples.
-->

Hãy cùng minh họa các mẫu trên.


```{.python .input}
#@tab all
d2l.show_images(images, 2, 9);
```


<!--
## The Probabilistic Model for Classification
-->

## Mô hình xác suất để Phân loại


<!--
In a classification task, we map an example into a category.
Here an example is a grayscale $28\times 28$ image, and a category is a digit.
(Refer to :numref:`sec_softmax` for a more detailed explanation.)
One natural way to express the classification task is via the probabilistic question: what is the most likely label given the features (i.e., image pixels)?
Denote by $\mathbf x\in\mathbb R^d$ the features of the example and $y\in\mathbb R$ the label.
Here features are image pixels, where we can reshape a $2$-dimensional image to a vector so that $d=28^2=784$, and labels are digits.
The probability of the label given the features is $p(y  \mid  \mathbf{x})$. If we are able to compute these probabilities, 
which are $p(y  \mid  \mathbf{x})$ for $y=0, \ldots,9$ in our example, then the classifier will output the prediction $\hat{y}$ given by the expression:
-->

Trong tác vụ phân loại, ta ánh xạ một mẫu tới một hạng mục.
Ví dụ, ở đây ta ánh xạ một ảnh xám kích thước $28\times 28$ tới hạng mục là một chữ số.
(Tham khảo :numref:`sec_softmax` để xem giải thích chi tiết hơn.)
Một cách diễn đạt tự nhiên về tác vụ phân loại là câu hỏi xác suất: nhãn nào là hợp lý nhất với các đặc trưng cho trước (tức là các pixel trong ảnh)?
Ký hiệu $\mathbf x\in\mathbb R^d$ là các đặc trưng và $y\in\mathbb R$ là nhãn của một mẫu.
Đặc trưng ở đây là các pixel trong ảnh $2$ chiều mà ta có thể biến đổi thành vector kích thước $d=28^2=784$, và nhãn là các chữ số.
Xác suất của nhãn khi biết trước đặc trưng là $p(y  \mid  \mathbf{x})$. Trong ví dụ của ta, nếu có thể tính toán các xác suất
$p(y  \mid  \mathbf{x})$ với $y=0, \ldots,9$, bộ phân loại sẽ đưa ra dự đoán $\hat{y}$ theo công thức:



$$\hat{y} = \mathrm{argmax} \> p(y  \mid  \mathbf{x}).$$


<!--
Unfortunately, this requires that we estimate $p(y  \mid  \mathbf{x})$ for every value of $\mathbf{x} = x_1, ..., x_d$.
Imagine that each feature could take one of $2$ values.
For example, the feature $x_1 = 1$ might signify that the word apple appears in a given document and $x_1 = 0$ would signify that it does not.
If we had $30$ such binary features, that would mean that we need to be prepared to classify any of $2^{30}$ (over 1 billion!) possible values of the input vector $\mathbf{x}$.
-->

Không may là việc này yêu cầu ước lượng $p(y  \mid  \mathbf{x})$ cho mọi giá trị $\mathbf{x} = x_1, ..., x_d$.
Hãy tưởng tượng mỗi đặc trưng nhận một giá trị nhị phân,
ví dụ, đặc trưng $x_1 = 1$ cho biết từ "quả táo" xuất hiện trong văn bản cho trước và $x_1 = 0$ biểu thị ngược lại.
Nếu có $30$ đặc trưng nhị phân như vậy, ta cần phân loại $2^{30}$ (hơn 1 tỷ!) vector đầu vào khả dĩ của $\mathbf{x}$.


<!--
Moreover, where is the learning? If we need to see every single possible example in order to predict 
the corresponding label then we are not really learning a pattern but just memorizing the dataset.
-->

Hơn nữa, như vậy không phải là học. Nếu cần xem qua toàn bộ các ví dụ khả dĩ để dự đoán
nhãn tương ứng thì chúng ta không thực sự đang học một khuôn mẫu nào mà chỉ là đang ghi nhớ tập dữ liệu.


<!--
## The Naive Bayes Classifier
-->

## Bộ phân loại Naive Bayes


<!--
Fortunately, by making some assumptions about conditional independence, 
we can introduce some inductive bias and build a model capable of generalizing from a comparatively modest selection of training examples.
To begin, let us use Bayes theorem, to express the classifier as
-->

May mắn thay, bằng cách đưa ra một số giả định về tính độc lập có điều kiện,
ta có thể đưa vào một số thiên kiến quy nạp và xây dựng một mô hình có khả năng tổng quát hóa từ một nhóm các mẫu huấn luyện với kích thước tương đối khiêm tốn.
Để bắt đầu, hãy sử dụng định lý Bayes để biểu diễn bộ phân loại bằng biểu thức sau


$$\hat{y} = \mathrm{argmax}_y \> p(y  \mid  \mathbf{x}) = \mathrm{argmax}_y \> \frac{p( \mathbf{x}  \mid  y) p(y)}{p(\mathbf{x})}.$$


<!--
Note that the denominator is the normalizing term $p(\mathbf{x})$ which does not depend on the value of the label $y$.
As a result, we only need to worry about comparing the numerator across different values of $y$.
Even if calculating the denominator turned out to be intractable, we could get away with ignoring it, so long as we could evaluate the numerator.
Fortunately, even if we wanted to recover the normalizing constant, we could.
We can always recover the normalization term since $\sum_y p(y  \mid  \mathbf{x}) = 1$.
-->

Lưu ý rằng mẫu số là số hạng chuẩn hóa $p(\mathbf{x})$ không phụ thuộc vào giá trị của nhãn $y$.
Do đó, chúng ta chỉ cần quan tâm tới việc so sánh tử số giữa các giá trị $y$ khác nhau.
Ngay cả khi việc tính toán mẫu số hóa ra là không thể, ta cũng có thể bỏ qua nó, miễn là ta có thể tính được tử số.
May mắn thay, ta vẫn có thể khôi phục lại hằng số chuẩn hóa nếu muốn.
Ta luôn có thể khôi phục số hạng chuẩn hóa vì $\sum_y p(y  \mid  \mathbf{x}) = 1$.


<!--
Now, let us focus on $p( \mathbf{x}  \mid  y)$.
Using the chain rule of probability, we can express the term $p( \mathbf{x}  \mid  y)$ as
-->

Bây giờ, hãy tập trung vào biểu thức $p( \mathbf{x}  \mid  y)$.
Sử dụng quy tắc dây chuyền cho xác suất, chúng ta có thể biểu diễn số hạng $p( \mathbf{x}  \mid  y)$ dưới dạng


$$p(x_1  \mid y) \cdot p(x_2  \mid  x_1, y) \cdot ... \cdot p( x_d  \mid  x_1, ..., x_{d-1}, y).$$


<!--
By itself, this expression does not get us any further. We still must estimate roughly $2^d$ parameters.
However, if we assume that *the features are conditionally independent of each other, given the label*, 
then suddenly we are in much better shape, as this term simplifies to $\prod_i p(x_i  \mid  y)$, giving us the predictor
-->

Biểu thức này tự nó không giúp ta được thêm điều gì. Ta vẫn phải ước lượng khoảng $2^d$ các tham số.
Tuy nhiên, nếu chúng ta giả định rằng *các đặc trưng khi biết nhãn cho trước là độc lập với nhau*,
thì số hạng này đơn giản hóa thành $\prod_i p(x_i  \mid  y)$, và ta có hàm dự đoán:


$$ \hat{y} = \mathrm{argmax}_y \> \prod_{i=1}^d p(x_i  \mid  y) p(y).$$


<!--
If we can estimate $\prod_i p(x_i=1  \mid  y)$ for every $i$ and $y$, and save its value in $P_{xy}[i, y]$, 
here $P_{xy}$ is a $d\times n$ matrix with $n$ being the number of classes and $y\in\{1, \ldots, n\}$.
In addition, we estimate $p(y)$ for every $y$ and save it in $P_y[y]$, with $P_y$ a $n$-length vector.
Then for any new example $\mathbf x$, we could compute
-->

Ta có thể ước lượng $\prod_i p(x_i=1  \mid  y)$ với mỗi $i$ và $y$, và lưu giá trị của nó trong $P_{xy}[i, y]$, 
ở đây $P_{xy}$ là một ma trận có kích thước $d\times n$ với $n$ là số lượng các lớp và $y\in\{1, \ldots, n\}$.
Cùng với đó, ta ước lượng $p(y)$ cho mỗi $y$ và lưu trong $P_y[y]$, với $P_y$ là một vector có độ dài $n$.
Sau đó, đối với bất kỳ mẫu mới $\mathbf x$ nào, ta có thể tính:


$$ \hat{y} = \mathrm{argmax}_y \> \prod_{i=1}^d P_{xy}[x_i, y]P_y[y],$$
:eqlabel:`eq_naive_bayes_estimation`


<!--
for any $y$. So our assumption of conditional independence has taken the complexity of our model from 
an exponential dependence on the number of features $\mathcal{O}(2^dn)$ to a linear dependence, which is $\mathcal{O}(dn)$.
-->

cho $y$ bất kỳ. Vì vậy, giả định của chúng ta về sự độc lập có điều kiện đã làm giảm độ phức tạp của mô hình từ
phụ thuộc theo cấp số nhân vào số lượng các đặc trưng $\mathcal{O}(2^dn)$ thành phụ thuộc tuyến tính, tức là $\mathcal{O}(dn)$.


<!--
## Training
-->

## Huấn luyện


<!--
The problem now is that we do not know $P_{xy}$ and $P_y$.
So we need to estimate their values given some training data first.
This is *training* the model. Estimating $P_y$ is not too hard.
Since we are only dealing with $10$ classes, we may count the number of occurrences $n_y$ for each of the digits and divide it by the total amount of data $n$.
For instance, if digit 8 occurs $n_8 = 5,800$ times and we have a total of $n = 60,000$ images, the probability estimate is $p(y=8) = 0.0967$.
-->

Vấn đề bây giờ là ta không biết $P_{xy}$ và $P_y$.
Vì vậy, ta trước tiên cần ước lượng giá trị của chúng với dữ liệu huấn luyện.
Đây là việc *huấn luyện* mô hình. Ước lượng $P_y$ không quá khó.
Do chỉ đang làm việc với $10$ lớp, ta có thể đếm số lần xuất hiện $n_y$ của mỗi chữ số và chia nó cho tổng số dữ liệu $n$.
Chẳng hạn, nếu chữ số 8 xuất hiện $n_8 = 5,800$ lần và ta có tổng số hình ảnh là $n = 60.000$, xác suất ước lượng sẽ là $p(y=8) = 0.0967$.


```{.python .input}
X, Y = mnist_train[:]  # All training examples

n_y = np.zeros((10))
for y in range(10):
    n_y[y] = (Y == y).sum()
P_y = n_y / n_y.sum()
P_y
```

```{.python .input}
#@tab pytorch
X = torch.stack([mnist_train[i][0] for i in range(len(mnist_train))], 
                dim=1).squeeze(0)
Y = torch.tensor([mnist_train[i][1] for i in range(len(mnist_train))])

n_y = torch.zeros(10)
for y in range(10):
    n_y[y] = (Y == y).sum()
P_y = n_y / n_y.sum()
P_y
```

```{.python .input}
#@tab tensorflow
X = tf.stack([train_images[i] for i in range(len(train_images))], axis=0)
Y = tf.constant([train_labels[i] for i in range(len(train_labels))])

n_y = tf.Variable(tf.zeros(10))
for y in range(10):
    n_y[y].assign(tf.reduce_sum(tf.cast(Y == y, tf.float32)))
P_y = n_y / tf.reduce_sum(n_y)
P_y
```


<!--
Now on to slightly more difficult things $P_{xy}$. Since we picked black and white images, 
$p(x_i  \mid  y)$ denotes the probability that pixel $i$ is switched on for class $y$.
Just like before we can go and count the number of times $n_{iy}$ such that an event occurs and divide it by 
the total number of occurrences of $y$, i.e., $n_y$.
But there is something slightly troubling: certain pixels may never be black (e.g., for well cropped images the corner pixels might always be white).
A convenient way for statisticians to deal with this problem is to add pseudo counts to all occurrences.
Hence, rather than $n_{iy}$ we use $n_{iy}+1$ and instead of $n_y$ we use $n_{y} + 1$.
This is also called *Laplace Smoothing*. It may seem ad-hoc, however it may be well motivated from a Bayesian point-of-view.
-->

Giờ hãy chuyển sang vấn đề khó hơn một chút là tính $P_{xy}$. Vì ta lấy các ảnh đen trắng,
$p(x_i \mid y)$ biểu thị xác suất điểm ảnh $i$ mang nhãn $y$.
Đơn giản giống như trên, ta có thể duyệt và đếm số lần $n_{iy}$ mà điểm ảnh $i$ mang nhãn $y$ và chia nó cho
tổng số lần xuất hiện $n_y$ của $y$.
Nhưng có một điểm hơi rắc rối: một số điểm ảnh nhất định có thể không bao giờ có màu đen (ví dụ, đối với các ảnh được cắt xén tốt, các điểm ảnh ở góc có thể luôn là màu trắng).
Một cách thuận tiện cho các nhà thống kê học để giải quyết vấn đề này là cộng thêm một số đếm giả vào tất cả các lần xuất hiện.
Do đó, thay vì $n_{iy} $, ta dùng $n_{iy} + 1$ và thay vì $n_y$, ta dùng $n_{y} + 1 $.
Phương pháp này còn được gọi là *Làm mượt Laplace* (*Laplace Smoothing*), có vẻ không chính thống nhưng hợp với quan điểm Bayes.


```{.python .input}
n_x = np.zeros((10, 28, 28))
for y in range(10):
    n_x[y] = np.array(X.asnumpy()[Y.asnumpy() == y].sum(axis=0))
P_xy = (n_x + 1) / (n_y + 1).reshape(10, 1, 1)

d2l.show_images(P_xy, 2, 5);
```

```{.python .input}
#@tab pytorch
n_x = torch.zeros((10, 28, 28))
for y in range(10):
    n_x[y] = torch.tensor(X.numpy()[Y.numpy() == y].sum(axis=0))
P_xy = (n_x + 1) / (n_y + 1).reshape(10, 1, 1)

d2l.show_images(P_xy, 2, 5);
```

```{.python .input}
#@tab tensorflow
n_x = tf.Variable(tf.zeros((10, 28, 28)))
for y in range(10):
    n_x[y].assign(tf.cast(tf.reduce_sum(
        X.numpy()[Y.numpy() == y], axis=0), tf.float32))
P_xy = (n_x + 1) / tf.reshape((n_y + 1), (10, 1, 1))

d2l.show_images(P_xy, 2, 5);
```


<!--
By visualizing these $10\times 28\times 28$ probabilities (for each pixel for each class) we could get some mean looking digits.
-->


Bằng cách trực quan hóa các xác suất $10\times 28\times 28$ này (cho mỗi điểm ảnh đối với mỗi lớp), ta có thể lấy được hình ảnh trung bình của các chữ số.


<!--
Now we can use :eqref:`eq_naive_bayes_estimation` to predict a new image.
Given $\mathbf x$, the following functions computes $p(\mathbf x \mid y)p(y)$ for every $y$.
-->

Giờ ta có thể sử dụng :eqref:`eq_naive_bayes_estimation` để dự đoán một hình ảnh mới.
Cho $\mathbf x$, hàm sau sẽ tính $p(\mathbf x \mid y)p(y)$ với mỗi $y$.


```{.python .input}
def bayes_pred(x):
    x = np.expand_dims(x, axis=0)  # (28, 28) -> (1, 28, 28)
    p_xy = P_xy * x + (1 - P_xy)*(1 - x)
    p_xy = p_xy.reshape(10, -1).prod(axis=1)  # p(x|y)
    return np.array(p_xy) * P_y

image, label = mnist_test[0]
bayes_pred(image)
```

```{.python .input}
#@tab pytorch
def bayes_pred(x):
    x = x.unsqueeze(0)  # (28, 28) -> (1, 28, 28)
    p_xy = P_xy * x + (1 - P_xy)*(1 - x)
    p_xy = p_xy.reshape(10, -1).prod(dim=1)  # p(x|y)
    return p_xy * P_y

image, label = mnist_test[0]
bayes_pred(image)
```

```{.python .input}
#@tab tensorflow
def bayes_pred(x):
    x = tf.expand_dims(x, axis=0)  # (28, 28) -> (1, 28, 28)
    p_xy = P_xy * x + (1 - P_xy)*(1 - x)
    p_xy = tf.math.reduce_prod(tf.reshape(p_xy, (10, -1)), axis=1)  # p(x|y)
    return p_xy * P_y

image, label = tf.cast(train_images[0], tf.float32), train_labels[0]
bayes_pred(image)
```


<!--
This went horribly wrong! To find out why, let us look at the per pixel probabilities.
They are typically numbers between $0.001$ and $1$. We are multiplying $784$ of them.
At this point it is worth mentioning that we are calculating these numbers on a computer, hence with a fixed range for the exponent.
What happens is that we experience *numerical underflow*, i.e., multiplying all the small numbers leads to something even smaller until it is rounded down to zero.
We discussed this as a theoretical issue in :numref:`sec_maximum_likelihood`, but we see the phenomena clearly here in practice.
-->

Điều này đã dẫn tới sai lầm khủng khiếp! Để tìm hiểu lý do tại sao, ta hãy xem xét xác suất trên mỗi điểm ảnh.
Chúng thường mang giá trị từ $0.001$ đến $1$ và ta đang nhân $784$ con số như vậy.
Ta đang tính những con số này trên máy tính, do đó sẽ có một phạm vi cố định cho số mũ.
Ta đã gặp vấn đề *tràn số dưới (underflow)*, tức là tích tất cả các số nhỏ hơn một sẽ dẫn đến một số nhỏ dần cho đến khi kết quả được làm tròn thành 0.
Ta đã thảo luận lý thuyết về vấn đề này trong :numref:`sec_maximum_likelihood`, và ở đây ta thấy hiện tượng này trong thực tế một cách rõ ràng.


<!--
As discussed in that section, we fix this by use the fact that $\log a b = \log a + \log b$, i.e., we switch to summing logarithms.
Even if both $a$ and $b$ are small numbers, the logarithm values should be in a proper range.
-->

Như đã thảo luận trong phần đó, ta khắc phục điều này bằng cách sử dụng tính chất $\log a b = \log a + \log b$, cụ thể là ta chuyển sang tính tổng các logarit.
Nhờ vậy ngay cả khi cả $a$ và $b$ đều là các số nhỏ, giá trị các logarit vẫn sẽ nằm trong miền thích hợp.


```{.python .input}
a = 0.1
print('underflow:', a**784)
print('logarithm is normal:', 784*math.log(a))
```

```{.python .input}
#@tab pytorch
a = 0.1
print('underflow:', a**784)
print('logarithm is normal:', 784*math.log(a))
```

```{.python .input}
#@tab tensorflow
a = 0.1
print('underflow:', a**784)
print('logarithm is normal:', 784*tf.math.log(a).numpy())
```


<!--
Since the logarithm is an increasing function, we can rewrite :eqref:`eq_naive_bayes_estimation` as
-->

Vì logarit là một hàm tăng dần, ta có thể viết lại :eqref:`eq_naive_bayes_estimation` thành


$$ \hat{y} = \mathrm{argmax}_y \> \sum_{i=1}^d \log P_{xy}[x_i, y] + \log P_y[y].$$


<!--
We can implement the following stable version:
-->

Ta có thể lập trình phiên bản ổn định sau:


```{.python .input}
log_P_xy = np.log(P_xy)
log_P_xy_neg = np.log(1 - P_xy)
log_P_y = np.log(P_y)

def bayes_pred_stable(x):
    x = np.expand_dims(x, axis=0)  # (28, 28) -> (1, 28, 28)
    p_xy = log_P_xy * x + log_P_xy_neg * (1 - x)
    p_xy = p_xy.reshape(10, -1).sum(axis=1)  # p(x|y)
    return p_xy + log_P_y

py = bayes_pred_stable(image)
py
```

```{.python .input}
#@tab pytorch
log_P_xy = torch.log(P_xy)
log_P_xy_neg = torch.log(1 - P_xy)
log_P_y = torch.log(P_y)

def bayes_pred_stable(x):
    x = x.unsqueeze(0)  # (28, 28) -> (1, 28, 28)
    p_xy = log_P_xy * x + log_P_xy_neg * (1 - x)
    p_xy = p_xy.reshape(10, -1).sum(axis=1)  # p(x|y)
    return p_xy + log_P_y

py = bayes_pred_stable(image)
py
```

```{.python .input}
#@tab tensorflow
log_P_xy = tf.math.log(P_xy)
# TODO: Look into why this returns infs
log_P_xy_neg = tf.math.log(1 - P_xy)
log_P_y = tf.math.log(P_y)

def bayes_pred_stable(x):
    x = tf.expand_dims(x, axis=0)  # (28, 28) -> (1, 28, 28)
    p_xy = log_P_xy * x + log_P_xy_neg * (1 - x)
    p_xy = tf.math.reduce_sum(tf.reshape(p_xy, (10, -1)), axis=1)  # p(x|y)
    return p_xy + log_P_y

py = bayes_pred_stable(image)
py
```


<!--
We may now check if the prediction is correct.
-->

Bây giờ ta có thể kiểm tra liệu dự đoán này có đúng hay không.


```{.python .input}
# Convert label which is a scalar tensor of int32 dtype
# to a Python scalar integer for comparison
py.argmax(axis=0) == int(label)
```

```{.python .input}
#@tab pytorch
py.argmax(dim=0) == label
```

```{.python .input}
#@tab tensorflow
tf.argmax(py, axis=0) == label
```


<!--
If we now predict a few validation examples, we can see the Bayes
classifier works pretty well.
-->

Nếu dự đoán một vài mẫu kiểm định, ta có thể thấy 
bộ phân loại Bayes hoạt động khá tốt.


```{.python .input}
def predict(X):
    return [bayes_pred_stable(x).argmax(axis=0).astype(np.int32) for x in X]

X, y = mnist_test[:18]
preds = predict(X)
d2l.show_images(X, 2, 9, titles=[str(d) for d in preds]);
```

```{.python .input}
#@tab pytorch
def predict(X):
    return [bayes_pred_stable(x).argmax(dim=0).type(torch.int32).item() 
            for x in X]

X = torch.stack([mnist_train[i][0] for i in range(10,38)], dim=1).squeeze(0)
y = torch.tensor([mnist_train[i][1] for i in range(10,38)])
preds = predict(X)
d2l.show_images(X, 2, 9, titles=[str(d) for d in preds]);
```

```{.python .input}
#@tab tensorflow
def predict(X):
    return [tf.cast(tf.argmax(bayes_pred_stable(x), axis=0), tf.int32).numpy()
            for x in X]

X = tf.stack(
    [tf.cast(train_images[i], tf.float32) for i in range(10, 38)], axis=0)
y = tf.constant([train_labels[i] for i in range(10, 38)])
preds = predict(X)
# TODO: The preds are not correct due to issues with bayes_pred_stable()
d2l.show_images(X, 2, 9, titles=[str(d) for d in preds]);
```


<!--
Finally, let us compute the overall accuracy of the classifier.
-->

Cuối cùng, hãy cùng tính toán độ chính xác tổng thể của bộ phân loại.


```{.python .input}
X, y = mnist_test[:]
preds = np.array(predict(X), dtype=np.int32)
float((preds == y).sum()) / len(y)  # Validation accuracy
```

```{.python .input}
#@tab pytorch
X = torch.stack([mnist_train[i][0] for i in range(len(mnist_test))], 
                dim=1).squeeze(0)
y = torch.tensor([mnist_train[i][1] for i in range(len(mnist_test))])
preds = torch.tensor(predict(X), dtype=torch.int32)
float((preds == y).sum()) / len(y)  # Validation accuracy
```

```{.python .input}
#@tab tensorflow
X = tf.stack([tf.cast(train_images[i], tf.float32) for i in range(
    len(test_images))], axis=0)
y = tf.constant([train_labels[i] for i in range(len(test_images))])
preds = tf.constant(predict(X), dtype=tf.int32)
# TODO: The accuracy is not correct due to issues with bayes_pred_stable()
tf.reduce_sum(tf.cast(preds == y, tf.float32)) / len(y)  # Validation accuracy
```


<!--
Modern deep networks achieve error rates of less than $0.01$.
The relatively poor performance is due to the incorrect statistical assumptions that we made in our model: 
we assumed that each and every pixel are *independently* generated, depending only on the label.
This is clearly not how humans write digits, and this wrong assumption led to the downfall of our overly naive (Bayes) classifier.
-->

Các mạng sâu hiện đại đạt tỷ lệ lỗi dưới $0,01$.
Hiệu suất tương đối kém ở đây là do các giả định thống kê không chính xác mà ta đã đưa vào trong mô hình:
ta đã giả định rằng mỗi và mọi pixel được tạo *một cách độc lập*, chỉ phụ thuộc vào nhãn.
Đây rõ ràng không phải là cách con người viết các chữ số, và giả định sai lầm này đã dẫn đến sự kém hiệu quả của bộ phân loại ngây thơ (*naive* Bayes) của chúng ta.


## Tóm tắt

<!--
* Using Bayes' rule, a classifier can be made by assuming all observed features are independent.  
* This classifier can be trained on a dataset by counting the number of occurrences of combinations of labels and pixel values.
* This classifier was the gold standard for decades for tasks such as spam detection.
-->

* Sử dụng quy tắc Bayes, một bộ phân loại có thể được tạo ra bằng cách giả định tất cả các đặc trưng quan sát được là độc lập.
* Bộ phân loại này có thể được huấn luyện trên tập dữ liệu bằng cách đếm số lần xuất hiện của các tổ hợp nhãn và giá trị điểm ảnh.
* Bộ phân loại này là tiêu chuẩn vàng trong nhiều thập kỷ cho các tác vụ như phát hiện thư rác.


## Bài tập

<!--
1. Consider the dataset $[[0,0], [0,1], [1,0], [1,1]]$ with labels given by the XOR of the two elements $[0,1,1,0]$.
What are the probabilities for a Naive Bayes classifier built on this dataset.
Does it successfully classify our points? If not, what assumptions are violated?
2. Suppose that we did not use Laplace smoothing when estimating probabilities and a data example arrived at testing time which contained a value never observed in training.
What would the model output?
3. The naive Bayes classifier is a specific example of a Bayesian network, where the dependence of random variables are encoded with a graph structure.
While the full theory is beyond the scope of this section (see :cite:`Koller.Friedman.2009` for full details), 
explain why allowing explicit dependence between the two input variables in the XOR model allows for the creation of a successful classifier.
-->

1. Xem xét tập dữ liệu $[[0,0], [0,1], [1,0], [1,1]]$ với các nhãn tương ứng là kết quả phép XOR của cặp số trong mỗi mẫu, tức $[0,1,1,0]$.
Các xác suất cho bộ phân loại Naive Bayes được xây dựng trên tập dữ liệu này là bao nhiêu?
Nó có phân loại thành công các điểm dữ liệu không? Nếu không, những giả định nào bị vi phạm?
2. Giả sử ta không sử dụng làm mượt Laplace khi ước lượng xác suất và có một mẫu dữ liệu tại thời điểm kiểm tra chứa một giá trị chưa bao giờ được quan sát trong quá trình huấn luyện.
Lúc này mô hình sẽ trả về giá trị gì?
3. Bộ phân loại Naive Bayes là một ví dụ cụ thể của mạng Bayes, trong đó sự phụ thuộc của các biến ngẫu nhiên được mã hóa bằng cấu trúc đồ thị.
Mặc dù lý thuyết đầy đủ nằm ngoài phạm vi của phần này (xem :cite:`Koller.Friedman.2009` để biết chi tiết),
hãy giải thích tại sao việc đưa sự phụ thuộc tường minh giữa hai biến đầu vào trong mô hình XOR lại có thể tạo ra một bộ phân loại thành công.


## Thảo luận
* Tiếng Anh: [MXNet](https://discuss.d2l.ai/t/418), [Pytorch](https://discuss.d2l.ai/t/1100), [Tensorflow](https://discuss.d2l.ai/t/1101)
* Tiếng Việt: [Diễn đàn Machine Learning Cơ Bản](https://forum.machinelearningcoban.com/c/d2l)


## Những người thực hiện
Bản dịch trong trang này được thực hiện bởi:

* Đoàn Võ Duy Thanh
* Lê Khắc Hồng Phúc
* Phạm Minh Đức
* Trần Yến Thy
* Phạm Hồng Vinh
* Nguyễn Văn Cường
* Đỗ Trường Giang
* Nguyễn Mai Hoàng Long