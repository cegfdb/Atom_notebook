# ___2020 - 05 - 02 Computer Vision___
***

- [Github openvino](https://github.com/openvinotoolkit/openvino.git)
- [Github Go imaging](https://github.com/disintegration/imaging)

# Basic
## 交互式标注 ginput
  ```py
  from skimage import data
  imm = data.camera()

  plt.imshow(imm, cmap='gray')
  aa = plt.ginput(4)
  aa = np.array(aa)
  plt.scatter(aa[:, 0], aa[:, 1], c='r')
  ```
  ![](images/cv_ginput.png)
## contour
  ```py
  fig, axes = plt.subplots(1, 4, figsize=(12, 3), sharey=False)
  axes[0].contour(imm, origin='image', cmap=plt.cm.gray)
  axes[1].contour(imm, levels=3, origin='lower', cmap=plt.cm.gray)
  axes[2].contour(imm, levels=[127.5], origin='upper', cmap=plt.cm.gray)

  axes[0].contour(imm, origin='image', cmap=plt.cm.gray)
  x = np.arange(1, 10)
  y = x.reshape(-1, 1)
  h = x * y
  cs = plt.contourf(h, levels=[10, 30, 50], colors=['#808080', '#A0A0A0', '#C0C0C0'], extend='both')
  cs.cmap.set_over('red')
  cs.cmap.set_under('blue')
  cs.changed()
  fig.tight_layout()
  ```
  ![](images/cv_contour.png)
## 直方图均衡化 histeq
  ```py
  axis('equal')
  axis('off')

  hist(imm.flatten(), 128)

  from skimage.exposure import equalize_hist
  plt.imshow(equalize_hist(imm), cmap=plt.cm.gray)
  ```
  ```py
  def hist_cumsum(imm, nbr_bins=256):
      imhist, bins = np.histogram(imm.flatten(), nbr_bins)
      cdf = imhist.cumsum() # cumulative distribution function
      cdf = 255 * cdf / cdf[-1] # 归一化
      return cdf, bins

  def histeq(imm, nbr_bins=256):
      """ 对一幅灰度图像进行直方图均衡化"""
      # 计算图像的直方图
      cdf, bins = hist_cumsum(imm, nbr_bins)
      # 使用累积分布函数的线性插值，计算新的像素值
      imm2 = np.interp(imm.flatten(), bins[:-1], cdf).reshape(imm.shape)
      return imm2

  from skimage import data
  from skimage.exposure import equalize_hist

  imm = data.camera()
  iee = histeq(imm)
  iee2 = equalize_hist(imm)

  fig, axes = plt.subplots(1, 4, figsize=(12, 3), sharey=False)
  axes[0].imshow(imm, cmap=plt.cm.gray)
  axes[1].imshow(iee, cmap=plt.cm.gray)
  axes[2].hist(imm.flatten(), 256, facecolor='r', alpha=0.5, label='original')
  axes[2].hist(iee.flatten(), 256, facecolor='g', alpha=0.5, label='histeq')
  axes[2].hist(iee2.flatten(), 256, facecolor='b', alpha=0.5, label='equalize_hist')
  axes[2].legend()
  axes[3].plot(hist_cumsum(imm)[0], label='original')
  axes[3].plot(hist_cumsum(iee)[0], label='histeq')
  axes[3].plot(hist_cumsum(iee2)[0], label='equalize_hist')
  axes[3].legend()
  fig.tight_layout()
  ```
  ![](images/cv_histeq.png)
## png to jpg
  ```py
  from skimage.io import imread, imsave
  import glob2

  for pp in glob2.glob('*.png'):
      print(pp)
      jj = os.path.basename(pp).split('.')[0] + '.jpg'
      # jj = os.path.join('jpg', jj)
      img = imread(pp)
      if img.shape[2] == 4:
          img[img[:, :, 3] == 0] = [255, 255, 255, 255]
      imsave(jj, img[:, :, :3])
  ```
  ```sh
  find ./* -iname '*.png'
  grep -srinI '\.png)'
  sed -i 's/\.png)$/.jpg)/' ./*.md
  ```
## 图像的主成分分析 PCA
  - **主成分分析 PCA** Principal Component Analysis，，在使用尽可能少维数的前提下，尽量多地保持训练数据的信息，PCA 产生的投影矩阵可以被视为将原始坐标变换到现有的坐标系，坐标系中的各个坐标按照重要性递减排列
  - 为了对图像数据进行 PCA 变换，图像需要转换成一维向量表示，将变平的图像堆积起来，可以得到一个矩阵，矩阵的一行表示一幅图像
  - 在计算主方向之前，所有的行图像按照平均图像进行中心化，然后计算协方差矩阵对应最大特征值的特征向量，通常使用 **SVD Singular Value Decomposition 奇异值分解** 方法来计算主成分
  - 当矩阵的维数很大时，SVD 的计算非常慢，所以此时通常不使用 SVD 分解，而使用另一种紧致技巧
  ```py
  def pca(X):
      """ 主成分分析：
          输入：矩阵X ，其中该矩阵中存储训练数据，每一行为一条训练数据
          返回：投影矩阵（按照维度的重要性排序）、方差和均值 """

      # 获取维数
      num_data,dim = X.shape

      # 数据中心化
      mean_X = X.mean(axis=0)
      X = X - mean_X

      if dim > num_data:
          # PCA- 使用紧致技巧
          M = np.dot(X, X.T) # 协方差矩阵
          e,EV = np.linalg.eigh(M) # 特征值和特征向量
          tmp = np.dot(X.T, EV).T # 这就是紧致技巧
          V = tmp[::-1] # 由于最后的特征向量是我们所需要的，所以需要将其逆转
          S = np.sqrt(e)[::-1] # 由于特征值是按照递增顺序排列的，所以需要将其逆转
          for i in range(V.shape[1]):
              V[:,i] /= S
      else:
          # PCA - 使用 SVD 方法
          U, S, V = np.linalg.svd(X)
          V = V[:num_data] # 仅仅返回前nun_data 维的数据才合理

      # 返回投影矩阵、方差和均值
      return V, S, mean_X
  ```
## 图像导数
  - 在很多应用中图像强度的变化情况是非常重要的信息,强度的变化可以用灰度图像 I 的 x 和 y 方向导数 Ix 和 Iy 进行描述
    ```py
    ∇I = [Ix, Iy]T
    ```
  - 梯度有两个重要的属性
    - **梯度的大小**，描述了图像强度变化的强弱
    - **梯度的角度** ``α=arctan2(Iy, Ix)``，描述了图像中在每个像素上强度变化最大的方向
  - **导数计算**
    - 可以用离散近似的方式来计算图像的导数，图像导数大多数可以通过卷积简单地实现
    - 在导数图像中，正导数显示为亮的像素，负导数显示为暗的像素，灰色区域表示导数的值接近于零
    ```py
    Ix = I * Dx
    Iy = I * Dy
    ```
    对于 Dx 和 Dy，通常选择 **Prewitt 滤波器**
    ```py
    Dx = [[-1, 0, 1], [-1, 0, 1], [-1, 0, 1]]
    Dy = [[-1, -1, -1], [0, 0, 0], [1, 1, 1]]
    ```
    或者 **Sobel 滤波器**
    ```py
    Dx = [[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]]
    Dy = [[-1, -2, -1], [0, 0, 0], [1, 2, 1]]
    ```
    ```py
    from skimage import data
    from skimage.filters import sobel, sobel_h, sobel_v

    imm = data.camera()
    imx = sobel_h(imm)  # x 方向导数
    imy = sobel_v(imm)  # y 方向导数

    magnitude = np.sqrt(imx ** 2 + imy ** 2)
    edges = sobel(imm)

    fig, axes = plt.subplots(1, 4, figsize=(12, 3), sharey=True)
    axes[0].imshow(imx, cmap=plt.cm.gray)
    axes[1].imshow(imy, cmap=plt.cm.gray)
    axes[2].imshow(magnitude, cmap=plt.cm.gray)
    axes[3].imshow(edges, cmap=plt.cm.gray)
    fig.tight_layout()
    ```
    ![](images/cv_sobel.png)
## 四点变换
  ```py
  import numpy as np
  import argparse
  import cv2

  def order_points(pts):
      rect = np.zeros((4, 2), dtype = "float32")
      s = pts.sum(axis = 1)
      rect[0] = pts[np.argmin(s)]
      rect[2] = pts[np.argmax(s)]

      diff = np.diff(pts, axis = 1)
      rect[1] = pts[np.argmin(diff)]
      rect[3] = pts[np.argmax(diff)]

      return rect

  def four_point_transform(image, pts):
      rect = order_points(pts)
      (tl, tr, br, bl) = rect

      widthA = np.sqrt(((br[0] - bl[0]) ** 2) + ((br[1] - bl[1]) ** 2))
      widthB = np.sqrt(((tr[0] - tl[0]) ** 2) + ((tr[1] - tl[1]) ** 2))
      maxWidth = max(int(widthA), int(widthB))

      heightA = np.sqrt(((tr[0] - br[0]) ** 2) + ((tr[1] - br[1]) ** 2))
      heightB = np.sqrt(((tl[0] - bl[0]) ** 2) + ((tl[1] - bl[1]) ** 2))
      maxHeight = max(int(heightA), int(heightB))

      dst = np.array([
              [0, 0],
              [maxWidth - 1, 0],
              [maxWidth - 1, maxHeight - 1],
              [0, maxHeight - 1]], dtype = "float32")

      M = cv2.getPerspectiveTransform(rect, dst)
      warped = cv2.warpPerspective(image, M, (maxWidth, maxHeight))

      return warped

  def image_warp_show(img_path, pts=None):
      fig, axes = plt.subplots(1, 2, figsize=(10, 5))
      imm = imread(img_path)
      axes[0].imshow(imm)
      if pts is None:
          pts = np.array(plt.ginput(4))
      axes[0].scatter(pts[:, 0], pts[:, 1])
      ww = four_point_transform(imm, pts)
      axes[1].imshow(ww)
      axes[0].axis('off')
      axes[1].axis('off')
      fig.tight_layout()
      return fig, axes, pts

  fig, axes, pts = image_warp_show('images/example_02.png')
  ```
  ![](images/cv_4_points_trans.png)
## inpaint
  ```py
  import numpy as np
  import matplotlib.pyplot as plt

  from skimage import data
  from skimage.restoration import inpaint

  image_orig = data.astronaut()[0:200, 0:200]

  # Create mask with three defect regions: left, middle, right respectively
  mask = np.zeros(image_orig.shape[:-1])
  mask[20:60, 0:20] = 1
  mask[160:180, 70:155] = 1
  mask[30:60, 170:195] = 1

  # Defect image over the same region in each color channel
  image_defect = image_orig.copy()
  for layer in range(image_defect.shape[-1]):
      image_defect[np.where(mask)] = 0

  image_result = inpaint.inpaint_biharmonic(image_defect, mask, multichannel=True)

  def ax_show_gray(ax, imm, title):
      ax.imshow(imm)
      ax.set_title(title)
      ax.set_axis_off()

  fig, axes = plt.subplots(ncols=4, nrows=1)
  ax_show_gray(axes[0], image_orig, 'Original image')
  ax_show_gray(axes[1], mask, 'Mask')
  ax_show_gray(axes[2], image_defect, 'Defected image')
  ax_show_gray(axes[3], image_result, 'Inpainted image')
  fig.tight_layout()
  ```
  ![](images/cv_inpaint.png)
## Video test function
  ```py
  import cv2

  def video_test(func=None, src=0, title="Opencv"):
      cap = cv2.VideoCapture(src)
      while True:
          grabbed, frame = cap.read()
          if grabbed != True:
              break
          if func != None:
              frame = func(frame)
          cv2.imshow(title, frame)
          key = cv2.waitKey(1) & 0xFF
          if key == ord("q"):
              break
      cap.release()
      cv2.destroyAllWindows()

  video_test(func=lambda frame: cv2.Canny(frame, 50, 200))
  ```
***

# 图像去噪
## 基本概念
  - **ROF (Rudin-Osher-Fatemi) 去噪模型** 图像去噪对于很多应用来说都非常重要，ROF 模型具有很好的性质，使处理后的图像更平滑，同时保持图像边缘和结构信息
  - **TV (Total Variation) 全变差** 定义为一幅灰度图像 I 的梯度范数之和，是一幅图像梯度的 L1 范数
    - 在 **连续** 表示的情况下，全变差表示为 `J(I) = ∫|∇I|dx`
    - 在 **离散** 表示的情况下，全变差表示为 `J(I) = ∑(x)|∇I|`
    - 滤波的结果是使得图像的 **全变差范数** 最小，同时尽可能接近原始图像
  - 在 ROF 模型里，**目标函数** 为寻找降噪后的图像 U，使下式最小
    ```py
    min(u)||I - U|| ^ 2 + 2λJ(U)
    ```
    其中范数 `||I-U||` 是去噪后图像 U 和原始图像 I 差异的度量，本质上该模型使去噪后的图像像素值 **平坦变化**，但是在图像区域的边缘上，允许去噪后的图像像素值 **跳跃变化**
  - **小波滤波器 Wavelet denoising filter** 根据图像的小波表示，将将小波域中的小值定义为噪声，并设置为 0，对于彩色图像通常是在 `YCbCr` 颜色空间中进行，因为在单独的颜色通道中去噪可能会导致更明显的噪声
  - **双边滤波 Bilateral filter**
    - 进行边缘保留滤波通常用到两个方法，**高斯双边滤波** 和 **均值迁移滤波**
    - 是一种边缘保留的噪声抑制滤波器，根据像素的 **空间邻近度 spatial closeness** 和 **辐射相似性 radiometric similarity** 对像素进行平均，同时考虑 **空域信息** 和 **灰度相似性**，达到 **保边去噪** 的目的
    - 双边滤波器比高斯滤波多了一个 **高斯方差**，是基于空间分布的高斯滤波函数
    - 在边缘附近，离的较远的像素不会太多影响到边缘上的像素值，这样就保证了边缘附近像素值的保存
    - 由于保存了过多的高频信息，对于彩色图像里的高频噪声，双边滤波器不能够干净的滤掉，只能够对于低频信息进行较好的滤波
## Skimage 示例
  ```py
  import numpy as np
  from skimage.util import random_noise
  from skimage import data, img_as_float, color
  from skimage.restoration import denoise_tv_chambolle, denoise_bilateral, denoise_wavelet, estimate_sigma

  original = img_as_float(data.chelsea()[100:250, 50:300])
  sigma = 0.155
  noisy = random_noise(original, var=sigma**2)

  fig, ax = plt.subplots(nrows=2, ncols=4, figsize=(8, 5), sharex=True, sharey=True, subplot_kw={'adjustable': 'box'})
  plt.gray()

  # Estimate the average noise standard deviation across color channels.
  sigma_est = estimate_sigma(noisy, multichannel=True, average_sigmas=True)
  # Due to clipping in random_noise, the estimate will be a bit smaller than the specified sigma.
  print("Estimated Gaussian noise standard deviation = {}".format(sigma_est))
  # Estimated Gaussian noise standard deviation = 0.14960113756936033

  def ax_show_gray(ax, imm, title):
      ax.imshow(imm)
      ax.set_title(title)
      ax.set_axis_off()

  ax_show_gray(ax[0, 0], noisy, 'Noisy')
  ax_show_gray(ax[0, 1], denoise_tv_chambolle(noisy, weight=0.1, multichannel=True), 'TV, weight=0.1')
  ax_show_gray(ax[0, 2], denoise_bilateral(noisy, sigma_color=0.05, sigma_spatial=15, multichannel=True), 'Bilateral, sigma_color=0.05')
  ax_show_gray(ax[0, 3], denoise_wavelet(noisy, multichannel=True, rescale_sigma=True), 'Wavelet denoising RGB')

  ax_show_gray(ax[1, 1], denoise_tv_chambolle(noisy, weight=0.2, multichannel=True), 'TV, weight=0.2')
  ax_show_gray(ax[1, 2], denoise_bilateral(noisy, sigma_color=0.1, sigma_spatial=15, multichannel=True), 'Bilateral, sigma_color=0.1')
  ax_show_gray(ax[1, 3], denoise_wavelet(noisy, multichannel=True, convert2ycbcr=True, rescale_sigma=True), 'Wavelet denoising YCbCr')
  ax_show_gray(ax[1, 0], original, 'Original')
  fig.tight_layout()
  ```
  ![](images/cv_skimage_denoise.png)
## Opencv 示例
  - **cv2.bilateralFilter**
    ```py
    bilateralFilter(src, d, sigmaColor, sigmaSpace[, dst[, borderType]]) -> dst
    ```
    - **src** 输入图像
    - **d** 在过滤期间使用的每个像素邻域的直径，如果 `d` 非 0，则 `sigmaSpace` 由 `d` 计算得出，如果 `sigmaColor` 没输入，则 `sigmaColor` 由 `sigmaSpace` 计算得出，通常考虑计算速度情况下，设置为 `5`
    - **sigmaColor** 色彩空间的标准方差，一般尽可能大，较大的参数值意味着像素邻域内较远的颜色会混合在一起， 从而产生更大面积的半相等颜色
    - **sigmaSpace** 坐标空间的标准方差，一般尽可能小，参数值越大意味着只要它们的颜色足够接近，越远的像素都会相互影响，当 `d > 0` 时，指定邻域大小而不考虑 `sigmaSpace`，否则 `d` 与 `sigmaSpace` 成正比
    ```py
    ''' 双边滤波 '''
    import cv2
    import numpy as np
    from skimage.util import random_noise
    from skimage import data, img_as_float, color

    original = img_as_float(data.chelsea()[100:250, 50:300])
    sigma = 0.155
    noisy = random_noise(original, var=sigma**2)

    ss = (noisy * 255).astype('uint8')
    dst_bi = cv2.bilateralFilter(ss, 0, 100, 5)
    dst_shift = cv2.pyrMeanShiftFiltering(ss, 10, 50)

    plt.imshow(np.hstack([ss, dst_bi, dst_shift]))
    plt.axis('off')
    plt.tight_layout()
    ```
    ![](images/cv_cv2_denoise.png)
## 自定义 tv denoise
  - **python 实现**
    ```py
    ''' np.roll 循环滚动数组中的元素值 '''
    aa = np.arange(10).reshape(2, 5)
    np.roll(aa, 2)
    # array([[8, 9, 0, 1, 2], [3, 4, 5, 6, 7]])
    np.roll(aa, -2)
    # array([[2, 3, 4, 5, 6], [7, 8, 9, 0, 1]])
    np.roll(aa, 1, axis=0)
    # array([[5, 6, 7, 8, 9], [0, 1, 2, 3, 4]])
    np.roll(aa, 1, axis=1)
    # array([[4, 0, 1, 2, 3], [9, 5, 6, 7, 8]])

    ''' 计算图像导数，可以维持导数图像大小 '''
    bb = np.arange(12).reshape(3, 4)
    np.roll(bb, -1, 0) - bb # 0 轴导数
    np.roll(bb, -1, 1) - bb # 1 轴导数
    ```
    **自定义实现 denoise**
    ```py
    import numpy as np

    def denoise(imm, U_init, tolerance=0.1, tau=0.125, tv_weight=100):
        """ 使用A. Chambolle（2005）在公式（11）中的计算步骤实现 Rudin-Osher-Fatemi（ROF）去噪模型
            输入：im 灰度图像，U_init U 的初始值，tolerance 停止条件，tau TV 正则项权值，tv_weight TV 步长
            输出：去噪和去除纹理后的图像，纹理残留
        """
        ww, hh = imm.shape # 噪声图像的大小
        Px, Py = imm, imm # 对偶域的 x y 分量
        U = U_init
        error = 1
        error_scale = np.sqrt(ww * hh)

        while (error > tolerance):
            U_pre = U

            # 原始变量的梯度
            GradUx = np.roll(U, -1, axis=1) - U # 变量U 梯度的 x 分量
            GradUy = np.roll(U, -1, axis=0) - U # 变量U 梯度的 y 分量

            # 更新对偶变量
            PxNew = Px + (tau / tv_weight) * GradUx
            PyNew = Py + (tau / tv_weight) * GradUy
            NormNew = np.maximum(1, np.sqrt(PxNew ** 2 + PyNew ** 2))

            Px = PxNew / NormNew # 更新x 分量（对偶）
            Py = PyNew / NormNew # 更新y 分量（对偶）

            # 更新原始变量
            RxPx = np.roll(Px, 1, axis=1) # 对x 分量进行向右x 轴平移
            RyPy = np.roll(Py, 1, axis=0) # 对y 分量进行向右y 轴平移

            DivP = (Px - RxPx) + (Py - RyPy) # 对偶域的散度
            U = imm + tv_weight * DivP # 更新原始变量

            # 更新误差
            error = np.linalg.norm(U - U_pre) / error_scale

        return U, imm - U # 去噪后的图像和纹理残余
    ```
    **合成的噪声图像测试**
    ```py
    from scipy.ndimage import filters

    # 使用噪声创建合成图像
    imm = zeros((500, 500))
    imm[100:400, 100:400] = 128
    imm[200:300, 200:300] = 255
    imm = imm + 30 * np.random.standard_normal((500, 500))

    U, T = denoise(imm, imm)
    G = filters.gaussian_filter(imm, 10)

    def ax_show_gray(ax, imm, title):
        ax.imshow(imm, cmap='gray')
        ax.set_title(title)
        ax.set_axis_off()

    fig, axes = plt.subplots(1, 4, figsize=(20, 5), sharey=False)
    ax_show_gray(axes[0], imm, "original")
    ax_show_gray(axes[1], U, "denoise")
    ax_show_gray(axes[2], T, "deleted noise")
    ax_show_gray(axes[3], G, "gaussian_filter")
    fig.tight_layout()
    ```
    ![](images/cv_denoise.png)
***

# Opencv 移动目标检测
## 根据第一帧图片检测图像变化
  ```py
  class DeltaDetect:
      def __init__(self, min_area=500):
          self.min_area = min_area         
          self.init_frame = None

      def __call__(self, frame):
          gray_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
          gray_frame = cv2.GaussianBlur(gray_frame, (21, 21), 0)
          if self.init_frame is None:
              self.init_frame = gray_frame
              return np.hstack([frame, np.zeros_like(frame), np.zeros_like(frame)])

          frame_delta = cv2.absdiff(self.init_frame, gray_frame)
          thresh = cv2.threshold(frame_delta, 25, 255, cv2.THRESH_BINARY)[1]
          thresh = cv2.dilate(thresh, None, iterations=2)

          cnts, _hierarchy = cv2.findContours(thresh.copy(), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
          bboxes = [cv2.boundingRect(cc) for cc in cnts if cv2.contourArea(cc) > self.min_area]

          for bb in bboxes:
              x, y, w, h = bb
              cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)
          text = "Unoccupied" if len(bboxes) == 0 else "Occupied"
          cv2.putText(frame, "Room Status: {}".format(text), (10, 20), cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 0, 255), 2)
          cv2.putText(frame, datetime.datetime.now().strftime("%A %d %B %Y %I:%M:%S%p"), (10, frame.shape[0] - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.35, (0, 0, 255), 1)

          thresh = cv2.cvtColor(thresh, cv2.COLOR_GRAY2BGR)
          frame_delta = cv2.cvtColor(frame_delta, cv2.COLOR_GRAY2BGR)

          return np.hstack([frame, thresh, frame_delta])

  video_test(func=DeltaDetect())
  ```
## 背景减除进行移动目标检测
  - **KNN 算法**
  - **MOG / MOG2**
    - 是一种基于高斯混合的背景 / 前景分割算法，通过混合 **K 高斯分布 (K = 3~5)** 对每个背景像素进行建模
    - 混合的权重表示这些颜色在场景中停留的 **时间比例**，可能的背景色是那些保持更长时间和更静态的颜色
    - **MOG2** 为每个像素选择适当的高斯分布，对于光照变化等的变化场景有更好的适应性
    - `cv2.createBackgroundSubtractorMOG2` 用于创建背景对象，参数 `detectShadows` 指定是否检测阴影，默认为 `True`，将检测并标记阴影，但会降低速度。 阴影将标记为灰色
  - **GMG**
    - 该算法结合 **统计学图像背景估计 statistical background image estimation** 和 **每像素贝叶斯分割 per-pixel Bayesian segmentation**
    - 该算法默认使用前 120 帧图像进行背景建模，采用概率前景分割算法，通过贝叶斯推理识别可能的前景对象，在前几帧图像中将会得到一个黑色窗口
    - 估计值是自适应的，新的观测值比旧的观测值权重更大，以适应可变光照
    - 应用了多种形态滤波操作，如 closing / opening，以去除不需要的噪声
    - 对于生成的结果，应用 opening 操作再去除以下噪声效果会更好
  - **Python 背景减除示例**
    ```py
    class BackgroundTest:
        def __init__(self, method='mog'):
            self.kernel = cv2.getStructuringElement(cv2.MORPH_OPEN, (5, 5))
            if method.lower == 'knn':
                self.fgbg = cv2.createBackgroundSubtractorKNN(detectShadows=True)
            else:
                self.fgbg = cv2.createBackgroundSubtractorMOG2(detectShadows=True)

        def __call__(self, frame):
            fgmask = self.fgbg.apply(frame)
            fgmask = cv2.morphologyEx(fgmask, cv2.MORPH_OPEN, self.kernel)
            return fgmask

    video_test(func=BackgroundTest(), src=os.path.expanduser('~/workspace/PyImageSearch/social-distance-detector/pedestrians.mp4'))
    ```
  - **根据前景面积检测运动物体**
    ```py
    class MotionDetect:
        def __init__(self, history=20, min_area=500, max_area=3000):
            self.history, self.min_area, self.max_area  = history, min_area, max_area    # 训练帧数
            self.bs = cv2.createBackgroundSubtractorKNN(detectShadows=True)  # 背景减除器，设置阴影检测
            self.bs.setHistory(self.history)
            self.kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (3, 3))
            self.frames = 0

        def __call__(self, frame):
            fg_mask = self.bs.apply(frame) # 获取 foreground mask
            fg_mask = cv2.morphologyEx(fg_mask, cv2.MORPH_ELLIPSE, self.kernel)

            if self.frames < self.history:
                self.frames += 1
                return np.hstack([frame, np.zeros_like(frame)])
            th = cv2.threshold(fg_mask.copy(), 244, 255, cv2.THRESH_BINARY)[1]
            th = cv2.erode(th, cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (3, 3)), iterations=2)
            dilated = cv2.dilate(th, cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (8, 3)), iterations=2)
            # 获取所有检测框
            contours, hier = cv2.findContours(dilated, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
            for c in contours:
                # 获取矩形框边界坐标
                x, y, w, h = cv2.boundingRect(c)
                # 计算矩形框的面积
                area = cv2.contourArea(c)
                if self.min_area < area < self.max_area:
                    cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)
            dilated = cv2.cvtColor(dilated, cv2.COLOR_GRAY2BGR)
            return np.hstack([frame, dilated])

    video_test(func=MotionDetect())
    ```
## YOLO 行人检测与社交距离检测
  ```py
  def yolo_detect_object(frame, net, out, obj_idx, min_conf=0.3, nms_thresh=0.3):
      hh, ww = frame.shape[:2]
      blob = cv2.dnn.blobFromImage(frame, 1 / 255.0, (416, 416), swapRB=True, crop=False)
      net.setInput(blob)
      layerOutputs = net.forward(out)
      boxes, centroids, confidences, results = [], [], [], []

      for output in layerOutputs:
          for detection in output:
              # extract the class ID and confidence (i.e., probability) of the current object detection
              scores = detection[5:]
              classID = np.argmax(scores)
              confidence = scores[classID]
              if classID == obj_idx and confidence > min_conf:
                  box = detection[0:4] * np.array([ww, hh, ww, hh])
                  centerX, centerY, width, height = box.astype("int")
                  xx = int(centerX - (width / 2))
                  yy = int(centerY - (height / 2))

                  boxes.append([xx, yy, int(width), int(height)])
                  centroids.append((centerX, centerY))
                  confidences.append(float(confidence))
      # apply non-maxima suppression to suppress weak, overlapping bounding boxes
      idxs = cv2.dnn.NMSBoxes(boxes, confidences, min_conf, nms_thresh)
      return [(confidences[ii[0]], boxes[ii[0]], centroids[ii[0]]) for ii in idxs]

  class SocialDistance:
      def __init__(self, model_path="yolo-coco", min_distance=50, resize_width=700, use_gpu=True):
          self.min_distance, self.resize_width = min_distance, resize_width
          config_path = os.path.join(model_path, 'yolov3.cfg')
          weights_path = os.path.join(model_path, 'yolov3.weights')
          labels_path = os.path.join(model_path, 'coco.names')

          with open(labels_path, 'r') as ff:
              LABELS = ff.read().strip().split("\n")
          personIdx = LABELS.index("person")

          net = cv2.dnn.readNetFromDarknet(config_path, weights_path)
          if use_gpu:
              net.setPreferableBackend(cv2.dnn.DNN_BACKEND_CUDA)
              net.setPreferableTarget(cv2.dnn.DNN_TARGET_CUDA)
          ln = net.getLayerNames()
          out = [ln[ii[0] - 1] for ii in net.getUnconnectedOutLayers()]

          self.person_detector = lambda frame: yolo_detect_object(frame, net, out, personIdx)

      def __call__(self, frame):
          hh, ww = frame.shape[:2]
          resize_height = int(hh / ww * self.resize_width)
          frame = cv2.resize(frame, (self.resize_width, resize_height))
          results = self.person_detector(frame)
          violate = set()
          if len(results) >= 2:
              centroids = np.array([ii[2] for ii in results])
              for id, ii in enumerate(centroids[:-1]):
                  dd = np.sqrt(((ii - centroids[id + 1:]) ** 2).sum(1)).tolist()
                  for jd, jj in enumerate(dd):
                      if jj < self.min_distance:
                          violate.add(id)
                          violate.add(id + jd + 1)

                  for (id, (prob, bbox, centroid)) in enumerate(results):
                      startX, startY, ww, hh = bbox
                      cX, cY = centroid
                      color = (0, 0, 255) if id in violate else (0, 255, 0)
                      cv2.rectangle(frame, (startX, startY), (startX + ww, startY + hh), color, 2)
                      cv2.circle(frame, (cX, cY), 5, color, 1)

                  text = "Social Distancing Violations: {}".format(len(violate))
                  cv2.putText(frame, text, (10, frame.shape[0] - 25), cv2.FONT_HERSHEY_SIMPLEX, 0.85, (0, 0, 255), 3)
          return frame

  video_path = os.path.expanduser('~/workspace/PyImageSearch/social-distance-detector/pedestrians.mp4')
  video_test(func=SocialDistance(), src=video_path)
  ```
## 目标检测更新背景
  ```py
  import insightface

  class DeltaDetect:
      def __init__(self, lines_plot=""):
          self.det = insightface.model_zoo.face_detection.retinaface_mnet025_v1()
          self.det.prepare(-1)      
          self.init_frame = None
          self.lines_plot = lines_plot.lower()
          self.max_dist_h = lambda lines: max([abs(ii[0] - ii[2]) for ii in lines.reshape(-1, 4) if abs(ii[0] - ii[2]) > abs(ii[1] - ii[3])] + [0])
          self.max_dist_v = lambda lines: max([abs(ii[1] - ii[3]) for ii in lines.reshape(-1, 4) if abs(ii[1] - ii[3]) > abs(ii[0] - ii[2])] + [0])

      def __call__(self, frame):
          bbs, pps = self.det.detect(frame)
          gray_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
          gray_frame = cv2.GaussianBlur(gray_frame, (3, 3), 0)
          bbs = bbs[:, :4].astype(int)
          if bbs.shape[0] == 0:
              self.init_frame = gray_frame
              return frame
          if self.init_frame is None:
              self.init_frame = np.zeros_like(gray_frame)

          frame_delta = cv2.absdiff(self.init_frame, gray_frame)
          mask = cv2.threshold(frame_delta, 50, 255, cv2.THRESH_BINARY)[1]
          mask = cv2.dilate(mask, cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (11, 11)), iterations=5) // 255
          canny_dst = cv2.Canny(gray_frame, 100, 200, apertureSize=3)
          lines = cv2.HoughLinesP(canny_dst * mask, 1, np.pi / 180.0, 40, np.array([]), 50, 20)
          lines = [] if lines is None else lines

          lines_sort_x, lines_sort_y = [], []
          for ii in lines:
              x1, y1, x2, y2 = ii[0][0], ii[0][1], ii[0][2], ii[0][3]
              vh_dd = abs(y2 - y1) - abs(x2 - x1)
              if vh_dd <= 0:
                  # it's a horizontal line， used to detect left / right border
                  lines_sort_y.append([y1, x1, y2, x2] if y1 < y2 else [y2, x2, y1, x1])  
              if vh_dd >= 0:
                  # it's a vertical line, used to detect top / bottom border
                  lines_sort_x.append([x1, y1, x2, y2] if x1 < x2 else [x2, y2, x1, y1])

          scores = []
          for bb in bbs:
              left, top, right, bottom = bb[:4].astype('int')
              score_left, score_right = self._score_func(lines_sort_x, left, right, top, bottom)
              score_top, score_bottom = self._score_func(lines_sort_y, top, bottom, left, right)
              scores.append(sum([score_left ** 2, score_right ** 2, score_top ** 2, score_bottom ** 2]))

              print(score_left, score_right, score_top, score_bottom)
              cv2.rectangle(frame, (left, top), (right, bottom), (0, 255, 0), 2)
              cv2.putText(frame, "{:.4f}".format(scores[-1]), (bb[0], bb[1]), cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 0, 255), 2)
          mask = np.expand_dims(mask, -1)
          mask = np.concatenate([mask, mask, mask], -1)
          frame = np.where(mask, frame, 0)

          if self.lines_plot == "left":
              lines = [ii for ii in lines_sort_x if ii[0] < left]
          elif self.lines_plot == "right":
              lines = [ii for ii in lines_sort_x if ii[2] > right]
          elif self.lines_plot == "top":
              lines = [[ii[1], ii[0], ii[3], ii[2]] for ii in lines_sort_y if ii[0] < top]
          elif self.lines_plot == "bottom":
              lines = [[ii[1], ii[0], ii[3], ii[2]] for ii in lines_sort_y if ii[2] > bottom]
          else:
              lines = [ii[0] for ii in lines]
          for ii in lines:
              # cv2.line(frame, (ii[0][0], ii[0][1]), (ii[0][2], ii[0][3]), (0, 0, 255), 3, cv2.LINE_AA)
              cv2.line(frame, (ii[0], ii[1]), (ii[2], ii[3]), (0, 0, 255), 3, cv2.LINE_AA)

          return frame

      def _score_func(self, lines_sort, left, right, top, bottom):
          score_left, score_right = [0.], [0.]
          for x1, y1, x2, y2 in lines_sort:
              y_min, y_max = min(y2, y1), max(y2, y1)
              y_dd = min(y_max, bottom) - max(top, y_min)
              if x1 < left and x2 < right:
                  # vertical line on the left of left border
                  score_left.append(y_dd)
              elif x2 > right and x1 > left:
                  # vertical line on the right of right border
                  score_right.append(y_dd)
          return max(score_left) / (bottom - top), max(score_right) / (bottom - top)

  video_test(func=DeltaDetect())
  ```
***

# Opencv 应用
## Square
  ```py
  import numpy as np
  import cv2

  def angle_cos(p0, p1, p2):
      d1, d2 = (p0 - p1).astype('float'), (p2 - p1).astype('float')
      return abs(np.dot(d1, d2) / np.sqrt(np.dot(d1, d1) * np.dot(d2, d2)))

  def find_squares(img):
      img = cv2.GaussianBlur(img, (5, 5), 0)
      squares = []
      for gray in cv2.split(img):
          for thrs in range(0, 255, 26):
              if thrs == 0:
                  bin = cv2.Canny(gray, 0, 50, apertureSize=5)
                  bin = cv2.dilate(bin, None)
              else:
                  _retval, bin = cv2.threshold(gray, thrs, 255, cv2.THRESH_BINARY)
              contours, _hierarchy = cv2.findContours(bin, cv2.RETR_LIST, cv2.CHAIN_APPROX_SIMPLE)
              for cnt in contours:
                  cnt_len = cv2.arcLength(cnt, True)
                  cnt = cv2.approxPolyDP(cnt, 0.02*cnt_len, True)
                  if len(cnt) == 4 and cv2.contourArea(cnt) > 1000 and cv2.isContourConvex(cnt):
                      cnt = cnt.reshape(-1, 2)
                      max_cos = np.max([angle_cos( cnt[i], cnt[(i+1) % 4], cnt[(i+2) % 4] ) for i in range(4)])
                      if max_cos < 0.1:
                          squares.append(cnt)
      return squares

  def square_test_func(frame):
      squares = find_squares(frame)
      cv2.drawContours(frame, squares, -1, (0, 255, 0), 3)
      return frame

  video_test(square_test_func, 0)
  ```
## Hough lines
  ```py
  def hough_line_detect_P(self, frame, mask=None):
      if len(frame.shape) == 3:
          frame = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
          frame = cv2.GaussianBlur(frame, (3, 3), 0)
      # ret, frame = cv2.threshold(frame, 0, 255, cv2.THRESH_OTSU)
      dst = cv2.Canny(frame, 100, 200, apertureSize=3)
      if mask is not None:
          dst *= mask
      lines = cv2.HoughLinesP(dst, 1, np.pi / 180.0, 40, np.array([]), 50, 20)
      return lines, dst

  def hough_line_detect_P(frame):
      frame = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
      frame = cv2.GaussianBlur(frame, (3, 3), 0)
      dst = cv2.Canny(frame, 100, 200, apertureSize=3)
      cdst = cv2.cvtColor(dst, cv2.COLOR_GRAY2BGR)
      # lines = cv2.HoughLinesP(dst, 1, np.pi / 180.0, 40, np.array([]), 50, 10)
      lines = cv2.HoughLinesP(dst, 1, np.pi / 180.0, 40, np.array([]), 50, 20)
      for ii in lines:
          cv2.line(cdst, (ii[0][0], ii[0][1]), (ii[0][2], ii[0][3]), (0, 0, 255), 3, cv2.LINE_AA)
      return cdst

  def hough_line_detect(frame):
      dst = cv2.Canny(frame, 50, 200)
      cdst = cv2.cvtColor(dst, cv2.COLOR_GRAY2BGR)
      lines = cv2.HoughLines(dst, 1, np.pi/180.0, 50, np.array([]), 0, 0)
      for ii in lines:
          rho = ii[0][0]
          theta = ii[0][1]
          a = np.cos(theta)
          b = np.sin(theta)
          x0, y0 = a * rho, b * rho
          pt1 = (int(x0 + 1000 * (-b)), int(y0 + 1000 * (a)))
          pt2 = (int(x0 - 1000 * (-b)), int(y0 - 1000 * (a)))
          cv2.line(cdst, pt1, pt2, (0, 0, 255), 3, cv2.LINE_AA)
      return cdst

  video_test(func=hough_line_detect_P)
  video_test(func=hough_line_detect)
  ```
  **skimage**
  ```py
  from skimage.transform import hough_line, hough_line_peaks
  from skimage.feature import canny

  def hough_line_detect_S(frame):
      edges = canny(frame, 2, 1, 25)
      # Classic straight-line Hough transform Set a precision of 0.5 degree.
      tested_angles = np.linspace(-np.pi / 2, np.pi / 2, 360)
      h, theta, d = hough_line(frame, theta=tested_angles)
      origin = np.array((0, frame.shape[1]))
      for _, angle, dist in zip(*hough_line_peaks(h, theta, d)):
          y0, y1 = (dist - origin * np.cos(angle)) / np.sin(angle)
          yield (origin, (y0, y1))
  ```
  ```py
  from skimage.draw import line
  img = np.zeros((15, 15), dtype=np.bool_)
  rr, cc = line(0, 0, 14, 14)
  img[rr, cc] = 1
  rr, cc = line(0, 14, 14, 0)
  img[cc, rr] = 1
  hspace, angles, dists = hough_line(img)
  hspace, angles, dists = hough_line_peaks(hspace, angles, dists)
  # [15, 15], [ 0.78101046, -0.78101046], [10.74418605,  0.51162791]

  from skimage.draw import line
  img = np.zeros((15, 15), dtype=np.bool_)
  rr, cc = line(2, 4, 10, 12)
  img[rr, cc] = 1
  rr, cc = line(0, 14, 14, 0)
  img[cc, rr] = 1
  hspace, angles, dists = hough_line(img)
  hspace, angles, dists = hough_line_peaks(hspace, angles, dists)
  # [15, 15], [ 0.78101046, -0.78101046], [10.74418605,  0.51162791]
  ```
  ```py
  from skimage.transform import probabilistic_hough_line
  from skimage.feature import canny
  from skimage.color import rgb2gray

  def hough_line_detect_SP(frame):
      # edges = canny(rgb2gray(frame), 2, 1, 25)
      edges = canny(rgb2gray(frame), sigma=2)
      lines = probabilistic_hough_line(edges, threshold=10, line_length=5, line_gap=3)
      for ii in lines:
          cv2.line(frame, ii[0], ii[1], (0, 0, 255), 3, cv2.LINE_AA)
      return frame

  ```
## Optical flow 光流
  ```py
  def draw_flow(img, flow, step=16):
      h, w = img.shape[:2]
      y, x = np.mgrid[step/2:h:step, step/2:w:step].reshape(2,-1).astype(int)
      fx, fy = flow[y, x].T
      lines = np.vstack([x, y, x+fx, y+fy]).T.reshape(-1, 2, 2)
      lines = np.int32(lines + 0.5)
      vis = cv2.cvtColor(img, cv2.COLOR_GRAY2BGR)
      cv2.polylines(vis, lines, 0, (0, 255, 0))
      for (x1, y1), (_x2, _y2) in lines:
          cv2.circle(vis, (x1, y1), 1, (0, 255, 0), -1)
      return vis

  def draw_hsv(flow):
      h, w = flow.shape[:2]
      fx, fy = flow[:, :, 0], flow[:, :, 1]
      ang = np.arctan2(fy, fx) + np.pi
      v = np.sqrt(fx * fx + fy * fy)
      hsv = np.zeros((h, w, 3), np.uint8)
      hsv[..., 0] = ang * (180 / np.pi / 2)
      hsv[..., 1] = 255
      hsv[..., 2] = np.minimum(v * 4, 255)
      bgr = cv2.cvtColor(hsv, cv2.COLOR_HSV2BGR)
      return bgr

  def warp_flow(img, flow):
      h, w = flow.shape[:2]
      flow = -flow
      flow[:, :, 0] += np.arange(w)
      flow[:, :, 1] += np.arange(h)[:, np.newaxis]
      res = cv2.remap(img, flow, None, cv2.INTER_LINEAR)
      # bgr = cv2.cvtColor(res, cv2.COLOR_GRAY2BGR)
      return res

  class OptflowTest:
      def __init__(self):
          self.pre, self.cur_glitch = None, None

      def __call__(self, frame):
          gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
          if self.pre is None:
              self.pre, self.cur_glitch = gray, frame
              return np.hstack([frame, frame, frame])

          flow = cv2.calcOpticalFlowFarneback(gray, self.pre, None, 0.5, 3, 15, 3, 5, 1.2, 0)
          self.pre = gray

          rgb_flow = draw_flow(gray, flow)
          hsv_flow = draw_hsv(flow)
          self.cur_glitch = warp_flow(self.cur_glitch, flow)
          return np.hstack([rgb_flow, hsv_flow, self.cur_glitch])

  video_test(func=OptflowTest())
  ```
## Track 物体跟踪
  ```py
  class ObjectTrack:
      def __init__(self, tracker='kcf', title="Opencv"):
          self.title = title
          OPENCV_OBJECT_TRACKERS = {
              "csrt": cv2.TrackerCSRT_create,
              "kcf": cv2.TrackerKCF_create,
              "boosting": cv2.TrackerBoosting_create,
              "mil": cv2.TrackerMIL_create,
              "tld": cv2.TrackerTLD_create,
              "medianflow": cv2.TrackerMedianFlow_create,
              "mosse": cv2.TrackerMOSSE_create
          }
          self.tracker = OPENCV_OBJECT_TRACKERS[tracker]()
          self.initBB = None

      def __call__(self, frame):
          # if the 's' key is selected, we are going to "select" a bounding box to track
          key = cv2.waitKey(1) & 0xFF
          if key == ord("s"):
              # select the bounding box of the object we want to track (make sure you press ENTER or SPACE after selecting the ROI)
              self.initBB = cv2.selectROI(self.title, frame, fromCenter=False, showCrosshair=True)
              # start OpenCV object tracker using the supplied bounding box coordinates, then start the FPS throughput estimator as well
              self.tracker = self.tracker.create()
              ret = self.tracker.init(frame, self.initBB)
              print("Init tracker:", ret)

          # check to see if we are currently tracking an object
          if self.initBB is not None:
              # grab the new bounding box coordinates of the object
              success, box = self.tracker.update(frame)
              if success:
                  x, y, w, h = [int(v) for v in box]
                  cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)
              cv2.putText(frame, "Success" if success else "Fail", (10, 20), cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 0, 255), 2)

          return frame

  video_test(func=ObjectTrack())
  ```
## QR code
  ```py
  class QrDetect:
      def __init__(self):
          self.qr = cv2.QRCodeDetector()

      def __call__(self, frame):
          ret, points = self.qr.detectMulti(frame)

          if ret:
              for pp in points:
                  cnt = pp.reshape((-1, 1, 2)).astype(np.int32)
                  rows, cols, _ = frame.shape
                  show_radius = 2.813 * ((rows / cols) if rows > cols else (cols / rows))
                  contour_radius = show_radius * 0.4
                  cv2.drawContours(frame, [cnt], 0, (0, 255, 0), int(round(contour_radius)))
                  tpl = cnt.reshape((-1, 2))
                  for x in tuple(tpl.tolist()):
                      color = (255, 0, 0)
                      cv2.circle(frame, tuple(x), int(round(contour_radius)), color, -1)
          return frame

  video_test(func=QrDetect())
  ```
## FFT blur detection
```py
import matplotlib.pyplot as plt
import numpy as np
import cv2

def detect_blur_fft(frame, center_size=60, resize_width=500):
    hh, ww = frame.shape[:2]
    resize_height = int(hh / ww * resize_width)
    frame = cv2.resize(frame, (resize_width, resize_height))
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)

    (h, w) = gray.shape
    (cX, cY) = (int(w / 2.0), int(h / 2.0))

    # compute the FFT to find the frequency transform,
    # then shift the zero frequency component (i.e., DC component located at the top-left corner)
    # to the center where it will be more easy to analyze
    fft = np.fft.fft2(gray)
    fftShift = np.fft.fftshift(fft)

    # zero-out the center of the FFT shift (i.e., remove low frequencies),
    # apply the inverse shift such that the DC component once again becomes the top-left,
    # and then apply the inverse FFT
    fftShift[cY - center_size : cY + center_size, cX - center_size : cX + center_size] = 0
    fftShift = np.fft.ifftshift(fftShift)
    recon = np.fft.ifft2(fftShift)

    # compute the magnitude spectrum of the reconstructed image,
    # then compute the mean of the magnitude values
    magnitude = 20 * np.log(np.abs(recon))
    mean = np.mean(magnitude)
    print(magnitude.min())

    vis_ret = np.hstack([frame, cv2.cvtColor((magnitude + 127.5).astype('uint8'), cv2.COLOR_GRAY2BGR)])
    return mean, vis_ret

def detect_blur_fft_test(frame):
    mean, frame = detect_blur_fft(frame)
    cv2.putText(frame, "FFT Mean: {}".format(mean), (10, frame.shape[0] - 25), cv2.FONT_HERSHEY_SIMPLEX, 0.85, (0, 0, 255), 3)
    return frame

  # check to see if we are visualizing our output
  if vis:
    # compute the magnitude spectrum of the transform
    magnitude = 20 * np.log(np.abs(fftShift))

    # display the original input image
    (fig, ax) = plt.subplots(1, 2, )
    ax[0].imshow(image, cmap="gray")
    ax[0].set_title("Input")
    ax[0].set_xticks([])
    ax[0].set_yticks([])

    # display the magnitude image
    ax[1].imshow(magnitude, cmap="gray")
    ax[1].set_title("Magnitude Spectrum")
    ax[1].set_xticks([])
    ax[1].set_yticks([])

    # show our plots
    plt.show()



  # the image will be considered "blurry" if the mean value of the
  # magnitudes is less than the threshold value
  return (mean, mean <= thresh)
```
```py
# USAGE
# python blur_detector_video.py

# import the necessary packages
from imutils.video import VideoStream
from pyimagesearch.blur_detector import detect_blur_fft
import argparse
import imutils
import time
import cv2

# construct the argument parser and parse the arguments
ap = argparse.ArgumentParser()
ap.add_argument("-t", "--thresh", type=int, default=10,
  help="threshold for our blur detector to fire")
args = vars(ap.parse_args())

# initialize the video stream and allow the camera sensor to warm up
print("[INFO] starting video stream...")
vs = VideoStream(src=0).start()
time.sleep(2.0)

# loop over the frames from the video stream
while True:
  # grab the frame from the threaded video stream and resize it
  # to have a maximum width of 400 pixels
  frame = vs.read()
  frame = imutils.resize(frame, width=500)

  # convert the frame to grayscale and detect blur in it
  gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
  (mean, blurry) = detect_blur_fft(gray, size=60,
    thresh=args["thresh"], vis=False)

  # draw on the frame, indicating whether or not it is blurry
  color = (0, 0, 255) if blurry else (0, 255, 0)
  text = "Blurry ({:.4f})" if blurry else "Not Blurry ({:.4f})"
  text = text.format(mean)
  cv2.putText(frame, text, (10, 25), cv2.FONT_HERSHEY_SIMPLEX,
    0.7, color, 2)

  # show the output frame
  cv2.imshow("Frame", frame)
  key = cv2.waitKey(1) & 0xFF

  # if the `q` key was pressed, break from the loop
  if key == ord("q"):
    break

# do a bit of cleanup
cv2.destroyAllWindows()
vs.stop()
```
***

# skimage segmentation
## Felzenszwalb Quickshift SLIC watershed
  This example compares four popular low-level image segmentation methods. As it is difficult to obtain good segmentations, and the definition of “good” often depends on the application, these methods are usually used for obtaining an oversegmentation, also known as superpixels. These superpixels then serve as a basis for more sophisticated algorithms such as conditional random fields (CRF).

  Felzenszwalb’s efficient graph based segmentation
  This fast 2D image segmentation algorithm, proposed in 1 is popular in the computer vision community. The algorithm has a single scale parameter that influences the segment size. The actual size and number of segments can vary greatly, depending on local contrast.

  1
  Efficient graph-based image segmentation, Felzenszwalb, P.F. and Huttenlocher, D.P. International Journal of Computer Vision, 2004

  Quickshift image segmentation
  Quickshift is a relatively recent 2D image segmentation algorithm, based on an approximation of kernelized mean-shift. Therefore it belongs to the family of local mode-seeking algorithms and is applied to the 5D space consisting of color information and image location 2.

  One of the benefits of quickshift is that it actually computes a hierarchical segmentation on multiple scales simultaneously.

  Quickshift has two main parameters: sigma controls the scale of the local density approximation, max_dist selects a level in the hierarchical segmentation that is produced. There is also a trade-off between distance in color-space and distance in image-space, given by ratio.

  2
  Quick shift and kernel methods for mode seeking, Vedaldi, A. and Soatto, S. European Conference on Computer Vision, 2008

  SLIC - K-Means based image segmentation
  This algorithm simply performs K-means in the 5d space of color information and image location and is therefore closely related to quickshift. As the clustering method is simpler, it is very efficient. It is essential for this algorithm to work in Lab color space to obtain good results. The algorithm quickly gained momentum and is now widely used. See 3 for details. The compactness parameter trades off color-similarity and proximity, as in the case of Quickshift, while n_segments chooses the number of centers for kmeans.

  3
  Radhakrishna Achanta, Appu Shaji, Kevin Smith, Aurelien Lucchi, Pascal Fua, and Sabine Suesstrunk, SLIC Superpixels Compared to State-of-the-art Superpixel Methods, TPAMI, May 2012.

  Compact watershed segmentation of gradient images
  Instead of taking a color image as input, watershed requires a grayscale gradient image, where bright pixels denote a boundary between regions. The algorithm views the image as a landscape, with bright pixels forming high peaks. This landscape is then flooded from the given markers, until separate flood basins meet at the peaks. Each distinct basin then forms a different image segment. 4

  As with SLIC, there is an additional compactness argument that makes it harder for markers to flood faraway pixels. This makes the watershed regions more regularly shaped. 5
  ```py
  from skimage.color import rgb2gray
  from skimage.filters import sobel
  from skimage.segmentation import felzenszwalb, slic, quickshift, watershed
  from skimage.segmentation import mark_boundaries
  from skimage.util import img_as_float
  from skimage.io import imread

  img = img_as_float(imread('./000067.dcm.png'))[:, :, :3]
  segments_fz = felzenszwalb(img, scale=100, sigma=0.5, min_size=50)
  segments_slic = slic(img, n_segments=250, compactness=10, sigma=1)
  segments_quick = quickshift(img, kernel_size=3, max_dist=6, ratio=0.5)
  gradient = sobel(rgb2gray(img))
  segments_watershed = watershed(gradient, markers=250, compactness=0.001)

  print(f"Felzenszwalb number of segments: {len(np.unique(segments_fz))}")
  print(f"SLIC number of segments: {len(np.unique(segments_slic))}")
  print(f"Quickshift number of segments: {len(np.unique(segments_quick))}")

  fig, ax = plt.subplots(2, 2, figsize=(10, 10), sharex=True, sharey=True)

  ax[0, 0].imshow(mark_boundaries(img, segments_fz))
  ax[0, 0].set_title("Felzenszwalbs's method")
  ax[0, 1].imshow(mark_boundaries(img, segments_slic))
  ax[0, 1].set_title('SLIC')
  ax[1, 0].imshow(mark_boundaries(img, segments_quick))
  ax[1, 0].set_title('Quickshift')
  ax[1, 1].imshow(mark_boundaries(img, segments_watershed))
  ax[1, 1].set_title('Compact watershed')

  for a in ax.ravel():
      a.set_axis_off()

  plt.tight_layout()
  plt.show()
  ```
  ![](images/skimage_seg_fsqw.png)
## Join segmentations
  When segmenting an image, you may want to combine multiple alternative segmentations. The skimage.segmentation.join_segmentations() function computes the join of two segmentations, in which a pixel is placed in the same segment if and only if it is in the same segment in both segmentations.
  ```py
  import numpy as np
  import matplotlib.pyplot as plt

  from skimage.filters import sobel
  from skimage.measure import label
  from skimage.segmentation import slic, join_segmentations
  from skimage.morphology import watershed
  from skimage.color import label2rgb, rgb2gray
  from skimage.io import imread

  img = (rgb2gray(imread('./000067.dcm.png')) * 255).astype(np.uint8)

  # Make segmentation using edge-detection and watershed.
  edges = sobel(img)

  # Identify some background and foreground pixels from the intensity values.
  # These pixels are used as seeds for watershed.
  markers = np.zeros_like(img)
  foreground, background = 1, 2
  markers[img < 20] = background
  markers[img > 30] = foreground

  ws = watershed(edges, markers)
  seg1 = label(ws == foreground)

  # Make segmentation using SLIC superpixels.
  seg2 = slic(img, n_segments=117, max_iter=160, sigma=1, compactness=0.75,
              multichannel=False)

  # Combine the two.
  segj = join_segmentations(seg1, seg2)

  # Show the segmentations.
  fig, axes = plt.subplots(ncols=2, nrows=2, figsize=(9, 5),
                           sharex=True, sharey=True)
  ax = axes.ravel()
  ax[0].imshow(img, cmap='gray')
  ax[0].set_title('Image')

  color1 = label2rgb(seg1, image=img, bg_label=0)
  ax[1].imshow(color1)
  ax[1].set_title('Sobel+Watershed')

  color2 = label2rgb(seg2, image=img, image_alpha=0.5)
  ax[2].imshow(color2)
  ax[2].set_title('SLIC superpixels')

  color3 = label2rgb(segj, image=img, image_alpha=0.5)
  ax[3].imshow(color3)
  ax[3].set_title('Join')

  for a in ax:
      a.axis('off')
  fig.tight_layout()
  plt.show()
  ```
  ![](images/skimage_seg_join.png)
## Morphological Snakes
  - **活动轮廓分割 snakes** 使用用户定义的轮廓或线进行初始化，然后该轮廓慢慢收缩

  Morphological Snakes 1 are a family of methods for image segmentation. Their behavior is similar to that of active contours (for example, Geodesic Active Contours 2 or Active Contours without Edges 3). However, Morphological Snakes use morphological operators (such as dilation or erosion) over a binary array instead of solving PDEs over a floating point array, which is the standard approach for active contours. This makes Morphological Snakes faster and numerically more stable than their traditional counterpart.

  There are two Morphological Snakes methods available in this implementation: Morphological Geodesic Active Contours (MorphGAC, implemented in the function morphological_geodesic_active_contour) and Morphological Active Contours without Edges (MorphACWE, implemented in the function morphological_chan_vese).

  MorphGAC is suitable for images with visible contours, even when these contours might be noisy, cluttered, or partially unclear. It requires, however, that the image is preprocessed to highlight the contours. This can be done using the function inverse_gaussian_gradient, although the user might want to define their own version. The quality of the MorphGAC segmentation depends greatly on this preprocessing step.

  On the contrary, MorphACWE works well when the pixel values of the inside and the outside regions of the object to segment have different averages. Unlike MorphGAC, MorphACWE does not require that the contours of the object are well defined, and it works over the original image without any preceding processing. This makes MorphACWE easier to use and tune than MorphGAC.
  ```py
  import numpy as np
  import matplotlib.pyplot as plt
  from skimage import data, img_as_float
  from skimage.io import imread
  from skimage.segmentation import (morphological_chan_vese,
                                    morphological_geodesic_active_contour,
                                    inverse_gaussian_gradient,
                                    checkerboard_level_set)


  def store_evolution_in(lst):
      """Returns a callback function to store the evolution of the level sets in
      the given list.
      """

      def _store(x):
          lst.append(np.copy(x))

      return _store


  # Morphological ACWE\
  image = rgb2gray(imread('./000067.dcm.png'))

  # Initial level set
  init_ls = checkerboard_level_set(image.shape, 6)
  # List with intermediate results for plotting the evolution
  evolution = []
  callback = store_evolution_in(evolution)
  ls = morphological_chan_vese(image, 35, init_level_set=init_ls, smoothing=3,
                               iter_callback=callback)

  fig, axes = plt.subplots(2, 2, figsize=(8, 8))
  ax = axes.flatten()

  ax[0].imshow(image, cmap="gray")
  ax[0].set_axis_off()
  ax[0].contour(ls, [0.5], colors='r')
  ax[0].set_title("Morphological ACWE segmentation", fontsize=12)

  ax[1].imshow(ls, cmap="gray")
  ax[1].set_axis_off()
  contour = ax[1].contour(evolution[2], [0.5], colors='g')
  contour.collections[0].set_label("Iteration 2")
  contour = ax[1].contour(evolution[7], [0.5], colors='y')
  contour.collections[0].set_label("Iteration 7")
  contour = ax[1].contour(evolution[-1], [0.5], colors='r')
  contour.collections[0].set_label("Iteration 35")
  ax[1].legend(loc="upper right")
  title = "Morphological ACWE evolution"
  ax[1].set_title(title, fontsize=12)


  # Morphological GAC
  gimage = inverse_gaussian_gradient(image)

  # Initial level set
  init_ls = np.zeros(image.shape, dtype=np.int8)
  init_ls[10:-10, 10:-10] = 1
  # List with intermediate results for plotting the evolution
  evolution = []
  callback = store_evolution_in(evolution)
  ls = morphological_geodesic_active_contour(gimage, 230, init_ls,
                                             smoothing=1, balloon=-1,
                                             threshold=0.69,
                                             iter_callback=callback)

  ax[2].imshow(image, cmap="gray")
  ax[2].set_axis_off()
  ax[2].contour(ls, [0.5], colors='r')
  ax[2].set_title("Morphological GAC segmentation", fontsize=12)

  ax[3].imshow(ls, cmap="gray")
  ax[3].set_axis_off()
  contour = ax[3].contour(evolution[0], [0.5], colors='g')
  contour.collections[0].set_label("Iteration 0")
  contour = ax[3].contour(evolution[100], [0.5], colors='y')
  contour.collections[0].set_label("Iteration 100")
  contour = ax[3].contour(evolution[-1], [0.5], colors='r')
  contour.collections[0].set_label("Iteration 230")
  ax[3].legend(loc="upper right")
  title = "Morphological GAC evolution"
  ax[3].set_title(title, fontsize=12)

  fig.tight_layout()
  plt.show()
  ```
  ![](images/skiamge_seg_morphological_snakes.png)
***

# CUDA 视频解码
## Opencv with CUDA
  - CUDA needs `gcc-8` / `g++-8`
    ```sh
    sudo apt install gcc-8 g++-8
    sudo rm /etc/alternatives/c++ && sudo ln -s /usr/bin/g++-8 /etc/alternatives/c++
    sudo rm /etc/alternatives/cc && sudo ln -s /usr/bin/gcc-8 /etc/alternatives/cc
    ```
  - NVCUVID needs `nvcuvid.h`, Download [NVIDIA Video Codec SDK](https://developer.nvidia.com/nvidia-video-codec-sdk#Download)
    ```sh
    unzip Video_Codec_SDK_9.1.23.zip
    sudo cp Video_Codec_SDK_9.1.23/include/* /usr/local/include/

    locate libnvcuvid.so
    # /usr/lib/x86_64-linux-gnu/libnvcuvid.so
    locate libnvidia-encode.so
    # /usr/lib/x86_64-linux-gnu/libnvidia-encode.so
    ```
  - Clone opencv_contrib
    ```sh
    git clone https://github.com/opencv/opencv_contrib.git
    ```
  - OpenGL support
    ```sh
    sudo apt install libgtkglext1 libgtkglext1-dev
    ```
  - Cmake and Build opencv
    ```sh
    cd opencv && mkdir build && cd build
    cmake -D CMAKE_BUILD_TYPE=Release -D WITH_CUDA=ON -D WITH_NVCUVID=ON -D ENABLE_FAST_MATH=1 -D CUDA_FAST_MATH=1 -D WITH_CUBLAS=1 -D WITH_OPENGL=ON \
      -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules \
      -D BUILD_OPENCV_PYTHON3=ON \
      -D PYTHON3_EXECUTABLE='/opt/anaconda3/bin/python3.7m' \
      ..

    ls python_loader/cv2/config-3.7.py
    make -j 8 && sudo make install

    pip install -e python_loader  # Or: python python_loader/setup.py install
    python -c 'import cv2; print(cv2.__version__)'
    # 4.3.0-dev
    ```
  - **C++ video reader test**
    ```sh
    cp ../samples/gpu/video_reader.cpp ./
    g++ video_reader.cpp  -L/usr/local/lib -lopencv_core -lopencv_highgui -lopencv_cudacodec -lopencv_videoio
    ./a.out /media/SD/PAD/PAD_video/fake_vis_1_10.avi
    ```
  - **Python test**
    ```py
    import cv2
    # video_path = '/media/SD/PAD/PAD_video/fake_vis_1_10.avi'
    video_path = os.path.expanduser('~/workspace/PyImageSearch/social-distance-detector/pedestrians.mp4')
    mm = cv2.TickMeter()
    crr = cv2.VideoCapture(video_path)
    ctt = []
    while True:
        mm.reset()
        mm.start()
        grabbed, cff = crr.read()
        if grabbed != True:
            break
        mm.stop()
        ctt.append(mm.getTimeMilli())
        cv2.imshow('cpu', cff)

    grr = cv2.cudacodec.createVideoReader(video_path)
    gttr, gttd = [], []
    while True:
        mm.reset()
        mm.start()
        grabbed, gff = grr.nextFrame()
        if grabbed != True:
            break
        mm.stop()
        gttr.append(mm.getTimeMilli())

        mm.reset()
        mm.start()
        nn = gff.download()
        mm.stop()
        gttd.append(mm.getTimeMilli())
        cv2.imshow('gpu', nn)

    gtt = np.add(gttr, gttd)
    print("CPU: %f ms/frame, GPU read: %f ms/frame, GPU download: %f ms/frame, GPU total: %f ms/frame" % (np.mean(ctt[40:]), np.mean(gttr[40:]), np.mean(gttd[40:]), np.mean(gtt[40:])))
    # CPU: 3.505913 ms/frame, GPU read: 0.468353 ms/frame, GPU download: 1.076169 ms/frame, GPU total: 1.544522 ms/frame

    plt.plot(ctt[40:], label='CPU')
    plt.plot(gtt[40:], label='GPU total')
    plt.plot(gttr[40:], label='GPU read')
    plt.plot(gttd[40:], label='GPU download')
    # Plot also the PyNvCodec one
    plt.plot(aa[0][40:], label='PyNvCodec total')
    plt.plot(aa[1][40:], label='PyNvCodec read')
    plt.plot(aa[2][40:], label='PyNvCodec download')
    plt.legend()
    plt.tight_layout()
    ```
    ![](images/cv_opecv_cuda_video_read.png)
## PyNvCodec
  - [Github VideoProcessingFramework](https://github.com/NVIDIA/VideoProcessingFramework)
  - [VideoProcessingFramework Building from source](https://github.com/NVIDIA/VideoProcessingFramework/wiki/Building-from-source)
  - **Init import**
    ```py
    import PyNvCodec as nvc
    import cv2
    import matplotlib.pyplot as plt

    encFilePath = './video.mp4'
    # encFilePath = 'rtsp://admin:admin1234@192.168.100.11:554/cam/realmonitor?channel=1&subtype=0'
    gpuID = 0
    nvDec = nvc.PyNvDecoder(encFilePath, gpuID)
    ```
  - **PyNvCodec DecodeSingleFrame + opencv**
    ```py
    rawFrameNV12 = np.ndarray(shape=(nvDec.Framesize()), dtype=np.uint8)
    nvDec.DecodeSingleFrame(rawFrameNV12)
    height, width = int(nvDec.Height() * 1.5), int(nvDec.Width())

    bb = rawFrameNV12.reshape(height, width)
    cc = cv2.cvtColor(bb, cv2.COLOR_YUV2RGB_NV12)
    plt.imsave('cc.jpg', cc)
    ```
  - **PyNvCodec DecodeSingleFrame + Converter**
    ```py
    rawFrameNV12 = np.ndarray(shape=(nvDec.Framesize()), dtype=np.uint8)
    nvDec.DecodeSingleFrame(rawFrameNV12)

    nvUpl = nvc.PyFrameUploader(nvDec.Width(), nvDec.Height(), nvc.PixelFormat.NV12, gpuID)
    rawSurface = nvUpl.UploadSingleFrame(rawFrameNV12)
    rawSurface.Empty()

    nvCvt = nvc.PySurfaceConverter(nvDec.Width(), nvDec.Height(), nvUpl.Format(), nvc.PixelFormat.RGB, gpuID)
    cvtSurface = nvCvt.Execute(rawSurface)
    cvtSurface.Empty()

    nvDwn = nvc.PySurfaceDownloader(nvDec.Width(), nvDec.Height(), nvCvt.Format(), gpuID)
    rawFrame = np.ndarray(shape=(cvtSurface.HostSize()), dtype=np.uint8)
    nvDwn.DownloadSingleSurface(cvtSurface, rawFrame)

    aa = rawFrame.reshape(nvDec.Height(), nvDec.Width(), 3)
    plt.imsave('dd.jpg', aa)
    ```
  - **PyNvCodec DecodeSingleSurface + Converter**
    ```py
    rawSurface = nvDec.DecodeSingleSurface()

    nvCvt = nvc.PySurfaceConverter(nvDec.Width(), nvDec.Height(), nvDec.Format(), nvc.PixelFormat.RGB, gpuID)
    cvtSurface = nvCvt.Execute(rawSurface)
    cvtSurface.Empty()

    nvDwn = nvc.PySurfaceDownloader(nvDec.Width(), nvDec.Height(), nvCvt.Format(), gpuID)
    rawFrame = np.ndarray(shape=(cvtSurface.HostSize()), dtype=np.uint8)
    nvDwn.DownloadSingleSurface(cvtSurface, rawFrame)

    aa = rawFrame.reshape(nvDec.Height(), nvDec.Width(), 3)
    plt.imsave('ee.jpg', aa)
    ```
  - **Test**
    ```py
    import cv2
    import PyNvCodec as nvc

    # encFilePath = os.path.expanduser('~/workspace/PyImageSearch/social-distance-detector/pedestrians.mp4')
    encFilePath = os.path.expanduser('~/workspace/pedestrians.mp4')

    gpuID = 0
    nvDec = nvc.PyNvDecoder(encFilePath, gpuID)
    nvCvt = nvc.PySurfaceConverter(nvDec.Width(), nvDec.Height(), nvDec.Format(), nvc.PixelFormat.RGB, gpuID)
    nvDwn = nvc.PySurfaceDownloader(nvDec.Width(), nvDec.Height(), nvCvt.Format(), gpuID)
    rawFrame = np.ndarray(shape=(nvDec.Width() * nvDec.Height() * 3), dtype=np.uint8)

    mm = cv2.TickMeter()
    nttr, nttd = [], []
    while True:
        mm.reset()
        mm.start()
        rawSurface = nvDec.DecodeSingleSurface()
        if rawSurface.Empty() == True:
            break
        cvtSurface = nvCvt.Execute(rawSurface)
        mm.stop()
        nttr.append(mm.getTimeMilli())

        mm.reset()
        mm.start()
        nvDwn.DownloadSingleSurface(cvtSurface, rawFrame)
        rawFrame.reshape(rawSurface.Height(), rawSurface.Width(), 3)
        mm.stop()
        nttd.append(mm.getTimeMilli())

    ntt = np.add(nttr, nttd)
    print("PyNvCodec read: %f ms/frame, PyNvCodec download: %f ms/frame, PyNvCodec total: %f ms/frame" % (np.mean(nttr[40:]), np.mean(nttd[40:]), np.mean(ntt[40:])))
    # PyNvCodec read: 0.356000 ms/frame, PyNvCodec download: 1.403157 ms/frame, PyNvCodec total: 1.759157 ms/frame
    ```
***

```py
sys.path.append(os.path.expanduser('~/workspace/samba/tdFace-flask/'))
from mtcnn_tf.mtcnn import MTCNN
det = MTCNN()
imm = imread('real_fake.png')

def hough_line_detect_P(frame):
    dst = cv2.Canny(frame, 100, 200)
    lines = cv2.HoughLinesP(dst, 1, np.pi / 180.0, 40, np.array([]), 50, 20)
    return lines

    cdst = cv2.cvtColor(dst, cv2.COLOR_GRAY2BGR)
    # lines = cv2.HoughLinesP(dst, 1, np.pi / 180.0, 40, np.array([]), 50, 10)
    for ii in lines:
        cv2.line(cdst, (ii[0][0], ii[0][1]), (ii[0][2], ii[0][3]), (0, 0, 255), 3, cv2.LINE_AA)
    return cdst

bbs, ccs, pps = det.detect_faces(imm)

up, left, down, right = bbs[0].astype('int')
# [ 363.3452, 1139.0464,  581.6208, 1302.2919]

ill = imm[up : down, 0 : left]
iuu = imm[0 : up, left : right]
irr = imm[up : down, right:]
idd = imm[down:, left : right]

fig, axes = plt.subplots(2, 4, figsize=(12, 3))
for ax_1, ax_2, ii in zip(axes[0], axes[1], [ill, iuu, irr, idd]):
    ax_1.imshow(ii)
    ax_2.imshow(hough_line_detect_P(ii)[:, :, ::-1])

up, left, down, right = bbs[1].astype('int')
# (328, 517, 671, 799)

ill = imm[up : down, 0 : left]
iuu = imm[0 : up, left : right]
irr = imm[up : down, right:]
idd = imm[down:, left : right]

fig, axes = plt.subplots(1, 4, figsize=(12, 3))
total = 0
for id, (ax, ii) in enumerate(zip(axes, [ill, iuu, irr, idd])):
    ax.imshow(ii)
    if ii.shape[0] != 0 and ii.shape[1] != 0:
        lines = hough_line_detect_P(ii)
        total += (max_dist_v(lines) / (down-up)) if id % 2 == 0 else (max_dist_h(lines) / (right-left))
        print(max_dist_h(lines), max_dist_h(lines) / (right-left), max_dist_v(lines), max_dist_v(lines) / (down-up))
        for jj in lines:
            ax.plot(jj[0][[0, 2]], jj[0][[1, 3]])


# max_dist_h = lambda lines: sqrt(max([(ii[0] - ii[2]) ** 2 + (ii[1] - ii[3]) ** 2 for ii in lines.squeeze() if abs(ii[0] - ii[2]) > abs(ii[1] - ii[3])]))
max_dist_h = lambda lines: max([abs(ii[0] - ii[2]) for ii in lines.squeeze() if abs(ii[0] - ii[2]) > abs(ii[1] - ii[3])] + [0])
max_dist_v = lambda lines: max([abs(ii[1] - ii[3]) for ii in lines.squeeze() if abs(ii[1] - ii[3]) > abs(ii[0] - ii[2])] + [0])
```