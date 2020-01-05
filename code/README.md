1.下载imagenet数据集,使用了预训练模型将下载下来的图像进行分类
  注:使用的预训练模型都是基础的模型,没有经过对抗训练的,efficientnet-b1,resnet50,densenet121,vgg16,inceptionv3都很容易得到
  如果还是需要的话,efficientnet-b1:'http://storage.googleapis.com/public-models/efficientnet/efficientnet-b1-f1951068.pth'
  其他的4个模型是使用的pytorch的官方模型会自动下载

2.将dev.csv文件中的需要的目标类别保存下来,将不需要的目标类别删除

3.使用target_atk_loop.py文件生成需要的csv文件
  target_atk_loop.py文件的基本思想是:使用每个图像的对应的目标类别的图像压缩到[-32,32]再叠加到原始图像中,生成对抗样本.
  这其中的一些技巧是:
	1.对原始图像使用高斯滤波,减少原始图像的纹理,
	2.对目标类别图像做自适应直方图均衡化,使得纹理更加清楚,
	3.对目标类别图像先clip到[50/255,200/255],再压缩到[-50/255,50/255],再clip到[-32/255,32/255],使得更多的像素点能达到32的变化,
	4.叠加到经过高斯滤波的原始图像中并裁剪到原始图像的[-32/255,32/255]的范围.
	5.将最好的一个目标攻击的图像和无目标攻击的图像对应的id及成功率记录到保存的csv文件中
	6.我们发现第600类(蜂巢?)和第490类(铁网?)是两个比较好的类,后面的无目标攻击都是在这两个类中进行
  使用target_atk_loop会生成一个csv文件,代表了每张图像搜索出来的对应的比较好的图像的,及其对应的操作
  使用atk_to_sub.py解析上面生成的csv文件,并生成对抗样本,这里发现对5个模型的目标攻击的成功率为Ttogap>(0.4*5)时是一个比较好的平衡点
  Ttogap代表目标类的成功率-原始类的成功率的差(理论上越大越好)
  Uuogap代表非原始类的任何一个类的成功率-原始类的成功率的差(理论上越大越好)

4.使用four_target_atk_loop_v2.py文件生成需要的csv文件
  four_target_atk_loop_v2.py的文件的基本思想和上面是一样的,但是这个使用的不只是一个目标类别的图像叠加到原始图像,而是使用4张图像叠加到原始图像
  主要区别是Ttogap设定为(0.9*5),因为我们发现使用4个不同的目标图像虽然我们可以使得我们线下的效果变好很多,但是线上却不是这样的,
  我们觉得还是线上和线下模型的差距导致的出现的这个问题.

5.总的来说,先使用target_atk_loop.py生成的文件,使用atk_to_sub解析时,将Ttogap设定为2.0是一个比较好的平衡点,得到初步的1216张对抗样本.
  使用four_target_atk_loop.py生成的文件,使用atk_to_sub_2解析时,只保留了Ttogap大于(0.9*5)的图像,并将这个得到的图像替换掉初步的1216张对抗样本,
  得到最终的1216张对抗样本.

